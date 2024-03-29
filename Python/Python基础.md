## **Python基础**

### **编译or解释**

Python语言是先编译再解释的语言。Python 在解释源程序时分为两步：

1. 将源码转为字节码
2. 将字节码转换为机器码
pyc 文件是由 Python 解释器将模块的源码转换为字节码

#### **__pyc__文件**

当我们的python文件被编译过，文件之间存在import关系，就会生成一个__pyc__文件夹

主要意义是加快启动速度。当我们的程序没有修改过，那么下次运行程序的时候，就可以跳过从源码到字节码的过程，直接加载 pyc 文件

1. import过的文件才会自动生成 pyc文件。
2. pyc文件不可以直接看到源码，可以被反编译
3. pyc的内容，是跟python的版本相关的，不同 版本编译后的pyc文件是不同的，2.5编译的pyc文件，2.4版本的python是无法执行的

备注：pyc文件是一种二进制文件，是一种跨平台的字节码，由python的虚拟机来执行的

### **应用场景**

数据处理和分析、服务端（如web开发）开发、人工智能（如机器学习）

### **典型产品**

Tensorflow

### **开发体验**

语法简洁,开发效率高

### **运行效率（性能）**

不够好，原因之一是跟其是解释型语言有关，其二是其多线程对多核处理器的资源使用能力不佳，虽然 Python 确实具有线程功能，但却只能使用单一核心

### **并发编程**

1. 使用threading库进行多线程编程时，因为GIL（全局解释器锁）的存在，同一时间只会有一个获得了 GIL 的线程在跑，其它的线程都处于等待状态等着 GIL 的释放，导致不能利用CPU物理多核的性能加速运算；

2. Python 在 2.6 里引入了 multiprocessing这个多进程标准库，让多进程的 python 程序编写简化到类似多线程的程度，大大减轻了 GIL 带来的不能利用多核的尴尬；

3. 但多进程的方案太重，还有个方案是把关键部分用 C/C++ 写成 Python 扩展，其它部分还是用 Python 来写，让 Python 的归 Python，C 的归 C。一般计算密集性的程序都会用 C 代码编写并通过扩展的方式集成到 Python 脚本里（如 NumPy 模块）。在扩展里就完全可以用 C 创建原生线程，而且不用锁 GIL，充分利用 CPU 的计算资源了；

虽然Python的多线程并并不能利用CPU多核加速计算，但并不是一无是处，我们得区分一下我们的应用是CPU密集型，还是IO密集型。

在IO密集型应用场景中，Python多线程虽然不能利用多核，但IO的时候会阻塞从而释放CPU资源，实现CPU和IO的并行，因此多线程用于IO密集场景，还是可以大幅提升速度的。具体可见这个代码示例：[单线程vs多线程爬虫性能对比](https://github.com/xiaoyuge/kingfish-python/blob/master/concurrent/multi_thread_craw.py)

而在CPU密集的场景，多线程确实不能提供更好的性能，但Python提供了multiprocessing，可以利用多核的优势加速计算，具体可见这个代码示例：[对于CPU密集型业务，对比单线程、多线程和多进程的性能](https://github.com/xiaoyuge/kingfish-python/blob/master/concurrent/thread_process_cpu_bound.py)

#### **Python多线程和多进程使用方法对比**

![multi-process-vs-multi-thread](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/multi-process-vs-multi-thread.png)

#### **协程：asyncio**

多线程有诸多优点且应用广泛，但也存在一定的局限性：

- 比如，多线程运行过程容易被打断，因此有可能出现 race condition 的情况；
- 再如，线程切换本身存在一定的损耗，线程数不能无限增加，因此，如果你的 I/O 操作非常 heavy，多线程很有可能满足不了高效率、高质量的需求;

首先区分一下Sync（同步）和 Async（异步）的概念：

- 所谓 Sync，是指操作一个接一个地执行，下一个操作必须等上一个操作完成后才能执行；
- 而 Async 是指**不同操作间可以相互交替执行**，如果其中的某个操作被 block 了，程序并不会等待，而是会找出可执行的操作继续执行；

不同于多线程，Asyncio 是单线程的，但其内部 event loop 的机制，可以让它并发地运行多个不同的任务，并且比多线程享有更大的自主控制权

Asyncio 和其他 Python 程序一样，是单线程的，它只有一个主线程，但是可以进行多个不同的任务（task），这里的任务，就是特殊的 future 对象。这些不同的任务，被一个叫做 event loop 的对象所控制。

我们可以假设任务只有两个状态：一是预备状态；二是等待状态。所谓的预备状态，是指任务目前空闲，但随时待命准备运行。而等待状态，是指任务已经运行，但正在等待外部的操作完成，比如 I/O 操作。

在这种情况下，event loop 会维护两个任务列表，分别对应这两种状态；并且选取预备状态的一个任务（具体选取哪个任务，和其等待的时间长短、占用的资源等等相关），使其运行，一直到这个任务把控制权交还给 event loop 为止。

当任务把控制权交还给 event loop 时，event loop 会根据其是否完成，把任务放到预备或等待状态的列表，然后遍历等待状态列表的任务，查看他们是否完成：

- 如果完成，则将其放到预备状态的列表；
- 如果未完成，则继续放在等待状态的列表；

而原先在预备状态列表的任务位置仍旧不变，因为它们还未运行。

这样，当所有任务被重新放置在合适的列表后，新一轮的循环又开始了：event loop 继续从预备状态的列表中选取一个任务使其执行…如此周而复始，直到所有任务完成。

Asyncio 中的任务，在运行过程中不会被打断，因此不会出现 race condition 的情况。尤其是在 I/O 操作 heavy 的场景下，Asyncio 比多线程的运行效率更高。因为 Asyncio 内部任务切换的损耗，远比线程切换的损耗要小；并且 Asyncio 可以开启的任务数量，也比多线程中的线程数量多得多

但需要注意的是，很多情况下，使用 Asyncio 需要特定第三方库的支持，比如aiohttp。而如果 I/O 操作很快，并不 heavy，那么运用多线程，也能很有效地解决问题

多线程、多进程和asyncio分别在什么场景下使用：

```Python
if io_bound:
    if io_slow:
        print('Use Asyncio')
    else:
        print('Use multi-threading')
else if cpu_bound:
    print('Use multi-processing')
```

换成白话就是：

- 如果是 I/O bound，并且 I/O 操作很慢，需要很多任务 / 线程协同实现，那么使用 Asyncio 更合适；
- 如果是 I/O bound，但是 I/O 操作很快，只需要有限数量的任务 / 线程，那么使用多线程就可以了；
- 如果是 CPU bound，则需要使用多进程来提高程序运行效率；

名词解释：

- CPU bound(CPU密集型)：CPU密集型也叫计算密集型，是指I/O在很短的时间内就可以完成，但CPU需要大量的计算和处理，特点是CPU占用率相当高，比如压缩解压缩、加密解密、正则表达式搜索等；

- I/O bound（I/O密集型）:I/O密集型是指系统运行过程中大部分的时间是CPU在等IO（硬盘/内存）的读写操作，CPU占用率较低，比如文件处理、网络爬虫、读写数据库等；

这里有两个例子演示如何使用协程来爬取网站：

- [示例1](https://github.com/xiaoyuge/kingfish-python/blob/master/concurrent/async_spider_blog.py)
- [示例2](https://github.com/xiaoyuge/kingfish-python/blob/master/concurrent/async_spider_wiki.py)

更多关于并发编程的代码示例见：[Python并发编程](https://github.com/xiaoyuge/kingfish-python/tree/master/concurrent)

#### **GIL是什么**

GIL（Global Interpreter Lock，即全局解释器锁）,GIL，是最流行的 Python 解释器 CPython 中的一个技术术语。它的意思是全局解释器锁，本质上是类似操作系统的 Mutex。每一个 Python 线程，在 CPython 解释器中执行时，都会先锁住自己的线程，阻止别的线程执行

当然，CPython 会做一些小把戏，轮流执行 Python 线程。这样一来，用户看到的就是“伪并行”——Python 线程在交错执行，来模拟真正并行的线程

CPython引进 GIL 其实主要是这么两个原因：

- 一是设计者为了规避类似于内存管理这样的复杂的竞争风险问题（race condition）；
- 二是因为 CPython 大量使用 C 语言库，但大部分 C 语言库都不是原生线程安全的（线程安全会降低性能和增加复杂度）

下面这张图，就是一个 GIL 在 Python 程序的工作示例。其中，Thread 1、2、3 轮流执行，每一个线程在开始执行时，都会锁住 GIL，以阻止别的线程执行；同样的，每一个线程执行完一段后，会释放 GIL，以允许别的线程开始利用资源

![GIL-work](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/GIL-work.jpg)

从图上会发现一个现象：为什么 Python 线程会去主动释放 GIL 呢？毕竟，如果仅仅是要求 Python 线程在开始执行时锁住 GIL，而永远不去释放 GIL，那别的线程就都没有了运行的机会。

CPython 中还有另一个机制，叫做 **check_interval**，意思是 CPython 解释器会去轮询检查线程 GIL 的锁住情况。每隔一段时间，Python 解释器就会强制当前线程去释放 GIL，这样别的线程才能有执行的机会

不同版本的 Python 中，check interval 的实现方式并不一样。早期的 Python 是 100 个 ticks，大致对应了 1000 个 bytecodes；而 Python 3 以后，interval 是 15 毫秒。当然，我们不必细究具体多久会强制释放 GIL，这不应该成为我们程序设计的依赖条件，我们只需明白，CPython 解释器会在一个“合理”的时间范围内释放 GIL 就可以了

![check-interval](https://github.com/xiaoyuge/Tech-Notes/blob/main/Python/resources/check-interval.jpg)

有了 GIL，并不意味着我们 Python 编程者就不用去考虑线程安全了。即使我们知道，GIL 仅允许一个 Python 线程执行，但前面我也讲到了，Python 还有 check interval 这样的抢占机制，可以看一下这个代码示例：[非线程安全代码示例](https://github.com/xiaoyuge/kingfish-python/blob/master/concurrent/non_thread_safe.py)

#### **如何绕过GIL**

1. 绕过 CPython，使用 JPython（Java 实现的 Python 解释器）等别的实现；
2. 把关键性能代码，放到别的语言（一般是 C++）中实现，然后再提供 Python的调用接口；

PS：如果你的应用真的对性能有超级严格的要求，比如 100us 就对你的应用有很大影响，那Python 可能不是你的最优选择

### **垃圾回收**

Python是带GC的，关于Python的GC，可以看一下这里的代码示例：[Python GC代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/gc)

#### **内存泄漏**

- 这里的泄漏，并不是说你的内存出现了信息安全问题，被恶意程序利用了，而是指程序本身**没有设计好，导致程序未能释放已不再使用的内存**;
- 内存泄漏也不是指你的内存在物理上消失了，而是意味着代码在分配了某段内存后，因为设计错误，**失去了对这段内存的控制，从而造成了内存的浪费**;

### **Python的GC总结**

1. 垃圾回收是 Python 自带的机制，用于自动释放不会再用到的内存空间；
2. 引用计数是其中最简单的实现，不过这只是充分非必要条件，因为循环引用需要通过不可达判定，来确定是否可以回收；
3. Python 的自动回收算法包括标记清除和分代收集，主要针对的是循环引用的垃圾收集；
4. 调试内存泄漏方面， objgraph 是很好的可视化分析工具;

### **函数**

在 Python 中，函数是一等公民（first-class citizen），函数也是对象。函数的几个核心概念：

- 我们可以把函数赋予变量：[示例](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/func_to_var.py)
- 我们可以把函数当作参数，传入另一个函数中：[示例](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/func_to_param.py)
- 我们可以在函数里定义函数，也就是函数的嵌套：[示例](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/func_inline_func.py)
- 函数的返回值也可以是函数对象（闭包）：[示例](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/func_closure.py)

### **装饰器**

所谓的装饰器，其实就是通过装饰器函数，来修改原函数的一些功能，使得原函数不需要修改

Decorators is to modify the behavior of the function through a wrapper so we don’t have to actually modify the function

一个装饰器的代码示例：[装饰器](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/decorator_impl.py)

装饰器的简洁优雅实现：[装饰器语法糖](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/decorator_sugar.py)

装饰器传递参数：[装饰器传参](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/decorator_pass_param.py)

装饰器还有更大程度的灵活性，装饰器可以接受原函数任意类型和数量的参数，除此之外，它还可以接受自己定义的参数：[装饰器传自定义参数](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/decorator_self_param.py)

函数被装饰以后，它的元信息变了。元信息告诉我们“它不再是以前的那个函数，而是被 wrapper() 函数取代了”

为了解决这个问题，我们通常使用内置的装饰器@functools.wrap，它会帮助保留原函数的元信息（也就是将原函数的元信息，拷贝到对应的装饰器函数里）:[装饰器保留元信息](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/decorator_hold_meta.py)

类也可以作为装饰器：[类装饰器](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/class_decorator.py)

Python也支持多个装饰器，比如写成下面这个样子：

```Python
@decorator1
@decorator2
@decorator3
def func():
    ...
```

它的执行顺序从里到外，所以上面的语句也等效于下面这行代码

```Python
decorator1(decorator2(decorator3(func)))
```

[多个装饰器示例](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/multi_decorator.py)

#### **装饰器的应用场景**

Python的装饰器的应用场景有点像AOP的应用场景，把一些常用的业务逻辑分离，提高程序可重用性，降低耦合度，提高开发效率

1. 身份验证

   如果认证通过，你就可以顺利登录；如果不通过，就抛出异常并提示你登录失败。

```python

import functools

def authenticate(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        request = args[0]
        if check_user_logged_in(request): # 如果用户处于登录状态
            return func(*args, **kwargs) # 执行函数post_comment() 
        else:
            raise Exception('Authentication failed')
    return wrapper
    
@authenticate
def post_comment(request, ...)
    ...
 
```

2. 日志记录

日志记录同样是很常见的一个案例。在实际工作中，如果你怀疑某些函数的耗时过长，导致整个系统的 latency（延迟）增加，所以想在线上测试某些函数的执行时间，那么，装饰器就是一种很常用的手段

```Python

import time
import functools

def log_execution_time(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        res = func(*args, **kwargs)
        end = time.perf_counter()
        print('{} took {} ms'.format(func.__name__, (end - start) * 1000))
        return res
    return wrapper
    
@log_execution_time
def calculate_similarity(items):
    ...
```

3. 输入合理性检查

   在大型公司的机器学习框架中，我们调用机器集群进行模型训练前，往往会用装饰器对其输入（往往是很长的 JSON 文件）进行合理性检查。这样就可以大大避免，输入不正确对机器造成的巨大开销

```Python

import functools

def validation_check(input):
    @functools.wraps(func)
    def wrapper(*args, **kwargs): 
        ... # 检查输入是否合法
    
@validation_check
def neural_network_training(param1, param2, ...):
    ...
```

4. 缓存

   以 Python 内置的 LRU cache 为例，LRU cache在 Python 中的表示形式是@lru_cache。@lru_cache会缓存进程中的函数参数和结果，当缓存满了以后，会删除 least recenly used 的数据，正确使用缓存装饰器，往往能极大地提高程序运行效率

   大型公司服务器端的代码中往往存在很多关于设备的检查，比如你使用的设备是安卓还是 iPhone，版本号是多少。这其中的一个原因，就是一些新的 feature，往往只在某些特定的手机系统或版本上才有（比如 Android v200+）

   这样一来，我们通常使用缓存装饰器，来包裹这些检查函数，避免其被反复调用，进而提高程序运行效率，比如写成下面这样

```python
@lru_cache
def check(param1, param2, ...) # 检查用户设备类型，版本号等等
    ...
```

### **迭代器**

#### **容器、可迭代对象和迭代器**

在 Python 中一切皆对象，对象的抽象就是类，而对象的集合就是容器，所有的容器都是可迭代的（iterable）

可迭代对象，通过 iter() 函数返回一个迭代器，再通过 next() 函数就可以实现遍历，for in 语句将这个过程隐式化

如何判断一个对象是否可迭代？这里有段代码用来演示如何判断：[判断是否可迭代](https://github.com/xiaoyuge/kingfish-python/blob/master/basic/iterable.py)

### **Python常用的库**

- **爬虫开发**：requests（http请求）、BeautifulSoup（html/xml解析） （[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/crawler)）
- **WEB开发**：flask/Django （[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/flask_app)）
- **数据处理**：Pandas（[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/pandas_case)）
- **Excel办公**：xlwings （[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/excel)）
- **科学计算**：Numpy （[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/numpy_case)）
- **数据可视化**：pyecharts （[代码示例](https://github.com/xiaoyuge/kingfish-python/tree/master/data_visual)）
