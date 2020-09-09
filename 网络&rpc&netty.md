
## Socket 源码及面试题
        1 Socket 整体结构
           Socket 的结构非常简单，Socket 就像一个壳一样，将套接字初始化、创建连接等各种操作包装了一下，其底层实现都是 SocketImpl 实现的，Socket 本身的业务逻辑非常简单。
 
            套接字的状态变更都是有对应操作方法的，比如套接字新建（createImpl 方法）后，状态就会更改成 created = true，连接（connect）之后，状态更改成 connected = true
        2 初始化
            指定代理类型（Proxy）创建套节点，一共有三种类型为：DIRECT（直连）、HTTP（HTTP、FTP 高级协议的代理）、SOCKS（SOCKS 代理），三种不同的代码方式对应的 SocketImpl 不同，分别是：PlainSocketImpl、HttpConnectSocketImpl、SocksSocketImpl，除了类型之外 Proxy 还指定了地址和端口；

            默认 SocksSocketImpl 创建，并且需要在构造器中传入地址和端口

            底层实现
              // stream 为 true 时，表示为stream socket 流套接字，使用 TCP 协议，比较稳定可靠，但占用资源多
              // stream 为 false 时，表示为datagram socket 数据报套接字，使用 UDP 协议，不稳定，但占用资源少
              private Socket(SocketAddress address, SocketAddress localAddr,
               boolean stream) throws IOException {}
                // 创建 socket
               createImpl(stream);
              // 如果 ip 地址不为空，绑定地址
               if (localAddr != null)
               // create、bind、connect 也是 native 方法
                bind(localAddr);
                connect(address);

               在构造 Socket 的时候，你可以选择 TCP 或 UDP，默认是 TCP；
               如果构造 Socket 时，传入地址和端口，那么在构造的时候，就会尝试在此地址和端口上创建套接字；
               Socket 的无参构造器只会初始化 SocksSocketImpl，并不会和当前地址端口绑定，需要我们手动的调用 connect 方法，才能使用当前地址和端口；
               Socket 我们可以理解成网络沟通的语言层次的抽象，底层网络创建、连接和关闭，仍然是 TCP 或 UDP 本身网络协议指定的标准，Socket 只是使用 Java 语言做了一层封装，从而让我们更方便地使用。 
        3 connect 连接服务端
             connect 方法主要用于 Socket 客户端连接上服务端，如果底层是 TCP 层协议的话，就是通过三次握手和服务端建立连接，为客户端和服务端之间的通信做好准备，
              public void connect(SocketAddress endpoint, int timeout) throws IOException {

             connect 方法要求有两个入参，第一个入参是 SocketAddress，表示服务端的地址，我们可以使用 InetSocketAddress 进行初始化，比如：new InetSocketAddress (“www.wenhe.com”, 2000)。

             第二入参是超时时间的意思（单位毫秒），表示客户端连接服务端的最大等待时间，如果超过当前等待时间，仍然没有成功建立连接，抛 SocketTimeoutException 异常，如果是 0 的话，表示无限等待。
        4 常用参数设置
           setTcpNoDelay
             此方法是用来设置 TCP_NODELAY 属性的，属性的注释是这样的：此设置仅仅对 TCP 生效，主要为了禁止使用 Nagle 算法，true 表示禁止使用，false 表示使用，默认是 false。
             总结算法开启关闭的场景：
              如果 Nagle 算法关闭，对于小数据包，比如一次鼠标移动，点击，客户端都会立马和服务端交互，实时响应度非常高，但频繁的通信却很占用不少网络资源；
             如果 Nagle 算法开启，算法会自动合并小数据包，等到达到一定大小（MSS）后，才会和服务端交互，优点是减少了通信次数，缺点是实时响应度会低一些。
             Socket 创建时，默认是开启 Nagle 算法的，可以根据实时性要求来选择是否关闭 Nagle 算法。 
           setSoLinger

             在我们调用 close 方法时，默认是直接返回的，但如果给 SO_LINGER 赋值，就会阻塞 close 方法，在 SO_LINGER 时间内，等待通信双方发送数据，如果时间过了，还未结束，将发送 TCP RST 强制关闭 TCP 。  
           setOOBInline

              如果希望接受 TCP urgent data（TCP 紧急数据）的话，可以开启该选项，默认该选项是关闭的，我们可以通过 Socket#sendUrgentData 方法来发送紧急数据。 
           setSoTimeout
              setSoTimeout 方法主要是用来设置 SO_TIMEOUT 属性的。
              注释上说：用来设置阻塞操作的超时时间，阻塞操作主要有：
                 ServerSocket.accept () 服务器等待客户端的连接；
                 SocketInputStream.read () 客户端或服务端读取输入超时；
                 DatagramSocket.receive()。
               我们必须在必须在阻塞操作之前设置该选项， 如果时间到了，操作仍然在阻塞，会抛出 InterruptedIOException 异常（Socket 会抛出 SocketTimeoutException 异常，不同的套接字抛出的异常可能不同）。 
               对于 Socket 来说，超时时间如果设置成 0，表示没有超时时间，阻塞时会无限等待
           setSendBufferSize
             setSendBufferSize 方法主要用于设置 SO_SNDBUF 属性的，入参是 int 类型，表示设置发送端（输出端）的缓冲区的大小，单位是字节。
             入参 size 必须大于 0，否则会抛出 IllegalArgumentException 异常。
             一般我们都是采取默认的，如果值设置太小，很有可能导致网络交互过于频繁，如果值设置太大，那么交互变少，实时性就会变低。
           setReceiveBufferSize
              setReceiveBufferSize 方法主要用来设置 SO_RCVBUF 属性的，入参是 int 类型，表示设置接收端的缓冲区的大小，单位是字节。
              入参 size 必须大于 0，否则会抛出 IllegalArgumentException 异常。
              一般来说，在套接字建立连接之后，我们可以随意修改窗口大小，但是当窗口大小大于 64k 时，需要注意：
                 必须在 Socket 连接客户端之前设置缓冲值；
                 必须在 ServerSocket 绑定本地地址之前设置缓冲值。
           setKeepAlive
              setKeepAlive 方法主要用来设置 SO_KEEPALIVE 属性，主要是用来探测服务端的套接字是否还是存活状态，默认设置是 false，不会触发这个功能。
              如果 SO_KEEPALIVE 开启的话，TCP 自动触发功能：如果两小时内，客户端和服务端的套接字之间没有任何通信，TCP 会自动发送 keepalive 探测给对方，对方必须响应这个探测（假设是客户端发送给服务端），预测有三种情况：
                  服务端使用预期的 ACK 回复，说明一切正常；
                  服务端回复 RST，表示服务端处于死机或者重启状态，终止连接；
                   没有得到服务端的响应（会尝试多次），表示套接字已经关闭了。     
           setReuseAddress
               setReuseAddress 方法主要用来设置 SO_REUSEADDR 属性，入参是布尔值，默认是 false。
                套接字在关闭之后，会等待一段时间之后才会真正的关闭，如果此时有新的套接字前来绑定同样的地址和端口时，如果 setReuseAddress 为 true 的话，就可以绑定成功，否则绑定失败。      
## ServerSocket 源码及面试题   
        类属性
          ServerSocket 的主要作用，是作为服务端的套接字，接受客户端套接字传递过来的信息，并把响应回传给客户端 
          ServerSocket 和 Socket 一样，底层都是依靠 SocketImpl 的能力，而 SocketImpl 底层能力的实现基本上都是 native 方法实现的。
        初始化 
           无参构造器做的事情比较简单，只指定了 SocketImpl 为 SocksSocketImpl 类；
           有参构造器有几种初始化的形式，我们一起来看一下参数最多的构造器的源码。  
             ServerSocket(int port, int backlog, InetAddress bindAddr)
             入参 port 指的是 ServerSocket 需要绑定本地那个端口。
             入参 backlog 指的是服务端接受客户端连接队列的最大长度
             入参 InetAddress 表示 ip 地址。
        bind 
          bind 方法主要作用是把 ServerSocket 绑定到本地的端口上，只有当我们使用无参构造器初始化 ServerSocket 时，才会用到这个方法，如果使用有参构造器的话，在初始化时就已经绑定到本地的端口上了。
           // 进行初始化
           ServerSocket serverSocket = new ServerSocket();
           // 进行绑定
            serverSocket.bind(new InetSocketAddress("localhost", 7007));
        accept

          accept 方法主要是用来 ServerSocket 接受来自客户端的套接字的，如果此时没有来自客户端的请求时，该方法就会一直阻塞，如果有通过 setSoTimeout 方法设置超时时间，那么 accept 只会在超时间内阻塞，过了超时时间就会抛出异常。

        说说你对 Socket 和 ServerSocket 的理解？
           两者我们都可以称为套接字，底层基于 TCP/UDP 协议，套接字对底层协议进行了封装，让我们使用时更加方便，Socket 常被使用在客户端，用于向服务端请求数据和接受响应，ServerSocket 常用于在服务端，用于接受客户端的请求并进行处理，两者其底层使用都是依靠 SocketImpl 的子类的 native 方法。
        说说对 SocketOptions 中的 SO_TIMEOUT 的理解？
           SocketOptions 类有很多属性设置，比如 SO_TIMEOUT 、SO_LINGER 等等，这些问题说一下自己的理解即可  
         在构造 Socket 的时候，我可以选择 TCP 或 UDP 么？应该如何选择？
            可以的，Socket 有三个参数的构造器，第三个参数表示你想使用 TCP 还是 UDP。   
         TCP 有自动检测服务端是否存活的机制么？有没有更好的办法？    
            我们可以通过 setKeepAlive 方法来激活该功能，如果两小时内，客户端和服务端的套接字之间没有任何通信，TCP 会自动发送 keepalive 探测给服务端，预测服务端有三种情况：
               服务端使用预期的 ACK 回复，说明一切正常；
               服务端回复 RST，表示服务端处于死机或者重启状态，终止连接；
               没有得到服务端的响应（会尝试多次），表示套接字已经关闭了。
            但我们并不建议使用这种方式，我们可以自己起一个定时任务，定时的访问服务端的特殊接口，如果服务端返回的数据和预期一致，说明服务端是存活的。 
## Socket结合线程池
        1 要求
            可以使用 Socket 和 ServiceSocket 以及其它 API；
            写一个客户端和服务端之间 TCP 通信的例子；
            服务端处理任务需要异步处理；
            因为服务端处理能力很弱，只能同时处理 5 个请求，当第六个请求到达服务器时，需要服务器返回明确的错误信息：服务器太忙了，请稍后重试~。

            我们应该使用 ServerSocket 的 backlog 的属性，把其设置成 5，但我们在上一章中说到 backlog 并不能准确代表限制的客户端连接数，而且我们还要求服务端返回具体的错误信息，即使 backlog 生效，也只会返回固定的错误信息，不是我们定制的错误信息。

            线程池似乎可以做这个事情，我们可以把线程池的 coreSize 和 maxSize 都设置成 4，把队列大小设置成 1，这样服务端每次收到请求后，会先判断一下线程池中的队列有没有数据，如果有的话，说明当前服务器已经马上就要处理第五个请求了，当前请求就是第六个请求，应该被拒绝。 
## IO 方式 Nio 如何实现多路复用
        java.io包  基于流模型实现 
          file抽象 输入输出流
          交互方式 是同步阻塞方式 在读取输入和输出流时，在读写动作完成之前线程一只会阻塞在那里 调用时线性顺序关系
          java.io简单直观 缺点io效率和扩展性存在局限性，容易造成性能瓶颈
        java.net包下 socket serversocket httpurlConnection 
        java.nio 1.4  提供channel select buffer  可以构建多路复用 同步阻塞Io程序 
         java 1.7 NIO2 AIO 异步非阻塞  基于事件和异步回调机制，简单理解为应用程序直接返回 而不会阻塞在那里 在后台完成，操作系统会通知对应线程完成后续工作。
         基础概念 
            区分同步和异步 同步是一种可靠有序的运行机制，当我们运行同步操作时，后续的任务是等待当前调用返回，才会进行下一步；
            异步相反，其他任务不需要等待当前调用返回，通常依靠事件、回调等机制实现任务的次序关系

            区分阻塞和非阻塞  在进行阻塞操作时，当前线程会处于阻塞状态无法从事其他任务，只有条件就绪才能继续
            比如ServerSocket新连接建立完毕 或者数据读取写入操作完成 ；非阻塞不管io操作是否结束，直接返回 相应操作在后台继续处理
           io 不仅时对文件操作 网络编程中 比如socket通信 都是典型的io操作
           输入流和输出流是用于读取或者写入字节的比如图片文件
           Reader和writer适用于操作字符增加了字符编解码功能
           BufferOutPutStream 带缓冲区的实现 避免频繁磁盘读写进而提高Io处理效率 这种数据利用了缓冲区 将批量数据进行一次操作 但在使用中千万别忘了flush
           Io 工具类都是先closeable接口  进行资源释放
             object
             closeable
                  file
                   RandomAccessFile
                   inputStream
                     filterInputstream - bufferedInputStream
                     byteArrayInputStrream
                   outPutstream
                     filterOutputstream->bufferedoutputStream
                     
                   reader
                   writer
             Java Nio
                buffer  除了布尔类型 都有对应buffer实现
                channel  类似于文件描述符 Nio中被用来支持批量IO操作的一种抽象
                file socket 更高层次的抽象 Channel更加底层操作系统的抽象的 
                 DMA 通过socket获取channel
                selector 是实现Nio多路复用的基础 可以检测到注册在selector上的多个channel 是否有channel 处于就绪状态
                 进而实现单线程对多个channel的高效管理 
                selector 同样依赖于底层操作系统
                  linux依赖于epoll windows 依赖于iocp
                charset 提供unicode字符串定义 Nio提供了相应的编解码器  
              Nio能解决什么问题
                   并发少的情况下 io 可以很好工作，并发多 连接数量急剧上升，线程切换会在高并发时开销很大
                    Nio步骤
                      通过selector.open创建一个selector作为类似调度员角色
                      创建一个ServerSocketChannel 并向selector注册 通过指定selectionkey.op_accept 告知调度员
                      关注新的连接请求
                       明确设置非阻塞模式 注册操作在阻塞模式下会抛出异常
                      selectot 阻塞在select操作当有channel发生连接请求时 会被唤醒
                       
                      Nio利用单线程轮询事件的机制高效定位就绪的channel，来决定 做什么 仅有select阶段是阻塞的 可以有效避免大量客户端连接时 频繁的线程切换操作 
## 文件拷贝方式 哪一种更高效

       java中比较典型的文件拷贝实现方式
         java.io fileInputStream  fileOutStream
         java.nio transferTo transferFrom
         nio 可能更快 利用现代操作系统底层机制 避免不必要的拷贝和上下文切换

       扩展：
         拷贝实现机制分析
            用户空间态和内核空间态 操作系统内核硬件驱动运行在内核空间态具有相对高的特权；用户空间则是给普通应用和服务使用的。

            当我们使用输入输出流进行读写时，实际上进行了多次上下文切换
            比如读取数据时，先在内核态将数据从磁盘读取到内核缓存，再切换到用户态将数据从内核缓存读取到用户缓存
            这种方式带来一定的额外开销可能会降低Io效率

            而基于nio transferto 在linux上则会使用零拷贝 数据传输不需要用户态参与省去了上下文切换的开销和不必要的内存拷贝
          java io /nio源码结构
            fielSystemprovider 只是一个抽象类 公共api通过serviceloader机制加载一系列文件系统实现，然后提供服务 
            copy方法不是利用transferto 而是本地用户态拷贝
             使用实践：
             使用缓存，减少io次数
             利用transfer机制 减少上下文切换和额外io操作
             减少不必要的转换 编解码序列化和反序列化
           Nio buffer
             基本属性
               capacity 数组长度
               position 操作的数据起始位置
               limit 操作的限额
               mark 记录上一次position位置
             基本操作
                创建一个byteBuffer 准备放入数据 capacity当然时缓冲区大小 position就是0 limit默认是capacity大小
                写入几个字节时 position会水涨船高 但是不可能超过limit
                当我们想把数据读出来时 需要调用flip方法 将position设置 为0 limit设置为以前的position那里
                从头在读一遍 调用rewind 
            Direct Buffer和垃圾收集
                allacate allacateDirect直接创建
                MappedByteBuffer 将文件按照大小直接映射在内存区域当程序访问时直接操作这块文件数据，省去了将数据从内核空间传向用户空间的消耗
                DirectBuffer 生命周期内内存地址都不会发生变更，进而安全进行访问
                减少堆内对象存储的可能额外维护工作 所以访问效率

               -XX:MaxDirectMemorySize=512M  
                如果出现内存不足，对外内存占用也是一种可能性
                不会主动回收directBuffer   cleaner和幻象引用 使用不当 出现oom

                回收建议
                   显示调用system.gc()强制出发
                   重复使用directBUffer
            跟踪和诊断对外内存占用  

                   -XX:NativeMemoryTracking={summary|detail}

           // 打印NMT信息
               jcmd <pid> VM.native_memory detail 

                jcmd <pid> VM.native_memory baseline

                jcmd <pid> VM.native_memory detail.diff
## 网络通信

        tcp拆包粘包机制
            5个字节  发了7个字节  粘包
            7个字节  发了三次      拆包

        产生原因：
           应用程序write写入的字节大小小于套接字发送缓冲区的大小（拆包）
           进行MSS大小的tcp分段 以太网帧的palyload大于MTU进行IP分片     
         netty 拆包粘包的解决方案
             消息定长处理 ，例如每个报文固定大小为200个，如果不够 空位补空格 
             在包尾部进行特殊字符进行分割  增加回车等；  
             （自定义协议栈）消息分为消息头和消息体 在消息头中包含消息总长的字段，然后进行业务处理
    
         java序列化
            没办法跨语言
            码流太大
            性能低
         Netty序列化
                jboss marshlling MessagePack框架
                     //责任链
                  sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingDecoder());
                  sc.pipeline().addLast(MarshallingCodeCFactory.buildMarshallingEncoder());

                GoogleProtobuf  基于Protobuf的kyro
                       //自定义协议栈
                ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
                ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
                        //解析client数据
                ch.pipeline().addLast(new ProtobufDecoder(RequestModule.Request.getDefaultInstance()));
                ch.pipeline().addLast(new ProtobufEncoder());
              

         数据可靠性通信场景分析和架构设计
             数据通信要求实时性高 且高性能 异构系统
             需要保障不同业务对应不同的实现
             支持负载均衡策略故障切换
             需要可靠性保障的支持 数据不允许丢失  


             netty-common  netty-client netty-server 三个项目
  
            Common包下
                proto文件夹定义.proto格式
                使用.bat文件生成数据文件到common.protobuf下
               封装数据格式
                定义数据  common.protobuf
              定义注解
                @Module @Cmd
              注解解析
                 NettyProcessBeanScanner类
                 在bean初始化后加载所有的bean 然后找到带有@Module @Ccmd的对象
                 创建 Invoker（反射调用类） 
                  加入到InvokerTable(map关系) 
                    ConcurrentHashMap<String/* module*/ ,Map<String/*cmd*/,Invoker>> invokerTable
                   

              以下两个开启异步线程池 不阻塞io主线程
                 client ->  ClientHandler
                
                 server ->  ServerHandler
                 ThreadPoolExecutor workerPool = new ThreadPoolExecutor(5,
            10,
            60L,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(4000),
            new ThreadPoolExecutor.DiscardPolicy());
           channelRead ->
             workerPool.submit(new MessageTask4Response(response,ctx));
              
           MessageTask4Request消息处理类
              invoke返回一个统一的消息类型 Result< T extends GeneratedMessageV3>

             创建请求和回送响应的一个数据封装类MessageBuilder
            
           
           haprroxy  长连接  4层 tcp   
               负载均衡  最小连接数策略
               keepalived 虚拟Ip
           nginx   短连接 7层 http            
## 高性能网络应用框架Netty   
         网络编程性能的瓶颈
           我们写过一个简单的网络程序 echo，采用的是阻塞式 I/O（BIO）。BIO 模型里，所有 read() 操作和 write() 操作都会阻塞当前线程的，如果客户端已经和服务端建立了一个连接，而迟迟不发送数据，那么服务端的 read() 操作会一直阻塞，所以使用 BIO 模型，一般都会为每个 socket 分配一个独立的线程，这样就不会因为线程阻塞在一个 socket 上而影响对其他 socket 的读写。BIO 的线程模型如下图所示，每一个 socket 都对应一个独立的线程；为了避免频繁创建、消耗线程，可以采用线程池，但是 socket 和线程之间的对应关系并不会变化。

           BIO 这种线程模型适用于 socket 连接不是很多的场景；但是现在的互联网场景，往往需要服务器能够支撑十万甚至百万连接，而创建十万甚至上百万个线程显然并不现实，所以 BIO 线程模型无法解决百万连接的问题。如果仔细观察，你会发现互联网场景中，虽然连接多，但是每个连接上的请求并不频繁，所以线程大部分时间都在等待 I/O 就绪。也就是说线程大部分时间都阻塞在那里，这完全是浪费，如果我们能够解决这个问题，那就不需要这么多线程了。  
            
           顺着这个思路，我们可以将线程模型优化为下图这个样子，可以用一个线程来处理多个连接，这样线程的利用率就上来了，同时所需的线程数量也跟着降下来了。这个思路很好，可是使用 BIO 相关的 API 是无法实现的，这是为什么呢？因为 BIO 相关的 socket 读写操作都是阻塞式的，而一旦调用了阻塞式 API，在 I/O 就绪前，调用线程会一直阻塞，也就无法处理其他的 socket 连接了 

           好在 Java 里还提供了非阻塞式（NIO）API，利用非阻塞式 API 就能够实现一个线程处理多个连接了。那具体如何实现呢？现在普遍都是采用 Reactor 模式，包括 Netty 的实现。所以，要想理解 Netty 的实现，接下来我们就需要先了解一下 Reactor 模式
         Reactor 模式
            下面是 Reactor 模式的类结构图，其中 Handle 指的是 I/O 句柄，在 Java 网络编程里，它本质上就是一个网络连接。Event Handler 很容易理解，就是一个事件处理器，其中 handle_event() 方法处理 I/O 事件，也就是每个 Event Handler 处理一个 I/O Handle；get_handle() 方法可以返回这个 I/O 的 Handle。Synchronous Event Demultiplexer 可以理解为操作系统提供的 I/O 多路复用 API，例如 POSIX 标准里的 select() 以及 Linux 里面的 epoll()
  
            Reactor 模式的核心自然是 Reactor 这个类，其中 register_handler() 和 remove_handler() 这两个方法可以注册和删除一个事件处理器；handle_events() 方式是核心，也是 Reactor 模式的发动机，这个方法的核心逻辑如下：首先通过同步事件多路选择器提供的 select() 方法监听网络事件，当有网络事件就绪后，就遍历事件处理器来处理该网络事件。由于网络事件是源源不断的，所以在主程序中启动 Reactor 模式，需要以 while(true){} 的方式调用 handle_events() 方法。

            void Reactor::handle_events(){ //通过同步事件多路选择器提供的 //select()方法监听网络事件 select(handlers); //处理网络事件 for(h in handlers){ h.handle_event(); }}// 在主程序中启动事件循环while (true) { handle_events(); 
         Netty 中的线程模型
            Netty 中最核心的概念是事件循环（EventLoop），其实也就是 Reactor 模式中的 Reactor，负责监听网络事件并调用事件处理器进行处理。在 4.x 版本的 Netty 中，网络连接和 EventLoop 是稳定的多对 1 关系，而 EventLoop 和 Java 线程是 1 对 1 关系，这里的稳定指的是关系一旦确定就不再发生变化。也就是说一个网络连接只会对应唯一的一个 EventLoop，而一个 EventLoop 也只会对应到一个 Java 线程，所以一个网络连接只会对应到一个 Java 线程

            一个网络连接对应到一个 Java 线程上，有什么好处呢？最大的好处就是对于一个网络连接的事件处理是单线程的，这样就避免了各种并发问题
            
            Netty 中的线程模型可以参考下图，这个图和前面我们提到的理想的线程模型图非常相似，核心目标都是用一个线程处理多个网络连接

            Netty 中还有一个核心概念是 EventLoopGroup，顾名思义，一个 EventLoopGroup 由一组 EventLoop 组成。实际使用中，一般都会创建两个 EventLoopGroup，一个称为 bossGroup，一个称为 workerGroup。为什么会有两个 EventLoopGroup 呢

            这个和 socket 处理网络请求的机制有关，socket 处理 TCP 网络连接请求，是在一个独立的 socket 中，每当有一个 TCP 连接成功建立，都会创建一个新的 socket，之后对 TCP 连接的读写都是由新创建处理的 socket 完成的。也就是说处理 TCP 连接请求和读写请求是通过两个不同的 socket 完成的
            
            在 Netty 中，bossGroup 就用来处理连接请求的，而 workerGroup 是用来处理读写请求的。bossGroup 处理完连接请求后，会将这个连接提交给 workerGroup 来处理， workerGroup 里面有多个 EventLoop，那新的连接会交给哪个 EventLoop 来处理呢？这就需要一个负载均衡算法，Netty 中目前使用的是轮询算法  
         用 Netty 实现 Echo 程序服务端 
               下面的示例代码基于 Netty 实现了 echo 程序服务端：首先创建了一个事件处理器（等同于 Reactor 模式中的事件处理器），然后创建了 bossGroup 和 workerGroup，再之后创建并初始化了 ServerBootstrap，代码还是很简单的，不过有两个地方需要注意一下
               

               第一个，如果 NettybossGroup 只监听一个端口，那 bossGroup 只需要 1 个 EventLoop 就可以了，多了纯属浪费

               第二个，默认情况下，Netty 会创建“2*CPU 核数”个 EventLoop，由于网络连接与 EventLoop 有稳定的关系，所以事件处理器在处理网络事件的时候是不能有阻塞操作的，否则很容易导致请求大面积超时。如果实在无法避免使用阻塞操作，那可以通过线程池来异步处理

               //事件处理器final EchoServerHandler serverHandler   = new EchoServerHandler();//boss线程组  EventLoopGroup bossGroup   = new NioEventLoopGroup(1); //worker线程组  EventLoopGroup workerGroup   = new NioEventLoopGroup();

               try {  ServerBootstrap b = new ServerBootstrap();  b.group(bossGroup, workerGroup)   .channel(NioServerSocketChannel.class)   .childHandler(new ChannelInitializer<SocketChannel>() {     @Override     public void initChannel(SocketChannel ch){       ch.pipeline().addLast(serverHandler);     }    });  //bind服务端端口    ChannelFuture f = b.bind(9090).sync();  f.channel().closeFuture().sync();} finally {  //终止工作线程组  workerGroup.shutdownGracefully();  //终止boss线程组  bossGroup.shutdownGracefully();}


               //socket连接处理器class EchoServerHandler extends     ChannelInboundHandlerAdapter {  //处理读事件    @Override  public void channelRead(    ChannelHandlerContext ctx, Object msg){      ctx.write(msg);  }  //处理读完成事件  @Override  public void channelReadComplete(    ChannelHandlerContext ctx){      ctx.flush();  }  //处理异常事件  @Override  public void exceptionCaught(    ChannelHandlerContext ctx,  Throwable cause) {      cause.printStackTrace();      ctx.close();  }}

               Netty 是一个款优秀的网络编程框架，性能非常好，为了实现高性能的目标，Netty 做了很多优化，例如优化了 ByteBuffer、支持零拷贝等等，和并发编程相关的就是它的线程模型了。Netty 的线程模型设计得很精巧，每个网络连接都关联到了一个线程上，这样做的好处是：对于一个网络连接，读写操作都是单线程执行的，从而避免了并发程序的各种问题
           - Netty
            EventLoop
            pipeLine 事件传播
           - 关于Netty的Reactor实现？
           - Netty的ByteBuf有哪些？（direct buf）==>zero copy(网络数据传输)  编解码
           - 内存与非内存Bytebuffer的区别与使用场景？
            GC回收
            堆外内存 ReferenceCountUtil.release() 引用计数法
           - 池化与非池化buffer的区别与使用场景？
           - 关于Netty的请求Buffer和响应Buffer?
          - Netty的ChannelPipeline设计模式？（责任链设计模式）
          - Netty的核心option参数配置？
          - Netty的ChannelInboundHandlerAdapter 使用异步 （不会自动释放）和 handler中未使用异步线程调用可以使用 SimpleChannelInboundHandler(自动释放资源)关系？
           服务端ctx.writeAndFlush会释放buffer
           客户端没有调用 
          - Netty的EventLoop核心实现？
          - Netty的连接管理事件接口有哪些常用方法（ChannelDuplexHandler）？
          - Netty的编解码与序列化手段
         - Netty的FastThreadLocal实现？
         - Netty中应用的装饰者 和 观察者模式在哪里体现？
















##  Dubbo
     --dubbo 中的 URL
       在互联网领域，每个信息资源都有统一的且在网上唯一的地址，该地址就叫 URL（Uniform Resource Locator，统一资源定位符），它是互联网的统一资源定位标志，也就是指网络地址。
       URL 本质上就是一个特殊格式的字符串。一个标准的 URL 格式可以包含如下的几个部分：
       protocol://username:password@host:port/path?key=value&key=value 
       
         protocol：URL 的协议。我们常见的就是 HTTP 协议和 HTTPS 协议，当然，还有其他协议，如 FTP 协议、SMTP 协议等。
         username/password：用户名/密码。 HTTP Basic Authentication 中多会使用在 URL 的协议之后直接携带用户名和密码的方式。
         host/port：主机/端口。在实践中一般会使用域名，而不是使用具体的 host 和 port。
         path：请求的路径。
         parameters：参数键值对。一般在 GET 请求中会将参数放到 URL 中，POST 请求会将参数放到请求体中。
       Dubbo 中任意的一个实现都可以抽象为一个 URL，Dubbo 使用 URL 来统一描述了所有对象和配置信息，并贯穿在整个 Dubbo 框架之中。这里我们来看 Dubbo 中一个典型 URL 的示例
         dubbo://172.17.32.91:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=32508&release=&side=provider&timestamp=1593253404714dubbo://172.17.32.91:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=32508&release=&side=provider&timestamp=1593253404714
       这个 Demo Provider 注册到 ZooKeeper 上的 URL 信息，简单解析一下这个 URL 的各个部分：
        protocol：dubbo 协议。
        username/password：没有用户名和密码。
        host/port：172.17.32.91:20880。
        path：org.apache.dubbo.demo.DemoService。
         parameters：参数键值对，这里是问号后面的参数。 
       在 dubbo-common 包中还提供了 URL 的辅助类：
         URLBuilder， 辅助构造 URL；
         URLStrParser， 将字符串解析成 URL 对象。
       Dubbo中URL示例
        URL再SPI中的应用
         Dubbo SPI 中有一个依赖 URL 的重要场景——适配器方法，是被 @Adaptive 注解标注的， URL 一个很重要的作用就是与 @Adaptive 注解一起选择合适的扩展实现类。
         在 dubbo-registry-api 模块中我们可以看到 RegistryFactory 这个接口，其中的 getRegistry() 方法上有 @Adaptive({"protocol"}) 注解，说明这是一个适配器方法，Dubbo 在运行时会为其动态生成相应的 “$Adaptive” 类型
         在生成的 RegistryFactory$Adaptive 类中会自动实现 getRegistry() 方法，其中会根据 URL 的 Protocol 确定扩展名称，从而确定使用的具体扩展实现类。我们可以找到 RegistryProtocol 这个类，并在其 getRegistry() 方法中打一个断点
         那么在 RegistryFactory$Adaptive 中得到的扩展名称为 zookeeper，此次使用的 Registry 扩展实现类就是 ZookeeperRegistryFactory。
        URL 在服务暴露中的应用
          Provider 在启动时，会将自身暴露的服务注册到 ZooKeeper 上，具体是注册哪些信息到 ZooKeeper 上呢？我们来看 ZookeeperRegistry.doRegister() 方法
          传入的 URL 中包含了 Provider 的地址（172.18.112.15:20880）、暴露的接口（org.apache.dubbo.demo.DemoService）等信息， toUrlPath() 方法会根据传入的 URL 参数确定在 ZooKeeper 上创建的节点路径，还会通过 URL 中的 dynamic 参数值确定创建的 ZNode 是临时节点还是持久节点。

        URL 在服务订阅中的应用
          Consumer 启动后会向注册中心进行订阅操作，并监听自己关注的 Provider。那 Consumer 是如何告诉注册中心自己关注哪些 Provider 呢
           ZookeeperRegistry 这个实现类，它是由上面的 ZookeeperRegistryFactory 工厂类创建的 Registry 接口实现，其中的 doSubscribe() 方法是订阅操作的核心实现

           consumer://...?application=dubbo-demo-api-consumer&category=providers,configurators,routers&interface=org.apache.dubbo.demo.DemoService...

           其中 Protocol 为 consumer ，表示是 Consumer 的订阅协议，其中的 category 参数表示要订阅的分类，这里要订阅 providers、configurators 以及 routers 三个分类；interface 参数表示订阅哪个服务接口，这里要订阅的是暴露 org.apache.dubbo.demo.DemoService 实现的 Provider

           通过 URL 中的上述参数，ZookeeperRegistry 会在 toCategoriesPath() 方法中将其整理成一个 ZooKeeper 路径，然后调用 zkClient 在其上添加监听。
## Dubbo的Spi机制？
        Dubbo 为了更好地达到 OCP 原则（即“对扩展开放，对修改封闭”的原则），采用了“微内核+插件”的架构。那什么是微内核架构呢？微内核架构也被称为插件化架构（Plug-in Architecture），这是一种面向功能进行拆分的可扩展性架构。内核功能是比较稳定的，只负责管理插件的生命周期，不会因为系统功能的扩展而不断进行修改。功能上的扩展全部封装到插件之中，插件模块是独立存在的模块，包含特定的功能，能拓展内核系统的功能。 
        
        微内核架构中，内核通常采用 Factory、IoC、OSGi 等方式管理插件生命周期，Dubbo 最终决定采用 SPI 机制来加载插件，Dubbo SPI 参考 JDK 原生的 SPI 机制，进行了性能优化以及功能增强。因此，在讲解 Dubbo SPI 之前，我们有必要先来介绍一下 JDK SPI 的工作原理
       JDK SPI
         SPI（Service Provider Interface）主要是被框架开发人员使用的一种技术。例如，使用 Java 语言访问数据库时我们会使用到 java.sql.Driver 接口，不同数据库产品底层的协议不同，提供的 java.sql.Driver 实现也不同，在开发 java.sql.Driver 接口时，开发人员并不清楚用户最终会使用哪个数据库，在这种情况下就可以使用 Java SPI 机制在实际运行过程中，为 java.sql.Driver 接口寻找具体的实现。
         
         当服务的提供者提供了一种接口的实现之后，需要在 Classpath 下的 META-INF/services/ 目录里创建一个以服务接口命名的文件，此文件记录了该 jar 包提供的服务接口的具体实现类。当某个应用引入了该 jar 包且需要使用该服务时，JDK SPI 机制就可以通过查找这个 jar 包的 META-INF/services/ 中的配置文件来获得具体的实现类名，进行实现类的加载和实例化，最终使用该实现类完成业务功能。 
       jdk SPI源码分析
          JDK SPI 的入口方法是 ServiceLoader.load() 
          在 ServiceLoader.load() 方法中，首先会尝试获取当前使用的 ClassLoader（获取当前线程绑定的 ClassLoader，查找失败后使用 SystemClassLoader），然后调用 reload() 方法，

          在 reload() 方法中，首先会清理 providers 缓存（LinkedHashMap 类型的集合），该缓存用来记录 ServiceLoader 创建的实现对象，其中 Key 为实现类的完整类名，Value 为实现类的对象。之后创建 LazyIterator 迭代器，用于读取 SPI 配置文件并实例化实现类对象。

          // 缓存，用来缓存 ServiceLoader创建的实现对象 
          private LinkedHashMap<String,S> providers = new LinkedHashMap<>(); 
          public void reload()   
          providers.clear(); // 清空缓存 
          lookupIterator = new LazyIterator(service, loader); // 迭代器 
          在前面的示例中，main() 方法中使用的迭代器底层就是调用了 ServiceLoader.LazyIterator 实现的。Iterator 接口有两个关键方法：hasNext() 方法和 next() 方法。这里的 LazyIterator 中的next() 方法最终调用的是其 nextService() 方法，hasNext() 方法最终调用的是 hasNextService() 方法，调用关系如下图所示：
          首先来看 LazyIterator.hasNextService() 方法，该方法主要负责查找 META-INF/services 目录下的 SPI 配置文件，并进行遍历

          在 hasNextService() 方法中完成 SPI 配置文件的解析之后，再来看 LazyIterator.nextService() 方法，该方法负责实例化 hasNextService() 方法读取到的实现类，其中会将实例化的对象放到 providers 集合中缓存起来

          以上就是在 main() 方法中使用的迭代器的底层实现。最后，我们再来看一下 main() 方法中使用ServiceLoader.iterator() 方法拿到的迭代器是如何实现的，这个迭代器是依赖 LazyIterator 实现的一个匿名内部类
       jdk SPI在JDBC中的应用
           JDK 中只定义了一个 java.sql.Driver 接口，具体的实现是由不同数据库厂商来提供的。这里我们就以 MySQL 提供的 JDBC 实现包为例进行分析。
           在 mysql-connector-java-*.jar 包中的 META-INF/services 目录下，有一个 java.sql.Driver 文件中只有一行内容，如下所示：
           com.mysql.cj.jdbc.Driver
           在使用 mysql-connector-java-*.jar 包连接 MySQL 数据库的时候，我们会用到如下语句创建数据库连接：
           String url = "jdbc:xxx://xxx:xxx/xxx"; 
           Connection conn = DriverManager.getConnection(url, username, pwd); 
           DriverManager 是 JDK 提供的数据库驱动管理器，其中的代码片段，如下所示：
            
           static { 
            loadInitialDrivers(); 
            println("JDBC DriverManager initialized"); 
           } 
            
           在调用 getConnection() 方法的时候，DriverManager 类会被 Java 虚拟机加载、解析并触发 static 代码块的执行；在 loadInitialDrivers() 方法中通过 JDK SPI 扫描 Classpath 下 java.sql.Driver 接口实现类并实例化
          
           private static void loadInitialDrivers() { 
           String drivers = System.getProperty("jdbc.drivers") 
           // 使用 JDK SPI机制加载所有 java.sql.Driver实现类 
           ServiceLoader<Driver> loadedDrivers =  
           ServiceLoader.load(Driver.class); 
           Iterator<Driver> driversIterator = loadedDrivers.iterator(); 
           while(driversIterator.hasNext()) { 
             driversIterator.next(); 
           } 
           String[] driversList = drivers.split(":"); 
           for (String aDriver : driversList) { // 初始化Driver实现类 
            Class.forName(aDriver, true, 
              ClassLoader.getSystemClassLoader()); 
           } 
           } 
           在 MySQL 提供的 com.mysql.cj.jdbc.Driver 实现类中，同样有一段 static 静态代码块，这段代码会创建一个 com.mysql.cj.jdbc.Driver 对象并注册到 DriverManager.registeredDrivers 集合中（CopyOnWriteArrayList 类型），如下所示：
      
           static { 
             java.sql.DriverManager.registerDriver(new Driver()); 
           } 
           
           在 getConnection() 方法中，DriverManager 从该 registeredDrivers 集合中获取对应的 Driver 对象创建 Connection，核心实现如下所示：
           private static Connection getConnection(String url, java.util.Properties info, Class<?> caller) throws SQLException { 
           // 省略 try/catch代码块以及权限处理逻辑 
           for(DriverInfo aDriver : registeredDrivers) { 
             Connection con = aDriver.driver.connect(url, info); 
             return con; 
            } 
           } 
       Dubbo SPI
        概述
          
          扩展点：通过 SPI 机制查找并加载实现的接口（又称“扩展接口”）。前文示例中介绍的 Log 接口、com.mysql.cj.jdbc.Driver 接口，都是扩展点。
          扩展点实现：实现了扩展接口的实现类。
          
          通过前面的分析可以发现，JDK SPI 在查找扩展实现类的过程中，需要遍历 SPI 配置文件中定义的所有实现类，该过程中会将这些实现类全部实例化。如果 SPI 配置文件中定义了多个实现类，而我们只需要使用其中一个实现类时，就会生成不必要的对象。例如，org.apache.dubbo.rpc.Protocol 接口有 InjvmProtocol、DubboProtocol、RmiProtocol、HttpProtocol、HessianProtocol、ThriftProtocol 等多个实现，如果使用 JDK SPI，就会加载全部实现类，导致资源的浪费。
          
          Dubbo SPI 不仅解决了上述资源浪费的问题，还对 SPI 配置文件扩展和修改。
          首先，Dubbo 按照 SPI 配置文件的用途，将其分成了三类目录。
          META-INF/services/ 目录：该目录下的 SPI 配置文件用来兼容 JDK SPI 。
          META-INF/dubbo/ 目录：该目录用于存放用户自定义 SPI 配置文件。
          META-INF/dubbo/internal/ 目录：该目录用于存放 Dubbo 内部使用的 SPI 配置文件。
          Dubbo 将 SPI 配置文件改成了 KV 格式，例如：
          dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
          其中 key 被称为扩展名（也就是 ExtensionName），当我们在为一个接口查找具体实现类时，可以指定扩展名来选择相应的扩展实现。例如，这里指定扩展名为 dubbo，Dubbo SPI 就知道我们要使用：org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol 这个扩展实现类，只实例化这一个扩展实现即可，无须实例化 SPI 配置文件中的其他扩展实现类
          
          使用 KV 格式的 SPI 配置文件的另一个好处是：让我们更容易定位到问题。假设我们使用的一个扩展实现类所在的 jar 包没有引入到项目中，那么 Dubbo SPI 在抛出异常的时候，会携带该扩展名信息，而不是简单地提示扩展实现类无法加载。这些更加准确的异常信息降低了排查问题的难度，提高了排查问题的效率。
        核心实现
        1. @SPI 注解
          Dubbo 中某个接口被 @SPI注解修饰时，就表示该接口是扩展接口，前文示例中的 org.apache.dubbo.rpc.Protocol 接口就是一个扩展接口  
          @SPI 注解的 value 值指定了默认的扩展名称，例如，在通过 Dubbo SPI 加载 Protocol 接口实现时，如果没有明确指定扩展名，则默认会将 @SPI 注解的 value 值作为扩展名，即加载 dubbo 这个扩展名对应的 org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol 这个扩展实现类，相关的 SPI 配置文件在 dubbo-rpc-dubbo 模块中
          ExtensionLoader 是如何处理 @SPI 注解的呢
          ExtensionLoader 位于 dubbo-common 模块中的 extension 包中，功能类似于 JDK SPI 中的 java.util.ServiceLoader。Dubbo SPI 的核心逻辑几乎都封装在 ExtensionLoader 之中
          Protocol protocol = ExtensionLoader 
          .getExtensionLoader(Protocol.class).getExtension("dubbo");
          
          ExtensionLoader 中三个核心的静态字段。
          **strategies（LoadingStrategy[]类型）:**LoadingStrategy 接口有三个实现（通过 JDK SPI 方式加载的），如下图所示，分别对应前面介绍的三个 Dubbo SPI 配置文件所在的目录，且都继承了 Prioritized 这个优先级接口，默认
          优先级是
          DubboInternalLoadingStrategy > DubboLoadingStrategy >  ServicesLoadingStrateg
          EXTENSION_LOADERS（ConcurrentMap<Class, ExtensionLoader>类型）
          ：Dubbo 中一个扩展接口对应一个 ExtensionLoader 实例，该集合缓存了全部 ExtensionLoader 实例，其中的 Key 为扩展接口，Value 为加载其扩展实现的 ExtensionLoader 实例。
          EXTENSION_INSTANCES（ConcurrentMap<Class<?>, Object>类型）：该集合缓存了扩展实现类与其实例对象的映射关系。在前文示例中，Key 为 Class，Value 为 DubboProtocol 对象。
          关注一下 ExtensionLoader 的实例字段。
          type（Class<?>类型）：当前 ExtensionLoader 实例负责加载扩展接口。
          cachedDefaultName（String类型）：记录了 type 这个扩展接口上 @SPI 注解的 value 值，也就是默认扩展名。
          cachedNames（ConcurrentMap<Class<?>, String>类型）：缓存了该 ExtensionLoader 加载的扩展实现类与扩展名之间的映射关系。
          cachedClasses（Holder<Map<String, Class<?>>>类型）：缓存了该 ExtensionLoader 加载的扩展名与扩展实现类之间的映射关系。cachedNames 集合的反向关系缓存。
          cachedInstances（ConcurrentMap<String, Holder<Object>>类型）：缓存了该 ExtensionLoader 加载的扩展名与扩展实现对象之间的映射关系。
          ExtensionLoader.getExtensionLoader() 方法会根据扩展接口从 EXTENSION_LOADERS 缓存中查找相应的 ExtensionLoader 实例
          得到接口对应的 ExtensionLoader 对象之后会调用其 getExtension() 方法，根据传入的扩展名称从 cachedInstances 缓存中查找扩展实现的实例
          // 根据扩展名从SPI配置文件中查找对应的扩展实现类 
          instance = createExtension(name); 
          在 createExtension() 方法中完成了 SPI 配置文件的查找以及相应扩展实现类的实例化，同时还实现了自动装配以及自动 Wrapper 包装等功能。其核心流程是这样的：
           1获取 cachedClasses 缓存，根据扩展名从 cachedClasses 缓存中获取扩展实现类。如果 cachedClasses 未初始化，则会扫描前面介绍的三个 SPI 目录获取查找相应的 SPI 配置文件，然后加载其中的扩展实现类，最后将扩展名和扩展实现类的映射关系记录到 cachedClasses 缓存中。这部分逻辑在 loadExtensionClasses() 和 loadDirectory() 方法中。
           2根据扩展实现类从 EXTENSION_INSTANCES 缓存中查找相应的实例。如果查找失败，会通过反射创建扩展实现对象。
           3自动装配扩展实现对象中的属性（即调用其 setter）。这里涉及 ExtensionFactory 以及自动装配的相关内容，本课时后面会进行详细介绍。
           4自动包装扩展实现对象。这里涉及 Wrapper 类以及自动包装特性的相关内容，本课时后面会进行详细介绍。
           5如果扩展实现类实现了 Lifecycle 接口，在 initExtension() 方法中会调用 initialize() 方法进行初始化。
        2. @Adaptive 注解与适配器
           @Adaptive 注解用来实现 Dubbo 的适配器功能，那什么是适配器呢？这里我们通过一个示例进行说明。Dubbo 中的 ExtensionFactory 接口有三个实现类，如下图所示，ExtensionFactory 接口上有 @SPI 注解，AdaptiveExtensionFactory 实现类上有 @Adaptive 注解
           AdaptiveExtensionFactory 不实现任何具体的功能，而是用来适配 ExtensionFactory 的 SpiExtensionFactory 和 SpringExtensionFactory 这两种实现。AdaptiveExtensionFactory 会根据运行时的一些状态来选择具体调用 ExtensionFactory 的哪个实现。
           @Adaptive 注解还可以加到接口方法之上，Dubbo 会动态生成适配器类。例如，Transporter接口有两个被 @Adaptive 注解修饰的方法：
           Dubbo 会生成一个 Transporter$Adaptive 适配器类，该类继承了 Transporter 接口：
           ExtensionLoader.createExtension() 方法，其中在扫描 SPI 配置文件的时候，会调用 loadClass() 方法加载 SPI 配置文件中指定的类
           
           loadClass() 方法中会识别加载扩展实现类上的 @Adaptive 注解，将该扩展实现的类型缓存到 cachedAdaptiveClass 这个实例字段上（volatile修饰）
            
           ExtensionLoader.getAdaptiveExtension() 方法获取适配器实例，并将该实例缓存到 cachedAdaptiveInstance 字段（Holder类型）中，核心流程如下：
           首先，检查 cachedAdaptiveInstance 字段中是否已缓存了适配器实例，如果已缓存，则直接返回该实例即可。
           然后，调用 getExtensionClasses() 方法，其中就会触发前文介绍的 loadClass() 方法，完成 cachedAdaptiveClass 字段的填充。
           如果存在 @Adaptive 注解修饰的扩展实现类，该类就是适配器类，通过 newInstance() 将其实例化即可。如果不存在 @Adaptive 注解修饰的扩展实现类，就需要通过 createAdaptiveExtensionClass() 方法扫描扩展接口中方法上的 @Adaptive 注解，动态生成适配器类，然后实例化。
           接下来，调用 injectExtension() 方法进行自动装配，就能得到一个完整的适配器实例。
           最后，将适配器实例缓存到 cachedAdaptiveInstance 字段，然后返回适配器实例。   
        3. 自动包装特性   
            Dubbo 将多个扩展实现类的公共逻辑，抽象到 Wrapper 类中，Wrapper 类与普通的扩展实现类一样，也实现了扩展接口，在获取真正的扩展实现对象时，在其外面包装一层 Wrapper 对象，你可以理解成一层装饰器 
            ExtensionLoader.loadClass() 方法中
            在 isWrapperClass() 方法中，会判断该扩展实现类是否包含拷贝构造函数（即构造函数只有一个参数且为扩展接口类型），如果包含，则为 Wrapper 类，这就是判断 Wrapper 类的标准。
            将 Wrapper 类记录到 cachedWrapperClasses（Set<Class<?>>类型）这个实例字段中进行缓存。
        4. 自动装配特性    
           在 createExtension() 方法中我们看到，Dubbo SPI 在拿到扩展实现类的对象（以及 Wrapper 类的对象）之后，还会调用 injectExtension() 方法扫描其全部 setter 方法，并根据 setter 方法的名称以及参数的类型，加载相应的扩展实现，然后调用相应的 setter 方法填充属性，这就实现了 Dubbo SPI 的自动装配特性
           injectExtension() 方法实现的自动装配依赖了 ExtensionFactory（即 objectFactory 字段），前面我们提到过 ExtensionFactory 有 SpringExtensionFactory 和 SpiExtensionFactory 两个真正的实现（还有一个实现是 AdaptiveExtensionFactory 是适配器）
           第一个，SpiExtensionFactory。 根据扩展接口获取相应的适配器，没有到属性名称：
           public <T> T getExtension(Class<T> type, String name) { 
           第二个，SpringExtensionFactory。 将属性名称作为 Spring Bean 的名称，从 Spring 容器中获取 Bean：
           public <T> T getExtension(Class<T> type, String name) {  
        5. @Activate注解与自动激活特性     
         
           @Activate 注解标注在扩展实现类上，有 group、value 以及 order 三个属性。
           group 属性：修饰的实现类是在 Provider 端被激活还是在 Consumer 端被激活。
           value 属性：修饰的实现类只在 URL 参数中出现指定的 key 时才会被激活。
           order 属性：用来确定扩展实现类的排序。
		
		  https://www.jianshu.com/p/3a3edbcd8f24
		- Dubbo的核心模型 invoker、invocation、filter
		- Dubbo的隐式传递?
		- Dubbo的泛化调用？
		- Dubbo的export与importer时机？
		- Dubbo的服务调用过程？
		- Dubbo的负载均衡策略？
		- Dubbo的集群容错？
		- IO NIO区别？
		- 多路复用的概念，Selector
		- Channel的概念、Bytebuf的概念，flip、position...
		- FileChannel 如何使用？
		- RAF使用，seek、skip方法
	


		自定义注解整合dubbo和spring
		1 / 启动类添加扫描注解 
		    RapidComponetnScan
               value
               basePackages
               basePackageClass 
            import
            
            rapidComponentScanRegister     

              rapidAnnotationBeanPostProcessor


