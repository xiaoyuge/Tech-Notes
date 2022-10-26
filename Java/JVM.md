## **JVM**
Sun官方定义的Java技术体系包括：
- java程序设计语言：
- 适配各种硬件平台的java虚拟机（jvm）；——『一次编写，到处运行』
- Class文件规范；
- Java API类库；
- 来自商业机构和开源社区的第三方Java类库；

**jdk**：jvm + Java API类库 + 编译（Javac）、执行（java）、调试（jstack、jmap、jhat、jps）、打包（jar）等一系列工具链\
**jre**：jvm + Java API类库(Java SE API 子集)

JDK提供的常用性能监控和调试工具：这些命令行工具大多数是JAVA_HOME/lib/tools.jar类库的一层薄薄的包装，他们的主要功能代码是在tools类库中实现，比如jstack命令的实现其实是一个叫做JStack.java的类

- **jps**：显示系统内所有的Hotspot虚拟机进程
- **jmap（Memory map for java）**：生成虚拟机的内存转储快照（heapdump文件），如果不使用jmap要想获取java堆转储快照，可以使用-XX：+HeapDumpOnOutOfMemoeryError，让虚拟机在OOM异常的时候自动生成dump文件，或通过-XX:+HeapDumpOnCtrlBreak参数可以使用【ctrl】+[break]快捷键让虚拟机生成dump文件；又或者在Linux系统下通过kill -3 命令发送进程退出信号『吓唬』一下虚拟机，也能拿到dump文件
- **jhat（JVM heap Dump Browser）**：用于分析heapdump文件（二进制文件），会建立一个http服务器，用户可以可视化的查看分析结果；
- **jstack（stack trace for java）**：显示虚拟机的线程快照（threaddump/javacore文件）,可以获取当前虚拟机内每一条线程正在执行的方法堆栈的集合；

jstack等命令的具体实现机制：依赖jvm的attach机制，是jvm提供一种jvm进程间通信的能力，能让一个进程传命令给另外一个进程，并让它执行内部的一些操作，比如说我们为了让另外一个jvm进程把线程dump出来，那么我们跑了一个jstack的进程，然后传了个pid的参数，告诉它要哪个进程进行线程dump

Attach能做些什么?
总结起来说，比如内存dump，线程dump，类信息统计(比如加载的类及大小以及实例个数等)，动态加载agent(使用过btrace的应该不陌生)，动态设置vm flag(但是并不是所有的flag都可以设置的，因为有些flag是在jvm启动过程中使用的，是一次性的)，打印vm flag，获取系统属性等

Attach在jvm里如何实现？
核心是AttachListener线程，该线程会从队列里不断取AttachOperation，然后找到请求命令对应的方法进行执行，比如我们一开始说的jstack命令，找到 { “threaddump”, thread_dump }的映射关系，然后执行thread_dump方法

Java虚拟机的内存管理：
Java虚拟机在执行java程序的过程中会把管理的内存划分为若干个不同的区域，有的区域随着虚拟机进程的启动而存在（如方法区、堆等），有些区域则随着用户线程的启动和结束而建立和销毁（如虚拟机栈、程序计数器、本地方法栈等）
jvm的运行时数据区包括:堆（所有线程共享）、方法区（所有线程共享）、虚拟机栈、本地方法栈和程序计数器（线程私有）
方法区是jvm规范规定的逻辑区域，不同的jvm实现可能不一样，比如 hotspot jvm 通过永久代（Permanent Generation）来实现方法区。

线程私有的内存区域（随着线程的启动和结束而建立和销毁）：
程序计数器：存储当前线程所执行的字节码指令的行号，字节码解释器工作的时候就是通过改变这个计数器的值来选取下一条要执行的字节码指令，如果执行的是native方法，这个计数器的值为空（Undefined），此内存区域是java虚拟机规范中唯一没有规定任何OutOfMemoryError情况的区域；
虚拟机栈：描述的是java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧（stack frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法从调用到执行完成的过程，就是一个栈帧在虚拟机栈入栈到出栈的过程，局部变量表所需内存空间的大小在编译期间即确定。这个区域有两种异常情况，一个是当线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError（如深度递归），一个是在上述情况下如果虚拟机栈可以动态扩展，但扩展时无法申请到足够的内存，则会抛出OutOfMemoryError；
本地方法栈：与虚拟机栈作用类似，只是为Native方法服务，也会抛出tackOverflowError和OutOfMemoryError；

线程共享的内存区域（堆和方法区）：
堆：所有的对象实例都在堆上分配（包括数组），堆中还可细分为新生代和老年代，新生代又可细分为Eden空间、from Survivor空间和to Survivor等空间，进一步划分的目的是为了更好地回收内存或者更快地分配内存，如果在堆中没有内存完成对象实例分配且堆也无法再扩展时，抛出OutOfMemoryError异常；
方法区：用于存储被虚拟机加载的类信息、常量、静态变量、即时编译器（JIT）编译后的代码等数据。方法区不等于永久代，仅仅是因为Hotspot虚拟机团队选择使用永久代来实现方法区并且把GC分代收集也扩展到了方法区，这样HotSpot虚拟机的垃圾收集器l可以像管理Java堆一样管理这部分内存，能够省去专门为方法区编写内存管理代码的工作。这一区域的内存垃圾回收目标主要针对废弃常量和类的卸载。当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

补充：
Hotspot虚拟机针对多个线程在堆上新建对象时进行内存分配的同步控制问题的解决方案，叫TLAB（Thread Local Allocation Buffer）：每个线程在堆中预先分配一小块内存区域，在给对象分配内存的时候，直接在自己这块【私有】内存中分配，当这部分区域用完之后，再分配新的【私有】区域。所以，【这部分Buffer内存是从堆中划分出来的，但是是本地线程私有的！！！】。这样每个线程都单独拥有一个空间，如果需要分配内存，就在自己的空间上分配，这样就不存在竞争的情况，可以大大提升分配效率。
因此，因为有了TLAB技术，堆内存并不是完完全全的线程共享，其eden区域中还是有一部分空间是分配给线程独享的。
注意：
1.我们说TLAB是线程独享的，但是只是在【分配】这个动作上是线程独享的，至于在读取、垃圾回收等动作上都是线程共享的；
  1.1 每个线程在初始化的时候都会去堆内存中申请一块TLAB，并不是说这个TLAB区域的内存其他线程完全无法访问了，其他线程的读取还是可以的，只不过无法在这个区域中分配内存而已；
  1.2 在TLAB分配之后，并不影响对象的移动和回收，即对象刚开始可能通过TLAB分配内存，存放在eden区，但还是会被垃圾回收或者移动到survivor space、old gen等；
 2.TLAB是在eden区分配的，因为eden区域本身不太大，而且TLAB空间的内存也非常小，默认情况下只占eden空间的1%，所以，必然存在一些大对象无法在TLAB中直接分配，这个时候可能在eden区或old gen进行分配，这种分配就需要进行同步控制，因此效率会低，所以我们经常说：小对象比大对象分配起来更加高效。
3.TLAB只是Hotspot虚拟机的一个优化方案，Java虚拟机规范中并没有关于TLAB的任何规定，所以，不代表所有的虚拟机都有这个特型；

Class文件结构（规范）
Java虚拟机不和包括Java在内的任何语言绑定，它只与『Class文件』这种特定的二进制文件格式所关联，Class文件中包含了Java虚拟机指令集和符号表以及若干其他辅助信息。
因此，任何一门语言都可以编译成能被Java虚拟机所接受的有效的Class文件。作为一个通用的、机器无关的执行平台，任何其他语言的实现者都可以将Java虚拟机作为语言的产品交付媒介。

Class文件是一组以8位字节为基础单位的二进制字节流，各个数据项目严格按照顺序紧凑地排列在Class文件之中，中间没有添加任何分隔符。当遇到需要占用8位字节以上空间的数据项时，则会按照高位在前（这种顺序称为Big-Endian,指最高位字节在地址最低位，最低位字节在地址最高位的顺序来存储数据）的方式分割成若干个8位字节进行存储。

Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：无符号数和表
无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节
表是由多个无符号数或者其他表作为数据项构成的符合数据类型，用于描述有层次关系的复合结构的数据，所有表习惯性地以『_info』结尾，整个Class文件本质上就是一张表
Class文件中的表数据项有：常量池、字段表、方法表和属性表

常量池
常量池可以理解为Class文件之中的资源仓库，是Class文件结构中与其他项目关联最多的数据类型，Class文件中很多数据项都要引用常量池中的常量，也是占用Class文件空间最大的数据项目之一。
常量池主要用于存放两大类型常量：字面量（literal）和 符号引用（symbolic reference）
字面量比如文本字符串、声明为final的常量值（基本类型）等；
符号引用指
1.类和接口的全限定名;
2.字段的名称和描述符，字段的描述符是指字段的数据类型，不包括定义字段的作用域（public、private、protected）、实例还是静态（static）、可变性（final）、并发可见性（volatile）、可否被序列化（transient）等；
3.方法的名称和描述符，方法的描述符是指方法的参数列表（包括数量、类型以及顺序）和返回值；——方法的参数列表严格按照参数的顺序放在一组小括号内()
PS：Java代码在进行Javac编译的时候，不像c和c++那样有『连接』这一步骤，而是在虚拟机加载class文件的时候进行动态连接，也就是说，在Class文件中不会保存各个方法、字段的最终内存地址信息，因此这些字段、方法的符号引用不经过运行期转换的话无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析，翻译到具体的内存地址。

class文件常量池 vs 运行时常量池
运行时常量池是方法区的一部分,运行时常量池相对于CLass文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入CLass文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用比较多的就是String类的intern()方法。

字段表
字段表用于描述接口或者类中声明的变量，包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量
一个字段的信息一部分用标志位来描述，一部分引用常量池中的常量来描述
用标志位来描述的：字段的作用域（public、private、protected修饰符）、实例还是静态变量（static）、可变性（final）、并发可见性（volatile）、可否被序列化（transient）
引用常量池来描述：字段名称和描述，描述指字段的数据类型

方法表
与字段表类似，一部分用标志位描述，一部分引用常量池中的常量来描述，方法的参数列表（包括参数数量、参数类型和顺序）和返回值通过引用常量池中的常量来描述
另外，方法中的java代码，经过编译器编译成字节码指令后，存放在方法属性表集合中的一个名为『code』的属性里面

属性表
在Class文件、字段表和方法表里都可以携带自己的属性表集合，以用于描述某些场景专有的信息
比如方法表使用的属性表：Code、Exceptions
比如字段表使用的属性表：ConstantValue
比如类文件使用的属性表：InnerClasses、SourceFile
其中，Code属性是Class文件中最重要的一个属性，如果把一个Java程序中的信息分为代码和元数据两部分，那么在整个class文件里，Code属性用于描述代码（方法体里的java代码），所有其他的数据项目都用于描述元数据（包括类、字段、方法定义及其他信息）。

类加载机制
类从被加载到虚拟机内存中开始，到卸载出内存为止，整个生命周期包括:
加载-》连接（验证-》准备-》解析）-》初始化-》使用-》卸载
上面的过程基本是按顺序进行的，除了【解析】阶段，可能在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定）。

什么情况会导致类被加载？
jvm虚拟机规范并没有强制约定，但严格规定了【有且仅有】如下5种情况必须立即对类进行【初始化】（而类加载、连接等阶段自然要在此之前开始）
1.遇到new（实例化对象）、getstatic（读取类的静态字段，不包括final字段）、putstatic（设置类的静态字段，不包括final字段）和invokestatic（调用类的静态方法）这4条字节码指令的时候；——对于getStatic和putStatic，只有直接定义这个字段的类才会被初始化，如果通过子类引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化；
2.使用java.lang.reflect包的方法对类进行反射调用的时候；
3.当初始化一个类的时候，如果发现其父类还没有初始化，则需要先进行父类的初始化；
4.当虚拟机启动的时候，需要指定一个要执行的主类（包含main(）方法的那个类)，虚拟机会先初始化这个主类；——有点类似第一点的invokestatic
5.JDK1.7的动态语言场景，如果java.lang.invoke.MethodHandle实例最后解析的结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄对应的类没有初始化，需要先触发其初始化；

以上5中情况，称为对一个类进行【主动引用】，除此之外，所有引用类的方式都不会触发初始化，称为【被动引用】

【加载】阶段做的事情：
1.通过一个类的全限定名来获取定义此类的二进制字节流；
2.将这个字节流代表的静态存储结构转化为方法区的运行时数据结构；
3.在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问接口（jvm规范并没有明确规定该对象是在堆中，对于HotSpot虚拟机，Class对象比较特殊，虽然是对象，但是是存放在方法区里的）；
加载阶段完成之后，虚拟机外部的二进制字节流就存储在方法区中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范并未规定此区域的具体数据结构。

【连接】阶段做的事情：
1.验证阶段：确保class文件的字节流中的信息符合虚拟机规范还要求且不会危害虚拟机安全，大致包括4个阶段的检验动作：文件格式验证、元数据验证、字节码验证和符号引用验证;
2.准备阶段：为类变量（非实例变量）分配内存空间并设置类变量的初始值（该变量类型的零值）；
3.解析阶段：将常量池中的符号引用替换成直接引用的过程，直接引用可以是指向目标的直接地址、相对偏移量或一个能间接定位到目标的句柄；

对于第2点，需要注意的一个点是：
局部变量不像类变量一样存在『准备阶段』，类变量有两次赋值过程，一次在准备阶段，赋予类型默认初始值，一次在初始化阶段，赋予程序猿定义的初始值。因此，对类变量来说，即使程序猿没有为类变量赋值也没关系，类变量仍然具有一个默认初值。但局部变量不一样，如果一个局部变量定义了但没有赋初始值是不能使用的，当然编译器在编译期间就会检查提示这一点。

【初始化】阶段做的事情：
执行类构造器<clinit>进行初始化，<clinit>合并了静态变量的赋值语句以及static块中的语句，顺序是由语句在源文件中出现的顺序所决定。虚拟机会保证在子类的<clinit>执行之前，父类的<clinit>方法已经执行完毕。虚拟机还会保证一个类的<clinit>方法在多线程环境中被正确地加锁、同步。如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>方法，其他线程都需要阻塞等待知道活动线程执行<clinit>方法完毕。

类加载器种类：
1.启动类加载器（Bootstrap Classloader）：c++语言实现，是虚拟机自身的一部分，负责加载存放在<JAVA_HOME>\lib目录下 或 -Xbootclasspath参数指定路径中的，并且是虚拟机能够识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库。该类加载器无法被java程序直接引用，用户在编写自定义加载器时，如果要将加载请求委派给启动类加载器，直接使用null即可；
2.扩展类加载器（Extension Classloader）：负责加载<JAVA_HOME>\lib\ext目录中的 或 被java.ext.dirs系统变量所指定的路径中的所有类库；
3.应用程序类加载器（Application Classloader）：是Classloader.getSystemClassloader()方法的返回值，所以一般也称为系统类加载器，负责加载用户类路径（classpath）上所指定的类库，如果没有自定义过自己的类加载器，一般情况下这个就是程序中的默认类加载器；

双亲委派模型：除了顶层的启动类加载器之外，其余的类加载器都应当有自己的父加载器，这里的父子关系不是以继承的关系来实现，而是以组合的关系来实现；
如果一个类加载器收到了类加载的请求，它首先不会自己尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的类加载请求最终都会被传送到顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求的时候，子加载器才会尝试自己去加载。这也导致子类加载器可以看到父类加载器加载的所有类，但父类加载器看不到子类加载器加载的所有类，不属于父子关系路径下的不同类加载器加载的类也互相看不见。

注意，双亲委派模型并不是一个强制性的约束模型，而是推荐的类加载器实现方式，双亲委派模型是可能被破坏的，比如下面的几种方式：
1.自定义类加载器覆盖父类的loadClass（）方法，而该方法的双亲委派的逻辑就可能被覆盖，因此jdk1.2之后不再提倡用户覆盖loadclass（）方法，而是覆盖findClass（）方法然后把自己的类加载逻辑写到findClass（）方法中；
2.模型本身缺陷，如果上层类加载器加载的基础类需要调用下层类加载器的类怎么办？线程上下文类加载器（Thread Context Classloader）；
3.对程序动态性的追逐：代码热替换、模块热部署等，如OSGI；

所以，请求加载类的类加载器叫这个类的【初始类加载器】，最终完成类加载的类加载器叫做这个类的【定义类加载器】

LoadClass方法的逻辑
1.首先会调用findLoadedClass（）方法判断该类文件是否已经被当前类加载器实例给加载了（注意是实例，意思是同一个类加载器即使不同实例，加载的同一个类也不同，即使是同一个类文件，如果是由不同的类加载器实例加载的，那么它们的类型是不相同的，强制转型会抛出ClassCastException）;
2.如果没有被加载，则执行双亲委派逻辑进行类加载；
3.如果双亲委派逻辑未完成类加载，则调用findclass（）方法进行类加载；

所以自定义类加载器实现自身的类加载逻辑的时候，尽量不要覆盖loadclass（）方法，而是覆盖findClass()方法

虚拟机字节码执行引擎
在不同的虚拟机实现里，执行引擎在执行java代码的时候可能会有解释执行（通过解释器执行）和编译执行（通过即使编译器jit生成本地机器代码）两种选择，或者两者兼备

运行时栈帧
栈帧（stack frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，是虚拟机运行时数据区中的虚拟机栈（VM Stack）的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态链接和方法返回地址等信息。每一个方法从调用开始到执行完成的过程，都对应一个栈帧在虚拟机栈里从入栈到出栈的过程。
在代码编译的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中。
对于执行引擎老说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧（current stack frame），与这个栈帧相关联的方法称为当前方法（current method）。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作。

栈帧中的元素
1.局部变量表
局部变量表（Local Variable Table）是一组变量存储空间，用于存放方法参数和方法内部定义的局部变量。在Java程序编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需要分配的局部变量表的最大容量。

2.操作数栈
操作数栈（Operand Stack）也称为操作栈，是一个后入先出（LIFO）的栈。同局部变量一样，操作数栈的最大深度也在编译的时候写入到Code属性的max_stacks数据项中。
当一个方法刚开始执行的时候，这个方法的操作数栈是空的，在方法执行的过程中，会有各种字节码指令往操作数栈中写入和提取内容，也即出栈/入栈操作。
Java虚拟机的解释执行引擎被称为『基于栈的执行引擎』，其中锁指的『栈』就是操作数栈。

3.动态链接
每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态链接（Dynamic Linking）。
字节码中的方法调用指令以常量池中指向方法的符号引用作为参数，这些符号引用一部分会在类加载阶段或者第一次使用阶段转化为直接引用，这种称为静态解析；另外一部分将在每一次运行期间转化为直接引用，这部分称为动态链接；

4.返回地址
当一个方法开始执行的时候，只有两种方式可以退出这个方法：一个是执行引擎遇到任意一个方法返回的字节码指令，这个时候可能有返回值传递给上层的方法调用者，称为正常完成出口；一个是在方法执行过程中遇到了异常，并且这个异常没有在方法体内得到处理，称为异常完成出口，一个方法使用异常完成出口的方式退出，是不会给上层调用者产生任何返回值的。
无论采用哪种退出方式，在方法退出后，都需要返回到方法被调用的位置，程序才能继续执行，方法返回时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层方法的执行状态。
方法退出的实际过程等同于将当前栈帧出栈，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值（如果有）压入调用者栈帧的操作数栈中，调整PC计数器的只以指向方法调用指令后的一条指令。

垃圾收集（GC，Garbage Collector）:
程序计数器、java虚拟机栈、本地方法栈都是线程私有的，随着线程的创建和结束而自生自灭，栈中的栈帧随着方法的进入和退出进行入栈和出栈操作，每个栈帧应该分配多少内存都是在编译期能够确定的。因此这几个区域的内存分配和回收都具备确定性，这几个区域不用过多考虑内存回收的问题。但堆和方法区就不一样了，一个接口多个实现类需要的内存可能不一样，一个方法中多个分支需要的内存也可能不一样，只有runtime的时候才能知道会创建哪些对象，这部分内存是动态分配和回收的，GC主要关注的也是这部分内存。

GC策略：
1.引用计数法：很难解决对象之间的循环引用问题，因此主流java虚拟机没有选用该方法来管理内存；
2.可达性分析法：通过一系列称为『GC Roots』的对象作为起始点，从这些起始点开始向下搜索，搜索所有走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达），则证明这个对象是不可用的，可以给gc回收。
在Java中，可作为GC Roots的对象包括如下几种：（1）虚拟机栈（栈帧中本地变量表中的变量）中引用的对象；（2）方法区中类静态变量引用的对象；（3）方法区常量池中常量引用的对象；（4）本地方法栈中JNI（即一般我们说的native方法）引用的对象；

对于从GC Roots不可达的对象，也不是马上且一定回收，会有两次标记过程：
1.第一次标记是标记出不可达对象，即没有与GC Roots通过任何引用链相连接；
2.对第一步标记出来的对象进行一次筛选，筛选条件是此对象是否有必要执行finalize（）方法，如果对象没有覆盖finalize方法，或者finalize方法已经被调用过，则视为『没必要执行』。对于被判定有必要执行finalize的对象（覆盖了finalize方法且没有被调用过的），放入F-queue队列中，并在稍后由一个低优先级的Finalizer线程去执行它（执行是指会触发finalize方法但不会等待其结束，主要出于安全考虑，不希望因为一个对象的finalize方法执行缓慢或发生死循环导致队列对等进而导致整个内存回收系统崩溃）。在finalizer线程执行finalize方法的时候，对象可以在finalize方法中拯救自己（只要重新跟引用链上的任一个对象建立关联即可，譬如把自己通过this关键字赋值给引用链上某个对象）；
3.执行完步骤2之后，会对所有不可达对象再进行第二次标记，如果第二次标记后该对象没有被移出『即将回收』集合（即没有通过步骤2逃脱收集），那就真的会被回收；

特别注意：任何对象的finalize()方法都只会被系统自动调用一次！！！！因此，如果某个对象通过调用finalize（）方法逃脱过，那么下一次gc的时候，他的finalize()方法不会被再次执行。

方法区回收（方法区在hotspot虚拟机中被实现为永久代）
永久代的垃圾收集主要回收两部分内容：
1.废弃常量：回收废弃常量与回收Java堆中的对象是类似的，以常量池中的字面量为例，例如一个字符串”abc”已经进入了常量池中，但是当前系统没有任何一个String对象叫做”abc”，那么”abc”常量可能会被系统清理出常量池；
2.无用的类：无用的类的满足条件：（1）该类所有的实例对象已经被GC；（2）加载该类的ClassLoader已经被GC；（3）该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法；
PS：GC的时机我们是不可控的，那么同样的我们对于Class的卸载也是不可控的；

Minor GC 和 Major GC（Full GC）的区别：
1.Minor GC又叫新生代GC，指发生在新生代的垃圾收集动作，非常频繁，一般回收速度也较快；
2.Major GC（Full GC）又叫老年代GC：指发生在老年代的GC，出现Major GC的时候，经常会伴随至少一次Minor GC，Major GC的速度一般会比Minor GC慢10倍以上；

垃圾收集算法：
1.标记-清除算法：首先标记出可以回收的对象，然后统一回收所有被标记的对象。有两点不足：一个是效率问题，标记和清除两个过程的效率都不高；二个是内存碎片问题，标记清除会产生大量内存碎片，而内存碎片可能导致分配较大对象的时候，无法找到足够的连续内存而不得不触发一次垃圾收集动作；
2.标记-整理算法：过程任然与标记清除一样，但第二部不是直接对可回收对象进行清除，而是把所有存活对象都向一端移动，然后直接清理调端边界之外的内存，这样就能解决内存碎片问题；
3.复制算法：将内存分为一块较大的Eden空间和两块较小的Survior空间，每次使用Eden和其中一块Survior，回收的时候，将Eden和Survior中还存活的对象一次性复制到另外一块Survior空间上，最后清理调Eden和刚才用过的Survior空间；

当代商业虚拟机基本都采用『分代收集』的策略，即根据堆分为的新生代和老年代，根据不同年代对象的特点使用不同的垃圾收集算法，新生代主要使用复制算法（因为新生代大量对象存活期短，每次gc都会有大量对象死去，因此复制存活对象的成本低），老年代主要使用标记-清除 或 标记-整理算法。

垃圾收集器：有垃圾收集器是新生代收集器（如Serial、ParNew、Parallel Scavenge，使用复制算法），有的垃圾收集器是老生代收集器（如CMS、Serial Old、Parallel Old，使用标记清除/整理算法），有的则是新生代和老生代都可以收集的收集器（如G1）

强应用 vs 软引用 vs 弱引用
强引用：我如果不使用这个对象了，比如置为 null，交给 jvm 的 gc 来进行回收，但是否回收以及什么时候回收是不知道的；
软引用：我如果不使用这个对象了，jvm 的 gc 回收的时候，如果内存不够了，就会被回收
弱引用：jvm 的 gc 只要垃圾回收 ，就会回收掉

内存对象的分配规则：
1.对象优先在新生代的Eden区分配：大多数情况下，对象在新生代Eden区中分配，当Eden区中没有足够的空间进行分配时，将发起一次Minor GC；
2.大对象直接进入老年代：所谓的大对象是指需要大量连续内存空间的java对象，最典型的大对象就是那种很长的字符串以及数组；
3.长期存活的对象将进入老年代：虚拟机给每个对象定义了一个对象年龄计数器，如果对象在Eden出生并经过一次MinorGC后仍然存活，并且能够被Survivor容纳的话，将被移动到Survior空间中，并且对象年龄为1.后续对象在Survior区中每熬过一次Minor GC，年龄就增加1岁，当其年龄增加到一定程度（可以通过虚拟机参数设置，默认是15），将会被晋升到老年代中；

JMM（Java内存模型）
Java虚拟机规范试图定义一种Java内存模型来屏蔽各种硬件和操作系统的内存访问差异，以实现让java程序在各种平台下都能达到一致的内存访问效果

Java内存模型主要目标是定义并发下共享变量的访问规则，以保证并发过程中共享变量访问的原子性、可见性和有序性。
注意：此处的变量指对象的实例字段、静态字段和构成数组对象的元素等并发线程可共享的变量，但不包括局部变量与方法参数，因为后者是线程私有的（注意：如果局部变量或方法参数是reference类型，那么引用的对象是在堆上是共享的，但reference本身是线程私有的）

Java内存模型定义了所有的共享变量都存储在主内存（main memory），每个线程还有自己的工作内存，工作内存中保存了该线程使用到的变量的主内存副本拷贝，线程对于变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量，不同线程之间无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存完成。
勉强对应关系：主内存对应java堆中对象实例数据部分，而工作内存则对应虚拟机栈中的部分区域。

Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这3个特征来建立的：
1.原子性：由java内存模型来直接保证的原子性变量操作包括read、load、assign、use、stor和write，我们可以认为基本数据类型（包括reference类型）的读写访问是具备原子性的（long和double的原子性依赖于虚拟机实现或通过volatile来保证）。如果应用场景需要更大范围的原子性，虚拟机是通过字节码指令monitorenter和monitierexit这两个操作来实现，对应到java代码中就是synchronized关键字；
2.可见性：指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。Java内存模型是通过在变量修改后将新值同步会主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此，区别在于volatile保证了新值能够立即同步到主内存，以及每次使用前立即从主内存刷新，而普调变量则不能保证这一点；
3.有序性：如果在本线程内观察，所有的操作都是有序的，但如果在一个线程中观察另一个线程，所有操作都是无序的。Java语言提供了volatile和synchronized关键字来保证线程之间操作的有序性；

volatile作用:我们知道锁的作用包括保障原子性、可见性以及有序性。volatile常被称为“轻量级锁”，其作用与锁有类似的地方——volatile也能够保障原子性（仅保障long/double型变量访问操作的原子性）、可见性以及有序性
1.保证long/double类型在32位系统读写的原子性：对于32位系统来说，long和double型数据的读写不是原子性的（因为Long和double有64位），也即如果两个线程同时对Long/Double进行读写的话，很可能会读到错误的中间结果，从而导致错误的计算结果。但不保证对共享变量赋值操作的原子性（比如i++），因为对共享变量进行的赋值操作实际上往往是一个复合操作，volatile并不能保障这些赋值操作的原子性；——扩展理解：从Java源代码编译成字节码指令后，即使编译出来只有一条字节码指令，也并不意味着执行这条指令就是一个原子操作，因为一条字节码指令在解释执行的时候，解释器可能要运行多条指令才能实现其语义，如果是JIT编译成本地机器指令执行，一条字节码指令也可能转化成若干条本地机器码指令；
2.保证线程之间的可见性：对于同一个volatile变量，一个线程（写线程）对该变量进行更新，其他线程（读线程）随后对该变量进行读取，这些线程总是可以读取到写线程对该变量所做的更新。换而言之，写线程更新一个volatile变量，读线程随后来读取该变量，那么这些读线程能够读取到写线程对该变量所做的更新这一点是有保障的；
3.保证有序性：对于访问（读、写）同一个volatile变量的多个线程而言，一个线程（写线程）在写volatile变量前所执行的内存读、写操作在随后读取该volatile变量的其他线程（读线程）看来是有序的;
PS：Java虚拟机对可见性和有序性的保障则是通过使用内存屏障实现的。