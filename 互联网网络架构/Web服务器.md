## **（1）Tomcat**

Tomcat类加载
Tomcat 是通过 Context 组件来加载管理 Web 应用的，我们先看一下JVM 的类加载机制，接着再谈谈 Tomcat 的类加载器如何打破 Java 的双亲委托机制

Java 的类加载，就是把字节码格式“.class”文件加载到 JVM 的方法区，并在 JVM 的堆区建立一个java.lang.Class对象的实例，用来封装 Java 类相关的数据和方法。那 Class 对象又是什么呢？你可以把它理解成业务类的模板，JVM 根据这个模板来创建具体业务类对象实例

JVM 并不是在启动时就把所有的“.class”文件都加载一遍，而是程序在运行过程中用到了这个类才去加载。JVM 类加载是由类加载器来完成的，JDK 提供一个抽象类 ClassLoader，这个抽象类中定义了三个关键方法：
* JVM 的类加载器是分层次的，它们有父子关系，每个类加载器都持有一个 parent 字段，指向父加载器；
* defineClass 是个工具方法，它的职责是调用 native 方法把 Java 类的字节码解析成一个 Class 对象，所谓的 native 方法就是由 C 语言实现的方法，Java 通过 JNI 机制调用；
* findClass 方法的主要职责就是找到“.class”文件，可能来自文件系统或者网络，找到后把“.class”文件读到内存得到字节码数组，然后调用 defineClass 方法得到 Class 对象；
* loadClass 是个 public 方法，说明它才是对外提供服务的接口，具体实现也比较清晰：首先检查这个类是不是已经被加载过了，如果加载过了直接返回，否则交给父加载器去加载。请你注意，这是一个递归调用，也就是说子加载器持有父加载器的引用，当一个类加载器需要加载一个 Java 类时，会先委托父加载器去加载，然后父加载器在自己的加载路径中搜索 Java 类，当父加载器在自己的加载范围内找不到时，才会交还给子加载器加载，这就是双亲委托机制


* BootstrapClassLoader 是启动类加载器，由 C 语言实现，用来加载 JVM 启动时所需要的核心类，比如rt.jar、resources.jar等；
* ExtClassLoader 是扩展类加载器，用来加载\jre\lib\ext目录下 JAR 包；
* AppClassLoader 是系统类加载器，用来加载 classpath 下的类，应用程序默认用它来加载类；
* 自定义类加载器，用来加载自定义路径下的类；
这些类加载器的工作原理是一样的，区别是它们的加载路径不同，也就是说 findClass 这个方法查找的路径不同

注意，类加载器的父子关系不是通过继承来实现的，比如 AppClassLoader 并不是 ExtClassLoader 的子类，而是说 AppClassLoader 的 parent 成员变量指向 ExtClassLoader 对象。同样的道理，如果你要自定义类加载器，不去继承 AppClassLoader，而是继承 ClassLoader 抽象类，再重写 findClass 和 loadClass 方法即可，Tomcat 就是通过自定义类加载器来实现自己的类加载逻辑。不知道你发现没有，如果你要打破双亲委托机制，就需要重写 loadClass 方法，因为 loadClass 的默认实现就是双亲委托机制。

Tomcat 的类加载器
Tomcat 的自定义类加载器 WebAppClassLoader 打破了双亲委托机制，它首先自己尝试去加载某个类，如果找不到再代理给父类加载器，其目的是优先加载 Web 应用自己定义的类。具体实现就是重写 ClassLoader 的两个方法：findClass 和 loadClass
Tomcat 的类加载器打破了双亲委托机制，没有一上来就直接委托给父加载器，而是先在本地目录下加载，为了避免本地目录下的类覆盖 JRE 的核心类，先尝试用 JVM 扩展类加载器 ExtClassLoader 去加载。那为什么不先用系统类加载器 AppClassLoader 去加载？很显然，如果是这样的话，那就变成双亲委托机制了，这就是 Tomcat 类加载器的巧妙之处

Tomcat 通过自定义类加载器 WebAppClassLoader 打破了双亲委托机制，具体来说就是重写了 JVM 的类加载器 ClassLoader 的 findClass 方法和 loadClass 方法，这样做的目的是优先加载 Web 应用目录下的类。除此之外，你觉得 Tomcat 的类加载器还需要完成哪些需求呢？或者说在设计上还需要考虑哪些方面？
——核心就是加载类的隔离和共享问题

Tomcat 作为 Servlet 容器，它负责加载我们的 Servlet 类，此外它还负责加载 Servlet 所依赖的 JAR 包。并且 Tomcat 本身也是一个 Java 程序，因此它需要加载自己的类和依赖的 JAR 包。首先让我们思考这一下这几个问题：
* 假如我们在 Tomcat 中运行了两个 Web 应用程序，两个 Web 应用中有同名的 Servlet，但是功能不同，Tomcat 需要同时加载和管理这两个同名的 Servlet 类，保证它们不会冲突，因此 Web 应用之间的类需要隔离
* 假如两个 Web 应用都依赖同一个第三方的 JAR 包，比如 Spring，那 Spring 的 JAR 包被加载到内存后，Tomcat 要保证这两个 Web 应用能够共享，也就是说 Spring 的 JAR 包只被加载一次，否则随着依赖的第三方 JAR 包增多，JVM 的内存会膨胀；
* 跟 JVM 一样，我们需要隔离 Tomcat 本身的类和 Web 应用的类；

第 1 个问题
 Tomcat 的解决方案是自定义一个类加载器 WebAppClassLoader， 并且给每个 Web 应用创建一个类加载器实例。我们知道，Context 容器组件对应一个 Web 应用，因此，每个 Context 容器负责创建和维护一个 WebAppClassLoader 加载器实例。这背后的原理是，不同的加载器实例加载的类被认为是不同的类，即使它们的类名相同。这就相当于在 Java 虚拟机内部创建了一个个相互隔离的 Java 类空间，每一个 Web 应用都有自己的类空间，Web 应用之间通过各自的类加载器互相隔离

第 2 个问题
本质需求是两个 Web 应用之间怎么共享库类，并且不能重复加载相同的类。我们知道，在双亲委托机制里，各个子加载器都可以通过父加载器去加载类，那么把需要共享的类放到父加载器的加载路径下不就行了吗，应用程序也正是通过这种方式共享 JRE 的核心类。因此 Tomcat 的设计者又加了一个类加载器 SharedClassLoader，作为 WebAppClassLoader 的父加载器，专门来加载 Web 应用之间共享的类。如果 WebAppClassLoader 自己没有加载到某个类，就会委托父加载器 SharedClassLoader 去加载这个类，SharedClassLoader 会在指定目录下加载共享类，之后返回给 WebAppClassLoader，这样共享的问题就解决了

第 3 个问题
如何隔离 Tomcat 本身的类和 Web 应用的类？我们知道，要共享可以通过父子关系，要隔离那就需要兄弟关系了。兄弟关系就是指两个类加载器是平行的，它们可能拥有同一个父加载器，但是两个兄弟类加载器加载的类是隔离的。基于此 Tomcat 又设计一个类加载器 CatalinaClassLoader，专门来加载 Tomcat 自身的类。这样设计有个问题，那 Tomcat 和各 Web 应用之间需要共享一些类时该怎么办呢？
老办法，还是再增加一个 CommonClassLoader，作为 CatalinaClassLoader 和 SharedClassLoader 的父加载器。CommonClassLoader 能加载的类都可以被 CatalinaClassLoader 和 SharedClassLoader 使用，而 CatalinaClassLoader 和 SharedClassLoader 能加载的类则与对方相互隔离。WebAppClassLoader 可以使用 SharedClassLoader 加载到的类，但各个 WebAppClassLoader 实例之间相互隔离
spring的加载问题
在 JVM 的实现中有一条隐含的规则，默认情况下，如果一个类由类加载器 A 加载，那么这个类的依赖类也是由相同的类加载器加载。比如 Spring 作为一个 Bean 工厂，它需要创建业务类的实例，并且在创建业务类实例之前需要加载这些类
前面提到，Web 应用之间共享的 JAR 包可以交给 SharedClassLoader 来加载，从而避免重复加载。Spring 作为共享的第三方 JAR 包，它本身是由 SharedClassLoader 来加载的，Spring 又要去加载业务类，按照前面那条规则，加载 Spring 的类加载器也会用来加载业务类，但是业务类在 Web 应用目录下，不在 SharedClassLoader 的加载路径下，这该怎么办呢？
于是线程上下文加载器登场了，它其实是一种类加载器传递机制。为什么叫作“线程上下文加载器”呢，因为这个类加载器保存在线程私有数据里，只要是同一个线程，一旦设置了线程上下文加载器，在线程后续执行过程中就能把这个类加载器取出来用。因此 Tomcat 为每个 Web 应用创建一个 WebAppClassLoader 类加载器，并在启动 Web 应用的线程里设置线程上下文加载器，这样 Spring 在启动时就将线程上下文加载器取出来，用来加载 Bean。
经过对如上问题的处理，最终tomact的类加载器层次结构如下所示
Tomcat类加载器的层次结构

Tomcat线程模型
对于一个网络 I/O 通信过程，比如网络数据读取，会涉及两个对象，一个是调用这个 I/O 操作的用户线程，另外一个就是操作系统内核。

一个进程的地址空间分为用户空间和内核空间，用户线程不能直接访问内核空间。当用户线程发起 I/O 操作后，网络数据读取操作会经历两个步骤：
1.用户线程等待内核将数据从网卡拷贝到内核空间；
2.内核将数据从内核空间拷贝到用户空间；

各种 I/O 模型的区别就是：它们实现这两个步骤的方式是不一样的。

同步阻塞 I/O：用户线程发起 read 调用后就阻塞了，让出 CPU。内核等待网卡数据到来，把数据从网卡拷贝到内核空间，接着把数据拷贝到用户空间，再把用户线程叫醒。


同步非阻塞 I/O：用户线程不断的发起 read 调用，数据没到内核空间时，每次都返回失败，直到数据到了内核空间，这一次 read 调用后，在等待数据从内核空间拷贝到用户空间这段时间里，线程还是阻塞的，等数据到了用户空间再把线程叫醒。



I/O 多路复用：用户线程的读取操作分成两步了，线程先发起 select 调用，目的是问内核数据准备好了吗？等内核把数据准备好了，用户线程再发起 read 调用。在等待数据从内核空间拷贝到用户空间这段时间里，线程还是阻塞的。那为什么叫 I/O 多路复用呢？因为一次 select 调用可以向内核查多个数据通道（Channel）的状态，所以叫多路复用。



异步 I/O：用户线程发起 read 调用的同时注册一个回调函数，read 立即返回，等内核将数据准备好后，再调用指定的回调函数完成处理。在这个过程中，用户线程一直没有阻塞。



Tomcat实现同步非阻塞IO
所谓的非阻塞，其实就是相对以前的 BIO，Tomcat 不再是用一个线程将一个请求从头处理到尾，而是分阶段来执行了
Tomcat 的 NioEndpoint 组件实现了 I/O 多路复用模型，Tomcat 的 NioEndpoint 组件虽然实现比较复杂，但基本原理就是上面两步。我们先来看看它有哪些组件，它一共包含 LimitLatch、Acceptor、Poller、SocketProcessor 和 Executor 共 5 个组件，它们的工作过程如下图所示：


LimitLatch 是连接控制器，它负责控制最大连接数，NIO 模式下默认是 10000，达到这个阈值后，连接请求被拒绝。LimitLatch 用来控制连接个数，当连接数到达最大时阻塞线程，直到后续组件处理完一个连接后将连接数减 1。请你注意到达最大连接数后操作系统底层还是会接收客户端连接，但用户层已经不再接收

Acceptor 跑在一个单独的线程里（通过线程组实现同时跑多个Acceptor线程），它在一个死循环里调用 accept 方法来接收新连接，一旦有新的连接请求到来，accept 方法返回一个 Channel 对象，接着把 Channel 对象交给 Poller 去处理

Poller 的本质是一个 Selector，也跑在单独线程里（通过线程组实现同时跑多个Poller线程）。Poller 在内部维护一个 Channel 数组，每个 Poller 线程都有自己的 Queue，它在一个死循环里不断检测 Channel 的数据就绪状态，一旦有 Channel 可读，就生成一个 SocketProcessor 任务对象扔给 Executor 去处理。

Executor 就是线程池，负责运行 SocketProcessor 任务类，SocketProcessor 的 run 方法会调用 Http11Processor 来读取和解析请求数据。Http11Processor 是应用层协议的封装，Http11Processor 读取 Channel 的数据来生成 ServletRequest 对象，它还会调用容器获得响应，再把响应通过 Channel 写出

如上过程是如何实现高并发的？
高并发就是能快速地处理大量的请求，需要合理设计线程模型让 CPU 忙起来，尽量不要让线程阻塞，因为一阻塞，CPU 就闲下来了。另外就是有多少任务，就用相应规模的线程数去处理。我们注意到 NioEndpoint 要完成三件事情：
1.接收连接；
2.检测 I/O 事件；
3.处理请求；
那么最核心的就是把这三件事情分开，用不同规模的线程数去处理，比如用专门的线程组去跑 Acceptor，并且 Acceptor 的个数可以配置；用专门的线程组去跑 Poller，Poller 的个数也可以配置；最后具体任务的执行也由专门的线程池来处理，也可以配置线程池的大小

Tomcat实现异步非阻塞IO
NIO 和 NIO.2（tomcat的AIO） 最大的区别是，一个是同步一个是异步。异步最大的特点是，应用程序不需要自己去触发数据从内核空间到用户空间的拷贝。
为什么是应用程序去“触发”数据的拷贝，而不是直接从内核拷贝数据呢？这是因为应用程序是不能访问内核空间的，因此数据拷贝肯定是由内核来做，关键是谁来触发这个动作。是内核主动将数据拷贝到用户空间并通知应用程序。还是等待应用程序通过 Selector 来查询，当数据就绪后，应用程序再发起一个 read 调用，这时内核再把数据从内核空间拷贝到用户空间
需要注意的是，数据从内核空间拷贝到用户空间这段时间，应用程序还是阻塞的。所以你会看到异步的效率是高于同步的，因为异步模式下应用程序始终不会被阻塞

异步非阻塞的工作过程
首先，应用程序在调用 read API 的同时告诉内核两件事情：
1.数据准备好了以后拷贝到哪个 Buffer；
2.调用哪个回调函数去处理这些数据;


下图展示了Nio2Endpoint 有哪些组件


从图上看，总体工作流程跟 NioEndpoint 是相似的。

LimitLatch 是连接控制器，它负责控制最大连接数。

Nio2Acceptor 扩展了 Acceptor，用异步 I/O 的方式来接收连接，跑在一个单独的线程里，也是一个线程组。Nio2Acceptor 接收新的连接后，得到一个 AsynchronousSocketChannel，Nio2Acceptor 把 AsynchronousSocketChannel 封装成一个 Nio2SocketWrapper，并创建一个 SocketProcessor 任务类交给线程池处理，并且 SocketProcessor 持有 Nio2SocketWrapper 对象。

Executor 在执行 SocketProcessor 时，SocketProcessor 的 run 方法会调用 Http11Processor 来处理请求，Http11Processor 会通过 Nio2SocketWrapper 读取和解析请求数据，请求经过容器处理后，再把响应通过 Nio2SocketWrapper 写出。

Nio2Endpoint 跟 NioEndpoint 的一个明显不同点是，Nio2Endpoint 中没有 Poller 组件，也就是没有 Selector。这是为什么呢？因为在异步 I/O 模式下，Selector 的工作交给内核来做了。

在异步 I/O 模型里，内核做了很多事情，它把数据准备好，并拷贝到用户空间，再通知应用程序去处理，也就是调用应用程序注册的回调函数。Java 在操作系统 异步 IO API 的基础上进行了封装，提供了 Java NIO.2 API，而 Tomcat 的异步 I/O 模型就是基于 Java NIO.2 实现的

Servlet
我们都知道 Servlet 从 3.0 开始加入了异步，从 3.1 开始又新增了对 IO 非阻塞的支持，那么这个和 Tomcat 线程模型中提到的异步非阻塞是一个概念吗？

从下面的 Tomcat  NIO 线程模型图中，我们可以清晰地看到，NIO 或 AIO 的概念是针对请求的接收来说，而 Servlet 的异步非阻塞主要是针对请求的处理，已经是到了 Tomcat 线程池那里了

我们先来看下 Servlet3.0 前后的变化对比，如下图所示：


概述一下就是，Servlet3.0 之前，Tomcat 线程在执行自定义 Servlet 时，如果过程中发生了 IO，那么 Tomcat 线程只能在那等着结果，这时线程是被挂起的，如果被挂起的多了，自然会影响对其他请求的处理。

所以在 Servlet3.0 之后，支持在这种情况下将这种等待的任务交给一个自定义的业务线程池去做，这样 Tomcat 线程可以很快地回到线程池，处理其他请求。而业务线程在执行完业务逻辑以后，通过调用指定的方法，告诉 Tomcat 线程池接下来可以将业务线程执行的结果返回给调用方，这样就实现了同步转异步的效果

这样做的好处，可能对提高系统的吞吐量有一定帮助，但从 JVM 层面来说，并没有减少工作量。业务线程在执行任务遇到 IO 时，依然会阻塞，现在只是由业务线程池代替了 Tomcat 线程池做了最耗时的那部分工作，这样也许可以将原来的 200 个 Tomcat 线程，拆分成 20 个 Tomcat 线程、180 个业务线程来配合工作

接着我们再聊一下 Servlet3.1 的非阻塞，这块简单来说，就是针对请求消息体的读取，这是个 IO 过程，以前是阻塞式读取，现在支持非阻塞读取了。实现的大致原理就是在读取数据时，新增一个监听事件，在读取完成后由 Tomcat 线程执行回调


## **（2）Openresty**
OpenResty 是一个兼具开发效率和性能的web服务开发平台，它的核心是基于 NGINX 的一个 C 模块（lua-nginx-module），该模块将 LuaJIT 嵌入到 NGINX 服务器中，并对外提供一套完整的 Lua API，透明地支持非阻塞 I/O，提供了轻量级线程、定时器等高级抽象。同时，围绕这个模块，OpenResty 构建了一套完备的测试框架、调试技术以及由 Lua 实现的周边功能库。你可以用 Lua 语言来进行字符串和数值运算、查询数据库、发送 HTTP 请求、执行定时任务、调用外部命令等，还可以用 FFI 的方式调用外部 C 函数。这基本上可以满足服务端开发需要的所有功能。

Openresty将 Nginx 扩展成了一个动态web服务器，Nginx是模块化设计的反向代理软件和HTTP Server，C语言开发。OpenResty是以Nginx为核心的Web开发平台，可以解析执行Lua脚本（OpenResty与Lua的关系，类似于Jvm与Java，不过Java可以做的事情太多了，OpenResty主要用来做Web、API等）

OpenResty 为什么要基于 Nginx?
利用Nginx的如下优势：
1.高并发高性能：基于IO多路复用机制（epoll）实现的IO模型，非阻塞，并发性能好（多进程单线程），一台nginx可支持千万并发连接、静态资源请求下百万rps；
3.高可靠性：指服务器可以持续不间断的长时间运行，因为其通常运行在企业内网的边缘节点上，这种场景需要4个9、5个9甚至更高的可用性，nginx采用多进程而非多线程的架构来保障高可靠，work进程之间互相隔离，master进程负责worker进程的监督管理，比如worker进程故障时候的新worker进程创建和failover；

解决了Nginx的什么问题
1.缺乏动态性：动态指的是程序可以在运行时、在不重新加载的情况下，去修改参数、配置，乃至修改自身的代码。具体到 Nginx 和 OpenResty 领域，你去修改上游、SSL 证书、限流限速阈值，而不用重启服务，就属于实现了动态。开源版本的 Nginx 并不支持动态特性，所以，你要对上游、SSL 证书做变更，就必须通过修改配置文件、重启服务的方式才能生效。而商业版本的 Nginx Plus 提供了部分动态的能力，你可以用 REST API 来完成更新；
2.二次开发成本较高：Nginx 采用 C 语言开发，二次开发门槛较高。市场应用广泛，更多是基于 nginx.conf 预留配置参数，如：反向代理、负载均衡、静态 web 服务器等，如果想让 Nginx 访问 MySQL ，定制化开发一些业务逻辑，难度很高

OpenResty 通过嫁接方式，将 Nginx 和 Lua 脚本相结合，既保留 Nginx 高并发优势，也拥有脚本语言的开发效率（不需要编译，随时开发随时解释执行，避免了c语言模块漫长的开发编译周期），也大大降低了开发门槛。
Lua 是最快的、动态脚本语言，接近 C 语言运行速度。LuaJIT 将一些常用的 lua 函数和工具库预编译并缓存，下次调用时直接使用缓存的字节码，速度很快。另外，Lua 支持协程，这个很重要。协程是用户态的操作，上下文切换不用涉及内核态，系统资源开销小；另外协程占用内存很小，初始 2KB

Openresty 相比Nginx的lua-nginx-module模块的区别
 lua-nginx-module这个 NGINX 的 C 模块确实是 OpenResty 的核心，但它并不等价于 OpenResty，除此之外，OpenResty还包括如下这些：
* NGINX C 模块：OpenResty 的项目命名都是有规范的，以 *-nginx-module命名的就是 NGINX 的 C 模块。OpenResty 中一共包含了 20 多个 C 模块，其中，最核心的就是 lua-nginx-module 和 stream-lua-nginx-module，前者用来处理七层流量，后者用来处理四层流量。这些 C 模块中，有些是需要特别注意的，虽然默认编译进入了 OpenResty，但并不推荐使用。 比如 redis2-nginx-module、redis-nginx-module 和 memc-nginx-module，它们是用来和 redis 以及 memcached 交互使用的。这些 C 库是 OpenResty 早期推荐使用的，但在 cosocket 功能加入之后，它们都已经被 lua-resty-redis 和 lua-resty-memcached 替代，处于疏于维护的状态。OpenResty 后面也不会开发更多的 NGINX C 库，而是专注在基于 cosocket 的 Lua 库上，后者才是未来；
* lua-resty- 周边库：OpenResty 官方仓库中包含 18 个 lua-resty-* 库，涵盖 Redis、MySQL、memcached、websocket、dns、流量控制、字符串处理、进程内缓存等常用库。除了官方自带的之外，还有更多的第三方库；
* 自己维护的 LuaJIT 分支：相对于 Lua，LuaJIT 增加了不少独有的函数；
* 测试框架：OpenResty 的测试框架是test-nginx，同样也是用 Perl 语言来开发的，从名字上就能看出来，它是专门用来测试 NGINX 相关的项目。OpenResty 官方的所有 C 模块和 lua-resty 库的测试案例，都是由 test-nginx 驱动的；
* 调试工具链：OpenResty 项目在如何科学和动态地调试代码上，花费了大量的精力，可以说是达到了极致，openresty-systemtap-toolkit 和 stapxx 这两个 OpenResty 的项目，都基于 systemtap 这个动态调试和追踪工具。使用 systemtap 最大的优势，便是实现活体分析，同时对目标程序完全无侵入；
* 打包工具：OpenResty 在不同发行操作系统（比如 CentOS、Ubuntu、MacOS 等）版本中的打包脚本，出于更细可控力度的目的，都是手工编写的；
* 工程化工具：比如lj-releng 是一个简单有效的 LuaJIT 代码检测工具，类似 luacheck，可以找出全局变量等潜在的问题。reindex 从名字来看是重建索引的意思，它其实是格式化 test-nginx 测试案例的工具，可以重新排列测试案例的编号，以及去除多余的空白符。reindex 可以说是 OpenResty 开发者每天都会用到的工具之一。opsboy 也是一个深藏不露的项目，主要用于自动化部署。OpenResty 每次发布版本前，都会在 AWS EC2 集群上做完整的回归测试，而这个回归测试正是由 opsboy 来部署和驱动的；


核心优势：动态性
动态指的是程序可以在运行时、在不重新加载的情况下，去修改参数、配置，乃至修改自身的代码。具体到 Nginx 和 OpenResty 领域，你去修改上游、SSL 证书、限流限速阈值，而不用重启服务，就属于实现了动态。至于动态和性能之间的关系，很显然，如果这几类操作不能动态地完成，那么频繁的 reload Nginx 服务，自然就会带来性能的损耗。
不过，我们知道，开源版本的 Nginx 并不支持动态特性，所以，你要对上游、SSL 证书做变更，就必须通过修改配置文件、重启服务的方式才能生效。而商业版本的 Nginx Plus 提供了部分动态的能力，你可以用 REST API 来完成更新，但这最多算是一个不够彻底的改良。
但是，在 OpenResty 中，这些桎梏都是不存在的，动态可以说就是 OpenResty 的杀手锏。你可能纳闷儿，为什么基于 Nginx 的 OpenResty 却可以支持动态呢？原因也很简单，Nginx 的逻辑是通过 C 模块来完成的，而 OpenResty 是通过脚本语言 Lua 来完成的——脚本语言的一大优势，便是运行时可以去做动态地改变

OpenResty应用场景
OpenResty 是一个被广泛使用的技术，但它并不能算得上是热门技术。说它应用广，是因为 OpenResty 现在是全球排名第五的 Web 服务器。我们经常用到的 12306 的余票查询功能，或者是京东的商品详情页，这些高流量的背后，其实都是 OpenResty 在默默地提供服务。说它并不热门，那是因为使用 OpenResty 来构建业务系统的比例并不高，使用者大都用 OpenResty 来处理入口流量，并没有深入到业务里面去，自然，对于 OpenResty 的使用也是浅尝辄止，满足当前的需求就可以了。这当然也与 OpenResty 没有像 Java、Python 那样有成熟的 Web 框架和生态有关。接近一半的 OpenResty 使用者，都把 OpenResty 用在 API 网关的开发上，Kong 和 orange 则是 OpenResty 领域中最流行的两个开源网关项目

其他应用场景还包括Faas（利用其动态特性）、边缘计算（利用 Nginx 和 LuaJIT 良好的多平台支持特性）等


OpenResty 整体架构（以及LuaJIT在架构中的位置）



OpenResty 的 worker 进程都是 fork master 进程而得到的， 其实， master 进程中的 LuaJIT 虚拟机也会一起 fork 过来。在同一个 worker 内的所有协程，都会共享这个 LuaJIT 虚拟机，Lua 代码的执行也是在这个虚拟机中完成的

标准 Lua 和 LuaJIT 的关系
标准 Lua 和 LuaJIT 是两回事儿，LuaJIT 只是兼容了 Lua 5.1 的语法。标准 Lua 现在的最新版本是 5.3，LuaJIT 的最新版本则是 2.1.0-beta3。在 OpenResty 几年前的老版本中，编译的时候，你可以选择使用标准 Lua VM ，或者 LuaJIT VM 来作为执行环境，不过，现在已经去掉了对标准 Lua 的支持，只支持 LuaJIT

为什么选择 LuaJIT？
最主要的原因，还是 LuaJIT 的性能优势。
标准 Lua 出于性能考虑，也内置了虚拟机，所以 Lua 代码并不是直接被解释执行的，而是先由 Lua 编译器编译为字节码（Byte Code），然后再由 Lua 虚拟机执行。
而 LuaJIT 的运行时环境，除了一个汇编实现的 Lua 解释器外，还有一个可以直接生成机器代码的 JIT 编译器。开始的时候，LuaJIT 和标准 Lua 一样，Lua 代码被编译为字节码，字节码被 LuaJIT 的解释器解释执行。但不同的是，LuaJIT 的解释器会在执行字节码的同时，记录一些运行时的统计信息，比如每个 Lua 函数调用入口的实际运行次数，还有每个 Lua 循环的实际执行次数。当这些次数超过某个随机的阈值时，便认为对应的 Lua 函数入口或者对应的 Lua 循环足够热，这时便会触发 JIT 编译器开始工作。
JIT 编译器会从热函数的入口或者热循环的某个位置开始，尝试编译对应的 Lua 代码路径。编译的过程，是把 LuaJIT 字节码先转换成 LuaJIT 自己定义的中间码（IR），然后再生成针对目标体系结构的机器码。所以，所谓 LuaJIT 的性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式

Web服务器

