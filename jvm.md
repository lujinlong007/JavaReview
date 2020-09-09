# jvm虚拟机：
## java内存结构
## 虚拟机内存模型
                通常可以把 JVM 内存区域分为下面几个方面，其中，有的区域是以线程为单位，而有的区域则是整个 JVM 进程唯一的。
                首先，程序计数器（PC，Program Counter Register）。在 JVM 规范中，每个线程都有它自己的程序计数器，并且任何时间一个线程都只有一个方法在执行，也就是所谓的当前方法。程序计数器会存储当前线程正在执行的 Java 方法的 JVM 指令地址；或者，如果是在执行本地方法，则是未指定值（undefined）
                第二，Java 虚拟机栈（Java Virtual Machine Stack），早期也叫 Java 栈。每个线程在创建时都会创建一个虚拟机栈，其内部保存一个个的栈帧（Stack Frame），对应着一次次的 Java 方法调用。
                前面谈程序计数器时，提到了当前方法；同理，在一个时间点，对应的只会有一个活动的栈帧，通常叫作当前帧，方法所在的类叫作当前类。如果在该方法中调用了其他方法，对应的新的栈帧会被创建出来，成为新的当前帧，一直到它返回结果或者执行结束。JVM 直接对 Java 栈的操作只有两个，就是对栈帧的压栈和出栈 栈帧中存储着局部变量表、操作数（operand）栈、动态链接、方法正常退出或者异常退出的定义等。
                第三，堆（Heap），它是 Java 内存管理的核心区域，用来放置 Java 对象实例，几乎所有创建的 Java 对象实例都是被直接分配在堆上。堆被所有的线程共享，在虚拟机启动时，我们指定的“Xmx”之类参数就是用来指定最大堆空间等指标
                堆也是垃圾收集器重点照顾的区域，所以堆内空间还会被不同的垃圾收集器进行进一步的细分，最有名的就是新生代、老年代的划分
                第四，方法区（Method Area）。这也是所有线程共享的一块内存区域，用于存储所谓的元（Meta）数据，例如类结构信息，以及对应的运行时常量池、字段、方法代码等。由于早期的 Hotspot JVM 实现，很多人习惯于将方法区称为永久代（Permanent Generation）。Oracle JDK 8 中将永久代移除，同时增加了元数据区（Metaspace
                第五，运行时常量池（Run-Time Constant Pool），这是方法区的一部分。如果仔细分析过反编译的类文件结构，你能看到版本号、字段、方法、超类、接口等各种信息，还有一项信息就是常量池。Java 的常量池可以存放各种常量信息，不管是编译期生成的各种字面量，还是需要在运行时决定的符号引用，所以它比一般语言的符号表存储的信息更加宽泛
                第六，本地方法栈（Native Method Stack）。它和 Java 虚拟机栈是非常相似的，支持对本地方法的调用，也是每个线程都会创建一个。在 Oracle Hotspot JVM 中，本地方法栈和 Java 虚拟机栈是在同一块儿区域，这完全取决于技术实现的决定，并未在规范中强制
            - 内存结构其他关注点
               直接内存（Direct Memory）区域，它就是我在专栏第 12 讲中谈到的 Direct Buffer 所直接分配的内存，也是个容易出现问题的地方。尽管，在 JVM 工程师的眼中，并不认为它是 JVM 内部内存的一部分，也并未体现 JVM 内存模型中。
               JVM 本身是个本地程序，还需要其他的内存去完成各种基本任务，比如，JIT Compiler 在运行时对热点方法进行编译，就会将编译后的方法储存在 Code Cache 里面；
               GC 等功能需要运行在本地线程之中，类似部分都需要占用内存空间。这些是实现 JVM JIT 等功能的需要，但规范中并不涉及。 

               Intern 字符串的缓存和静态变量曾经都被分配在永久代上，而永久代已经被元数据区取代。但是，Intern 字符串缓存和静态变量并不是被转移到元数据区，而是直接在堆上分配
            - Java 对象是不是都创建在堆上的呢：对象实例都是分配在堆上 
            - 什么是 OOM 问题，它可能在哪些内存区域发生      
                 OOM 如果通俗点儿说，就是 JVM 内存不够用了，javadoc 中对OutOfMemoryError的解释是，没有空闲内存，并且垃圾收集器也无法提供更多内存
                 在抛出 OutOfMemoryError 之前，通常垃圾收集器会被触发，尽其所能去清理出空间
                 我们去分配一个超大对象，类似一个超大数组超过堆的最大值，JVM 可以判断出垃圾收集并不能解决这个问题，所以直接抛出 OutOfMemoryError。
                 在java.nio.BIts.reserveMemory() 方法中，我们能清楚的看到，System.gc() 会被调用，以清理空间，这也是为什么在大量使用 NIO 的 Direct Buffer 之类时，通常建议不要加下面的参数，毕竟是个最后的尝试，有可能避免一定的内存不足问题。
                 -XX:+DisableExplicitGC
                 从我前面分析的数据区的角度，除了程序计数器，其他区域都有可能会因为可能的空间不足发生 OutOfMemoryError，简单总结如下
                     堆内存不足是最常见的 OOM 原因之一，抛出的错误信息是“java.lang.OutOfMemoryError:Java heap space”，原因可能千奇百怪，例如，可能存在内存泄漏问题；也很有可能就是堆的大小不合理，比如我们要处理比较可观的数据量，但是没有显式指定 JVM 堆大小或者指定数值偏小；或者出现 JVM 处理引用不及时，导致堆积起来，内存无法释放等。
                     而对于 Java 虚拟机栈和本地方法栈，这里要稍微复杂一点。如果我们写一段程序不断的进行递归调用，而且没有退出条件，就会导致不断地进行压栈。类似这种情况，JVM 实际会抛出 StackOverFlowError；当然，如果 JVM 试图去扩展栈空间的的时候失败，则会抛出 OutOfMemoryError。
                     对于老版本的 Oracle JDK，因为永久代的大小是有限的，并且 JVM 对永久代垃圾回收（如，常量池回收、卸载不再需要的类型）非常不积极，所以当我们不断添加新类型的时候，永久代出现 OutOfMemoryError 也非常多见，尤其是在运行时存在大量动态类型生成的场合；类似 Intern 字符串缓存占用太多空间，也会导致 OOM 问题。对应的异常信息，会标记出来和永久代相关：“java.lang.OutOfMemoryError: PermGen space
                     随着元数据区的引入，方法区内存已经不再那么窘迫，所以相应的 OOM 有所改观，出现 OOM，异常信息则变成了：“java.lang.OutOfMemoryError: Metaspace”。
                     直接内存不足，也会导致 OOM，
            - 如何监控和诊断 JVM 堆内和堆外内存使用   
                可以使用综合性的图形化工具，如 JConsole、VisualVM（注意，从 Oracle JDK 9 开始，VisualVM 已经不再包含在 JDK 安装包中）等。这些工具具体使用起来相对比较直观，直接连接到 Java 进程，然后就可以在图形化界面里掌握内存使用情况。以 JConsole 为例，其内存页面可以显示常见的堆内存和各种堆外部分使用状态。也可以使用命令行工具进行运行时查询，如 jstat 和 jmap 等工具都提供了一些选项，可以查看堆、方法区等使用数据。或者，也可以使用 jmap 等提供的命令，生成堆转储（Heap Dump）文件，然后利用 jhat 或 Eclipse MAT 等堆转储分析工具进行详细分析。如果你使用的是 Tomcat、Weblogic 等 Java EE 服务器，这些服务器同样提供了内存管理相关的功能。另外，从某种程度上来说，GC 日志等输出，同样包含着丰富的信息

                这里有一个相对特殊的部分，就是是堆外内存中的直接内存，前面的工具基本不适用，可以使用 JDK 自带的 Native Memory Tracking（NMT）特性，它会从 JVM 本地内存分配的角度进行解读
            - 常用命令：jstat、jmap、jstack等    
			- 新生代（Eden S0 S1）、老年代 、MetaSpace （比例）
               按照通常的 GC 年代方式划分，Java 堆内分为：
               1. 新生代
                 新生代是大部分对象创建和销毁的区域，在通常的 Java 应用中，绝大部分对象生命周期都是很短暂的。其内部又分为 Eden 区域，作为对象初始分配的区域；两个 Survivor，有时候也叫 from、to 区域，被用来放置从 Minor GC 中保留下来的对象
               
                 JVM 会随意选取一个 Survivor 区域作为“to”，然后会在 GC 过程中进行区域间拷贝，也就是将 Eden 中存活下来的对象和 from 区域的对象，拷贝到这个“to”区域。这种设计主要是为了防止内存的碎片化，并进一步清理无用对象

                 从内存模型而不是垃圾收集的角度，对 Eden 区域继续进行划分，Hotspot JVM 还有一个概念叫做 Thread Local Allocation Buffer（TLAB），据我所知所有 OpenJDK 衍生出来的 JVM 都提供了 TLAB 的设计。这是 JVM 为每个线程分配的一个私有缓存区域，否则，多线程同时分配内存时，为避免操作同一地址，可能需要使用加锁等机制，进而影响分配速度，你可以参考下面的示意图。从图中可以看出，TLAB 仍然在堆上，它是分配在 Eden 区域内的。其内部结构比较直观易懂，start、end 就是起始地址，top（指针）则表示已经分配到哪里了。所以我们分配新对象，JVM 就会移动 top，当 top 和 end 相遇时，即表示该缓存已满，JVM 会试图再从 Eden 里分配一块儿
               2. 老年代  
                 放置长生命周期的对象，通常都是从 Survivor 区域拷贝过来的对象。当然，也有特殊情况，我们知道普通的对象会被分配在 TLAB 上；如果对象较大，JVM 会试图直接分配在 Eden 其他位置上；如果对象太大，完全无法在新生代找到足够长的连续空闲空间，JVM 就会直接分配到老年代
               3. 永久代
               这部分就是早期 Hotspot JVM 的方法区实现方式了，储存 Java 类元数据、常量池、Intern 字符串缓存，在 JDK 8 之后就不存在永久代这块儿了
            - jvm性能调优、参数配置 
               最大堆体积 -Xmx value
               初始的最小堆体积 -Xms value
               老年代和新生代的比例 -XX:NewRatio=value 默认数值是 2，意味着老年代是新生代的 2 倍大；换句话说，新生代是堆大小的 1/3。
               也可以不用比例的方式调整新生代的大小，直接指定下面的参数，设定具体的内存大小数值
                  -XX:NewSize=value
               Eden 和 Survivor 的大小是按照比例设置的，如果 SurvivorRatio 是 8，那么 Survivor 区域就是 Eden 的 1/8 大小，也就是新生代的 1/10，因为 YoungGen=Eden + 2*Survivor，JVM 参数格式是  
                  -XX:SurvivorRatio=value
               在 JVM 内部，如果 Xms 小于 Xmx，堆的大小并不会直接扩展到其上限，也就是说保留的空间（reserved）大于实际能够使用的空间（committed）。当内存需求不断增长的时候，JVM 会逐渐扩展新生代等区域的大小，所以 Virtual 区域代表的就是暂时不可用（uncommitted）的空间
            - 内存溢出分析：堆内、堆外 （含义、如何设置）
                 在 JMC 或 JConsole 的内存管理界面，会统计部分非堆内存，但提供的信息相对有限
                 依赖 NMT 特性对 JVM 进行分析，它所提供的详细分类信息，非常有助于理解 JVM 内部实现
                 开启 NMT 并选择 summary 模式 -XX:NativeMemoryTracking=summary
                 为了方便获取和对比 NMT 输出，选择在应用退出时打印 NMT 统计信息
                 -XX:+UnlockDiagnosticVMOptions -XX:+PrintNMTStatistics

                 第一部分非常明显是 Java 堆，我已经分析过使用什么参数调整，不再赘述。
                 第二部分是 Class 内存占用，它所统计的就是 Java 类元数据所占用的空间，JVM 可以通过类似下面的参数调整其大小
                 -XX:MaxMetaspaceSize=value
                 你可以使用下面的小技巧，调整启动类加载器元数据区，这主要是为了对比以加深理解 
                 -XX:InitialBootClassLoaderMetaspaceSize=30720
                 下面是 Thread，这里既包括 Java 线程，如程序主线程、Cleaner 线程等，也包括 GC 等本地线程。你有没有注意到，即使是一个 HelloWorld 程序，这个线程数量竟然还有 25。似乎有很多浪费，设想我们要用 Java 作为 Serverless 运行时，每个 function 是非常短暂的，如何降低线程数量呢？如果你充分理解了专栏讲解的内容，对 JVM 内部有了充分理解，思路就很清晰了：JDK 9 的默认 GC 是 G1，虽然它在较大堆场景表现良好，但本身就会比传统的 Parallel GC 或者 Serial GC 之类复杂太多，所以要么降低其并行线程数目，要么直接切换 GC 类型；JIT 编译默认是开启了 TieredCompilation 的，将其关闭，那么 JIT 也会变得简单，相应本地线程也会减少。
                 下面是替换了默认 GC，并关闭 TieredCompilation 的命令行
                 得到的统计信息如下，线程数目从 25 降到了 17，消耗的内存也下降了大概 1/3
                 接下来是 Code 统计信息，显然这是 CodeCache 相关内存，也就是 JIT compiler 存储编译热点方法等信息的地方，JVM 提供了一系列参数可以限制其初始值和最大值等，例如：-XX:InitialCodeCacheSize=value -XX:ReservedCodeCacheSize=value
                 很明显，CodeCache 空间下降非常大，这是因为我们关闭了复杂的 TieredCompilation，而且还限制了其初始大小

                 面就是 GC 部分了，就像我前面介绍的，G1 等垃圾收集器其本身的设施和数据结构就非常复杂和庞大，例如 Remembered Set 通常都会占用 20%~30% 的堆空间。如果我把 GC 明确修改为相对简单的 Serial GC，会有什么效果呢
                 使用命令 -XX:+UseSerialG
                 可见，不仅总线程数大大降低（25 → 13），而且 GC 设施本身的内存开销就少了非常多。据我所知，AWS Lambda 中 Java 运行时就是使用的 Serial GC，可以大大降低单个 function 的启动和运行开销
                 Compiler 部分，就是 JIT 的开销，显然关闭 TieredCompilation 会降低内存使用。其他一些部分占比都非常低，通常也不会出现内存使用问题，请参考官方文档。唯一的例外就是 Internal（JDK 11 以后在 Other 部分）部分，其统计信息包含着 Direct Buffer 的直接内存，这其实是堆外内存中比较敏感的部分，很多堆外内存 OOM 就发生在这里，。原则上 Direct Buffer 是不推荐频繁创建或销毁的，如果你怀疑直接内存区域有问题，通常可以通过类似 instrument 构造函数等手段，排查可能的问题
			- 垃圾回收算法（引用计数、标记压缩、清除、复制算法、分区）、垃圾收集器
                垃圾收集器
                 GC，Garbage Collector）是和具体 JVM 实现紧密相关的，不同厂商（IBM、Oracle），不同版本的 JVM，提供的选择也不同。接下来，我来谈谈最主流的 Oracle JDK

                 Serial GC，它是最古老的垃圾收集器，“Serial”体现在其收集工作是单线程的，并且在进行垃圾收集过程中，会进入臭名昭著的“Stop-The-World”状态。当然，其单线程设计也意味着精简的 GC 实现，无需维护复杂的数据结构，初始化也简单，所以一直是 Client 模式下 JVM 的默认选项
                 从年代的角度，通常将其老年代实现单独称作 Serial Old，它采用了标记 - 整理（Mark-Compact）算法，区别于新生代的复制算法
                 Serial GC 的对应 JVM 参数是 -XX:+UseSerialGC
                 
                 ParNew GC，很明显是个新生代 GC 实现，它实际是 Serial GC 的多线程版本，最常见的应用场景是配合老年代的 CMS GC 工作，下面是对应参数-XX:+UseConcMarkSweepGC -XX:+UseParNewGC

                 CMS（Concurrent Mark Sweep） GC，基于标记 - 清除（Mark-Sweep）算法，设计目标是尽量减少停顿时间，这一点对于 Web 等反应时间敏感的应用非常重要，一直到今天，仍然有很多系统使用 CMS GC。但是，CMS 采用的标记 - 清除算法，存在着内存碎片化问题，所以难以避免在长时间运行等情况下发生 full GC，导致恶劣的停顿。另外，既然强调了并发（Concurrent），CMS 会占用更多 CPU 资源，并和用户线程争抢
                 
                 Parallel GC，在早期 JDK 8 等版本中，它是 server 模式 JVM 的默认 GC 选择，也被称作是吞吐量优先的 GC。它的算法和 Serial GC 比较相似，尽管实现要复杂的多，其特点是新生代和老年代 GC 都是并行进行的，在常见的服务器环境中更加高效
                 -XX:+UseParallelGC

                 另外，Parallel GC 引入了开发者友好的配置项，我们可以直接设置暂停时间或吞吐量等目标，JVM 会自动进行适应性调整，例如下面参数
                 -XX:MaxGCPauseMillis=value-XX:GCTimeRatio=N

                 G1 GC 这是一种兼顾吞吐量和停顿时间的 GC 实现，是 Oracle JDK 9 以后的默认 GC 选项。G1 可以直观的设定停顿时间的目标，相比于 CMS GC，G1 未必能做到 CMS 在最好情况下的延时停顿，但是最差情况要好很多。G1 GC 仍然存在着年代的概念，但是其内存结构并不是简单的条带式划分，而是类似棋盘的一个个 region。Region 之间是复制算法，但整体上实际可看作是标记 - 整理（Mark-Compact）算法，可以有效地避免内存碎片，尤其是当 Java 堆非常大的时候，G1 的优势更加明显。G1 吞吐量和停顿表现都非常不错，并且仍然在不断地完善，与此同时 CMS 已经在 JDK 9 中被标记为废弃（deprecated），所以 G1 GC 值得你深入掌握 
                垃圾收集的原理和基础概念
                   自动垃圾收集的前提是清楚哪些内存可以被释放。这一点可以结合我前面对 Java 类加载和内存结构的分析，来思考一下。主要就是两个方面，最主要部分就是对象实例，都是存储在堆上的；还有就是方法区中的元数据等信息，例如类型不再使用，卸载该 Java 类似乎是很合理的 

                   对于对象实例收集，主要是两种基本算法，引用计数和可达性分析
                   引用计数算法，顾名思义，就是为对象添加一个引用计数，用于记录对象被引用的情况，如果计数为 0，即表示对象可回收。这是很多语言的资源回收选择，例如因人工智能而更加火热的 Python，它更是同时支持引用计数和垃圾收集机制。具体哪种最优是要看场景的，业界有大规模实践中仅保留引用计数机制，以提高吞吐量的尝试。Java 并没有选择引用计数，是因为其存在一个基本的难题，也就是很难处理循环引用关系

                   另外就是 Java 选择的可达性分析，Java 的各种引用关系，在某种程度上，将可达性问题还进一步复杂化，具体请参考专栏第 4 讲，这种类型的垃圾收集通常叫作追踪性垃圾收集（Tracing Garbage Collection）。其原理简单来说，就是将对象及其引用关系看作一个图，选定活动的对象作为 GC Roots，然后跟踪引用链条，如果一个对象和 GC Roots 之间不可达，也就是不存在引用链条，那么即可认为是可回收对象。JVM 会把虚拟机栈和本地方法栈中正在引用的对象、静态属性引用的对象和常量，作为 GC Roots

                   方法区无用元数据的回收比较复杂，我简单梳理一下。还记得我对类加载器的分类吧，一般来说初始化类加载器加载的类型是不会进行类卸载（unload）的；而普通的类型的卸载，往往是要求相应自定义类加载器本身被回收，所以大量使用动态类型的场合，需要防止元数据区（或者早期的永久代）不会 OOM。在 8u40 以后的 JDK 中，下面参数已经是默认的：-XX:+ClassUnloadingWithConcurrentMark  
                垃圾回收算法
                   复制（Copying）算法，我前面讲到的新生代 GC，基本都是基于复制算法，过程就如专栏上一讲所介绍的，将活着的对象复制到 to 区域，拷贝过程中将对象顺序放置，就可以避免内存碎片化。这么做的代价是，既然要进行复制，既要提前预留内存空间，有一定的浪费；另外，对于 G1 这种分拆成为大量 region 的 GC，复制而不是移动，意味着 GC 需要维护 region 之间对象引用关系，这个开销也不小，不管是内存占用或者时间开销

                   标记 - 清除（Mark-Sweep）算法，首先进行标记工作，标识出所有要回收的对象，然后进行清除。这么做除了标记、清除过程效率有限，另外就是不可避免的出现碎片化问题，这就导致其不适合特别大的堆；否则，一旦出现 Full GC，暂停时间可能根本无法接受

                   标记 - 整理（Mark-Compact），类似于标记 - 清除，但为避免内存碎片化，它会在清理过程中将对象移动，以确保移动后的对象占用连续的内存空间 
                垃圾收集过程的理解
                    第一，Java 应用不断创建对象，通常都是分配在 Eden 区域，当其空间占用达到一定阈值时，触发 minor GC。仍然被引用的对象（绿色方块）存活下来，被复制到 JVM 选择的 Survivor 区域，而没有被引用的对象（黄色方块）则被回收。注意，我给存活对象标记了“数字 1”，这是为了表明对象的存活时间

                    经过一次 Minor GC，Eden 就会空闲下来，直到再次达到 Minor GC 触发条件，这时候，另外一个 Survivor 区域则会成为 to 区域，Eden 区域的存活对象和 From 区域对象，都会被复制到 to 区域，并且存活的年龄计数会被加 1

                    类似第二步的过程会发生很多次，直到有对象年龄计数达到阈值，这时候就会发生所谓的晋升（Promotion）过程，如下图所示，超过阈值的对象会被晋升到老年代。这个阈值是可以通过参数指定：-XX:MaxTenuringThreshold=n

                    后面就是老年代 GC，具体取决于选择的 GC 选项，对应不同的算法。下面是一个简单标记 - 整理算法过程示意图，老年代中的无用对象被清除后， GC 会将对象进行整理，以防止内存碎片化。


                    通常我们把老年代 GC 叫作 Major GC，将对整个堆进行的清理叫作 Full GC，但是这个也没有那么绝对，因为不同的老年代 GC 算法其实表现差异很大，例如 CMS，“concurrent”就体现在清理工作是与工作线程一起并发运行的 
                GC的新发展
                   GC 仍然处于飞速发展之中，目前的默认选项 G1 GC 在不断的进行改进，很多我们原来认为的缺点，例如串行的 Full GC、Card Table 扫描的低效等，都已经被大幅改进，例如， JDK 10 以后，Full GC 已经是并行运行，在很多场景下，其表现还略优于 Parallel GC 的并行 Full GC 实现。即使是 Serial GC，虽然比较古老，但是简单的设计和实现未必就是过时的，它本身的开销，不管是 GC 相关数据结构的开销，还是线程的开销，都是非常小的，所以随着云计算的兴起，在 Serverless 等新的应用场景下，Serial GC 找到了新的舞台。
                   jdk11 中新的gc
                   Epsilon GC，简单说就是个不做垃圾收集的 GC，似乎有点奇怪，有的情况下，例如在进行性能测试的时候，可能需要明确判断 GC 本身产生了多大的开销，这就是其典型应用场景
                   ZGC，这是 Oracle 开源出来的一个超级 GC 实现，具备令人惊讶的扩展能力，比如支持 T bytes 级别的堆大小，并且保证绝大部分情况下，延迟都不会超过 10 ms。虽然目前还处于实验阶段，仅支持 Linux 64 位的平台，但其已经表现出的能力和潜力都非常令人期待
            - gc调优思路
                对于 GC 调优来说，首先就需要清楚调优的目标是什么？从性能的角度看，通常关注三个方面，内存占用（footprint）、延时（latency）和吞吐量（throughput），大多数情况下调优会侧重于其中一个或者两个方面的目标，很少有情况可以兼顾三个不同的角度。当然，除了上面通常的三个方面，也可能需要考虑其他 GC 相关的场景，例如，OOM 也可能与不合理的 GC 相关参数有关；或者，应用启动速度方面的需求，GC 也会是个考虑的方面。
                简要地介绍其上下文，然后将诊断思路和调优实践过程表述出来

                基本的调优思路可以总结为
                   理解应用需求和问题，确定调优目标。假设，我们开发了一个应用服务，但发现偶尔会出现性能抖动，出现较长的服务停顿。评估用户可接受的响应时间和业务量，将目标简化为，希望 GC 暂停尽量控制在 200ms 以内，并且保证一定标准的吞吐量

                   掌握 JVM 和 GC 的状态，定位具体的问题，确定真的有 GC 调优的必要。具体有很多方法，比如，通过 jstat 等工具查看 GC 等相关状态，可以开启 GC 日志，或者是利用操作系统提供的诊断工具等。例如，通过追踪 GC 日志，就可以查找是不是 GC 在特定时间发生了长时间的暂停，进而导致了应用响应不及时
                   
                   这里需要思考，选择的 GC 类型是否符合我们的应用特征，如果是，具体问题表现在哪里，是 Minor GC 过长，还是 Mixed GC 等出现异常停顿情况；如果不是，考虑切换到什么类型，如 CMS 和 G1 都是更侧重于低延迟的 GC 选项

                   通过分析确定具体调整的参数或者软硬件配置

                   验证是否达到调优目标，如果达到目标，即可以考虑结束调优；否则，重复完成分析、调整、验证这个过程
            - G1调优
                G1 GC 的内部结构和主要机制。
                  从内存区域的角度，G1 同样存在着年代的概念，但是与我前面介绍的内存结构很不一样，其内部是类似棋盘状的一个个 region 组成 
                  region 的大小是一致的，数值是在 1M 到 32M 字节之间的一个 2 的幂值数，JVM 会尽量划分 2048 个左右、同等大小的 region，这点可以从源码heapRegionBounds.hpp中看到。当然这个数字既可以手动调整，G1 也会根据堆大小自动进行调整

                  在 G1 实现中，年代是个逻辑概念，具体体现在，一部分 region 是作为 Eden，一部分作为 Survivor，除了意料之中的 Old region，G1 会将超过 region 50% 大小的对象（在应用中，通常是 byte 或 char 数组）归类为 Humongous 对象，并放置在相应的 region 中。逻辑上，Humongous region 算是老年代的一部分，因为复制这样的大对象是很昂贵的操作，并不适合新生代 GC 的复制算法
                
                  region 设计有什么副作用
                  region 大小和大对象很难保证一致，这会导致空间的浪费。不知道你有没有注意到，我的示意图中有的区域是 Humongous 颜色，但没有用名称标记，这是为了表示，特别大的对象是可能占用超过一个 region 的。并且，region 太小不合适，会令你在分配大对象时更难找到连续空间，这是一个长久存在的情况，请参考OpenJDK 社区的讨论。这本质也可以看作是 JVM 的 bug，尽管解决办法也非常简单，直接设置较大的 region 大小，参数如下：-XX:G1HeapRegionSize=<N, 例如16>M
                回收算法 
                  在新生代，G1 采用的仍然是并行的复制算法，所以同样会发生 Stop-The-World 的暂停。
                  在老年代，大部分情况下都是并发标记，而整理（Compact）则是和新生代 GC 时捎带进行，并且不是整体性的整理，而是增量进行的  

                  习惯上人们喜欢把新生代 GC（Young GC）叫作 Minor GC，老年代 GC 叫作 Major GC，区别于整体性的 Full GC。但是现代 GC 中，这种概念已经不再准确，对于 G1 来说：
                  Minor GC 仍然存在，虽然具体过程会有区别，会涉及 Remembered Set 等相关处理。老年代回收，则是依靠 Mixed GC。并发标记结束后，JVM 就有足够的信息进行垃圾收集，Mixed GC 不仅同时会清理 Eden、Survivor 区域，而且还会清理部分 Old 区域。可以通过设置下面的参数，指定触发阈值，并且设定最多被包含在一次 Mixed GC 中的 region 比例。
                  –XX:G1MixedGCLiveThresholdPercent
                  –XX:G1OldCSetRegionThresholdPercent

                  从 G1 内部运行的角度，下面的示意图描述了 G1 正常运行时的状态流转变化，当然，在发生逃逸失败等情况下，就会触发 Full GC。

                  G1 相关概念非常多，有一个重点就是 Remembered Set，用于记录和维护 region 之间对象的引用关系。为什么需要这么做呢？试想，新生代 GC 是复制算法，也就是说，类似对象从 Eden 或者 Survivor 到 to 区域的“移动”，其实是“复制”，本质上是一个新的对象。在这个过程中，需要必须保证老年代到新生代的跨区引用仍然有效

                  G1 的很多开销都是源自 Remembered Set，例如，它通常约占用 Heap 大小的 20% 或更高，这可是非常可观的比例。并且，我们进行对象复制的时候，因为需要扫描和更改 Card Table 的信息，这个速度影响了复制的速度，进而影响暂停时间 
                G1 其他
                   上面提到了 Humongous 对象的分配和回收，这是很多内存问题的来源，Humongous region 作为老年代的一部分，通常认为它会在并发标记结束后才进行回收，但是在新版 G1 中，Humongous 对象回收采取了更加激进的策略。我们知道 G1 记录了老年代 region 间对象引用，Humongous 对象数量有限，所以能够快速的知道是否有老年代对象引用它。如果没有，能够阻止它被回收的唯一可能，就是新生代是否有对象引用了它，但这个信息是可以在 Young GC 时就知道的，所以完全可以在 Young GC 中就进行 Humongous 对象的回收，不用像其他老年代对象那样，等待并发标记结束。
                   
                   在 8u20 以后字符串排重的特性，在垃圾收集过程中，G1 会把新创建的字符串对象放入队列中，然后在 Young GC 之后，并发地（不会 STW）将内部数据（char 数组，JDK 9 以后是 byte 数组）一致的字符串进行排重，也就是将其引用同一个数组。你可以使用下面参数激活： 
                   -XX:+UseStringDeduplication
                   ，这种排重虽然可以节省不少内存空间，但这种并发操作会占用一些 CPU 资源，也会导致 Young GC 稍微变慢

                   类型卸载是个长期困扰一些 Java 应用的问题，在专栏第 25 讲中，我介绍了一个类只有当加载它的自定义类加载器被回收后，才能被卸载。元数据区替换了永久代之后有所改善，但还是可能出现问题。


                   G1 的类型卸载有什么改进吗？很多资料中都谈到，G1 只有在发生 Full GC 时才进行类型卸载，但这显然不是我们想要的。你可以加上下面的参数查看类型卸载：
                   -XX:+TraceClassUnloading幸好现代的 G1 已经不是如此了，8u40 以后，G1 增加并默认开启下面的选项：
                   -XX:+ClassUnloadingWithConcurrentMark


                   我们知道老年代对象回收，基本要等待并发标记结束。这意味着，如果并发标记结束不及时，导致堆已满，但老年代空间还没完成回收，就会触发 Full GC，所以触发并发标记的时机很重要。早期的 G1 调优中，通常会设置下面参数，但是很难给出一个普适的数值，往往要根据实际运行结果调整-XX:InitiatingHeapOccupancyPercent在 JDK 9 之后的 G1 实现中，这种调整需求会少很多，因为 JVM 只会将该参数作为初始值，会在运行时进行采样，获取统计数据，然后据此动态调整并发标记启动时机。对应的 JVM 参数如下，默认已经开启：-XX:+G1UseAdaptiveIHOP


                   在现有的资料中，大多指出 G1 的 Full GC 是最差劲的单线程串行 GC。其实，如果采用的是最新的 JDK，你会发现 Full GC 也是并行进行的了，在通用场景中的表现还优于 Parallel GC 的 Full GC 实现。

                   打印垃圾收集日志
                     掌握 GC 调优信息收集途径。掌握尽量全面、详细、准确的信息，是各种调优的基础，不仅仅是 GC 调优。我们来看看打开 GC 日志，这似乎是很简单的事情，可是你确定真的掌握了吗？除了常用的两个选项，-XX:+PrintGCDetails-XX:+PrintGCDateStamps还有一些非常有用的日志选项，很多特定问题的诊断都是要依赖这些选项：-XX:+PrintAdaptiveSizePolicy // 打印G1 Ergonomics相关信息
  
                     我们知道 GC 内部一些行为是适应性的触发的，利用 PrintAdaptiveSizePolicy，我们就可以知道为什么 JVM 做出了一些可能我们不希望发生的动作。例如，G1 调优的一个基本建议就是避免进行大量的 Humongous 对象分配，如果 Ergonomics 信息说明发生了这一点，那么就可以考虑要么增大堆的大小，要么直接将 region 大小提高

                     如果是怀疑出现引用清理不及时的情况，则可以打开下面选项，掌握到底是哪里出现了堆积。-XX:+PrintReferenceGC 

                     另外，建议开启选项下面的选项进行并行引用处理。-XX:+ParallelRefProcEnabled

                     JDK 9 中 JVM 和 GC 日志机构进行了重构，其实我前面提到的 PrintGCDetails 已经被标记为废弃，而 PrintGCDateStamps 已经被移除，指定它会导致 JVM 无法启动。可以使用下面的命令查询新的配置参数。java -Xlog:help
                   
                   通用实践
                     如果发现 Young GC 非常耗时，这很可能就是因为新生代太大了，我们可以考虑减小新生代的最小比例。-XX:G1NewSizePercent
                     降低其最大值同样对降低 Young GC 延迟有帮助。-XX:G1MaxNewSizePercent

                     如果我们直接为 G1 设置较小的延迟目标值，也会起到减小新生代的效果，虽然会影响吞吐量。如果是 Mixed GC 延迟较长，我们应该怎么做呢？还记得前面说的，部分 Old region 会被包含进 Mixed GC，减少一次处理的 region 个数，就是个直接的选择之一。

                     在上面已经介绍了 G1OldCSetRegionThresholdPercent 控制其最大值，还可以利用下面参数提高 Mixed GC 的个数，当前默认值是 8，Mixed GC 数量增多，意味着每次被包含的 region 减少。-XX:G1MixedGCCountTarget
            - GC停顿、吞吐量，进入老年代阈值、大对象回收问题等 
			- CPU飙升：死锁、线程阻塞
			- 关于GC: minor major full
 			- stw，安全点等
 			-- jvm的逃逸分析  & Tlab & 消除伪共享 & UNsafe & 
-性能调优
    性能调优四班斧
     1借助监控预防问题 发现问题
     2借助工具定位问题
     3定期复盘问题，防止同类问题再现
     4定好规范，一定程度上规避问题
     --应用性能调优
     工具
     --Skywalking
         是什么 适用于分布式系统的性能监控工具
         作用：
         Skywaking架构刨析
           webappui--Http GraphQl格式        
           Agent/探针-gRpc或者HTTP json格式   
           Backend(OAP) OAP config
           H2 后端存储
         核心概念
            服务
            服务实例
            端点
            追踪  
         java agent 配置方式
           系统属性 -Dskywalking
           代理选项 option=value
           系统环境变量 ${key:option}
           优先级 代理选项》系统属性》系统环境变量》配置文件
           不变化直接配置文件  变化的使用以上三种方式
         配置实战 
           设置采样
           并打印sql详情  
           使用-d的形式配置 
           -Dskywalking.agent.sample_n_per_3_secs=1
           -Dskywalking.plugin.mysql.trace_sql_parameters=true
           -Dskywalking.plugin.mysql.sql_parameters_max_length=200
           -Dskywalking.agent.service_name=foodie-dev  
         java agent插件
            内置插件 plugins 启动时只启动他
            引导插件 bootstrap  thread/http 
            可选插件 optional  
            启用和禁用  放置plugins目录中默认启动 移除plugins目录    
          监控spring bean 

          拷贝 apm-spring-annotation-plugin-6.6.0.jar 到plugins目录下
         监控任意代码
           移动java包apm-customize-enhance-plugin-6.6.0.jar 到plugins   
           编写增强规则
            配置文件路径  
            plugin.customize.enhance_file=C:/Users/lujinlong/Desktop/agent/customize_enhance.xml
          配置不成功 原因分析
           可能存在问题的地方：
           skywalking.yaml里面没有指定enhance配置文件；另外对于windows，路径得写成例如C:/xxx 或者C:\\xxx。常识性的问题不要犯
           检查enhance配置文件是否有问题，例如static是否为true、方法所属的类是不是正确、方法的签名是否正确、类名全路径是否正确等等。
           此外，可以分析下java agent的日志。搜索enhance，里面有增强过程的。  
          告警 动态配置
          linux
             1、lsof -i:端口号
             2、netstat -tunlp|grep 端口号  
          windows 
            查看端口是否被占用 netstat -ano|findstr 8080
            关闭某个端口 taskkill /PID 22876 /F
     --Springboot actuator  
        操作步骤
         1pom中添加依赖 
          <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
         2yml配置文件中打开端口
          management:
           endpoint:
             health:
               show-details: always #暴漏默认端点 info health  
          3全部暴漏端口
            endpoints:
              web:
                 exposure:
                     include: '*' #暴漏所有端点   
          4设置info端点的值（告警webhook定时任务中读取相关人员email 发送邮件）
             info:
               project-name: foodie-dev
                author: fuck
                  owner-emial: xxx  #配置作者信息                
        监控数据可视化
           spring boot admin
              集成Client模式
                 admin client  注册到Admin Server
                 admin server 定时请求 /actuator
                 内存记录 server_name IP port
                开发admin代码步骤
                服务端步骤
                 1添加依赖 spring-boot-admin-server
                 2添加depmedencyManagement  版本管理
                   <!--整合spring boot admin
		            添加spring boot admin依赖的时候不用自己添加版本
		           -->
                 3 启动类添加 @EnableAdminServer
                 4 添加配置文件 配置端口 8988
                客户端步骤
                 1.添加依赖spring-boot-admin-starter-client
                 2.添加epmedencyManagement  版本管理
                   <!--整合spring boot admin
		            添加spring boot admin依赖的时候不用自己添加版本
		           -->
		         3.添加配置文件 节点  boot:
                                       admin:
                                          client:
                                      url: http://localhost:8988  
              服务发现模式
                  微服务注册  服务发现组件Eureka
                  admin server 注册 服务发现组件eureka  通过服务发现组件找到微服务 actuator节点
                开发步骤
                开发eureka server
                  
                   1.添加依赖spring-cloud-starter-netflix-eureka-server
                   2.启动类添加EnableEurekaServer
                   3配置文件添加  
                     eureka:
                      client:
                         service-url:
                             defaultZone: http://localhost:8761/eureka
                        #是否要从其他server中获取数据
                       fetch-registry: false
                       #是否要注册到其他server上
                      register-with-eureka: false  
                改造foodie-dev 
                  1 添加eureka-client 依赖 以及cloud 版本管理器
                  2 添加配置文件节点  application:
                          name: foodie-dev   
                      eureka:
                         client:
                             service-url:
                            defaultZone: http://localhost:8761/eureka       
                改造 adminserver
                  1 添加eureka-client 依赖及 cloud版本管理器 
                  2 添加配置文件节点
                     eureka:
                       client:
                         service-url:
                          defaultZone: http://localhost:8761/eureka
                      spring:
                         application:
                         name: admin-server

           Promethus+grafana
     --javaMelody
        1. 添加依赖
           <dependency>
             <groupId>net.bull.javamelody</groupId>
             <artifactId>javamelody-spring-boot-starter</artifactId>
            <version>1.85.0</version>
          </dependency>  
        2.添加配置 
          javamelody:
            # 是否启用 默认true Enable JavaMelody auto-configuration (optional, default: true)
            enabled: true
           # 排除不想监控的数据源 Data source names to exclude from monitoring (optional, comma-separated)
            excluded-datasources: secretSource,topSecretSource
            #是否启用 spring service controller监控  Enable monitoring of Spring services and controllers (optional, default: true)
            spring-monitoring-enabled: true
   
            # 是否记录http请求 log http requests:
              log: true
            #n哪些路径不需要监控
             # to exclude images, css, fonts and js urls from the monitoring:
              #url-exclude-pattern: (/webjars/.*|/css/.*|/images/.*|/fonts/.*|/js/.*)
   
            #用来转换 http请求 将其中动态的部分删除
             #http-transform-pattern: \d+
    
            authorized-users: admin:pwd
             # 指定存储数据的路径to change the default storage directory:
              storage-directory: /tmp/javamelody-${server.port}}

               # javamedelody的访问路径 to change the default "/monitoring" path:
               #monitoring-path: /admin/performance
            management-endpoint-monitoring-enabled: true
              #和actuator配合使用
            #1 路径挂到actuator下面 actuator/monitoring
             #2 设置的账号密码失效     
     --tomcatManager
         1.在conf目录下 tomcat-user.xml里添加
          <role rolename="manager-gui"/>
          <role rolename="manager-script"/>
          <role rolename="manager-jmx"/>
          <role rolename="manager-status"/>
           <user username="admin" password="123456" roles="manager-gui,manager-script,manager-jmx,manager-status"/> 
         2图形界面管理     
         3命令管理界面
         4只支持传统的部署方式 不支持springboot 部署方式
         5ip地址访问manager 需要修改 mete-info 下 context 配置文件 
         查看内存泄漏 线程dump
     --PSI Probe
         增强版的tomcat管理监控工具 

         发布 psi-probe-web-3.4.2.war

         除了 tomcat  jboss也支持  项目发布难  
     --ELK  / Sleuth +Zipkin    Hystrix+Dashboard    Sentinel+DashBoard
     技术
     --对象池
         场景 维护一些很大 创建很慢的对象 提高性能
         对象池框架  Commons-pools2
         (抽象类)objectPool 
           -->GenericObjectpool （实现类）GenericKeyedObjectPool
          方法 borrowObject 借出对象
          returnObject  将对象池归还
           invalidateObject 失效一个对象
          addObject  增加一个空闲对象
          clear  清空空闲的所有对象 并释放资源
          close  关闭对象池 释放相关资源
          getNumidle 获得空闲的对象数量
          getNumActive  获得被借出对象数量
         Factory 创建和管理poolObject  一般自己扩展
         poolObject  包装原有的对象 从而让对象池管理  一般用DefaultpooledObject
       
         --开发一个带监控的数据库连接池  替代业界连接池
          巩固对象池用法
          收获若干Spring bOOT开发小技巧
          如何扩展Actuator
          开发步骤 
           
           1.创建ConnnectionPooledObjectFactory implements PooledObjectFactory<Connection>
           2 覆写 makeObject方法
               用DefaultPooledObject包装connection 返回	
           3  创建自定义datasource  DMDataSource implements DataSource
                 构造器中初始化
                 this.pool = new GenericObjectPool<>(new ConnnectionPooledObjectFactory());
                 覆写getconnection  
                   使用pool.borrowObject
           4添加带监控的端点DataSourceEndpoint
                类上加注解@@Endpoint(id = "datasource") 表示可以用actuator/datasource 访问 
                @ReadOperation 访问时访问这个方法  添加对应属性
                 GenericObjectPool<Connection> pool = dataSource.getPool();
                 HashMap<String, Object> map = Maps.newHashMap();
                 map.put("numActive",pool.getNumActive());
                 map.put("numIdle",pool.getNumIdle());
                 map.put("createdCount",pool.getCreatedCount());
           5启动类添加bean 
                @Bean
               @Primary
                public DataSource dataSource(){
                     return  new DMDataSource();
                }

               @Bean
               public DataSourceEndpoint dataSourceEndpoint(){
                 DataSource dataSource = this.dataSource();
                  return new DataSourceEndpoint((DMDataSource)dataSource);
                }
           6由于和proxy类冲突 配置文件中 排除   
               excluded-datasources: dataSource  

            以上代码并未实现回收资源statement resultset 以及复用Connec'tion     

            实现myConnection
             覆写所有方法
              // 如果底层的Connection已经关闭
            if (this.isClosed()) {
               try {
                objectPool.invalidateObject(this);
               } catch (Exception e) {
                throw new SQLException(e);
              }
             }
             // 底层Connection没有关闭，可以继续复用
             else {
               try {
                 objectPool.returnObject(this);//会调用factory passivateObject 
               } catch (Exception e) {
                this.connection.close();
                throw new SQLException(e);
               }
              }
              passivateObject  关闭statement
               MyConnection myConnection = p.getObject();
                Statement statement = myConnection.getStatement();
                if (statement != null) {
                statement.close();
               }  
          abandon 于 evict区别 
          abandon 时对象池对象的一个状态 对象还在对象池里
          evict 清理对象的而过程  清理的不一定时abandon对象
          配置和注意点
     --线程池  
         （特殊的对象池）
          重用已存在的线程
          控制并发
          功能强大
               线程池参数
                 ThreadPoolExecutor(10,
                  10,
                  10l,
                  TimeUnit.SECONDS,
                  new LinkedBlockingDeque<>(),
                  Executors.defaultThreadFactory(),
                  new ThreadPoolExecutor.AbortPolicy());

                  corePoolSize 核心线程
                  MaximumPoolSize 最大线程数（核心线程+非核心线程）
                  KeepAliveTime 允许空闲时间 默认情况下指非核心线程的空闲时间，如果设置allowCoreThreadTimeOut=true
                 表实核心线程 空闲多久也会被回收
                  Timeunit 等待时间单位
                  woekQueue 存储等待的任务 传入BlockingQueue
                  threadFactory  线程工厂 default创建的线程拥有相同的优先级，非守护线程 有线程名称
                   privilegedThreadFactory 继承上面的factory  运行在这个线程中具有相同的访问控制和classloader

                 rejectHnadler  拒绝任务的策略 
                  abortPolicy 抛出异常
                  callerRusPolicy 用调用者所在的线程执行任务
                  DiscardOldestPolicy 丢弃队列中最靠前的任务
                  DisCardPolicy 丢弃任务


               
                  核心线程--》正式工  core
                  非核心线程--》临时工   
                  正式员工+临时员工  maxsize
                  keepalive 

                 线程池核心api -执行类
                  execute 提交任务 交给线程池执行
                  submit 提交任务 能够返回执行结果
                  shutdown  关闭线程池 等待任务都执行完毕
                  shutdownNow 关闭线程池 不等任务执行完
                  监控类api
                   gettaskCount返回线程池已经执行和未执行的线程数量
                   getCompletedTaskCount 已完成的任务数
                   getPooolSize 线程池当前的线程数量
                   getAciveCount 线程池中正在执行的任务数
                 线程池状态 
                  running 
                    ->(shutdown) shutdown  
                    ->(shutdownnow) stop 
                                       队列为空&线程数量为0  
                                       tidying 
                                             terminated   
               blockingqueue详解 选择策略
                    是什么
                     阻塞队列 
                       当队列为空 获取对象阻塞 当队列满，放入队列阻塞
                       多线程环境下 自动管理线程的等待与唤醒
                       api  插入add odder put 删除 remove poll take poll 检查 element peek
                     有界 arrayBlock  LinkedBlock(有参数)  Synchronous

                     无界 LinkedTransfer Delay  Priority LinkedBlock(无参数)
                    调优技巧
                      core  max  workQueue的容量
                       系统资源消耗  上下文切换
                          线程小  队列大 多余任务排队
                       任务常阻塞  
                           增大max
                       队列小  线程应该变大  cpu利用率高 线程切换开销变大
               ScheduledThreadPoolExecutor
                     扩展ThreadPoolExecutor
                     提供延时执行 周期性执行 

                     schedule（runable/callable）
                     executor.scheduleAtFixedRate(new Runnable() {
                      @Override
                     public void run() {
                         System.out.println("scheduleAtFixedRate"+new Date());
                     }
                       },
                      //第一次执行任务时 延时多久
                     0,
                      //隔多久执行
                     3,TimeUnit.SECONDS);
                     executor.scheduleWithFixedDelay(new Runnable() {
                                            @Override
                                            public void run() {
                                                System.out.println("scheduleWithFixedDelay"+new Date());
                                            }
                                        },
                      //第一次执行任务时 延时多久
                      0,
                      //每次执行完 延时多久隔多久执行
                      3,TimeUnit.SECONDS
                      );

                      Timer 单线程  如果阻塞比较久  会影响到下一个线程
                      如果跑出异常  会导致其他任务不能执行
               ForkJoinPool 
                    原理 
                     一个线程一个队列
                     如果有线程执行完，会去查看其它线程对应的队列是否有未完成的任务
                     自己任务后进先出  别人任务 先进先出

                     并行执行  工作窃取

                     RecursiveTask 实现 ForkJoin 
                     覆写 compute函数  不能处理异常  逻辑较为复杂
                     ParallelStream 底层使用 ForkJoinPool
               Executors 使用
               
                 cache  内部实现 core 0 workQueue  sychronous
                 fix     内部使用  linkedQueue 无参构造  无界队列       
               线程池调优实战
                  任务类型
                    cpu密集型任务 挖矿 计算
                    IO密集型  增删改查任务
                    混合型任务 

                线程数调优
                    太多会造成切换  太少浪费计算机资源
                    
                    cpu密集型 n+1(某一个线程暂停中断 cpu有一个空闲) 理论上n  没有切换 没有资源浪费

                    IO密集型任务  2n 处理io 不需要占用cpu

                    估算公式
                    N*u*(1+WT/ST) n cpu核心数 u 目标cpu的利用率
                    wt 线程等待时间  ST 线程运行时间
                BlockingQueue调优 
                   单个任务占用内存
                   线程池计划占用内存

                业务评估、综合压测逐步调整   
     --连接池
         数据库连接池
            DBCP2(commons pool2)   druid（为监控而生）  tomcat  hikari  
         Redis连接池
            Jedis  Lettuce  Redisson
         HTTp连接池
           httpClietn OKhTTP自带连接池
           RestRTemplate Feign 底层使用以上连接池  
         连接池调优
            获取连接-执行业务  -归还连接
            经验值 连接数=2N+可用磁盘数  业务估算+压测 、分离连接池 2个连接池
            数据库  报表连接池、业务增删连接池
     --异步化
         同步  程序按照定义的顺序执行，每一行都必须等待上一行执行后才能执行

         异步   程序执行时，无须等待异步调用就能直接返回，即可执行后面操作  

         异步适用场景
         耗时操作中，将不影响主流程的操作异步执行，降低响应时间
         实现异步的方式
          创建线程
          线程池
          @Async注解 
              1@Async必须返回void 或者future
              2建议将@  @Async标准的方法放到独立的类中  this调用异步方法会失效 原因是@ASYNC是通过动态代理实现的  this会是代理失效

              如果需要异步和同步方法同时在一个类中
              AsyncDemoAnnotation bean = applicationContext.getBean(AsyncDemoAnnotation.class);
               Future<String> log = bean.saveOpLog();
              3建议自定义queue大小  因为spring线程池默认使用 ThreadPoolTaskExecutor 
               默认无界队列  linkedBlockingQueue
               在配置文件中  设置队列大小 task:
                 pool:
                    queue-capacity: 200
          

            生产者消费者模式  
         远程调用异步化
          异步版本的AsyncRestTempalte

           ListenableFuture<ResponseEntity<IMOOCJSONResult>> future = this.asyncRestTemplate.postForEntity

           future.addCallback

          webclient
              Mono<IMOOCJSONResult> mono = this.webClient.get().uri("http://localhost:8088/index/subCat/{rootCatId}", 1)
                .retrieve().bodyToMono(IMOOCJSONResult.class);

              Mono<IMOOCJSONResult> mono = this.webClient.post().uri("http://localhost:8088/address/update")
                .contentType(MediaType.APPLICATION_JSON)
                .body(BodyInserters.fromObject(addressBO))
                .retrieve()
                .bodyToMono(IMOOCJSONResult.class);
                

                 IMOOCJSONResult result = mono.block();
         
          mq 生产者消费者  
     --锁优化
        --synchronized
          如何使用
             java内置的关键字
             确保互斥
              并发问题出现条件
               1存在共享数据
               2多个线程操作共享数据
             修饰实例方法 给当前实例加锁 进入同步代码要获取实例的锁
             修饰静态方法 给当前类加锁  进入同步代码之前要获得指定对象的锁
             修饰代码块  给指定对象加锁 ，进入同步代码之前要获得对象的锁

          底层原理

            早期：基于对象内部的监视器实现 （用户态切换到内核态）
                 监视器依赖于操作系统的MutexLock 互斥锁
           
             1.6
             对象存储（堆内如何存储） 
             对象头 
               Mark Word 
                 无锁 
                 偏向锁 -XX:+UseBiasedLocking（开启偏向锁） -XX : BiasedLockingStartupDelay=0（启动多久后开启偏向锁）
                 -XX:+UseHeavyMonitors 禁用偏向锁级轻量级锁
                （偏向于已经拿到锁的线程）
                 锁升级过程
                 x访问线程同步方法->检查Mark work的锁状态(无锁还是偏向锁)-> 偏向锁（检查是否指向线程x）-是 执行同步代码块/否 cas替换线程id-成功>执行同步代码、失败 撤销偏向锁 （说明存在大量竞争） -> 暂停线程y并检查状态 ->升级为轻量级锁  ->cas 操作  -失败> x线程自选获得锁->(自选操作依然失败) 升级到重量级锁->释放锁


                 轻量级锁 
                 重量级锁（操作系统）

                 各种锁的场景
                    偏向锁 加锁和解锁都不需要额外的消耗 如果线程存在额外的竞争 会带来额外的锁撤销的消耗  适用于只有一个线程访问同步块
                    轻量级锁  竞争的线程不会阻塞 提高了程序响应速度  ；如果始终获取不到会使用自旋消耗cpu 适用于 追求响应时间， 锁占用时间短
                    重量级锁  线程竞争不使用自旋 不会消耗cpu  线程阻塞 响应缓慢；追求吞吐量 所占用时间较长
                 GC标记
               类型指针 
               数组长度
             实例数据 
             对其填充

             其他优化机制
              -server  -XX:+DoEscapeAnalysis -XX:+EliminateLocks 开启逃逸分析 以及锁消除
              -server  -XX:-DoEscapeAnalysis -XX:-EliminateLocks 消除
              逃逸分析
                分析变量是否会逃出作用域
              锁消除
              锁粗化  将多个连续的加锁 解锁连接在一起，扩展成范围更大的锁
                //进行锁粗化：整合成一次锁请求、同步、释放
               synchronized (lock) {
                 // do some thing
                 // 能够很快执行完毕，且无需同步的代码
                   // do other thing
                 }
        --ReentranLock
           lock 表示获得锁
           tryLock 表示尝试获得锁 如果失败返回false
           tryLock(unit) 带超时的
           lockInterrupttibly 表示当前线程interrupt就抛异常 如果没有就去获得锁
           unlock 解锁

           互斥性

           可重入性

            try {
            lock.lock();
            lock.lock();
            i = i + 1;
             } finally {
            lock.unlock();
            lock.unlock();
            }

           公平性
              线程谁先请求 谁先获得锁
              公平锁 如果另一个线程持有锁或者有其他线程在等待队列中等待这个锁，那么新发出的请求的线程将被放入到队列中
              非公平锁 当锁被某个线程持有时，新发出请求的线程会被放入队列中；发出请求的时候，锁变成可用状态，这个线程会跳过队列中等待线程获得锁
              默认是非公平锁 非公平锁性能高

            Condition
              替换传统线程通信中，Object的wait notify
              await -wait
              signal notify
              sigalall notifyall

             操作Condition 必须在lock 和unlock 保护下
                多个线程之间的协作
          
           实现原理  AQS(AbstractQueuedSynchronizer)
              用于构建锁和同步容器的框架
              简化同步细节
                node类型的双向链表  node线程封装  
                 head 头结点  tail 尾结点 cas操作添加节点（在方法addWaiter(是否独占/共享模式)）
                 native方法 var1 var2-->确定内存中对象
                 enq
              tryLock
                 getstate
                  1cas/设置独占线程
                  2已经锁了 state+1 
              unlock
                tryRelease 
                  state-1
              lock sync.lock
                非公平  cas操作 设置独占/acquiredQueued(addWaiter)    
        以上对比
          相同 互斥；可重入性
          不同 syn jvm实现  reen jdk实现
          性能 jdk1.6之前 性能差
               jdk1.6差不多
          使用 sync 方便简洁 jvm保证加锁解锁
               reen 手动加锁解锁
          功能 reen 可指定公平非公平  非常丰富api  Condition

          优先synchronized  
         --ReentrantReadWriteLock  
          线程x 线程y 
             读多写少
             ReadLock 读锁 共享锁
             WriteLock 写锁 排它锁
             线程允许获取读锁的条件
               不能有其他线程的写锁
               没有写请求 或者有写请求 但调用线程和持有锁线程是同一个 
             线程允许获得写锁的条件
               不能有读锁
               不能有写锁
            三个特性
              公平性
              可重入性
              锁降级 先获得写锁  获得读锁      释放写锁
            ReentrantReadWriteLock  与ReenTranLock
               与ReenTranLock 完全互斥
                ReentrantReadWriteLock 读锁共享 写锁互斥
         --StampedLock  上面的锁的增强版
           
            写线程排他的 
             饥饿问题 1个写线程  读线程500
           特性
             获得锁的方法 都会返回stamp 如果stamp=0 表示操作失败 其他值表示成功
             释放锁的方法 需要用stamp作为参数，参数的值必须和加锁时返回的stamp一致
            提供三种访问模式
              writing   类似于写锁
              Reading  悲观读模式 类似于读锁
              Optimistic reading 乐观读模式

              悲观读 在执行悲观读的过程中 不允许写操作
              乐观读 在执行乐观读的过程中 允许写操作

             提供了读锁于写锁的相互转换的功能
             不可重入
              // 获得一个乐观读锁
              long stamp = sl.tryOptimisticRead();
                // 验证获得乐观锁之后，有没有发生过写操作
              if (!sl.validate(stamp)) {
           锁优化的五板斧
           减少锁的持有时间
           减少锁的粒度 ConcurrentHashMap segment    	
           锁粗化 频繁加锁解锁
           锁分离  读写分离ReenTrantLock  操作分离（不同业务使用不同锁 linkedBlockingQueue  putLock takeLock）
           Cas
jvm 调优
  理论
    内存结构
      线程共享
        堆 
          jdk 8之前
           新生代 Eden From to（伊甸园和存活区） 
           老年代  Tenured
           持久代  Perm Gen
          jdk8 之后 
           新生代
           老年代
        方法区（堆和元空间有交集）
           jdk8 之后  静态变量 字符串常量池 存放到堆里
           原空间 MetaSpace (本地内存) 元空间  
           方法1区  类信息（类的版本 字段描述 方法描述 接口和父类等描述
           class文件常量池 ）  运行时常量池  存放到元空间

           静态常量池  
             存放class文件常量池；
              字面量 例如文本字符串 final修饰的常量
              符号引用 例如类和接口的全限定名 字段的名称和描述符 方法的名称和描述符

           运行时常量池
               当类加载到内存中后 jvm会将静态常量池的内容移动到运行时常量池；运行时常量池存储的编译器的字面量和符号引用
           字符串常量池 
              理解为运行时常量池分出来
              类加载到内存时 字符串  会存到字符串常量池    
      线程隔离   
        虚拟机栈
           栈帧（局部变量表 操作数栈  指向运行时常量池的引用 方法返回地址 
           动态链接） java方法
        本地方法栈  native 方法
        程序计数器 各个线程字节码的地址 （线程争抢 返回时需要知道执行到哪儿）
      实战演示
        堆 局部变量 demo=0x000888
        方法区 JVMtEST1.class main方法描述信息
               Demo.class 成员变量描述信息
        堆 0x000888(name="aaaa")       
    加载机制
       .java
       .class
        加载
          读取类的二进制流
          转换为方法区数据结构 并存放到方法区
          在java堆里产生java.lang.class对象
         链接
           验证 验证class文件是否符合规范 
                1文件格式的验证 是否以0XCADEBABER开头
                版本是否合理
                2 元数据的验证  是否有父类 是否继承了final类  非抽象类实现了所有抽象方法
                3字节码验证
                   运行检查 栈数据类型和操作码参数吻合
                   跳转指令指向合理的位置


                4符号引用验证
                   常量池描述类是否存在
                   访问的方法或字段是否存在并且有足够的权限
                    -Xverify:none（关闭验证）   
           准备 为类的静态变量分配内存 初始为系统的初始值
               final static 修饰的变量 
            解析 符号引用转换为直接引用
         初始化  
           执行class init方法 由              
            编译器自动收集类的静态变量的赋值和静态语句块合并而成，也叫类构造器
                  初始化的顺序和源文件的顺序一致
                  子类的cint被调用之前 会先调用父类的cint
                  jvm保证cint方法的线程安全
           初始化时 如果类实例化一个新对象 回到用init 方法进行初始化 并执行对应的构造方法内的代码
         使用
         卸载  
    编译器优化 
       字节码是如何运行的
         解释执行：由解释器一行一行翻译执行  优势在于没有编译等待时间，性能较差
         编译执行 把字节码编译成机器吗 直接执行机器码 运行效率高很多，带来了额外的开销
       查询运行模式  
         java -version
         java -Xint -version  设置jvm的执行模式为解释执行
         -XComp jvm优先以编译模式运行
         -Xmixed 混合模式运行
       一般情况  
          一开始由解释器解释执行
          当虚拟机发现某个方法或代码块运行特别频繁，就会认为是热点代码，为了提高热点代码执行效率，进行即时编译 JIT 把这些热点代码编译为本地平台相关的机器码，并执行各种层次的优化
       两个编译器
         HotSpot  C1
          是一个简单快速的编译器
          主要关注局部性优化
          适用于执行时间较短或对启动性能有要求的程序 例如GUI
           也被成为Client Compiler
         HotSpot c2
          长期运行的服务端应用程序做性能调优的编译器
          适用于执行时间较长或对峰值有性能要求的程序
          Server Compiler
       分层编译
         0 解释执行
         1 简单C1编译 会用C1编译器进行一些简单的优化，不开启profiling
         2 受限的C1编译 仅执行带方法调用次数以及循环回边执行次数Profiling的c1编译  
         3 完全C1编译 会执行带有所有Profiling的c1 代码
         4 c2编译  会使用c2编译器进行优化 该级别会启用一些编译耗时的优化
         级别越高 应用启动越慢，优化开销越高 峰值性能越高 
       jdk8 默认开启分层编译
       只想开启C2 -XX:-TieredCompilation(禁用中间编译层123)
       只想开启C1 -XX:+TieredCompilation -XX:TieredStopLevel=1
       如何找到热点代码
         基于采样的热点探测
         基于计数器的热点探测（默认）
          方法调用计数器：统计方法被调用的次数 C1=1500 C2=10000
          也可以指定-XX:CompileThreshold=x指定阈值
          回边计数器 用于统计一个方法循环体中代码执行的次数，在字节码中遇到流向后跳转的指令成为回边
          C1=13995 c210700  -XX:OnStackReplacePercentage=x指定阈值
        方法调用计数器执行流程
           方法入口  是否存在以编译版本
           存在 执行编译后的代码
           不存在 方法调用计数器值+1
             两计数器之和是否超过阈值
               超过想编译器提交编译请求
               没有超过以解释方式运行代码  

           相关参数 -XX:-UseCounterDecay关闭热度衰减
                   -XX:CounterHalfLifeTime设置半衰周期  
         回边计数器执行流程
             向编译器调教OSR编译              
      编译优化三种方式       
      方法内联
         把目标方法的代码复制到发起调用的方法之中，避免发生真正的方法调用
         // 内联后
          private static int addInline(int x1, int x2, int x3, int x4) {
             return x1 + x2 + x3 + x4;
          }
        方法内联的条件
          方法体足够小
            热点方法 如果方法体小于325字节会尝试内联 可用-XX:FreqInlineSize修改大小
            非热点方法  如果方法体小于35字节 会尝试内联 -XX:MaxInlineSize修改大小
          被调用的方法运行时的实现被可以唯一确定
            static private  final方法
            public的实例方法 执指向的实现可能是自身 父类 子类；
            当且仅当唯一确定方法的实现 才可以实现内联
        建议
         尽量让方法体小
         尽量使用 final private static 关键字
         场景下修改jvm参数
        内联可能带来的问题

          CodeCache的益处（热点代码的缓冲区）240M 导致JVM退化为解释执行模式
        方法内联的参数

          打印内联 -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining   
          @ 56   com.imooc.jvm.jvm.InlineTest2::add1 (12 bytes)   inline (hot)
                                @ 2   com.imooc.jvm.jvm.InlineTest2::add2 (4 bytes)   inline (hot)
                                @ 7   com.imooc.jvm.jvm.InlineTest2::add2 (4 bytes)   inline (hot)
                                470毫秒

          -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:FreqInlineSize=1（超过多大不内联）
           580毫秒
      逃逸分析 
         -XX:+DoeScapeAnalysis 
         全局变量赋值逃逸 
            public static SomeClass someClass;

           // 全局变量赋值逃逸
           public void globalVariablePointerEscape() {
                 someClass = new SomeClass();
           }
         方法返回值逃逸
          // 方法返回值逃逸
   
           public SomeClass methodPointerEscape() {
                return new SomeClass();
          }
          实例引用逃逸
          // 实例引用传递逃逸
          public void instancePassPointerEscape() {
             this.methodPointerEscape()
            .printClassName(this);
            }
         线程逃逸
          赋值给类变量或者可以在其他线程访问的实例变量
      逃逸状态标记
         全局级别逃逸 一个对象可能从方法或者当前线程中逃逸
          对象作为返回值  或者静态天亮
         参数级别逃逸
          对象被作为参数传递一个方法 但是在这个方法之外无法访问 其他线程不可见
         无逃逸   
      标量替换
        标量 不能被进一步分解的量 基础数据类型、对象引用
        聚合量  可以进一步分解的量  字符串

        通过逃逸分析确定该对象不会被外部访问 并且对象可以被进一步分解时，JVM不会创建该对象 而是创建该对象的成员变量来替代
        -XX:+EliminateAllpcations 开启标量替换 jdk8默认开启
      栈上分配      
        通过逃逸分析 能够确认不会被外部访问，直接在栈上分配对象
         -XX:+EliminatesLocks 锁消除 
    垃圾收集算法
       什么场景下该使用什么垃圾回收策略
         在对内存要求苛刻的场景 想办法提高对象的回收效率 多回收掉一些对象 腾出更多空间
         cpu使用率高情况下 降低高并发垃圾回收的频率，CPU更多的执行你的义务而不是垃圾回收
       堆（对象） 方法区（常量 类信息）
       对象在什么时候被回收
          引用计数法  通过对象的引用计数器来判断该对象是否被引用
            循环引用出现问题
          可达性分析 
            以根对象作为起点向下搜索，走过的路径被称为引用链，如果某个对象到根对象没有引用链相连接时，就认为对象不可达 可以回收。
          GC ROOTS  包含哪些对象？
            虚拟机栈中的引用对象
            方法区中类静态属性的对象
            方法区中常量引用的对象
            本地方法栈中引用的对象
       引用
         强引用 
           比如Object obj=new Object()的引用
           只要强引用在 永远不会被回收的是对象
         软引用 Soft reference 
            形如SoftReference<String> sr = new SoftReference<> ("jhello")
            用来描述一些有用但非必须的对象
            软引用关联的对象 只有在内存不足的时候才会回收
         弱引用 Weak Reference  
            弱引用也是用来描述非必须对象的
            无论内存是否充足 都会回收被弱引用关联的对象
         虚引用   
            不影响对象的生命周期 如果对象只有虚引用 那么它就和没有引用一样
            在任何时候都可能被回收；
            必须和ReferenceQueue 配合使用；
            当垃圾回收期准备回收一个对象时，如果发现它还有虚引用就会在回收对象之前 把这个虚引用加入到与之关联的引用队列中。
            程序可以通过判断引用队列中是否已经加入了虚引用 来了解被引用的对象是否将要被垃圾回收；
            如果程序发现某个虚引用已经被加入到引用队列那么就可以在所引用的对象的内存被回收之前采取必要的行动
       可达性分析注意点 
        
         一个对象即使不可达 也不一定被回收
       垃圾回收过程
           对象有没有引用链和根对象连接
           对象有无必要执行finalize
            有必要 将对象放入到F-Queue 虚拟机将自动创建低优先级的finalizer执行对象的finalize（）
              回收
              对象在finalize()中重新建立连接 ->从 F-Queue队列中移除

            无必要
              对象未重写finalize 或者虚拟机已经调用过finalize()
               回收   
       finalize()建议
       
         避免使用finalize()方法  操作不当可能会导致问题
         finalize  优先级低 何时被调用无法确定 因为什么时候发生GC不确定 
         使用try catch finally 代替     
    垃圾回收算法
         标记清除 Mark-Sweep
           标记需要清除的对象
           清理掉要回收对象
           垃圾清理之后存在内存碎片
          标记整理 Mark-Compact
            标记需要回收的对象
            把所有存活的对象压缩到内存的一端
            清理掉边界外的所有空间 
          复制算法 Copy
            把内存分为两块每次只使用一块
            将正在使用的内存中的存活对象复制到未使用的内存中，然后清除掉正在使用的内存中的所有对象
            交换两次内存的角色

           三种算法对比  
             标记清除  实现简单 存在内存碎片分配内存速度受影响
             标记整理  无碎片 整理内存开存在开销
             复制       性能好 无碎片  内存利用率低
           分代收集算法
               商业虚拟机都采用分代收集
               根据对象的存活周期 把内存分为多个区域 不同的区域使用不同的垃圾回收算法

             回收类型
             新生代回收  Minor Gc|Young Gc
             老年代回收  Major Gc
             整个堆回收  Full GC 约等于Major gc


           对象分配过程
                意外
                新建对象不一定分配到伊甸园
                1 对象大于-XX:PretenureSizeThreshold 直接分配到老年代
                2 新生代空间不够
                对象不一定要达到年龄才进入老年代
                动态年龄： 如果Survivor空间中所有相同年龄对象大小的总和大于Survivor空间的一半 那么年龄大于该年龄的对象就可以直接进入老年代
            触发垃圾回收的条件-
              新生代-伊甸园空间不足 Minor GC
              老年代  Full Gc
                老年代空间不足、
                元空间不足、
                要晋升到老年代的对象所占用的空间大于老年代的剩余空间
                显式调用System.gc
                  建议垃圾回收器执行垃圾回收
                  -XX：+DisableExplicitGc参数 忽略掉System.gc的调用
             分代好处
                更有效的清除不需要的对象
                提升了垃圾回收的效率
             分代收集算法的调优原则
                合理设置Survivor区的大小 避免内存消耗
                让GC尽量发生在新生代 尽量减少Full GC发生的频率






           增量算法 每次只收集一部分  
    垃圾收集器
        算法；提供理论支持
        收集器 利用算法 实现垃圾回收的实践
        新生代 Serial ParNew Parallel Scavenge
        老年代 CMS Serial Old Parallel Old
        跨年代 G1  

        Stop The World
        STW 全局停顿 java代码停止运行 native代码可以运行 但不能与Jvm进行交互
         原因：多半是由于垃圾回收导致 也可能由Dump线程 死锁检查 Dump堆导致
         危害：服务停止 没有响应 主从切换 危害生产环境

        并行收集
          多个垃圾收集线程并行工作，但是收集的过程中，用户线程（业务线程）还是处于等待状态的

        并发收集
          用户线程和垃圾收集线程 一同工作
        吞吐量
           cpu用于运行用户代码的时间与Cpu总消耗时间的比值
           公式：运行用户代码时间/(运行用户代码时间+垃圾收集时间) 
         新生代
         Serial收集器
         最基本的发展历史最悠久 底层采用复制算法
         执行过程
           用户线程运行到 safepoint 全部暂停 GC线程进行垃圾回收  回收完毕 用户线程继续运行
         特点：单线程  简单高效   收集过程全程 stw
         适用于场景  
           客户端 -client
           单核机器        
         ParNew
         Serial收集器的多线程版本
         收集过程： GC线程属于多线程
         特点：多线程
              使用-xx:ParallelGCThreads设置垃圾收集线程数量
         使用场景
           CMs配合使用   
         Parallel Scavenge      
           达到可控制的吞吐量
           -XX:MaxGCPauseMillis 控制最大的垃圾收集停顿时间
           -XX:GCTimeRatio 设置吞吐量的大小 取值0-100，系统花费不超过1/(1+n)的时间用于垃圾收集
           自适应GC策略 可用 -XX:+UseAdptiveSizePolicy 打开
           打开自适应策略后 无需手动设置新生代的大小（-Xmn） Eden 与Survivor区（-XX:SurvivorRatio）的比列等参数
           虚拟机根据系统的运行状况收集性能监控信息

           适用掺进 比较注重吞吐量的场景
         老年代
         Serial Old
          Serial老年代版本  标记整理
          可以和所有的新生代配合使用
          CMS 收集器出现故障的时候 会用Serial Old作为后备
         Parallel Old 
           Parallel Scavenge 收集器的老年代版本
           标记整理算法
           适用场景  与Parallel Scavenge 配合使用
         CMS  Concurrent Mark Sweep
           并发收集
           标记清除

           初始标记
              标记根对象能直接关联到的对象  STW
           并发标记  
              找出所有根对象能关联到的对象
              并发执行 无STW
           并发预清理  （不一定执行）
              重新标记那些在并发标记阶段 引用被更新的对象，从而减少后面标记阶段的工作量
              并发执行 无Stw
              可使用-XX:CMSPrecleanEnabled关闭并发预清理阶段
           并发可中止的预清理阶段  
              和并发预清理做的事情一样 并发执行 无STW
              当Eden的使用量大于CMSScheduledRemarkEdenSizeThreshold的阈值默认2M 才会执行该阶段
              作用 允许我们能够控制预清理阶段的结束时机
              扫描多长时间 Eden区使用占比达到一定阈值
           重新标记
              修改并发标记期间 因为用户线程继续运行，导致标记发生变动的那对象的标记 
              一般来说 重新标记花费的时间会比初始标记阶段长一些，但比并发标记的时间短
               存在Stw
            并发清除
              基于标记的结果 清除掉前面标记出来的垃圾    
              并发执行 无stw
                为什么不是并发整理  实现难
            并发重置
              清理本次CMS的上下文信息 为下一次GC做准备   
         优点：STW时间比较短  大多数并发执行
         缺点：并发阶段可能导致应用吞吐量的降低 CPU资源比较敏感
               无法处理浮动垃圾
               不能等到老年代几乎满了才开始收集
                 预留内存不够  ConcurrentMode Failure ->Serial Old 作为后备
                 可使用CMSInitiatingOccupancyFration设置老年代占比达到多少就触发垃圾回收 默认68%
               内存碎片
                  标记清除算法
                  UseCMSCompactAtFullCollection 在完成Full GC后是否进行内存碎片整理 默认开启
                  CMSFullGCsBeforeCompaction 进行几次Full GC后进行内存碎片整理 默认0

         使用场景
            希望系统停顿时间短 响应速度快 比如服务端应用程序       
         G1收集器
         堆内存布局 若干个大小相等的region 取值范围1-32M  2的n次幂
         通过参数-XX:G1HeapRegionSize指定Region的大小

         Eden
         Survivor
         Old
         Humongous 存储大对象 也作为老年代的一部分

         设计思想
         内存分块
         跟踪每个region里面的垃圾堆积的价值大小 
         构建一个优先列表 根据允许的收集时间 优先回收价值高的Region
         垃圾收集机制
         young gc
           所有Eden Region都满了的情况 触发 young gc
           伊甸园的对象会转移到Survivor region里面去
           原先Survivor region中的对象转移到新的Survivor region中 或者晋升到 Old Region
           空闲的Region 会被放到空闲列表中等待下次被使用
         mixed gc
           当老年代大小占整个堆的百分比达到一定阈值
           -XX：InitiatingHeapingOccupancyPercent指定 默认45%
           回收所有的young region 同时回收部分 Old Region 
           初始标记
           并发标记
           最终标记
           筛选回收 
              对各个Region的回收价值和成本进行排序
              根据用户所期望的停顿时间来指定回收计划，并选择一些region回收 
          回收过程：
            选择一些列region 构成一个回收集
            把决定回收的region的存活对象复制到空的region
            删除掉需回收的region 无内存碎片
            存在STW 
         full gc   
           复制对象内存不够 或者无法分配足够内存    
           使用Serial Old模式

        G1 优化原则  减少FULL GC   

          增大预留内存 增大-XX:G1ReservePercent 默认为堆10%
          更早地回收垃圾  减少 -XX:InitiatingHeapOccupancyPercent  老年代达到该值就触发mixed gc默认45% 
          增加并发阶段使用的线程数 增大 -XX:ConcGCThreads
        特点
          作用在整个堆
          可控的停顿时间
            MaxGCPauseMillis 
          占用内存较大的应用  6G以上 G1  6G以下CMS
          替换CMS垃圾收集器    
        如何选择垃圾收集器
          关注点
            计算类 吞吐量  Parallel SCavenge
            响应时间 cms g1
            桌面端 启动慢  Serial 
          基础设施
          JDK
  工具 
       监控工具
         jps
           q				只显示进程号
           -m				显示传递给main方法的参数
           -l				显示应用main class的完整包名应用的jar文件完整路径名
           -v				显示传递给JVM的参数
           -V				禁止输出类名、JAR文件名和传递给main方法的参数，仅显示本地JVM标识符的列表。
         jstat
            jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
            jstat -class -t -h3 17416 1000 10
             <option>      指定参数，取值可用jstat -options查看
              <vmid>        VM标识，格式为<lvmid>[@<hostname>[:<port>]]
                <lvmid>：如果lvmid是本地VM，那么用进程号即可; 
                <hostname>：目标JVM的主机名;
                <port>：目标主机的rmiregistry端口;
              -t            用来展示每次采样花费的时间
             <lines>       每抽样几次就列一个标题，默认0，显示数据第一行的列标题
             <interval>    抽样的周期，格式使用:<n>["ms"|"s"]，n是数字，ms/s是时间单位，默认是ms
              <count>       采样多少次停止
             -J<flag>      将<flag>传给运行时系统，例如：-J-Xms48m
             -? -h --help  Prints this help message.
             -help         Prints this help message.
       故障排查工具
        
         jinfo
          查看jvm参数  调整jvm参数
          打印系统属性和JVM参数
          jinfo 42342
          只打印系统属性
          jinfo -sysprops 17416
          只打印jvm参数 
          jinfo  -flags 17416
          查看jvm参数是否开启
          jinfo  -flag UseG1GC 17416

          启动应用时  指定 -XX:+PrintFlagsFinal

          查看可以动态调整的参数java -XX:+PrintFlagsFinal | grep manageable
          动态调整
          jinfo -flag +HeapDumpAfterFullGC 17416
          jinfo -flag MinHeapFreeRatio=60 17416 
         jmap 展示对象堆内存映射或内存详细信息
          jmap [options] pid
          jmap -clstats pid java堆的类加载统计信息
          jmap -finalizerinfo 等待finalization的对象
          jmap -histo:live  统计活动对象
          jmap -dump:live,format:b,file=filepath  将堆dump到filename
         jstack  打印当前虚拟机线程快照(出现长时间等待  死锁 死循环)
          jstack  -l（锁）  -e（额外信息）
         jhat  分析堆Dump文件 
         jcmd 
          jcmd <pid | main class> <command ...|PerfCounter.print|-f file>
           jcmd com.imooc.Application PerfCounter.print
           jcmd 17416 PerfCounter.print
         jhsdb
           jhsdb clhsdb --pid 25448 
            交互式的
           jhsdb hsdb --pid 25448  图形化界面
           
           jhsdb jstack 
           jhsdb jmap 
           jhsdb jinfo
           jhsdb debugd 非常
       可视化工具
         jhsdb
         基于jmx
          jconsole  
          VisualVM
         JMC  可视化分析JFR JMX
           JFR在生产环境中堆吞吐量的影响不大于1%
           动态配置 无需重启
           对应用完全透明

          JFR产生的数据不向前兼容
           不支持 。dump文件  可以使用.jfr
           定位线上问题的神器
           性能调优  分析调优目标  
       远程连接

         jstatd
           远程配置
            添加policy文件
            服务端启动
            jstatd -J-Djava.security.policy=./jstatd.all.policy \ -J-Djava.rmi.server.hostname=micro\
            -J-Djava.rmi.server.logCalls=true \ -p 7412
           客户端使用VisualVm远程连接  
            不能使用Sample 


         jmx 多行输入 空格\enter
           服务端设置
           java -Djava.rmi.server.hostname=micro \
           -Dcom.sun.management.jmxremote.port=1233 \
           -Dcom.sun.management.jmxremote.rmi.port=1240 \
           -Dcom.sun.management.jmxremote.authenticate=false \
           -Dcom.sun.management.jmxremote.ssl=false \
           -jar eureka-server-0.0.1-SNAPSHOT.jar
           设置防火墙规则
           客户端使用visualvm远程连接
              可以使用sampler
           开启用户认证
           1创建 access password文件
           2java -Djava.rmi.server.hostname=micro \
           -Dcom.sun.management.jmxremote.port=1232 \
           -Dcom.sun.management.jmxremote.rmi.port=1240 \
           -Dcom.sun.management.jmxremote.authenticate=true \
           -Dcom.sun.management.jmxremote.access.file=/root/jmxremote.access \
           -Dcom.sun.management.jmxremote.password.file=/root/jmxremote.password \
           -Dcom.sun.management.jmxremote.ssl=false \
           -jar eureka-server-0.0.1-SNAPSHOT.jar

            keytool -genkeypair \
            -alias visualvm \
            -keyalg RSA \n      -validity 365 \n      -storetype pkcs12 \n      -keystore visualvm.keystore \n      -storepass visualvm-key \n      -keypass visualvm-key \n      -dname "CN=damu, OU=itmuch, O=itmuch.com, L=NanJing, S=JiangSu, C=CN"

            开启ssl
                

           ssh
             动态端口映射  安全
       第三方工具
         MAT 分析内存 找到内存泄露 集合使用

         浅堆
         保留集
         保留堆
         前导对象集的保留集

         支配树
         jitwatch  jit编译监控
  实战

      预热jvm参数选项
        java -help  查看标准选项
        java -X     查看附加选项
        java -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsInitial
        查看高级选项  hsdb  flags 查看高级选项
        boolen -XX:+ 打开  - 关闭
        其他类型 key=value
        高级选项的分类
          高级运行时选项
          高级JIT编译器选项
          高级可服务选项
          高级垃圾收集选项
         不该使用的选项
         CMS JDK11 标记为过时  废弃 淘汰  删除
        jdk8 垃圾收集相关参数
         -Xms50m -Xmx50m 
         -XX:+PrintGCDetails 打印垃圾回收详情
         -XX:+PrintGCDateStamps 打印垃圾回收日期
         -XX:+PrintGCTimeStamps 打印垃圾回收时间
         -XX:+PrintGCPause 打印垃圾回收原因
         -Xloggc:gclog.log 垃圾回收日志
      实战 
         #jdk8  垃圾收集打印日志参数
         -Xms50m -Xmx50m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -Xloggc:gclog.log
         #打印类加载 和偏斜锁信息
         -XX:+TraceClassLoading -XX:+TraceBiasedLocking
         -Xms50m -Xmx50m -Xlog:gc*=trace:file=xloggxc.log
          jdk9之前 日志格式不统一 
          之后 统一日志管理Xlog 统一配置
          -Xlog:class+load=debug,biasedlocking=debug:file=trace.log
      cpu过高问题定位
           top+jstack

           top第一行 查找最高占用率的进程
           top -Hp  pid 查找进程中占用率最高的线程 

           print %x  pid 

           jstack 10509 > 1.txt

           cat 1.txt  | grep -A 30  294a
         jmc  需要开启 jmx
         
         可能导致cpu占用率过高 的场景和解决方案

            无限while循环
              尽量避免无限循环
              让循环执行的慢一点（循环中加sleep）
            频繁GC
              降低GC频率
            频繁创建新对象
              合理使用单例
            序列化和反序列化
              使用合理的api实现功能
              选择好用的序列化和反序列
            正则表达式
               改写正则表达式 降低回溯发生
            频繁的线程上下文切换
               降低切换的频率         
      内存溢出
        堆内存溢出
          演示堆内存代码 并定位问题
           -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
           while (true) {
            oomTest.oomList.add(UUID.randomUUID().toString());
            } 
           定位使用	mat 或者visualvm导入 dump文件
            查看对象的引用链

          总结堆内存溢出的场景与解决方案
            内存泄露
            非内存泄露（调大堆内存） 存储结构不合理

          分析foodie-dev项目中可能存在的堆内存溢出的代码并解决
            pagesize 大小太大导致堆内存溢出
            添加 validated注解  并在输入参数中@Max
        栈溢出
          java虚拟机规范 
           如果线程请求的栈最大深度大于虚拟机所允许的最大深度，将会抛出StackOverflowError
           如果虚拟机的栈内存允许动态扩展，当无法申请到足够内存时，会抛出OutOfMemmoryError
          hotSpot 虚拟机
            栈内存不可扩展
            不区分虚拟机栈和本地方法栈 统一用Xss设置栈的大小
          有的虚拟机可以用Xss设置虚拟机栈 Xoss设置本地方法栈  
            Xss144k 
            局部变量会使用栈空间  导致栈的深度变小
          如何运行更多的线程
            减少Xss配置
            栈能分配的内存
               机器总内存-操作系统内存-堆内存-方法区内存-程序计数器内存-直接内存
            尽量杀死其他程序
            操作系统对线程的限制
               linux 系统对线程的限制
                 * cat /proc/sys/kernel/threads-max
                 * - 作用：系统支持的最大线程数，表示物理内存决定的理论系统进程数上限，一般会很大
                 * - 修改：sysctl -w kernel.threads-max=7726
                 * =======
                 * cat /proc/sys/kernel/pid_max
                 * - 作用：查看系统限制某用户下最多可以运行多少进程或线程
                 * - 修改：sysctl -w kernel.pid_max=65535
                 * =======
                 * cat /proc/sys/vm/max_map_count
                 * - 作用：限制一个进程可以拥有的VMA(虚拟内存区域)的数量，虚拟内存区域是一个连续的虚拟地址空间区域。
                 * 在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。
                 * - 修改：sysctl -w vm.max_map_count=262144
                 * =======
                 * * ulimit –u
                 * * - 作用：查看用户最多可启动的进程数目
                 * * - 修改：ulimit -u 65535   
             -Xms=10m -Xmx=10m -Xss=144k -XX:MetaspaceSize=10m     
        方法区溢出
           线程共享 ，用来存储被虚拟机加载的类型信息、常量、静态变量
            jdk8之前 Perm Gen 实现
            jdk8之后 被Metaspace实现
            一部分在堆内存 一部分在元空间

            // intern()：native方法
            // 如果字符串常量池里面已经包含了等于字符串x的字符串；那么
            // 就返回常量池中这个字符串的引用
            // 如果常量池中不存在，那么就会把当前字符串添加到常量池，
            // 并返回这个字符串的引用
              set.add(String.valueOf(i++).intern());

             /**
             * JDK 6: -XX:PermSize=6m -XX:MaxPermSize=6m
              * 报永久代溢出(java.lang.OutOfMemoryError: PermGen space)
               * ==========
               * JDK 7: -XX:PermSize=6m -XX:MaxPermSize=6m
                * 不报错，原因：JDK 7把字符串常量池放到堆了，设置-Xmx6m会报堆内存溢出
               * ==========
               * JDK 8+：同JDK 7
              */ 

             -XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=10m 
               Hello enhancedOOMObject = (Hello) enhancer.create();
             ava.lang.OutOfMemoryError: Metaspace 

             方法区总结
                不同JDK版本 方法区存放结构不同，相同的代码在不同版本异常不同
               
             溢出原因
               常量池里对象太大

               加载的类的种类太多
                 cglib  动态代理的操作库生成了大量的动态类
                 JSP项目 
                 脚本语言动态类加载 Groovy基本
              避免方法区溢出的方案
                 根据JDK版本 为常量池保留足够空间
                 jdk 设大PermSize MaxPerSize
                 jdk7 以上 设大Xms Xmx
                 防止类加载过多导致的溢出
                   jdk7之前 设大PerSize MaxPerSize 
                   jdk8 留空元空间相关的配置 或设置合理的元空间大小（元空间使用本地内存  动态扩展）
        直接内存溢出
           jvm虚拟机规范 没有定义这块内存
           不属于虚拟机运行数据区
           直接内存时一块由操作系统直接管理的内存 
           也叫堆外内存，可以使用Unsafe或者ByteBuffer分配直接内存  
           可用-XX:MaxDirectMemorySize控制 默认是0 表示不限制

           为什么要有直接内存？
             性能高
           直接内存使用场景
              有很大的数据需要存储，生命周期很长
              频繁的Io操作 比如并发网络通信  

           添加的代码如果报找不到主类 需要清空一下 
           
           分配直接内存

            Unsafe.allocateMemory(size)

            ByteBuffer.allocateDirect(size)   
              put flip get

            Unsafe 分配的内存溢出java.lang.OutOfMemoryError 没有小尾巴 
            -XX:MaxDirectMemorySize=100m对Unsafe不起作用

            ByteBuffer直接内存溢出报错是java.lang.OutOfMemoryError: Direct buffer memory
            // 2. -XX:MaxDirectMemorySize对ByteBuffer有效

            直接内存的经验之谈 

              堆Dump文件看不出问题或者比较小 可以考虑是直接内存的问题

              配置内存时，应给直接内存预留足够空间
           直接内存的总结:
              堆外内存 Io效率较高
              Unsafe  oom  无视设置
              ByteBuffer 底层是用unsafe类实现  包装异常  OOM带小尾巴  设置生效    
        代码缓冲区
           存储编译后的代码      
           -XX:ReservedCodeCacheSize=3000K
           compilation: disabled (not enough contiguous free space left)
           jit 编译器被禁用 已经编译的方式继续编译  没编译的只能以解释执行
           配置合理的代码缓冲区大小  代码缓冲区有清理过程

           总结
             设置合理的代码缓冲区大小 默认240m   
             如果平时项目性能OK 但是性能突然下降 ，业务没问题可排查是否又代码缓冲区满导致 
      分析GC日志

         -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+UseSerialGC -Xmx50m -Xloggc:gcnew.log 

         #jdk相关
         Java HotSpot(TM) 64-Bit Server VM (25.191-b12) for windows-amd64 JRE (1.8.0_191-b12), built on Oct  6 2018 09:29:03 by "java_re" with MS VC++ 10.0 (VS2010)
         #内存相关
         Memory: 4k page, physical 33411236k(13243500k free), swap 39364556k(7378648k free)
         #当前应用的jvm参数
         CommandLine flags: -XX:InitialHeapSize=52428800 -XX:MaxHeapSize=52428800 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseSerialGC 
         #年轻代 
          2020-08-07T12:02:28.475+0800: 0.634: [GC (Allocation Failure) 2020-08-07T12:02:28.476+0800: 0.634: [DefNew: 13696K->1664K(15360K), 0.0953249 secs] 13696K->2118K(49536K), 0.0966149 secs] [Times: user=0.00 sys=0.00, real=0.10 secs] 

         #Full Gc
          2020-08-07T12:02:31.896+0800: 4.055: [Full GC (Metadata GC Threshold) 2020-08-07T12:02:31.896+0800: 4.055: [Tenured: 6245K->6678K(34176K), 0.0254347 secs] 9070K->6678K(49536K), [Metaspace: 20586K->20586K(1067008K)], 0.0257918 secs] [Times: user=0.06 sys=0.00, real=0.03 secs] 

          -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+UseConcMarkSweepGC -Xmx50m -Xloggc:gccms.log 

         可视化分析工具
          gceasy  不开源
          gcviewer  开源  不完全支持xlog  
      项目变慢
          stop th world 过长    

         项目依赖的资源导致变慢 （数据库 网络）

         Code Cache 满了

         线程争抢过于激烈


         服务器问题
           操作系统问题
           其他进程争抢资源

         线程dump可视化工具 
          fastthread  perfama          
      tlab 实战
         thread local allocation buffer 线程私有分配缓冲区
         线程专用的内存分配区域，jvm会为每个线程分配一块tlab区域 占用eden空间
         为什么要有tlab
           加锁对象分配
         tlab  空间较小 所以大对象无法在tlab分配 只能直接分配到线程共享的堆里面
         分配独享 使用共享
         使用默认配置 不需要修改配置

         对象分配总结
            尝试栈上分配  成功  栈上分配
            不成功  尝试tlab分配（是否足够小）  成功 tlab分配
            失败 尝试Eden分配  成功eden 分配
            失败 直接在老年代分配
          // -XX:+UseTLAB: 484ms
          // -XX:-UseTLAB: 1239ms   
      xxfox  在线查询jvm 参数 只能查询jdk8
        参数查询  参数检查 参数变迁 参数生成   
