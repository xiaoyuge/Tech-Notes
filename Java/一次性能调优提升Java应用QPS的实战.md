# **一次性能调优提升Java应用QPS的实战**

## **背景**

故事的时间回到2019年9月1号学校开学前一天的那个夜晚，晚上20点左右，全国所有的小朋友都端坐在电视机前，观看例行的《开学第一课》，这样一个本与我负责的系统没有任何肉眼可见端倪的事件，却让我负责的系统突然报警了，系统的核心页面-播放详情页无法打开，进而导致正常用户无法浏览、下单和支付，也即系统的核心链路不可用了，虽然通过紧急的限流降级等手段，及时止损暂时解决了问题，但事后的故障原因分析是必不可少的。

通过事后盘查，事故起因让人啼笑皆非。我们的某一个CP（Content Provider）为了蹭开学第一课的热度，把他在我们平台上传的某个课程视频的标题，改成了《开学第一课-xxxxxx》，而实际上我所在的某视频平台并没有购买《开学第一课》的直播版权，但因为该视频平台的影响力，很多家长以为它会在开学前一天晚上同步直播《开学第一课》，于是，在那天晚上的20点左右，一大波家长或熊孩子涌入该视频平台开始搜索《开学第一课》，结果是并没有搜到预期的内容却把鸡贼CP蹭热度的课程视频给搜了出来，于是，在瞬间的大量搜索点击流量下，我们的系统就无辜躺枪了。

这次故障事后，我拉着我的架构师和相关核心技术人员，一起重新review了我们系统的架构，并通过持续的性能调优手段大幅提升了系统的QPS，出于在团队内总结分享的目的，我让架构师把这次调优的经过总结成了一篇文章，下面就是出自这位架构师的总结。

## **导读**

作为一名开发人员，你负责的系统可能会出现高并发的挑战。当然这是好事，因为这既反映出您的业务在蒸蒸日上，对你而言也是一个锤炼技术能力的机会。本文记录了笔者在工作过程中的一次 Java Web 应用性能优化过程，包括：如何去排查性能问题，如何利用工具，以及如何去解决等。现抛砖引玉分享给大家，以供大家参考。

## **问题背景**

本次性能优化的起源，是一次线上事故引起的，当时一个接口的访问量突然暴增，系统不堪重负而导致接口响应变慢，最终客户端调用接口时，出现大量 socket timeout 异常。事后笔者任务就是对这个接口排查性能瓶颈并解决，以提升 QPS 指标。

经过梳理，总结这个模块的架构如下：

![arch-old](https://github.com/xiaoyuge/Tech-Notes/blob/main/Java/resources/arch-old.png)

当然这只是一个简图，真实的业务逻辑还是要比这个复杂一点，但总的说来它的业务逻辑还是不复杂的，主要的流程还是图上反映的那样，即：从缓存中读数据，读到后返回给客户端（没读到就抛异常告诉客户端无数据）。

交代一下服务器环境配置：Web 服务器是 CentOS 系统， CPU 是 4 核的，JVM 堆内存大小为 4 G。程序是基于 SpringBoot 1.5.2 构建的，内嵌的 Web 容器 Tomcat 。分布式缓存用的是 CouchBase（后续简称 cb）。

按理说这个接口的 QPS 应该很高才对，但实际测下来只有 1400，所以笔者的任务就是排查性能瓶颈并解决，以提升 QPS 指标。

## **安装接口压测工具 wrk**

工欲善其事，必先利其器。 要进行性能优化，对接口进行压测是经常要做的事，所以我们希望有一款简单好用的压测工具，最终我们选择了 wrk 。 我们找了一台 CentOS 的机器作为压测机安装 wrk ，安装方法如下：

```bash
sudo yum groupinstall 'Development Tools'
sudo yum install openssl-devel
sudo yum install git
git clone <https://github.com/wg/wrk.git> wrk
cd wrk
make
```

安装好后，我们对一台 web 服务器直接发起 HTTP 请求进行压测：

```bash
./wrk/wrk -t300 -c400 -d30s "http://10.41.135.74:8982/api/v3/dynamic/product/column/detail?productId=204918601"
```

执行的结果如下：

![try-wrk]()

参数说明：

- -t300：指起 300 个线程来执行请求，每个线程在执行完一个HTTP请求后会立即执行下一个HTTP请求；
- -c400：指在连接池中建立400个连接，用于 HTTP 请求，而不是每次请求都重新建立连接；
- -d30s：指本轮压测持续时间为 30 秒；

结果说明：

- Requests/sec：指每秒处理的请求数量，也就是我们说的 QPS （Query Per Second）；
- Transfer/sec：指每秒传输的数据量大小；

从上图看，目前的 QPS 在 1400 左右。

## **引入 Perf4j**

perf4j 是一款用于诊断 Java 应用程序性能的开源工具，它可以统计各阶段耗时统计，帮助我们找出性能瓶颈。为了知晓在高并发情况下，程序各阶段的耗时，我们先引入 perf4j 开源项目，步骤如下：

1. 在项目 pom.xml 中添加 perf4j 依赖:

```xml
<dependency>
 <groupId>org.perf4j</groupId>
 <artifactId>perf4j</artifactId>
</dependency>
```

2. 在 Application 类（或其它 Configuration 类）中添加 TimingAspect 的 Bean 定义：

```Java
@Bean
public TimingAspect timingAspect() {
 return new TimingAspect();
}
```

注意： 我们的系统的日志库用的是 slf4j，因此要引的是 org.perf4j.slf4j.aop 包里面的 TimingAspect 类。

3. 对想要监测的代码，在方法上添加 @Profiled 注解，如：

```Java
@Profiled
@RequestMapping(name = "请求 Card 化页面数据", value = "/cpage", method = RequestMethod.GET)
public String getCpageResult(
        @RequestParam(value = "cpageCode", required = false) String cpageCode,
        @RequestParam(value = "previewMode", required = false) String previewModeQuery) {
  // ...
}
```

或用 org.perf4j.StopWatch 类：

```Java
public ColumnDetailV2 queryColumnDetailV2FromCache(Long qipuId) {
    org.perf4j.StopWatch stopWatch = new Slf4JStopWatch("queryColumnDetailV2FromCache");
    stopWatch.start("4-1-genColumnDetailV2CacheKey");
    String cacheKey = genColumnDetailV2CacheKey(qipuId);
    stopWatch.stop();
    stopWatch.start("4-2-getCmsInfoFromCB");
    String result = cmsCouchbaseOp.getCmsInfoFromCB(cacheKey);
    stopWatch.stop();
    if (StringUtils.isBlank(result)) {
        LOGGER.warn("queryColumnDetailV2FromCache: {} not exist, result: {}", cacheKey, result);
        return null;
    } else {
        stopWatch.start("4-3-parseColumnDetailV2");
        ColumnDetailV2 columnDetailV2 = JSON.parseObject(result, ColumnDetailV2.class);
        stopWatch.stop();
        return columnDetailV2;
    }
}
```

注意 SpringBoot 框架自身也提供了一个 StopWatch 类（在包 org.springframework.util 中），但它没有 org.perf4j 提供的 StopWatch 类的功能强大。

4. 在 logback 日志文件中定义 Perf4j 的 Appender（推荐 Perf4j 的日志单独输出到一个文件），如：

```xml
<property name="LOG_HOME" value="/data/logs/kpp-lighthouse" />

<appender name="Perf4jFileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
   <File>${LOG_HOME}/lighthouse-cms-web-perf4.log</File>
   <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <!--文件的路径与名称,{yyyy-MM-dd.HH}精确到小时,则按小时分割保存-->
      <FileNamePattern>${LOG_HOME}/lighthouse-cms-web-perf4.%d{yyyy-MM-dd.HH}.log.gz</FileNamePattern>
      <!-- 如果当前是按小时保存，则保存48小时(=2天)内的日志（建议生产环境改成7天） -->
      <MaxHistory>48</MaxHistory>
   </rollingPolicy>
   <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
      <pattern>%m%n</pattern>
   </encoder>
</appender>

<appender name="Perf4jAppender" class="org.perf4j.logback.AsyncCoalescingStatisticsAppender">
    <!--
       TimeSlice 用来设置聚集分组输出的时间间隔，默认是 30000 ms，
       在生产环境中可以适当增大以供减少写文件的次数
    -->
    <param name="TimeSlice" value="10000"/>
    <appender-ref ref="Perf4jFileAppender"/>
</appender>

<!-- Perf4j 默认用名称为 org.perf4j.TimingLogger 的 Logger -->
<logger name="org.perf4j.TimingLogger" level="INFO" additivity="false">
  <appender-ref ref="Perf4jAppender"/>
</logger>
```

引入完了后，笔者用 org.perf4j.StopWatch 的方式在程序代码中添加性能监测代码。 然后在本地（IDEA debug模式）运行程序，调用几次后 per4j 输出的统计数据如下：

![per4j-log]()

上图中的 Tag 列是笔者添加性能监测代码时，给每段代码起的名字，这里解释它们的意义：

- 1-0-queryColumnDetail 是整个接口（Controller层的方法）的耗时统计；
- 1-x-xxxx 是在 1-0-queryColumnDetail 中的各段代码的耗时统计；
- 2-0-queryColumnDetailStatic 等同于 1-4-queryColumnDetailStatic （queryColumnDetail 方法调用 queryColumnDetailStatic 方法）
- 2-x-xx 是 2-0-queryColumnDetailStatic 中的各段代码耗时统计。
- 以此类推......

我们可以看到：接口的总耗时只有 25 ms，主要耗时在读 cb 缓存（23ms），其中读 cb 为 19 ms ，将 json 解析为对象 3 ms 。

## **实施压测**

为了进行压测，我们将线上一台机器切走流量，把含性能监控日志的代码部署到这台机器上， 后续我们将对它进行 N 轮压测：压测 -> 优化 -> 再压测 -> 再优化...... 直到达到一个让人接受的 QPS 指标为止。
注意： 如果是服务启动过，要先压测一轮以进行预热，并且这次的数据不作准

### **第 0 轮：无压力测试**

我们先了解一下在无压力的情况程序性能表现如何，如果发现哪段代码耗时较长，就排查这段代码好了。不压测，只访问接口一次，查看性能统计数据如下：

![ptest-0]()

可以看出来，没有压力时，整个接口的总时长就 3 ms，也就是说，程序没有哪段代码有明显的性能损耗，只有压力上上去了才有。

### **第 1 轮： 诊断是否是日志造成的性能损耗**

笔者首先猜想可能是日志输出较多造成的性能损耗，因为日志导致的性能问题的案例比较常见（笔者在以前的项目中就遇到过类似的问题）。

但是猜归猜，还需要实际测试来确认这个猜想，所以笔者打算在有日志输出时压测一次，在无日志输出时再压测一次，比较下它们的 QPS 差异。

这里先交待一下原来的 logback.xml 日志配置如下：

```xml
<property name="LOG_HOME" value="/data/logs/kpp-lighthouse" />
<!-- 正常 info 级别的程序日志输出 -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>${LOG_HOME}/lighthouse-cms-dynamic.log</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <!--日志文件输出的文件名-->
    <FileNamePattern>${LOG_HOME}/lighthouse-cms-dynamic.%d{yyyy-MM-dd}.log.gz</FileNamePattern>
    <!--日志文件保留天数-->
    <MaxHistory>60</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
    <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的[如配置80,则80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志] -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <includeCallerData>true</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="FILE"/>
</appender>
<logger name="o.s">
    <level value="DEBUG" />
</logger>
<logger name="c.i.input.dao">
    <level value="DEBUG" />
</logger>
<root level="INFO">
    <appender-ref ref="ASYNC_DAILY_ROLLING_FILE"/>
</root>
```

日志配置用的是 RollingFileAppender，是一种很常见的日志配置方法（细心的读者们发现什么端倪了么？）

在以上日志配置下，笔者准备 600 个连接，起 300 并发线程，压测 60 秒：

```bash
./wrk -t300 -c600 -d60s <http://10.41.135.74:8982/api/v3/dynamic/product/column/detail?productId=204918601>
```

结果如下：

![ptest-log-1-1]()

QPS 只有 1400 左右。

然后我们将所有日志级别都调成 ERROR 级别，再压测：

![ptest-log-1-2]()

发现 QPS 从 1400 提升到 2400，因此确认了日志输出对程序性能有较大损耗。

### **第 2 轮：优化日志配置**

虽然从上一轮我们知道是日志输出的问题，但笔者也不敢真将日志输出都关闭，万一哪天出线上 BUG 了，没有日志岂不是得抓狂？？

经过笔者对日志配置仔细分析，发现有几个可优化项：

1. 将 ImmediateFlush 改为 false :为 false 的意思是去掉写日志时立即刷磁盘的操作，理论上这样可以提升日志写入的性能；
2. 将 queueSize 值改大：queueSize 值是指定日志缓存队列（一个 BlockingQueue）的最大容量，改大点可以提升性能（但也不要太大，以免占用太多内存）。 笔者计算了一下： 一条日志按 500B 算（根据当前业务场景），10K条日志也就5M，因此这个值从 512 改到 1W ；
3. 将 includeCallerData 从 true 改为 false ：includeCallerData 表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，为了提升性能，改为 false，即：当 event 被加入 BlockingQueue 时，event 关联的调用者数据不会被提取，只有线程名这些比较简单的数据；

调整参数后的日志配置如下：

```xml
<!-- 正常 info 级别的程序日志输出 -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>${LOG_HOME}/lighthouse-cms-dynamic.log</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--日志文件输出的文件名-->
        <FileNamePattern>${LOG_HOME}/lighthouse-cms-dynamic.%d{yyyy-MM-dd}.log.gz</FileNamePattern>
        <!--日志文件保留天数-->
        <MaxHistory>60</MaxHistory>
    </rollingPolicy>
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!-- 优化项1：去掉写日志时立即刷磁盘的操作，以提升日志写入的性能。 -->
        <ImmediateFlush>false</ImmediateFlush>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的[如配置80,则80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志] -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 优化项2：本项是暂存日志记录的 BlockingQueue 的最大容量，改大点可以提升性能（但也不要太大，会占用内存），
        一条日志500B算，10K条日志也就5M，因此这个值从 512 改到 1W 。 -->
    <queueSize>10000</queueSize>

    <!-- 优化项3：includeCallerData表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，
        为了提升性能，默认为false，即：当event被加入BlockingQueue时，event关联的调用者数据不会被提取，只有线程名这些比较简单的数据
        因此这个值从 true 改为 false 。
    -->
    <includeCallerData>false</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="FILE"/>
</appender>
<root level="INFO">
    <appender-ref ref="ASYNC_DAILY_ROLLING_FILE"/>
</root>
```

改后再进行压测，结果如下：

![ptest-log-2]()

优化日志配置后，QPS提升 到 2300 ，虽不如完全关闭日志后的 2400 ，但也相差不多了。

### **第 3 轮： 分析系统性能瓶颈**

QPS 到 2300 就是极限了么？ 笔者觉得应该还远没到程序的最佳状态，但下一步应该怎么优化呢？我们不妨先尝试分析下压测状态下的机器性能，如 CPU 内存 网络IO 等指标，看能不能找出性能瓶颈和优化思路。

我们先查看 java 进程的性能：

- 用 ps -ef | grep java 查看 java 程序的进程id(这步开发者应该都会吧，这里就省略截图了)；
- 用 pidstat -p [进程id] [打印的间隔时间] [打印次数] 命令来打印出 java 进程的 CPU 使用情况：

![ptest-cpu-1]()

笔者发现，一旦压测开始， CPU 直接涨到 100% （上图红框所示）。（这里显示的 CPU 为每个核的平均值，而不是加和值，因此虽然是 4 核，100% 指 CPU 已打满了）。

- 为了知道是具体哪个线程占 CPU 高，用 top -Hp [进程id] 来查看线程的 CPU 占用情况：

![ptest-cpu-2]()

发现是 id 为 63099 的线程占 CPU 最高，为 46% 。

- 用 printf "%x\n" [进程id] 打印出此线程 id 的十六进制值： f67b 。
- 用 jstack [进程id] |grep -B 1 -C 20 [线程id十六进制值] 打印出此线程的堆栈信息（建议多打印几次），如下：

![ptest-cpu-3]()

![ptest-cpu-4]()

通过以上堆栈信息，发现是下面这个日志输出的 Appender 线程占用了大量的 CPU 资源：

```xml
<appender name="ASYNC_DAILY_ROLLING_FILE" class="ch.qos.logback.classic.AsyncAppender">
```

具体来说，有两个问题：

1. logback 用 ArrayBlockingQueue 缓存日志对象，似乎在 take 端 和 poll 端存在资源竞争（ArrayBlockingQueue 元素的存和取用的是同一把锁）;
2. 仍有获取 caller 数据的情况（还记得第 2 轮的优化项 3 么？）;

对第 1 个问题，网上查了下，logback 的低版本在创建 ArrayBlockingQueue 时，用的是公平锁，的确会有性能问题 （它在高版本作了优化，将这个地方改成使用非公平锁），由于在我们项目中升级 logback jar 的版本会牵扯比较大， 笔者简单起见直接 hack 改它的源码，将类 AsyncAppenderBase 的源码单独拷贝出来：

![ptest-cpu-5]()

然后进行修改：

```Java
public class AsyncAppenderBase<E> extends UnsynchronizedAppenderBase<E> implements AppenderAttachable<E> {

    // 省略其它代码...

    public void start() {
        if (!this.isStarted()) {
            if (this.appenderCount == 0) {
                this.addError("No attached appenders found.");
            } else if (this.queueSize < 1) {
                this.addError("Invalid queue size [" + this.queueSize + "]");
            } else {
                this.blockingQueue = new ArrayBlockingQueue(this.queueSize, false);
                if (this.discardingThreshold == -1) {
                    this.discardingThreshold = this.queueSize / 5;
                }

                this.addInfo("Setting discardingThreshold to " + this.discardingThreshold);
                this.worker.setDaemon(true);
                this.worker.setName("AsyncAppender-Worker-" + this.getName());
                super.start();
                this.worker.start();
            }
        }
    }
    
    // 省略其它代码...
```

其实只改了一行源代码，就是将： this.blockingQueue = new ArrayBlockingQueue(this.queueSize); 改成： this.blockingQueue = new ArrayBlockingQueue(this.queueSize, false); （SpringBoot 的类加载机制，是优先加载我们项目中的代码，因此这样就替换掉了在 logback JAR包中的同名类）。

对第 2 个问题，终于在 logback.xml 中找到了问题所在，原来日志配置中有提取 method 信息（元素中有 %method），还是会触发读取调用者信息：

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <!-- 省略其它代码 ... -->
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <!-- 优化项1：去掉写日志时立即刷磁盘的操作，以提升日志写入的性能。 -->
        <ImmediateFlush>false</ImmediateFlush>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS}-|%thread-|%level-|%X{requestId}-|%X{req.remoteHost}-|%X{req.xForwardedFor}-|%X{req.requestURI}-|%X{req.queryString}-|%X{req.userAgent}-|%logger{50}:%method-|%msg%n</pattern>
    </encoder>
</appender>
```

所以把 %method 去掉。
改了这两个地方后，部署代码再压测：

![ptest-cpu-6]()

发现 QPS 提升到 2500，虽然不是很明显，但也算前进了一步。

### **第 4 轮： 优化 JVM 参数**

经过上一轮优化后，我们再看一下 CPU 情况。通过 top -Hp [进程id]，我们查看下压测状态下占 CPU 高的前几个线程：

```bash
13753 root 20 0 7905m 4.3g 6584 R 24.0 56.0 3:48.70 java
13713 root 20 0 7905m 4.3g 6584 S 12.3 56.0 0:20.75 java
13716 root 20 0 7905m 4.3g 6584 S 12.3 56.0 0:20.71 java
13715 root 20 0 7905m 4.3g 6584 S 12.0 56.0 0:20.57 java
13714 root 20 0 7905m 4.3g 6584 S 11.6 56.0 0:20.85 java
20154 root 20 0 7905m 4.3g 6584 R 8.7 56.0 0:10.30 java
20137 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.28 java
20141 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.24 java
20159 root 20 0 7905m 4.3g 6584 S 8.0 56.0 0:10.23 java
14080 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:11.44 java
20129 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:10.92 java
20138 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.27 java
20140 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.32 java
20144 root 20 0 7905m 4.3g 6584 R 7.7 56.0 0:10.25 java
20151 root 20 0 7905m 4.3g 6584 S 7.7 56.0 0:10.27 java
......
```

- 排名第 1 位的线程，占 CPU 24% ，经排查它仍是日志线程；

- 排名第 2 ~ 5 位的线程，每个占 CPU 12%（4个加起来就是 48%），经排查它们都是 GC 线程（用于 YGC）：

![ptest-jvm-1]()

- 排名第 6 位及以后线程，平均每个占比 7~8%，经排查大多都是 Web 容器的线程，阻塞在等待 CB 调用的返回上;

虽然这三个可能都是问题，但我们最好先聚焦于解决最大的一个问题，把最大问题解决了再解决下一个问题，这样投入产出比最高（类似于“贪心算法”，哈哈）。
 
那问题来了，我们怎么判断哪个问题是最大问题呢？ 由于当前的主要症状是 CPU 被打满，第二个问题吃的 CPU 最多（且按以往经验 GC 线程没有道理吃这么多 CPU）， 所以我们选择先解决第二个问题，即 GC 线程的问题。

我们用 jstat -gcutil [进程id] [打印的间隔时间（毫秒）] [打印的次数] 命令来查一下 GC 的情况：

![ptest-jvm-3]()

每列解释如下：

- S0：幸存1区当前使用比例
- S1：幸存2区当前使用比例
- E：伊甸园区使用比例
- O：老年代使用比例
- M：元数据区使用比例
- CCS：压缩使用比例
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

上图的红框部分，是压测开始时的 GC 信息，从图上 S0 区、S1 区在压测开始后就不停的捣腾， 并且 YGC （代表年轻代垃圾回收的累积次数）这列数字增加得很快，说明 YGC 比较严重。
 那 YGC 到底有多严重，以及 YGC 的具体情况是怎样的， 我们用 jstat -gcnewcapacity [进程id] [打印的间隔时间（毫秒）] [打印的次数] 命令输出 JVM 新生代的具体信息：

![ptest-jvm-4]()

每列解释如下：

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0CMX：最大幸存0区大小
- S0C：当前幸存0区大小
- S1CMX：最大幸存1区大小
- S1C：当前幸存1区大小
- ECMX：最大伊甸园区大小
- EC：当前伊甸园区大小
- YGC：年轻代垃圾回收次数
- FGC：老年代回收次数

上图的红框部分，是压测开始时的 YGC 总次数，我们发现 YGC 居然每 1 秒要执行 6 次。 最重要的是，NGCMX（新生代最大容量）只有 340 M，S0CMX（最大幸存 0 区大小）、S1CMX（最大幸存 1 区大小）都只有 34 M，有点太小了。

因此我们在 JVM 启动命令中设置 JVM 参数， 添加 -Xmn1536m 及 -XX:SurvivorRatio=6 两个参数：

- -Xmn1536m 是指新生代最大容量为 1.5G（总堆内存为 4 G）；
- -XX:SurvivorRatio=6 是指 S0(幸存0区) : S1(幸存1区) : EC(伊甸园区) 的比例为 2:2:6。

设置好 JVM 参数后，我们再压测：

![ptest-jvm-5]()

发现 QPS 提升到 2680（接近 2700）。 同时查看 GC 情况（如下图），发现 GC 线程的 CPU 占用率降下来了；而YGC 的次数，也由 1 秒 6 次，变成了 2 秒 1 次。

![ptest-jvm-6]()

### **第 5 轮： 将 Web 容器从 Tomcat 改为 Undertow**

经过上一轮优化后，发现 GC 线程的 CPU 占用率降下来了。然后我们再看最大问题是哪个。

目前不再有明显 CPU 高的线程，最高还是日志线程 14%，其次是 Web 容器中的 Worker 线程，平均为 7%。 但日志线程只有一个，而 Web 容器中的 Worker 线程则非常多了，因此对 Web 容器的优化是我们下一步的主要方向。

在网上查资料，有的文章分析说 Undertow 的性能比 Tomcat 好（是SpringBoot 三大内嵌容器 Tomcat、Jetty、Undertow 中最好的一个）， 因此我们尝试将 SpringBoot 中默认的 Tomcat 改为 Undertow 试试看。

首先，在 pom.xml 配置如下：

```xml
<!-- 网上说 Undertow 的性能比 Tomcat 好，试试看？ -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <!-- 移除掉默认支持的 Tomcat -->
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<!-- 添加 Undertow 容器 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

然后，在 application.yml 文件中配置 Undertow 的相关参数：

```yml
server:
    port: 8982
    undertow:
        io-threads: 4          # 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置每个CPU核心一个线程
        worker-threads: 256    # 阻塞任务线程池, 当执行类似servlet请求阻塞操作, undertow会从这个线程池中取得线程,它的值设置取决于系统的负载
                               # 这个值要根据阻塞系数来设置，并不是所谓的越大越好，太大反而会因竞争资源导致性能下降！！
        # 以下的配置会影响buffer,这些buffer会用于服务器连接的IO操作,有点类似netty的池化内存管理
        # 每块buffer的空间大小,越小的空间被利用越充分，不要设置太大，以免影响其他应用，合适即可
        # 每个区分配的buffer数量 , 所以 pool 的大小是buffer-size * buffers-per-region
        buffer-size: 1048
        buffers-per-region: 1048
        direct-buffers: true   #  是否分配的直接内存(NIO直接分配的堆外内存)，建议开启可以使用堆外内存减少byte数组的回收开销。
        accesslog:
            enabled: false
```

再压测:

![ptest-undertow-1]()

发现 QPS 提升到 3800，果然 Undertow 的性能比 Tomcat 好不少。

### **第 6 轮： 添加加上内存缓存**

之前发现 Web 容器的 Worker 线程大多阻塞在调用 cb 上，考虑到当前场景下 cb 中的数据是静态（每 5 分钟更新一次），是可以进行本地内存缓存的。 具体方案为：如果内存缓存有数据，就直接读内存中的存缓数据，否则就从 cb 中读数据，并放在内存缓存中。

但是要注意两点：

1. 将内存缓存设置一个比较短的过期时间。
2. 将内存缓存的大小控制在可接受的范围。

业界有不少成熟的内存缓存开源项目，经过比较，我们选择了 Guava 作为内存缓存的实现方案，代码如下：

```Java
@Configuration
@EnableCaching
public class GuavaCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        GuavaCacheManager cacheManager = new GuavaCacheManager();
        cacheManager.setCacheBuilder(
                CacheBuilder.newBuilder().
                        expireAfterWrite(10, TimeUnit.SECONDS).
                        maximumSize(10000));
        return cacheManager;
    }
}
```

经过计算，本场景下一条缓存平均大小为 10KB，1W 条就是 100M，而 JVM 堆内存有 4G，理论上是完全撑得住的。 在 cb 中的数据，是每 5 分钟刷新一次，因此内存缓存设置为 10 秒过期，也完全没有问题。

然后在我们在读 cb 的地方加上内存缓存，代码如下：

```Java
@Cacheable(value = "column-detail", key = "#qipuId")
public ColumnDetailV2 queryColumnDetailV2FromCache(Long qipuId) {
    LOGGER.info("queryColumnDetailV2FromCache, qipuId = {}", qipuId);
    String cacheKey = genColumnDetailV2CacheKey(qipuId);
    String result = cmsCouchbaseOp.getCmsInfoFromCB(cacheKey);
    if (StringUtils.isBlank(result)) {
        LOGGER.warn("queryColumnDetailV2FromCache: {} not exist, result: {}", cacheKey, result);
        return null;
    } else {
        return JSON.parseObject(result, ColumnDetailV2.class);
    }
}
```

关键是第 1 行代码：@Cacheable(value = "column-detail", key = "#qipuId") 即缓存块名为 column-detail，其中每条缓存的 key 为参数 qipuId 的值。
考虑到加上内存缓存后，处理请求的 Worker 线程 Blocking 时间会大大减少，这里改小了 undertow 的线程数：

```yml
server:
    port: 8982
    undertow:
        ioThreads: 4
        workerThreads: 32
```

io 线程数改为 4 （因为被测机器的 CPU 只有 4 核）， worker 线程数改为 32 （即 io 线程数的 8 倍）。 注意：并不是线程数越多越好，因为太多线程数，竞争资源反而会加剧性能损耗。

将修改的代码部署到压测环境，再压测：

![ptest-cache-1]()

QPS 进一步提升到 5000，说明在“恰当”的场景下，在分布式缓存的基础上再加上内存缓存，能更进一步提升性能。

## **总结与展望**

由于 5000 的 QPS 对当前业务场景已经足够用了，因此笔者也不打算花时间继续优化了。
 最后总结下做性能优化的一些思路和原则：

- **数据驱动**：通过“压测 + 分析数据”的方式，来指导我们优化的方向以及明确要解决的问题，而不要想当然（即使是猜，也要先想办法验证你的猜想再动手优化）；
- **聚焦最大瓶颈**：如果有多个优化方向，则先评估哪一个方向的问题是最大瓶颈，然后集中精力去优化这个最大瓶颈，优化完后再评估下一个最大瓶颈并解决，如此迭代前进；
- **利用工具**： 现在业界性能优化的工具还是比较多的，用好这些工具可以事半功倍，有些特殊场景即使没有好用的工具，也可以花点精力开发一些小工具来使用。

那实际上还有没有优化空间呢？笔者判断应该还是有不少优化空间的，感兴趣读可以继续尝试更多的方案哦！
