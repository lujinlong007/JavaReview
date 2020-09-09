# jdk基础
## java平台理解

               JRE(JVM java类库 一些模块)
               JDK 是jre的超级 提供了更多工具  编译器和各种诊断工具
                java源代码  --->javac 编译为字节码--》运行时 jvm内嵌的解释器将字节码转换为最终的机器码
                JIT 动态编译  -->运行时 将热点代码编译为机器码 属于编译执行而不是解释执行
               语言特性： 泛型  lambda       
               基础类库：集合 IO/NIO  网络 并发
               class loader :Bootstrap  application  extension 自定义的classloader
               类加载的大致过程  加载 验证 链接 初始化  
               常见的垃圾收集器 SerialGC parallelGC CMS G1
                 一次编译 到处执行
                java 分为编译时和运行时 编译源代码生成。class文件和jvm跨平台抽象 屏蔽操作系统和硬件的细节
                 jdk8实际是解释和编译混合的一种模式 -XMIXED

                JIT compiler
                C1客户端 C2服务端
                 -xint 只能解释执行 不对代码进行编译
                 -xcomp 关闭解释器。不进行解释执行
      --  Exception 和Error有什么区别
               都继承子Throwable类 只有throwable类型的实例才能被抛出throw和捕获catch
                  Exception 是程序正常运行中，可以预料的情况 可能并且应该被捕获，进行处理
                     exception 又分为可检查和不可检查 
                     可检查在源代码里必须进行显示捕获，这是编译检查的一部分
                     不可检查就是运行是异常 类似NullPointerExcepiton arrayOutBoundsException
                          可以编码避免的逻辑错误，具体根据需要俩判断是否需要捕获，并不会在编译器强制要求。

                  Error 正常情况下不大可能出现的情况  处于非正常和不可恢复的状态 OutMemoryError
                 
                  NoClassDefFoundError  
                   如果这个类在编译时时可用的，在运行时找不到这个类的定义时候，就会抛出（jvm运行时系统抛出）
                  ClassNotFoundException(类加载器加载class文件 class.forname() classloader.loadclass() 如果没有在classpath下找到 ClassNotFoundException是一个检查异常)
                 try-catch-finally throw throws
                 try-with-resources mutiple catch  扩展了Autocloseable closeable对象
                 try(BufferReader br = new BufferReader()){
                 	 do something
                 }
                 catch(){

                 }

                 实践  不要捕获通用异常  而应该捕获特定异常
                       throw RuntimeException
                       不要生吞异常
                       throw early catch late  在更高层面有了清晰的业务逻辑 更清楚合适的处理方式
                       try-catch 产生额外的性能开销  影响jvm对代码的优化
                        不要利用异常控制代码流程
                        每次实例化一个Exception 都会对当前线程进行快照，相对重的操作
      -- 常用关键字理解
         -static 静态 全局 一旦被修饰，说明修饰的东西在一定范围内是共享的，谁都可以访问，注意并发读写问题；
          修饰的对象  
          修饰类变量 如果该变量是public ,任何类都可以直接访问，而且无需初始化类，直接类名.static变量访问
           如我们定义了：public static List<String> list = new ArrayList();这样的共享变量。
           两个解决方案：把线程不安全的arrayList 换成CopyOnwriteArrrayList
                        每次访问手动加锁
          方法  代表该方法和当前类无关，任何类都可以直接访问；该方法内部只能调用static修饰的方法 util类里面的各种方法，全部static修饰
          方法块  静态块用于在类启动时做一些初始化的操作 也只能调用被static修饰的变量，并且变量写在前面。
                  public static List<String> list = new ArrayList();
             // 进行一些初始化的工作
            static {
                list.add("1");
                }
            初始化时机
             父类静态变量初始化 父类静态块初始化 子类静态变量初始化 子类静态块初始化 main方法执行 父类构造器初始化 子类构造器初始化
              被static修饰的方法，在类初始化的时候并不会初始化，只有当自己被调用才会执行
          -final
             意思是不变的，三种场景：
               被final修饰的类 表明该类无法继承
              被final修饰的方法，表明该方法无法被覆写
              被final修饰的变量 表明变量在声明的时候，就必须初始化完成，以后也不能修改其内存地址
               list map被final修饰后，可以修改其内部值得，但是无法修改其内存地址
          -try catch finally
             try确定代码执行范围 catch捕获发生的异常，finally用来执行一定要执行的代码块
             fiannly先执行后，再抛出catch的异常
             catch中发生了未知异常，finally还会执行，避免catch吃掉异常，先把try抛出的异常打印出来
          -volatile 
              多线程共享 & 阻止指令重排序 
              可见的 修饰某个共享变量 ，当共享变量被被修改是，通知到其他线程上，其他线程就能会知道当前共享变量的值被修改了
              多核cpu下 为了提高效率 线程直接会和cpu缓存打交道，而不是内存，主要是因为cpu缓存执行速度更快，线程要拿c,会直接从cpu缓存中拿，cpu缓存中没有，就会从内存中拿；
              线程读永远拿cpu缓存的值 cpu缓存和内存中的值并不是时刻都同步的；volatile 内存主动通知缓存，该值已失效，需要重新拉取
          -transient 
             用来修饰类变量 当前不变不需要进行序列化。
           -default
             使用在接口上，对于该接口，子类无需强制实现，但是自己必须有默认实现。
             
             变量和方法被static和final关键字修饰的目的：
              变量和方法与类无关，直接使用，比较方便
              强调内存地址不可变，方法不可被覆写，强调方法内部的稳定性
      -- final关键字的作用 (方法、变量、类) finally finalize
                 final 修饰的class 代表不可扩展 
                 final的变量时不可修改的
                 final的方法也是不可重写的

                 finally 则是保证代码一定被执行的一种机制
                  try-finally try-catch-finally 关闭jdbc连接 保证unlock
                 finalize 是基础类java.lang。object的一个方法  保证对象被垃圾收集前完成特定资源的回收
                 
                  final 修饰类或方法  告知不许修改，保证平台安全的必要手段
                  final 修饰参数或者变量 可以清楚避免意外赋值导致的编程错误
                  final 变量产生不可变的效果 在并发编程中，明确不能在赋值final变量 有助于减少额外的同步开销
                  也可以省去一些防御性拷贝的必要
                 
                  finalize什么时候执行 使用不当会造成程序死锁 挂起

                  final 不是immutable
                   final List<String> strList = new List<>()
                    final只能约束strList应用不会被赋值，strList对象行为不被final影响
                    List.of创建不可变的List
                    实现immutable的类
                      将class声明成final
                      将所有成员变量定义为private 和final 不需要实现setter方法
                      构造对象是 成员变量深度拷贝
                      确实需要实现getter方法 使用copy-on-write原则 创建私有的copy
                  finalize 在垃圾回收前被调用 特殊公民
                    资源用完立即显示释放 或者利用资源池来重用
                    java.lang.ref.Cleaner 利用PhantomReference  实现所谓的post-mortem清理机制
                    利用幻象引用和引用队列 保证对象被彻底销毁前做一些类似的资源回收的工作，关闭文件描述符    
                    每个Cleaner操作都是独立的有自己的线程 可以避免死锁

                   将state 定义为static  避免普通的内部类隐含着对外部对象的强引用  
                     内部类如果不是static  本身对外面那个类有引用关系
			-- 强引用、弱引用、虚引用、软引用
			     不同类型的引用 主要体现在对象不同的可达性状态和对垃圾收集的影响
			     强引用 
			      在 Java 中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，即使该对象以后永远都不会被用到 JVM 也不会回收。因此强引用是造成 Java 内存泄漏的主要原因之一 如果没有其它引用关系，只要超出作用域或者显示将其引用赋值给null,就可以被垃圾回收
               软引用
             软引用需要用 SoftReference 类来实现，对于只有软引用的对象来说，当系统内存足够时它不会被回收，当系统内存空间不足时它会被回收。软引用通常用在对内存敏感的程序中 jvm会确保在抛出OutofMemoryError之前 清理软引用对象
                弱引用
              弱引用需要用 WeakReference 类来实现，它比软引用的生存期更短，对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管 JVM 的内存空间是否足够，总会回收该对象占用的内存。 构建一种没有特定约束的关系 维护一种非强制的映射关系，如果试图获得对象还在 就是用它 否则重新实例化  实现缓存的选择
                虚引用
                虚引用需要 PhantomReference 类来实现，它不能单独使用，必须和引用队列联合使用。虚引用的主要作用是跟踪对象被垃圾回收的状态。
             提供一种确保对象被finalize以后 做某些事情的机制。

               对象可达性状态流转分析

                   对象创建--> 对象初始化--> 强引用状态-->
                    软引用 ->弱引用->finalize->幻象引用->unreachable

                    强可达 当一个对象可以有一个或者多个线程可以不通过各种引用访问到  比如创建一个对象 创建它的线程对它强可达
                    软可达  只能通过软引用才能访问的对象
                    弱可达 只能通过弱引用访问 十分临近finalize 状态，当弱引用被清楚时，就符合finalizede 条件了
                    幻象可达 没有强软弱引用关联 并且被finalize过了只有幻象引用指向这个对象
                    不可达状态 对象可以被清除了
                     除了幻象引用 对象如果没有被销毁 都可通过get方法获取原对象
                 判断对象可达性 时jvm垃圾收集器决定如何处理对象的一部分考虑
                    检查弱引用指向对象的是否被垃圾收集 诊断是否有特定内存泄漏的一个思路
                    引用队列
                    显示的影响软引用垃圾收集
                             -XX:SoftRefLRUPolicyMSPerMB=3000
                     诊断JVM引用情况
                        
                     -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintReferenceGC     
                     finally  声明对象Reachability fence   
      -- int  integer      
               integer 是int  对应的包装类 有一个int类型的字段存储数据 
                静态工厂方法 valueOf 利用缓存机制 
                自动装箱和拆箱 语法糖  发生在编译阶段 生成的字节码是一致的。
                javac 自动把装箱操作转换为Integer.valueOf()  把拆箱操作转换为Integer.int.Value
                避免无意中装箱和拆箱操作
                  源码分析  
                     IntegerCache  Integer内部缓存数据类
                     其中的成员变量 value 也是不可变的  private final int value;
                  原始类型线程安全
                     原始类型要是用并发相关手段AtomicInteger
                     float double 甚至不能保证更新的原子性
                  原始数据类型和引用数据类型局限性
                     原始数据类型和java泛型并不能配合使用
                      java的对象都是引用类型 如果是一个原始数据类型数组 它在内存里是一段连续内存 而对象数组则不然 数据存储的是引用 对象往往是分散在存储在堆的不同位置。
      -- String StringBuffer StringBuilder
                 String 典型的immutable类 被声明成final class   所有属性也是final 由于它的不可变性 类似拼接 裁剪字符串 都会产生新的String 对象
                 StringBuffer 解决拼接产生太多中间对象提供的一个类  保证了线程安全 也带来了额外的性能开销
                 Stringbuilder  去掉了线程安全部分
                  字符串设计和实现
                   String 保证基础线程安全 无法改变内部数据   拷贝构造函数
                   StringBuffer 和StringBuilder底层利用可修改数组 二者都继承了AbstractStringBuilder
                    内部数组创建 多大 目前时字符串长度加16  扩容会产生额外开销 抛弃原有数组 创建新数组 还要进行arrayCopy
                    jdk8 会把string拼接转换成Stringbuilder操作
                  字符串缓存  
                       常见应用进行堆转储 25%对象时字符串  如果能避免创建字符串可有效降低内存消耗和对象创建开销
                       jdk6 使用intern 字符串存在PermGen  空间有限 基本很难被FullGc之外的垃圾收集照顾
                       jdk8中MetaSpace替代  缓存大小也被增大 
                        打印具体大小-XX:+PrintStringTableStatistics
                        使用intern 显示排重机制 显式调用 污染代码 
                       G1 gc -XX:+UseStringDeduplication
                          字符串的基础操作 都是直接利用jvm内部的intrinsic机制  特殊优化的本地代码 根本不是java生成的字节码
                   String 自身的演化
                     jdk6 直接char数组
                     jdk9 将储存方式从char数组 改变为一个BYTE数组外加一个标识编码的coder
      --动态代理
                 动态类型和静态类型就是 区分语言类型信息是在运行时检查还是编译期检查
                 强类型和弱类型 不同类型变量赋值是否需要强制类型转换
                 java是静态的强类型语言 提供了反射机制也具有了部分动态的语言的能力

                 动态代理是一种方便运行时构建代理 动态处理代理方法调用的机制
                   RPC调用 面向切面编程
                   jdk自身提供的动态代理机制 主要利用了上面提到的反射机制
                   其他的实现方式 字节码操作 cglib asm javassist
                 反射机制及其演进
                   java.lang.reflect 
                    AccessibleObject.setAccessible() 运行时修改成员访问限制
                    自定义高性能Nio 显示释放DirectBuffer 用反射绕开限制
                 动态代理
                    代理可以看作时对调用目标的一个封装  这样对目标代码的调用不是直接发生的，而是通过代理完成
                    通过代理可以让调用者和实现者之间解耦 比如RPC调用框架内部的寻址序列化 反序列化 通过代理可以提供更加友好的界面
                    静态代理 RMI  rmic生成stub    

                    实现对应的InvocationHandler  
                       以接口为纽带 为被调用目标构建代理对象 进而应用程序就可使用代理间接运行调用目标的逻辑 代理为应用程序插入逻辑提供便利的入口。

                   jdk proxy优势
                      最小化依赖关系  jdk本身支持 更加可靠；代码实现简单
                   cglib 
                      有时候调用目标可能不便于实现额外接口；
                      只操作我们关心的类，不必要其他相关类增加工作量
                      高性能   
                    Aop是对OOP的一个补充
                    OOP 对于跨越不同对象分类分散 逻辑表现力不够  ，比如在不同模块的特定阶段做一些事情 类似日志 用户鉴权 全局性异常处理 性能监控 事务处理
      --接口和抽象类的区别
              接口时对行为的抽象 它是抽象方法的集合 利用接口可以达到api定义和实现的分离；
              接口，不能实例化；不能包含任何非常量成员，任何filed都隐含着 public static final的意义；同时 没有非静态方法实现，也就是说要么抽象方法，要么静态方法；

                 抽象类是不能实例化的类 用abstract修饰，目的是代码重用，
                 除了不能实例化 形式上和一般的java类并没有太大区别，可以有一个或者多个抽象方法，也可以没有抽象方法
                 抽象类大多用于抽取相关java类的共同实现或者共同成员变量，然后通过代码继承达到代码复用的结果
                 Collection框架 abstractList
              
                 接口implments  抽象类extends  

                  不支持多继承 可以实现多个接口，但不能通过继承多个类来重用逻辑
                  任何接口添加抽象方法，相应的所有实现类必须实现这个接口否则会出现编译错误；抽象类不会出现
                  接口的职责不仅仅限于抽象方法的集合，有一类没有任何接口 通常叫做marker interface,目的就是为了申明某些东西
                  java 8functional interface 只有一个抽象方法的接口 Runnable callable
                  java 8 接口也可以有方法的实现  可以定义default method
                  collection 中增加lambda stream相关的功能  就是default method

                封装 继承 多态
                封装 隐藏内部实现细节 避免外部调用者接触到内部细节，简化操作界面
                继承 代码复用  紧耦合的关系 父类代码修改 子类行为也变动
                多态  重写父子类中相同名字和参数的方法 不同的实现；重载时相同的名字 但是不同的参数
                    方法名称和参数一致 但是返回值不一样 编译不通过
                SOLID设计原则
                   单一职责  类或对象最好只有单一职责
                   开关原则  对扩展开放封对修改关闭   设计保证平滑的扩展性尽量避免新增功能修改已有实现
                   里式替换   用到继承关系时  可以用父类或者基类的地方 都可以用子类替换
                   接口分离   
                   依赖反转  实体应该依赖于抽象而不是实现  高层次模块 不应该依赖于低层次模块 而是基于抽象。
                OOP
                   JAVA 10 中引入了本地方法推断和var 类型  
      - 泛型、泛型继承、泛型擦除
         java的泛型可以说是假泛型  完全是一种编译器技巧  编译器会自动将类型转换成对应的特定类型 使用泛型必须保证转换成Object
      - jdk ServiceLoader
      - 关于精度损失问题：int、long 超过最大值
      - 关于注解：元注解的种类、继承java.lang.Annotation、注解的基础类型、注解的常用方法      
      -- 关于ClassLoader，类加载器，双亲委派模型  
          概述
           一般来说，我们把 Java 的类加载过程分为三个主要步骤：加载、链接、初始化
           首先是加载阶段（Loading），它是 Java 将字节码数据从不同的数据源读取到 JVM 中，并映射为 JVM 认可的数据结构（Class 对象），这里的数据源可能是各种各样的形态，如 jar 文件、class 文件，甚至是网络数据源等；如果输入数据不是 ClassFile 的结构，则会抛出 ClassFormatError
           加载阶段是用户参与的阶段，我们可以自定义类加载器，去实现自己的类加载过程
           第二阶段是链接（Linking），这是核心的步骤，简单说是把原始的类定义信息平滑地转化入 JVM 运行的过程中。这里可进一步细分为三个步骤
           验证（Verification），这是虚拟机安全的重要保障，JVM 需要核验字节信息是符合 Java 虚拟机规范的，否则就被认为是 VerifyError，这样就防止了恶意信息或者不合规的信息危害 JVM 的运行，验证阶段有可能触发更多 class 的加载。
           准备（Preparation），创建类或接口中的静态变量，并初始化静态变量的初始值。但这里的“初始化”和下面的显式初始化阶段是有区别的，侧重点在于分配所需要的内存空间，不会去执行更进一步的 JVM 指令。
           解析（Resolution），在这一步会将常量池中的符号引用（symbolic reference）替换为直接引用。在Java 虚拟机规范中，详细介绍了类、接口、方法和字段等各个方面的解析。
           最后是初始化阶段（initialization），这一步真正去执行类初始化的代码逻辑，包括静态字段赋值的动作，以及执行类定义中的静态初始化块内的逻辑，编译器在编译阶段就会把这部分逻辑整理好，父类型的初始化逻辑优先于当前类型的逻辑


           再来谈谈双亲委派模型，简单说就是当类加载器（Class-Loader）试图加载某个类型的时候，除非父加载器找不到相应类型，否则尽量将这个任务代  理给当前加载器的父加载器去做。使用委派模型的目的是避免重复加载 Java 类型

           普通原始类型静态变量和引用类型（即使是常量），是需要额外调用 putstatic 等 JVM 指令的，这些是在显式初始化阶段执行，而不是准备阶段调用
          扩展
             三种类加载器
               启动类加载器（Bootstrap Class-Loader），加载 jre/lib 下面的 jar 文件，如 rt.jar。它是个超级公民，即使是在开启了 Security Manager 的时候，JDK 仍赋予了它加载的程序 AllPermission。    
               # 指定新的bootclasspath，替换java.*包的内部实现java -Xbootclasspath: your_App # a意味着append，将指定目录添加到bootclasspath后面java -Xbootclasspath/a: your_App # p意味着prepend，将指定目录添加到bootclasspath前面java -Xbootclasspath/p: your_App
               我们一般可以使用下面方法获取父加载器，但是在通常的 JDK/JRE 实现中，扩展类加载器 getParent() 都只能返回 null。public final   ClassLoader getParent()
               扩展类加载器（Extension or Ext Class-Loader），负责加载我们放到 jre/lib/ext/ 目录下面的 jar 包，这就是所谓的 extension 机制。该目录也可以通过设置 “java.ext.dirs”来覆盖。java -Djava.ext.dirs=your_ext_dir HelloWorld

               应用类加载器（Application or App Class-Loader），就是加载我们最熟悉的 classpath 的内容。这里有一个容易混淆的概念，系统（System）类加载器，通常来说，其默认就是 JDK 内建的应用类加载器，但是它同样是可能修改的，比如：java -Djava.system.class.loader=com.yourcorp.YourClassLoader HelloWorld

               如果我们指定了这个参数，JDK 内建的应用类加载器就会成为定制加载器的父亲，这种方式通常用在类似需要改变双亲委派模式的场景

             通常类加载机制有三个基本特征
               双亲委派模型。但不是所有类加载都遵守这个模型，有的时候，启动类加载器所加载的类型，是可能要加载用户代码的，比如 JDK 内部的 ServiceProvider/ServiceLoader机制，用户可以在标准 API 框架上，提供自己的实现，JDK 也需要提供些默认的参考实现。 例如，Java 中 JNDI、JDBC、文件系统、Cipher 等很多方面，都是利用的这种机制，这种情况就不会用双亲委派模型去加载，而是利用所谓的上下文加载器

               可见性，子类加载器可以访问父加载器加载的类型，但是反过来是不允许的，不然，因为缺少必要的隔离，我们就没有办法利用类加载器去实现容器的逻辑

               单一性，由于父加载器的类型对于子加载器是可见的，所以父加载器中加载过的类型，就不会在子加载器中重复加载。但是注意，类加载器“邻居”间，同一类型仍然可以被加载多次，因为互相并不可见
             java9变化
               前面提到的 -Xbootclasspath 参数不可用了。API 已经被划分到具体的模块，所以上文中，利用“-Xbootclasspath/p”替换某个 Java 核心类型代码，实际上变成了对相应的模块进行的修补，可以采用下面的解决方案：首先，确认要修改的类文件已经编译好，并按照对应模块（假设是 java.base）结构存放， 然后，给模块打补丁：java --patch-module java.base=your_patch yourApp
               由于 Jigsaw 项目引入了 Java 平台模块化系统（JPMS
               扩展类加载器被重命名为平台类加载器（Platform Class-Loader），而且 extension 机制则被移除。也就意味着，如果我们指定 java.ext.dirs 环境变量，或者 lib/ext 目录存在，JVM 将直接返回错误！建议解决办法就是将其放入 classpath 里  
               部分不需要 AllPermission 的 Java 基础模块，被降级到平台类加载器中，相应的权限也被更精细粒度地限制起来。
               rt.jar 和 tools.jar 同样是被移除了！JDK 的核心类库以及相关资源，被存储在 jimage 文件中，并通过新的 JRT 文件系统访问，而不是原有的 JAR 文件系统。虽然看起来很惊人，但幸好对于大部分软件的兼容性影响，其实是有限的，更直接地影响是 IDE 等软件，通常只要升级到新版本就可以了。

               增加了 Layer 的抽象， JVM 启动默认创建 BootLayer，开发者也可以自己去定义和实例化 Layer，可以更加方便的实现类似容器一般的逻辑抽象
               结合了 Layer，目前的 JVM 内部结构就变成了下面的层次，内建类加载器都在 BootLayer 中，其他 Layer 内部有自定义的类加载器，不同版本模块可以同时工作在不同的 Layer
             自定义类加载器
               实现类似进程内隔离，类加载器实际上用作不同的命名空间，以提供类似容器、模块化的效果。例如，两个模块依赖于某个类库的不同版本，如果分别被不同的容器加载，就可以互不干扰
               应用需要从不同的数据源获取类定义信息，例如网络数据源，而不是本地文件系统。
               或者是需要自己操纵字节码，动态修改或者生成类型    
               解自定义类加载过程
               通过指定名称，找到其二进制实现，这里往往就是自定义类加载器会“定制”的部分，例如，在特定数据源根据名字获取字节码，或者修改或生成字节码。
               然后，创建 Class 对象，并完成类加载过程。二进制信息到 Class 对象的转换，通常就依赖defineClass，我们无需自己实现，它是 final 方法。有了 Class 对象，后续完成加载过程就顺理成章了  
             就提到了由于字节码是平台无关抽象，而不是机器码，所以 Java 需要类加载和解释、编译，这些都导致 Java 启动变慢。谈了这么多类加载，有没有什么通用办法，不需要代码和其他工作量，就可以降低类加载的开销呢？  
      -- 有哪些方法可以在运行时动态生成一个Java类？
           概述
             我们可以从常见的 Java 类来源分析，通常的开发过程是，开发者编写 Java 代码，调用 javac 编译成 class 文件，然后通过类加载机制载入 JVM，就成为应用运行时可以使用的 Java 类了。
             从上面过程得到启发，其中一个直接的方式是从源码入手，可以利用 Java 程序生成一段源码，然后保存到文件等，下面就只需要解决编译问题了。有一种笨办法，直接用 ProcessBuilder 之类启动 javac 进程，并指定上面生成的文件作为输入，进行编译。最后，再利用类加载器，在运行时加载即可

             你可以考虑使用 Java Compiler API，这是 JDK 提供的标准 API，里面提供了与 javac 对等的编译器功能，具体请参考java.compiler相关文档。进一步思考，我们一直围绕 Java 源码编译成为 JVM 可以理解的字节码，换句话说，只要是符合 JVM 规范的字节码，不管它是如何生成的，是不是都可以被 JVM 加载呢？我们能不能直接生成相应的字节码，然后交给类加载器去加载呢

             当然也可以，不过直接去写字节码难度太大，通常我们可以利用 Java 字节码操纵工具和类库来实现，比如在专栏第 6 讲中提到的ASM、Javassist、cglib 等。

             重新审视我们谈到的动态代理，本质上不就是在特定的时机，去修改已有类型实现，或者创建新的类型。

           扩展
             类从字节码到 Class 对象的转换，在类加载过程中，这一步是通过下面的方法提供的功能，或者 defineClass 的其他本地对等实现
             protected final Class<?> defineClass(String name, byte[] b, int off, int len,                                   ProtectionDomain protectionDomain)protected final Class<?> defineClass(String name, java.nio.ByteBuffer b,                                   ProtectionDomain protectionDomain)
             可以看出，只要能够生成出规范的字节码，不管是作为 byte 数组的形式，还是放到 ByteBuffer 里，都可以平滑地完成字节码到 Java 对象的转换过程。
             JDK 提供的 defineClass 方法，最终都是本地代码实现的。
             static native Class<?> defineClass1(ClassLoader loader, String name, byte[] b, int off, int len,                                  ProtectionDomain pd, String source);

             更进一步，我们来看看 JDK dynamic proxy 的实现代码。你会发现，对应逻辑是实现在 ProxyBuilder 这个静态内部类中，ProxyGenerator 生成字节码，并以 byte 数组的形式保存，然后通过调用 Unsafe 提供的 defineClass 入口。

             byte[] proxyClassFile = ProxyGenerator.generateProxyClass(      proxyName, interfaces.toArray(EMPTY_CLASS_ARRAY), accessFlags);try {  Class<?> pc = UNSAFE.defineClass(proxyName, proxyClassFile,                                   0, proxyClassFile.length,                                   loader, null);  reverseProxyCache.sub(pc).putIfAbsent(loader, Boolean.TRUE);  return pc;} catch (ClassFormatError e) {// 如果出现ClassFormatError，很可能是输入参数有问题，比如，ProxyGenerator有bug}

             前面理顺了二进制的字节码信息到 Class 对象的转换过程，似乎我们还没有分析如何生成自己需要的字节码，接下来一起来看看相关的字节码操纵逻辑
             JDK 内部动态代理的逻辑，可以参考java.lang.reflect.ProxyGenerator的内部实现。我觉得可以认为这是种另类的字节码操纵技术，其利用了DataOutputStrem提供的能力，配合 hard-coded 的各种 JVM 指令实现方法，生成所需的字节码数组

             这种实现方式的好处是没有太多依赖关系，简单实用，但是前提是你需要懂各种JVM 指令，知道怎么处理那些偏移地址等，实际门槛非常高，所以并不适合大多数的普通开发场景。
             幸好，Java 社区专家提供了各种从底层到更高抽象水平的字节码操作类库，我们不需要什么都自己从头做。JDK 内部就集成了 ASM 类库，虽然并未作为公共 API 暴露出来，但是它广泛应用在，如java.lang.instrumentation API 底层实现，或者Lambda Call Site生成的内部逻辑中，这些代码的实现我就不在这里展开了，

             从相对实用的角度思考一下，实现一个简单的动态代理，都要做什么？如何使用字节码操纵技术，走通这个过程呢？

             对于一个普通的 Java 动态代理，其实现过程可以简化成为：提供一个基础的接口，作为被调用类型（com.mycorp.HelloImpl）和代理类之间的统一入口，如 com.mycorp.Hello。
             实现InvocationHandler，对代理对象方法的调用，会被分派到其 invoke 方法来真正实现动作。
             通过 Proxy 类，调用其 newProxyInstance 方法，生成一个实现了相应基础接口的代理类实例，可以看下面的方法签名。

             public static Object newProxyInstance(ClassLoader loader,                                    Class<?>[] interfaces,                                    InvocationHandler h)

             我们分析一下，动态代码生成是具体发生在什么阶段呢？不错，就是在 newProxyInstance 生成代理类实例的时候

             字节码操纵技术，除了动态代理，还可以应用在什么地方？
               这个技术似乎离我们日常开发遥远，但其实已经深入到各个方面，也许很多你现在正在使用的框架、工具就应用该技术，下面是我能想到的几个常见领域。各种 Mock 框架ORM 框架IOC 容器部分 Profiler 工具，或者运行时诊断工具等生成形式化代码的工具

      -- string long 源码解析和面试题
       -- string 
         -不变性  
            hashMap的key建议使用不可变类，比如string这种不可变类；不可变是指类的值一旦被初始化，就不能被改变了，如果被改变，将会是新的类 
            String s="hello";           s="world";           把s的引用指向了新的内存地址  
           源码分析
            String 被final修饰 说明该类不可能你被继承，任何对string操作的方法，都不会被继承覆写
            string中保存的数据是一个char类型的数组value，数组也是被final修饰的，value的权限是private，外部访问不到，也没开放对value进行复制的方法，所以value一旦产生，内存地址根本无法被修改。
         -字符串乱码
            二进制转换操作，没有强制规定文件编码，不同的环境默认的文件编码不一致导致的。
            解决办法 所有使用字符串的地方都用到 utf-8 newString getBytes
         -首字母大小写   
            项目被spring托管，有时候会通过applicationContext.getBean(className); className必须满足首字母小写
            substring()底层使用Arrays.copyOfRange()
         -相等判断
            equals 和 equalsIgnoreCase。后者判断相等时，会忽略大小写    
            equals源码分析
              判断内存地址是否相同
              待比较的对象是否是string（anObject instanceof String）,如果不是string,直接返回
              两个字符串的长度是否相等，不相等直接返回
              一次比较每个字符是否相等，若有一个不相等，直接返回不相等；
          -替换删除
              replace 替换所有字符、replaceAll 批量替换字符串、replaceFirst 替换遇到的第一个字符串三种场景。
          -拆分和合并
              split("分隔符",数值) 
               Guava工具类Splitter.on(',').trimResults()去掉空格。omitEmptyStrings()//去掉空值.splitToList(string)    
              join
                Guava工具类Joinner.on(',').skkipNulls 
        --Long
           -缓存        
            long自己实现缓存机制，缓存到-128-127内所以的long值，如果是这个范围的，就不会初始化，直接从缓存中读取。
            源码解析
             // 缓存，范围从 -128 到 127，+1 是因为有个 0
              static final Long cache[] = new Long[-(-128) + 127 + 1];
              static块  容器初始化 进行加载
            -使用long时，推荐使用valueOf 少使用parseLong方法
               本身缓存机制，缓存-128到127范围内的值，减少开销，parseLong没有这个机制  
      -- Arrays、Collections、Objects常用方法       
         -工具类通用特征
           构造器私有
           工具类的工具方法必须被static final关键字修饰  
           尽量不在工具类方法中，对共享变量做有修改的操作访问（如果必须要做，必须加锁）
         -Arrays
           排序 
             Arrays.sort 入参支持int long double 各种基本数据类型；要排序的数组，外部的排序器；
             list.toArray(array);  Arrays.sort(array, Comparator.comparing(SortDTO::getSortTarget));
           二分查找法  
           Arrays.binarySearch(array, new SortDTO("200"),
                    Comparator.comparing(SortDTO::getSortTarget));
               使用注意，如果被搜索的数组是无序的，一定要先排序，否则二分搜索很可能搜索不到，    
             源代码实现
                // a：我们要搜索的数组，fromIndex：从那里开始搜索，默认是0； toIndex：搜索到何时停止，默认是数组大小
             // key：我们需要搜索的值 c：外部比较器
             private static <T> int binarySearch0(T[] a, int fromIndex, int toIndex,
                                     T key, Comparator<? super T> c)
              如果比较器c是空的 直接使用key的compareTo进行排序
              开始位置小于结束为止 一直循环假设low=0 high=10 mid=5
              比较中间值和给定值得大小关系
                如果数组中间值小于给定值，说明要找投入值在中间值的右边
                low=mid+1
                反之，在中间值得左边
                high=mid-1
                其他情况 找到了
           拷贝  
             add(扩容)  remove（删除元素不是最后一个）都会进行拷贝
             copyOf 拷贝整个数组  copyOfRange拷贝部分
             源码实现
              copyOfRange
               计算需要拷贝的长度
               初始化新数组
               调用native方法System.arraycopy 参数被拷贝的数组 从数组哪里开始 目标数组 从目的数组哪里开始拷贝 拷贝的长度

          -collections     
              求集合中的最大值 最小值
                max提供两种类型的方法，一个需要传外部排序器，一个不需要传排序器，但是需要集合中的元素强制实现Comparable接口
                max方法泛型定义 必须继承Object并且实现Comparable接口；因为max底层使用的Comparable的compareTo方法进行排序
              多种类型的集合
               线程安全的集合
                集合方法都是synchronized打头的 而且底层所有的方法都打上了synchronized
               不可变的集合 
                 unmodifiable开头  会从原集合中得到一个不可变的新集合，新集合只能访问，无法修改；一旦修改，就会抛出异常
          -Objects        
             相等判断
              Objects 有提供 equals 和 deepEquals 两个方法来进行相等判断，前者是判断基本类型和自定义类的，后者是用来判断数组的

             判断是否为空
              isNull 和 nonNull 对于对象是否为空返回 Boolean 值，requireNonNull 方法更加严格，如果一旦为空，会直接抛出异常
          -引申
             ArrayList 初始化之后，不能被修改，该怎么办
              可以使用 Collections 的 unmodifiableList 的方法，该方法会返回一个不能被修改的内部类集合，这些集合类只开放查询的方法，对于调用修改集合的方法会直接抛出异常    
      -- Lambda 源码            
             无法通过idea查看源码和其他底层结构
             异常判断法
               我们可以在代码执行中主动抛出异常，打印出堆栈，堆栈会说明其运行轨迹，一般这种方法简单高效，基本上可以看到很多情况下的隐藏代码，我们来试一下

               从异常的堆栈中，我们可以看到 JVM 自动给当前类建立了内部类（错误堆栈中出现多次的 $ 表示有内部类），内部类的代码在执行过程中，抛出了异常，但这里显示的代码是 Unknown Source，所以我们也无法 debug 进去，一般情况下，异常都能暴露出代码执行的路径，我们可以打好断点后再次运行，但对于 Lambda 表达式而言，通过异常判断法我们只清楚有内部类，但无法看到内部类中的源码
              javap命令法
                javap -verbose Lambda.class
                汇编指令中我们很容易找到 Constant pool 打头的一长串类型，我们叫做常量池，官方英文叫做 Run-Time Constant Pool，我们简单理解成一个装满常量的 table ，table 中包含编译时明确的数字和文字，类、方法和字段的类型信息等等。table 中的每个元素叫做 cpinfo，cpinfo 由唯一标识 ( tag ) + 名称组成，

                InvokeDynamic 表示动态的调用方法，后面我们会详细说明；
                Fieldref 表示字段的描述信息，如字段的名称、类型；
                NameAndType 是对字段和方法类型的描述；
                MethodHandle 方法句柄，动态调用方法的统称，在编译时我们不知道具体是那个方法，但运行时肯定会知道调用的是那个方法；
                MethodType 动态方法类型，只有在动态运行时才会知道其方法类型是什么。

                MethodHandles 和 LambdaMetafactory 都是 java.lang.invoke 包下面的重要方法，invoke 包主要实现了动态语言的功能，我们知道 java 语言属于静态编译语言，在编译的时候，类、方法、字段等等的类型都已经确定了，而 invoke 实现的是一种动态语言，也就是说编译的时候并不知道类、方法、字段是什么类型，只有到运行的时候才知道


                

                比如这行代码：Runnable runnable = () -> System.out.println(“lambda is run”); 在编译器编译的时候 () 这个括号编译器并不知道是干什么的，只有在运行的时候，才会知道原来这代表着的是 Runnable.run() 方法。invoke 包里面很多类，都是为了代表这些 () 的，我们称作为方法句柄（ MethodHandler ），在编译的时候，编译器只知道这里是个方法句柄，并不知道实际上执行什么方法，只有在执行的时候才知道，那么问题来了，JVM 执行的时候，是如何知道 () 这个方法句柄，实际上是执行 Runnable.run() 方法的呢？

                metafactory 方法入参 caller 代表实际发生动态调用的位置，invokedName 表示调用方法名称，invokedType 表示调用的多个入参和出参，samMethodType 表示具体的实现者的参数，implMethod 表示实际上的实现者，instantiatedMethodType 等同于 implMethod。

                1：从汇编指令的 simple 方法中，我们可以看到会执行 Runnable.run 方法；

                2：在实际的运行时，JVM 碰到 simple 方法的 invokedynamic 指令，会动态调用 LambdaMetafactory.metafactory 方法，执行具体的 Runnable.run 方法。

                所以可以把 Lambda 表达值的具体执行归功于 invokedynamic JVM 指令，正是因为这个指令，才可以做到虽然编译时不知道要干啥，但动态运行时却能找到具体要执行的代
      -- 常用的 Lambda 表达式使用场景解析和应用     
         数据准备
         常用方法
          filter  predicate test()
            // list 在下图中进行了初始化
             List<String> newList = list.stream()
            // 过滤掉我们希望留下来的值
            // StringUtils.equals(str,"hello") 表示我们希望字符串是 hello 能留下来
            // 其他的过滤掉
           .filter(str -> StringUtils.equals(str, "hello"))
            // Collectors.toList() 帮助我们构造最后的返回结果
            .collect(Collectors.toList());
          map  function<t,r>
             map 方法可以让我们进行一些流的转化，比如原来流中的元素是 A，通过 map 操作，可以使返回的流中的元素是 B
             List<String> codes = students.stream()
             //StudentDTO::getCode 是 s->s.getCode() 的简写
             .map(StudentDTO::getCode)
             .collect(Collectors.toList()); 
             TestMap 所有学生的学号为 ["W199","W25","W3","W1"]
          mapToInt
             mapToInt 方法的功能和 map 方法一样，只不过 mapToInt 返回的结果已经没有泛型，已经明确是 int 类型的流了
              students.stream()
             .mapToInt(s->Integer.valueOf(s.getId()+""))
              // 一定要有 mapToObj，因为 mapToInt 返回的是 IntStream，因为已经确定是 int 类型了
              // 所有没有泛型的，而 Collectors.toList() 强制要求有泛型的流，所以需要使用 mapToObj
              // 方法返回有泛型的流
               .mapToObj(s->s)
              .collect(Collectors.toList());
                // 计算学生总分
              Double sumScope = students.stream()
                .mapToDouble(s->s.getScope())
               // DoubleStream/IntStream 有许多 sum（求和）、min（求最小值）、max（求最大值）、average（求平均值）等方法
               .sum();
          flatMap
            flatMap 方法也是可以做一些流的转化，和 map 方法不同的是，其明确了 Function 函数的返回值的泛型是流，源码如下：      
            // 计算学生所有的学习课程，flatMap 返回 List<课程> 格式
           List<Course> courses = students.stream().flatMap(s->s.getLearningCources().stream())
             .collect(Collectors.toList());
          distinct 
             List<String> names = students.stream()
             .map(StudentDTO::getName)
             .distinct()
             .collect(Collectors.toList());   
          Sorted   
            // 直接连起来写
            List<String> codes = students.stream()
              .map(StudentDTO::getCode)
              // 等同于 .sorted(Comparator.naturalOrder()) 自然排序
             .sorted()
             .collect(Collectors.toList());
             // 自定义排序器
            List<String> codes2 = students.stream()
              .map(StudentDTO::getCode)
             // 反自然排序
             .sorted(Comparator.reverseOrder())
            .collect(Collectors.toList());
             log.info("TestSorted 反自然排序 is {}",JSON.toJSONString(codes2))
          peek    
            students.stream().map(StudentDTO::getCode)
           .peek(s -> log.info("当前循环的学号是{}",s))
            .collect(Collectors.toList())
          limit 
             limit 方法会限制输出值个数，入参是限制的个数大小    
                 List<String> codes = students.stream()
                 .map(StudentDTO::getCode)
                 .limit(2L)
                 .collect(Collectors.toList())
          reduce        
             Double sum1 = students.stream()
             .map(StudentDTO::getScope)
             // 第一个参数表示成绩的基数，会从 100 开始加
              .reduce(100D,(scope1,scope2) -> scope1+scope2)
          findFirst
            findFirst 表示匹配到第一个满足条件的值就返回     
             Long id = students.stream()
             .filter(s->StringUtils.equals(s.getName(),"小美"))
             // 同学中有两个叫小美的，这里匹配到第一个就返回
              .findFirst()
             .get().getId();

              // 防止空指针
             Long id2 = students.stream()
             .filter(s->StringUtils.equals(s.getName(),"小天"))
             .findFirst()
             // orElse 表示如果 findFirst 返回 null 的话，就返回 orElse 里的内容
             .orElse(new StudentDTO()).getId();

               Optional<StudentDTO> student= students.stream()
              .filter(s->StringUtils.equals(s.getName(),"小天"))
               .findFirst();

                if(student.isPresent()){
                   log.info("testFindFirst 小天同学的 ID {}",student.get().getId());
                 return;
               
          groupingBy && toMap   
            groupingBy 是能够根据字段进行分组，toMap 是把 List 的数据格式转化成 Map 的格式  
              // 学生根据名字进行分类
            Map<String, List<StudentDTO>> map1 = students.stream()
               .collect(Collectors.groupingBy(StudentDTO::getName));
              Map<String, Set<String>> map2 = students.stream()
           .collect(Collectors.groupingBy(StudentDTO::getName,
                                  Collectors.mapping(StudentDTO::getCode,Collectors.toSet())));   
             // 学生转化成学号为 key 的 map
            Map<String, StudentDTO> map3 = students.stream()
           //第一个入参表示 map 中 key 的取值
           //第二个入参表示 map 中 value 的取值
           //第三个入参表示，如果前后的 key 是相同的，是覆盖还是不覆盖，(s1,s2)->s1 表示不覆盖，(s1,s2)->s2 表示覆盖
            .collect(Collectors.toMap(s->s.getCode(),s->s,(s1,s2)->s1));                      
