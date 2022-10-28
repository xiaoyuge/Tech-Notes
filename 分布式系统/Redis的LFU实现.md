
最近看了Redis的LFU缓存淘汰的实现，被惊艳到了。同样是敲代码，有人把它上升成了艺术。

Redis在4.0之后支持配置缓存淘汰使用LFU（Least Frequently Used, 最少频繁使用算法）。先简单回顾下LFU算法：

LFU算法
LFU是缓存淘汰算法。一般我们可用于缓存的空间有限，但是我们需要缓存的数据可能是无限，为了能够提升缓存的命中率，我们希望留在缓存中的会是我们之后较短时间内最可能有到的那部分，这时候缓存淘汰就有用武之地了。

LFU算法的思路是： 我们记录下来每个（被缓存的/出现过的）数据的请求频次信息，如果一个请求的请求次数多，那么它就很可能接着被请求。

在数据请求模式比较稳定（没有对于某个数据突发的高频访问这样的不稳定模式）的情况下，LFU的表现还是很不错的。在数据的请求模式大多不稳定的情况下，LFU一般会有这样一些问题：

1）微博热点数据一般只是几天内有较高的访问频次，过了这段时间就没那么大意义去缓存了。但是因为在热点期间他的频次被刷上去了，之后很长一段时间内很难被淘汰；
2）如果采用只记录缓存中的数据的访问信息，新加入的高频访问数据在刚加入的时候由于没有累积优势，很容易被淘汰掉；
3）如果记录全部出现过的数据的访问信息，会占用更多的内存空间；

对于上面这些问题，其实也都有一些对应的解决方式，相应的出现了很多LFU的变种。如：Window-LFU、LFU*、LFU-Aging。在Redis的LFU算法实现中，也有其解决方案。

近似计数算法 – Morris算法
Redis记录访问次数使用了一种近似计数算法——Morris算法。Morris算法利用随机算法来增加计数，在Morris算法中，计数不是真实的计数，它代表的是实际计数的量级。

算法的思想是这样的：算法在需要增加计数的时候通过随机数的方式计算一个值来判断是否要增加，算法控制 在计数越大的时候，得到结果“是”的几率越小。



我们来玩个逃脱游戏，游戏规则是这样的：

从上面的游戏看，最顺利的情况下，经过7次摇号，你就能逃离了。但是实际呢？概率是个很神奇的东西。





上面按游戏规则试了10次的结果的图（都是一张图，后面几张是为了能看清楚，对某个范围进行了放大的结果）。横轴为楼层数，纵轴为达到对应楼层尝试的次数。

从上面的图能看到， 曲线整体都是越来约陡的，也就是想要得到“是”越来越难。虽然对于同一层，几次游戏可能使用的次数会不同，但是基本可以认为是在一个大致范围内的。按着上面的图，我们可以说：如果你到了第三层，你应该尝试了上千次。这个时候，3这个数字已经不仅仅代表3了，它代表的是一个量级，代表的一个近似计数。

近似计数算法一个很重要的参数就是概率的变化，上面的游戏里，概率一直以上层的0.1降低，如果这个降低的速率更快，比如是上层的概率的0.05，那么可以想象，想要上一层变得更难了，上一层楼我要试的次数会更多，那么曲线会变得更陡，能标识得近似数范围也会变得更大。

Redis中的LFU实现
Redis中关于缓存淘汰的核心源码都在evict.c文件中，其中实现LFU的主要方法是：LFUGetTimeInMinutes、LFUTimeElapsed、LFULogIncr、LFUDecrAndReturn。下面具体来说：

Redis中实现LFU算法的时候，有这个两个重要的可配置参数：

server.lfu_log_factor : 能够影响计数的量级范围，即上表中的factor参数；
server.lfu_decay_time: 控制LFU计数衰减的参数。
关于这两个参数的作用接着往下看。

访问计数
前面说了Redis的LFU实现使用了近似计数算法，这个算法的特点是能够用一个较小的数表示一个很大的量级，所以对于Redis来说统计频次不需要太多空间和内容，只需要一个不那么大的数就行（这个特性解决了前面说的LFU的常见问题3）。正好，Redis的LRU算法实现里，用了一个24位的redisObject->lru字段，拿到LFU中正好合用。Redis没有全部用掉这24位，只拿了其中8位用来做计数，剩下的16位另作别用。

 *          16 bits      8 bits
 *     +----------------+--------+
 *     + Last decr time | LOG_C  |
 *     +----------------+--------+
1
2
3
4
8个bit位最大为255，从Redis文档中贴出来的数据（如下表）可以看到，不同的factor的值能够控制计数代表的量级的范围，当factor为100时，能够最大代表10M，也就是千万级别的命中数。



第一列的factor，就是前头说的配置项server.lfu_log_factor，那么这个配置的作用就比较明显了，就是用来控制概率衰减的速率。具体怎么控制的，我们来看下LFULogIncr方法的实现：

/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really implemented. Saturate it at 255. */
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}

从方法实现我们可以看到，LFULogIncr先通过生成随机数的方式获取了double值r。

    double r = (double)rand()/RAND_MAX;

这里的rand()方法会生成一个 0 ~ RAND_MAX 之间的随机数，所以r的范围也就是0~1之间。我们可以认为r rr的值是随机的，那么我们可以认为：
r的值在0 ~ 1范围内，也就是r<=1，的概率为100%（1）；
r的值在0 ~ 0.9范围内，也就是r<=0.9，的概率为90%（0.9）；
…
r的值在0 ~ 0.1范围内，也就是r<=0.1，的概率为10%（0.1）；
r的值在0 ~ 0.01范围内，也就是r<=0.01，的概率为1%（0.01）；

这个r就可以看成是上头说的那个游戏里的摇到的号，不过它是摇到在号段内就行了，难度小很多。这个号段，就是接下来生成的一个和计数值有关联的另一个值p：

    double p = 1.0/(baseval*server.lfu_log_factor+1);

其中的baseval是根据计数值计算出来的，所以当计数值一定的时候，p的计算公式为：
当计数值小于等于LFU_INIT_VAL时： p = 1
当计数值大于LFU_INIT_VAL时： p = 1 ÷ ( ( c o u n t e r − L F U _ I N I T _ V A L ) × f a c t o r + 1 )

我们先忽略LFU_INIT_VAL，这个稍后来说，我们先简化p pp的计算公式为：
p=1÷(counter×factor+1)

公式中的factor就是前头说的Redis配置项server.lfu_log_factor，我们先假定我们配置它为10。那么有：
c o u n t e r = 1 , p = 1 ÷ ( 1 × 10 + 1 ) = 1 11 ≈ 0.090909 
c o u n t e r = 2 , p = 1 ÷ ( 2 × 10 + 1 ) = 1 21 ≈ 0.047619 
c o u n t e r = 3 , p = 1 ÷ ( 3 × 10 + 1 ) = 1 31 ≈ 0.032258 


上面的图就是p随counter变化的曲线图。除了factor为0的之外，可以看到随着counter的增大，p变得越来越小，也就是想要增加一次counter就越来越难，也就需要更多次数才可能增加一次counter。
而factor越大，在相同的counter数下，p值越小。 

现在我们回过头来看官方的数据，其实就能看出来些什么了。


用上面实现的逻辑来跑下：


对比官方的数据，大体上是差不多的。列表中的数据，不知道是否有留意到，当factor为0时，要计数100次，需要的counter值是104。前面说了，factor为0时，p始终是等于1的，也就是每次增加计数值的概率都是100%，那么计数100次，应该只需要counter是100就行了，为什么现在得是104呢？这个就牵扯涉及到了前面我们暂时忽略掉的一个值 – LFU_INIT_VAL。LFU_INIT_VAL的值是Redis设定的常量，等于5，关于这个值的作用，源码中有这样的注释：

New keys don’t start at zero, in order to have the ability to collect some accesses before being trashed away, so they start at COUNTER_INIT_VAL. The logarithmic increment performed on LOG_C takes care of COUNTER_INIT_VAL when incrementing the key, so that keys starting at COUNTER_INIT_VAL (or having a smaller value) have a very high chance of being incremented on access.

（注释里头说的是COUNTER_INIT_VAL，但是实际源码里用的就是LFU_INIT_VAL）

Redis中在创建新对象的时候，它的LFU计数值不是从0开始的，这个在createObject（object.c文件中）方法中能看到。当使用LFU的时候，它的lru值是这样初始化的：

        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;

初始计数值会直接就是LFU_INIT_VAL，也就是5，这点就是为了解决前头说到的LFU算法的第常见问题2：新增的缓存可能还没开始累积优势就被淘汰了。给新增缓存一个5的计数值，那么至少能保证新增缓存比真正冷（计数值低于5）的数据要晚淘汰。

可是初始计数值就是从5开始的，为什么会有出现计数值低于5的数据呢？这是Redis为了解决LFU解决前头说的常见问题1引入的手段 – 计数衰减。

计数衰减
在缓存被访问时，会更新数据的访问计数，更新的步骤是：先在现有数据的计数上进行计数衰减，再对完成衰减后的计数进行增加。

计数衰减的实现在LFUDecrAndReturn方法中：

/* Return the current time in minutes, just taking the least significant
 * 16 bits. The returned time is suitable to be stored as LDT (last decrement
 * time) for the LFU implementation. */
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime/60) & 65535;
}

/* Given an object last access time, compute the minimum number of minutes
 * that elapsed since the last access. Handle overflow (ldt greater than
 * the current 16 bits minutes time) considering the time as wrapping
 * exactly once. */
unsigned long LFUTimeElapsed(unsigned long ldt) {
    unsigned long now = LFUGetTimeInMinutes();
    if (now >= ldt) return now-ldt;
    return 65535-ldt+now;
}

/* If the object decrement time is reached decrement the LFU counter but
 * do not update LFU fields of the object, we update the access time
 * and counter in an explicit way when the object is really accessed.
 * And we will times halve the counter according to the times of
 * elapsed time than server.lfu_decay_time.
 * Return the object frequency counter.
 *
 * This function is used in order to scan the dataset for the best object
 * to fit: as we check for the candidate, we incrementally decrement the
 * counter of the scanned objects if needed. */
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    return counter;
}

前面说到过Redis是使用原来用于LRU的24位的redisObject->lru中的8位来进行计数的，剩下的16位另作它用 – 用于计数值的衰减。开头说的Redis配置项 – server.lfu_decay_time，也是用于控制计数的衰减的。

server.lfu_decay_time 的代表的含义是计数衰减的周期长度，单位是分钟。当时间过去一个周期，计数值就会减1。（按照源码中注释说的，当计数值大于两倍的COUNTER_INIT_VAL时，计数值是以减半进行衰减的，小于两倍的COUNTER_INIT_VAL时每次减一。但是现在从源码看来都是直接减一的，应该这块之前这么弄有问题所以改了，注释没改）

衰减周期的计算就是redisObject->lru中那16位，它记录的就是上次进行衰减的时间。衰减周期数就等于从上次衰减到现在经过的时间除以衰减周期长度 server.lfu_decay_time。

    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;

计数衰减用于解决前面说的问题1，即类似微博热点数据，短时间内计数值会冲到非常高，这样后续很难被淘汰的问题。

缓存淘汰的执行
使用LFU方式进行缓存缓存淘汰，其实和使用LRU方式的执行过程基本完全一致，只是把idle换成了 255 - counter。