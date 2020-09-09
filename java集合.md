 # 集合类总结
 ## ArrayList源码解析和设计思路 
 ### 整体架构：
        index(0开始) elementData数组本身
        DEFAULT_CAPACAITY 数组的初始大小默认是10
        size当前数组的大小 int 没有使用volatile修饰 非线程安全
        modCount统计当前数组被修改的版本次数，数组结构有变动，就会+1
 ### 类注释
       允许put null 值，会自动扩容
       size isEmpty get set add 等方法时间复杂度都是O(1)
       是非线程安全的 多线程情况下 推荐使用线程安全类Collections.synchronizedList
       增强for循环，或者迭代使用过程中，如果数组大小被改变，会快速失败，抛出异常。
 ### 源码解析
	          初始化 
	           无参数初始化 ，数组大小为空
	           指定初始数据初始化 elementData是保存数组的容器 默认为null
	           如果给定的集合数据有值
	              判断集合类型是否是Object类型 会转成Objectp[]   
	            注意 ArrayList无参数构造器初始化时，默认大小时空数组，并不是10 10时第一次扩容时的数组值
	          新增和扩容实现
	            新增就是往数组中添加元素，主要分为两步
	              判断是否需要扩容，如果需要进行扩容操作
	              直接赋值
	              add()
	              //确保数组大小是否足够，不够执行扩容，size 为当前数组的大小
                   ensureCapacityInternal(size + 1);  // Increments modCount!!
                     如果初始化数组大小时，有给定初始化值，以给定的大小为准，不走if逻辑
                     确保容积足够 
                      ensureExplicitCapacity
                       记录数组被修改
                       如果我们期望的最小容量大于目前数组的长度，那么就扩容
                       grow
                       把现有的数据拷贝到新的数组里面去
                        如果扩容的值小于我们期望的值，扩容后的值就等于我们的期望值
                        如果扩容后的值大于jvm所能分配的数组的最大值，就用Integer最大值
                        通过复制进行扩容 Arrays.copyOf()
                     扩容的规则并不是翻倍，时原来容量大小+容量大小的一半
                     ArrayList 中数组的最大值时Integer.max_value 超过这个值，jvm就不会给数组分配内存空间了

                  //直接赋值，线程不安全的
                   elementData[size++] = e;
               扩容的本质
                     System.arraycopy(elementData, 0, newElementData, 0,Math.min(elementData.length,newCapacity))       
		       删除   
                   数组索引删除 根据值删除 批量删除

                   如果要删除的值时null 找到第一个值时null的删除
                   如果要删除的值不是null,找到第一个和要删除的值相等

                   fastRemove 
                     记录数组的结构变化 modCount
                      numMoved 表示删除 index 位置的元素后，需要从 index 后移动多少个元素到前面去
                       if (numMoved > 0)
                     // 从 index +1 位置开始被拷贝，拷贝的起始位置是 index，长度是 numMoved
                     System.arraycopy(elementData, index+1, elementData, index, numMoved)
                     //数组最后一个位置赋值 null，帮助 GC
                      elementData[--size] = null;
                迭代器
                  迭代器 要素
                  cursor 迭代的过程中，下一个元素的位置，默认从0开始
                  lastRet 新增场景，表实上一次迭代的过程中，索引的位置，删除场景为 -1 
                  expectedModCount 表实迭代过程中 期望的版本号 modCount 数组实际的版本号
                  三个方法
                  hasNext还有没有值可以迭代
                  next如果有值 迭代的值时多少
                  remove删除当前的迭代的值
                迭代器源码
                  public boolean hasNext() {
                         return cursor != size;//cursor 表示下一个元素的位置，size 表示实际大小，如果两者相等，说明已经没有元素可以迭代了，如果不等，说明还可以迭代
                   } 
                  迭代过程中检查版本是否被修改，有被修改直接抛出异常 
                  checkForComodification     if (modCount != expectedModCount)    
                  下一次迭代时，返回元素位置 cursor = i + 1;
                  next 方法就干了两件事情，第一是检验能不能继续迭代，第二是找到迭代的值，并为下一次迭代做准备（cursor+1）。
                  remove
                    //迭代过程中，判断版本号有无被修改，有被修改，抛 ConcurrentModificationException 异常
                   checkForComodification();
                   cursor = lastRet;
                 // -1 表示元素已经被删除，这里也防止重复删除
                  lastRet = -1;
                   // 删除元素时 modCount 的值已经发生变化，在此赋值给 expectedModCount
                  // 这样下次迭代时，两者的值是一致的了
                   expectedModCount = modCount;
             时间复杂度
              O（1）
             线程安全
              只有当 ArrayList 作为共享变量时，才会有线程安全问题，当 ArrayList 是方法内的局部变量时，是没有线程安全的问题的。
              ArrayList 有线程安全问题的本质，是因为 ArrayList 自身的 elementData、size、modConut 在进行各种操作时，都没有加锁，而且这些变量的类型并非是可见（volatile）的，所以如果多个线程对这些变量进行操作时，可能会有值被覆盖的情况。
              类注释中推荐我们使用 Collections#synchronizedList 来保证线程安全，SynchronizedList 是通过在每个方法上面加上锁来实现，虽然实现了线程安全，但是性能大大降低，
## LinkedList 源码解析
           场景 适用于集合元素先入先出和先入后出的场景
### 整体架构
    底层是一个双向链表
    链表每个节点我们叫做Node Node有prev属性 代表前一个节点 next属性代表后一个节点
    first 是双向链表的头节点 它的前一个结点是null
    last是双向链表的尾巴节点 后一个节点是null
    当链表中没有数据时 first last 是同一个节点 前后都指向null
     因为是双向链表，只要机器内存足够强大 是没有大小限制的。
### 源码解析
             -追加 新增
                追加可以加到头部也可以加到尾部
                 add默认尾部 addFirst头部添加
     
 	            从尾部追加
                  linkLast(e)
                  把尾节点暂存
                  新建节点 // l 是新节点的前一个节点，当前值是尾节点值 // e 表示当前新增节点，当前新增节点后一个节点是 null	
		          final Node<E> newNode = new Node<>(l, e, null);	
 	              // 新建节点追加到尾部  last = newNode;
		         //如果链表为空（l 是尾节点，尾节点为空，链表即空），头部和尾部是同一个节点，都是新建的节点
                   if (l == null)
                      first = newNode;![图片描述]
                    //否则把前尾节点的下一个节点，指向当前尾节点。
                    else
                           l.next = newNode;
                       //大小和版本更改
                     size++;
                        modCount++; 
	            从头部追加
	               linkFirst
		            // 头节点赋值给临时变量 final Node<E> f = first;  
		             / 新建节点，前一个节点指向null，e 是新建节点，f 是新建节点的下一个节点，目前值是头节点的值
                     final Node<E> newNode = new Node<>(null, e, f);
		               // 头节点为空，就是链表为空，头尾节点是一个节点
                       if (f == null)
                               last = newNode;
                         //上一个头节点的前一个节点指向当前节点
                        else
                          f.prev = newNode;
	               -节点删除
       	                 节点删除的方式和追加类似，我们可以选择从头部删除，也可以选择从尾部删除，删除操作会把节点的值，
		                  前后指向节点都置为 null，帮助 GC 进行回收。
	                头部删除
	                 //从头删除节点 f 是链表头节点
		              unlinkFirst(Node<E> f) 
  	                   // 拿出头节点的值，作为方法的返回值
                           final E element = f.item;
		                 // 拿出头节点的下一个节点
                           final Node<E> next = f.next;
                         //帮助 GC 回收头节点
                        f.item = null;
                        f.next = null;
		              // 头节点的下一个节点成为头节点
                     first = next;
		             //如果 next 为空，表明链表为空
                    if (next == null)
                          last = null;
                      //链表不为空，头节点的前一个节点指向 null
                       else
                            next.prev = null;
                              //修改链表大小和版本
                             size--;
                                  modCount++;
                           LinkedList 新增和删除速度很快。
             - 节点查询
                链表查询某一个节点是非常慢的 需要挨个循环查找才行
	             // 根据链表索引位置查询节点
	              // 如果 index 处于队列的前半部分，从头开始找，size >> 1 是 size 除以 2 的意思。
                if (index < (size >> 1)) {
                    Node<E> x = first;
                   // 直到 for 循环到 index 的前一个 node 停止
	                   // 直到 for 循环到 index 的前一个 node 停止
                       for (int i = 0; i < index; i++)
                         x = x.next;
       	             else {// 如果 index 处于队列的后半部分，从尾开始找
                     Node<E> x = last;
                         / / 直到 for 循环到 index 的后一个 node 停止
                         for (int i = size - 1; i > index; i--)
                           x = x.prev;
                   LinkedList 并没有采用从头循环到尾的做法，而是采取了简单二分法，首先看看 index 是在链表的前半部分，还是后半部分。如果是前半部分，就从头开始寻找，反之亦然。
	        通过这种方式，使循环的次数至少降低了一半，提高了查找的性能，
         - 方法对比
                LinkedList 实现了 Queue 接口，在新增、删除、查询等方面增加了很多新的方法，这些方法在平时特别容易混淆，在链表为空的情况下，返回值也不太一样，我们列一个表格，方便大家记录：

                 方法含义	返回异常	返回特殊值	底层实现
                 新增	    add(e)	   offer(e)	 底层实现相同
                 删除  	remove()	poll(e)	 链表为空时，remove 会抛出异常，poll 返回 null。
                查找	   element()	peek()	 链表为空时，element 会抛出异常，peek 返回 null。
         -迭代器
                双向迭代器 
                  ListIterator  
	                迭代顺序	方法
                 从尾到头迭代方法	hasPrevious、previous、previousIndex
                    从头到尾迭代方法	hasNext、next、nextIndex
               尾部迭代器特殊
                    // 如果上次节点索引位置大于 0，就还有节点可以迭代
                   public boolean hasPrevious() {
                   return nextIndex > 0;
                     }
	            去前一个节点
	                // next 为空场景：1:说明是第一次迭代，取尾节点(last);2:上一次操作把尾节点删除掉了
                    / / next 不为空场景：说明已经发生过迭代了，直接取前一个节点即可(next.prev)
                       lastReturned = next = (next == null) ? last : next.prev;
	
	                 迭代器删除
	            /  / lastReturned 是本次迭代需要删除的值，分以下空和非空两种情况：
             // lastReturned 为空，说明调用者没有主动执行过 next() 或者 previos()，直接报错
          // lastReturned 不为空，是在上次执行 next() 或者 previos()方法时赋的值
                 if (lastReturned == null)
             throw new IllegalStateException();
		   //删除当前节点
             unlink(lastReturned);
               // next == lastReturned 的场景分析：从尾到头递归顺序，并且是第一次迭代，并且要删除最后一个元素的情况下
              // 这种情况下，previous() 方法里面设置了 lastReturned = next = last,所以 next 和 lastReturned会相等
                 if (next == lastReturned)
                    // 这时候 lastReturned 是尾节点，lastNext 是 null，所以 next 也是 null，这样在 previous() 执行时，发现 next 是 null，就会把尾节点赋值给 next
                        next = lastNext;
                 else
                 nextIndex--;
                  lastReturned = null;
## list涉及到问题
          -扩容类问题
	        1 ArrayList 无参数构造器构造，现在 add 一个值进去，此时数组的大小是多少，下一次扩容前最大可用大小是多少？
              此处数组的大小是 1，下一次扩容前最大可用大小是 10，因为 ArrayList 第一次扩容时，是有默认值的，默认值是 10，在第一次 add 一个值进去时，数组的可用大小被扩容到 10 了
	        2 如果我连续往 list 里面新增值，增加到第 11 个的时候，数组的大小是多少？
              这里的考查点就是扩容的公式，当增加到 11 的时候，此时我们希望数组的大小为 11，但实际上数组的最大容量只有 10，不够了就需要扩容，扩容的公式是：oldCapacity + (oldCapacity>> 1)，oldCapacity 表示数组现有大小，
               目前场景计算公式是：10 + 10 ／2 = 15，然后我们发现 15 已经够用了，所以数组的大小会被扩容到 15。
            3 数组初始化，被加入一个值后，如果我使用 addAll 方法，一下子加入 15 个值，那么最终数组的大小是多少？
             第一题中我们已经计算出来数组在加入一个值后，实际大小是 1，最大可用大小是 10 ，现在需要一下子加入 15 个值，那我们期望数组的大小值就是   16，此时数组最大可用大小只有 10，明显不够，需要扩容，扩容后的大小是：10 + 10 ／2 = 15，这时候发现扩容后的大小仍然不到我们期望的值 16，这时候源码中有一种策略如下：

             // newCapacity 本次扩容的大小，minCapacity 我们期望的数组最小大小
             // 如果扩容后的值 < 我们的期望值，我们的期望值就等于本次扩容的大小
             if (newCapacity - minCapacity < 0)
             newCapacity = minCapacity;
           所以最终数组扩容后的大小为 16。
 
            4 现在我有一个很大的数组需要拷贝，原数组大小是 5k，请问如何快速拷贝？
              因为原数组比较大，如果新建新数组的时候，不指定数组大小的话，就会频繁扩容，频繁扩容就会有大量拷贝的工作，造成拷贝的性能低下，所以回答说新建数组时，指定新数组的大小为 5k 即可。
 	        5为什么说扩容会消耗性能？
             扩容底层使用的是 System.arraycopy 方法，会把原数组的数据全部拷贝到新数组上，所以性能消耗比较严重
	        6 源码扩容过程有什么值得借鉴的地方？
             是扩容的思想值得学习，通过自动扩容的方式，让使用者不用关心底层数据结构的变化，封装得很好，1.5 倍的扩容速度，可以让扩容速度在前期缓慢上升，在后期增速较快，大部分工作中要求数组的值并不是很大，所以前期增长缓慢有利于节省资源，在后期增速较快时，也可快速扩容。
               扩容过程中，有数组大小溢出的意识，比如要求扩容后的数组大小，不能小于 0，不能大于 Integer 的最大值。
	  
         -删除类问题 
	        1  ArrayList 数组，我们通过增强 for 循环进行删除
	        会报错。因为增强 for 循环过程其实调用的就是迭代器的 next () 方法，当你调用 list#remove () 方法进行删除时，modCount 的值会 +1，而这时候迭代器中的 expectedModCount 的值却没有变，导致在迭代器下次执行 next () 方法时，expectedModCount != modCount 就会报 ConcurrentModificationException 的错误。
  
          2 如果删除时使用 Iterator.remove () 方法可以删除么，为什么？
            可以的，因为 Iterator.remove () 方法在执行的过程中，会把最新的 modCount 赋值给 expectedModCount，这样在下次循环过程中，modCount 和 expectedModCount 两者就会相等。
         -对比类问题
           1 ArrayList 和 LinkedList 有何不同？
           可以先从底层数据结构开始说起，然后以某一个方法为突破口深入，比如：最大的不同是两者底层的数据结构不同，ArrayList 底层是数组，LinkedList 底层是双向链表，两者的数据结构不同也导致了操作的 API 实现有所差异，拿新增实现来说，ArrayList  会先计算并决定是否扩容，然后把新增的数据直接赋值到数组上，
            而 LinkedList 仅仅只需要改变插入节点和其前后节点的指向位置关系即可。
	       2 ArrayList 和 LinkedList 应用场景有何不同
             ArrayList 更适合于快速的查找匹配，不适合频繁新增删除，像工作中经常会对元素进行匹配查询的场景比较合适，LinkedList 更适合于经常新增和删除，对查询反而很少的场景。	
	      3  ArrayList 和 LinkedList 两者有没有最大容量
             ArrayList 有最大容量的，为 Integer 的最大值，大于这个值 JVM 是不会为数组分配内存空间的，LinkedList  底层是双向链表，理论上可以无限大。但源码中，LinkedList 实际大小用的是 int 类型，这也说明了 LinkedList 不能超过 Integer 的最大值，不然会溢出
	      4 ArrayList 和 LinkedList 是如何对 null 值进行处理的
             ArrayList 允许 null 值新增，也允许 null 值删除。删除 null 值时，是从头开始，找到第一值是 null 的元素删除；LinkedList 新增删除时对 null 值没有特殊校验，是允许新增和删除的。  
	      5  ArrayList 和 LinedList 是线程安全的么，为什么？
            当两者作为非共享变量时，比如说仅仅是在方法里面的局部变量时，是没有线程安全问题的，只有当两者是共享变量时，才会有线程安全问题。主要的问题点在于多线程环境下，所有线程任何时刻都可对数组和链表进行操作，这会导致值被覆盖，甚至混乱的情况。
             如果有线程安全问题，在迭代的过程中，会频繁报 ConcurrentModificationException 的错误，意思是在我当前循环的过程中，数组或链表的结构被其它线程修改了。
	      6  如何解决线程安全问题？
              Java 源码中推荐使用 Collections#synchronizedList 进行解决，Collections#synchronizedList 的返回值是 List 的每个方法都加了 synchronized 锁，保证了在同一时刻，数组和链表只会被一个线程所修改，或者采用 CopyOnWriteArrayList 并发 List 来解决	   
	     -双向链表
           双向链表中双向的意思是说前后节点之间互相有引用，链表的节点我们称为 Node。Node 有三个属性组成：其前一个节点，本身节点的值，其下一个节点，假设 A、B 节点相邻，A 节点的下一个节点就是 B，B 节点的上一个节点就是 A，两者互相引用，在链表的头部节点，我们称为头节点。头节点的前一个节点是 null，尾部称为尾节点，尾节点的后一个节点是 null，如果链表数据为空的话，头尾节点是同一个节点，本身是 null，指向前后节点的值也是 null。
          新增：我们可以选择从链表头新增，也可以选择从链表尾新增，如果是从链表尾新增的话，直接把当前节点追加到尾节点之后，本身节点自动变为尾节点。
           删除：把删除节点的后一个节点的 prev 指向其前一个节点，把删除节点的前一个节点的 next 指向其后一个节点，最后把删除的节点置为 null 即可。	       
## vector arraylist linkedList 

                都是实现集合框架中的list  有序集合  都能提供按照位置进行定位 添加 删除的操作；都能提供迭代器遍历容器内容

                vector 线程安全的动态数组  内部使用对象数组保存数据，可以根据需要自动增加容量 扩容提高一倍

                ArrayList 更加广泛的动态数组实现 本身不是线程安全  扩容增加50%

                linkedList 双向链表 不需要扩容 也不是线程安全的
                    vector arraylist  适合随机访问 除了在尾部插入和删除元素 性能相对较差  ，比如在中间插入一个元素 需要移动后续所有元素
                    linkedList 进行节点插入删除要高效的很多 随机访问性能要比数组慢
                                          
                    list  方便的访问 插入 删除
                    Set  set不允许重复元素  保证唯一性的场合
                    queue/deque  支持先入先出 后入先出
                   TreeSet 代码里默认是利用TreeMap实现的  创建一个Dummy对象present 作为value
                           插入的对象以键的形式放入到TreeMap里面
                      同理HashSet 也是以hashMap基础实现的 
                      treeSET 支持自然顺序访问 但是添加删除 包含等操作相对低效
                      hashset利用hash算法 提供常数时间内的添加删除包含操作 不保证有序
                      LinkedHashSet   内部构建一个记录插入顺序的双向链表 提供了顺序遍历的能力 
                      遍历元素时  hashset 性能受自身容量限制 除非有必要不要将其背后的hashMap容量设置过大
                      LinkedHashSet 遍历元素只和元素多少有关系
			
			     Collections 类中提供synchronized方法 使集合支持并发
			     Collections.sort()->Arrays.sort()(太小数据集 java直接采用二分插入排序)
			       原始数据类型采用双轴快速排序
			       对象数据类型采用TimSort(归并和二分排序)

			       Collections  默认方法的形式实现在里面
			        jdk9 提供静态工厂方法 List.of  没有使用可变参数  可变参数需要jvm额外开销
## hashMap 源码解析
		    整体架构
		      数组+链表+红黑树 其中当链表的长度超过8时并且数组大小大于64时，链表会转换成红黑树，当红黑树的大小小于等于6m会退化为链表	        
		     hashmap 数组的元素可能是个Node 也可能是个链表 也可能是个红黑树。
		    类注释
		      允许null值 不同于hashtable 是线程不安全的
		      loadfactor的默认值时0.75  较高的值会减少空间开销（扩容减少，数组大小增长速度变慢） 但增加了查找成本 （hash 冲突增加，链表长度变长）
		      扩容条件 数组的容量<需要的数组大小/loadfactor
		      如果有很多的数据需要存储到HashMap 建议hashmap的容量一开始就设置较大，减少不断扩容
		      hashmap 是非线程安全的  外部加锁或者使用synchronizedMap
		      迭代过程中，hashMap结构被修改会快速失败
		    常见属性
		      default_initail_capatity 初始容量
		      maximum_capaticty 最大容量
		      负载因子 default_load_factor=0.75
		      桶上的链表长度大于等于8时 链表转换成红黑树 treeify_threshold =8
		      桶上的红黑树大小小于等于6时 转换为链表 untreeify_threshold=6
		      当数组容量大于64时 链表才会转换为红黑树 min_treeify_capacity=64  
		      modCount 记录迭代过程中 hashMap结构是否发生变化 如果有变化，迭代会fail-fast
		      size  hashmap的实际大小可能不准
		      存放数据的数组
		      Node<k,v>[] table
		      //链表的节点
              static class Node<K,V> implements Map.Entry<K,V>
              //红黑树的节点
               static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>
               // 扩容的门槛，有两种情况
              // 如果初始化时，给定数组大小的话，通过 tableSizeFor 方法计算，数组大小永远接近于 2 的幂次方，比如你给定初始化大小     19，实际上初始化大小为 32，为 2 的 5 次方。
             // 如果是通过 resize 方法进行扩容，大小 = 数组容量 * 0.75
             int threshold;
           -新增 
             新增 key，value 大概的步骤如下：

                1空数组有无初始化，没有的话初始化；
                2如果通过 key 的 hash 能够直接找到值，跳转到 6，否则到 3；
                3如果 hash 冲突，两种解决方案：链表 or 红黑树；
                4如果是链表，递归循环，把新元素追加到队尾；
                5如果是红黑树，调用红黑树新增的方法；
                6通过 2、4、5 将新元素追加成功，再根据 onlyIfAbsent 判断是否需要覆盖；
                7判断是否需要扩容，需要扩容进行扩容，结束。 
               // 入参 hash：通过 hash 算法计算出来的值。
               // 入参 onlyIfAbsent：false 表示即使 key 已经存在了，仍然会用新值覆盖原来的值，默认为 false
               final V putVal(int hash, K key, V value, boolean onlyIfAbsent  
                 // n 表示数组的长度，i 为数组索引下标，p 为 i 下标位置的 Node 值
                    Node<K,V>[] tab; Node<K,V> p; int n, i;
                  //如果数组为空，使用 resize 方法初始化
                if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length;
                  // 如果当前索引位置是空的，直接生成新的节点在当前索引位置上
                if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
                // 如果当前索引位置有值的处理方法，即我们常说的如何解决 hash 冲突
                  // e 当前节点的临时变量
                  Node<K,V> e; K k;
                  // 如果 key 的 hash 和值都相等，直接把当前下标位置的 Node 值赋值给临时变量
                 if (p.hash == hash &&
                 ((k = p.key) == key || (key != null && key.equals(k))))
                  e = p;
                  // 如果是红黑树，使用红黑树的方式新增
                  else if (p instanceof TreeNode)
                  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                // 是个链表，把新节点放到链表的尾端
              else {
               // 自旋
                 for (int binCount = 0; ; ++binCount) {
                   // e = p.next 表示从头开始，遍历链表
                  // p.next == null 表明 p 是链表的尾节点
                  if ((e = p.next) == null) {
                    // 把新节点放到链表的尾部 
                    p.next = newNode(hash, key, value, null);
                    // 当链表的长度大于等于 8 时，链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 链表遍历过程中，发现有元素和新增的元素相等，结束循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                //更改循环的当前元素，使 p 在遍历过程中，一直往后移动。
                p = e;
               }
               }
                // 说明新节点的新增位置已经找到了
            if (e != null) {
              V oldValue = e.value;
              // 当 onlyIfAbsent 为 false 时，才会覆盖值 
               if (!onlyIfAbsent || oldValue == null)
                 e.value = value;
               afterNodeAccess(e);
              // 返回老值
              return oldValue;
              }
               // 记录 HashMap 的数据结构发生了变化
               ++modCount;
              //如果 HashMap 的实际大小大于扩容的门槛，开始扩容
             if (++size > threshold)
                 resize();
               afterNodeInsertion(evict);
         -链表新增 、
            链表查询的时间复杂度是 O (n)，红黑树的查询复杂度是 O (log (n))。在链表数据不多的时候，使用链表进行遍历也比较快，只有当链表数据比较多的时候，才会转化成红黑树，但红黑树需要的占用空间是链表的 2 倍，考虑到转化时间和空间损耗，所以我们需要定义出转化的边界值。
            当链表的长度是 8 的时候，出现的概率是 0.00000006，不到千万分之一，所以说正常情况下，链表的长度不可能到达 8 ，而一旦到达 8 时，肯定是 hash 算法出了问题，所以在这种情况下，为了让 HashMap 仍然有较高的查询性能，所以让链表转化成红黑树，我们正常写代码，使用 HashMap 时，几乎不会碰到链表转化成红黑树的情况，毕竟概念只有千万分之一。
         -红黑树新增节点
            1首先判断新增的节点在红黑树上是不是已经存在，判断手段有如下两种：
              1.1. 如果节点没有实现 Comparable 接口，使用 equals 进行判断；
              1.2. 如果节点自己实现了 Comparable 接口，使用 compareTo 进行判断。
            2新增的节点如果已经在红黑树上，直接返回；不在的话，判断新增节点是在当前节点的左边还是右边，左边值小，右边值大；
            3自旋递归 1 和 2 步，直到当前节点的左边或者右边的节点为空时，停止自旋，当前节点即为我们新增节点的父节点；
            4把新增节点放到当前节点的左边或右边为空的地方，并于当前节点建立父子节点关系；
            5进行着色和旋转，结束

            红黑树的新增，要求大家对红黑树的数据结构有一定的了解。面试的时候，一般只会问到新增节点到红黑树上大概是什么样的一个过程，着色和旋转的细节不会问，因为很难说清楚，但我们要清楚着色指的是给红黑树的节点着上红色或黑色，旋转是为了让红黑树更加平衡，提高查询的效率，总的来说都是为了满足红黑树的 5 个原则：

              节点是红色或黑色
              根是黑色
              所有叶子都是黑色
              从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点
              从每个叶子到根的所有路径上不能有两个连续的红色节点
         //balanceInsertion 对红黑树进行着色或旋转，以达到更多的查找效率，着色或旋转的几种场景如下
            //着色：新节点总是为红色；如果新节点的父亲是黑色，则不需要重新着色；
            如果父亲是红色，那么必须通过重新着色或者旋转的方法，再次达到红黑树的5个约束条件
            //旋转： 父亲是红色，叔叔是黑色时，进行旋转
            //如果当前节点是父亲的右节点，则进行左旋
            //如果当前节点是父亲的左节点，则进行右旋
           -查找
             HashMap 的查找主要分为以下三步：
               根据 hash 算法定位数组的索引位 置，equals 判断当前节点是否是我们需要寻找的 key，是的话直接返回，不是的话往下。
               判断当前节点有无 next 节点，有的话判断是链表类型，还是红黑树类型。
               分别走链表和红黑树不同类型的查找方法。
             红黑树查找
                红黑树查找的代码很多，我们大概说下思路，实际步骤比较复杂，可以去 github 上面去查看源码：
                    从根节点递归查找；
                    根据 hashcode，比较查找节点，左边节点，右边节点之间的大小，根本红黑树左小右大的特性进行判断；
                     判断查找节点在第 2 步有无定位节点位置，有的话返回，没有的话重复 2，3 两步；
                    一直自旋到定位到节点位置为止。
                    如果红黑树比较平衡的话，每次查找的次数就是树的深度
		-- hashtable hashmap treemap
			    以键值对存储和操作数据的容器类型
			    hashtable 早期的hash表实现 本身同步 不支持null键值对
                hashmap   不是同步的 支持null 键和值
                   hasnmap进行put get操作 可以达到常数时间的性能  绝大多数减脂存储的首选
                treeMap 基于红黑树的一种提供顺序访问的map 它的get put remove操作都是哦（log(n)）时间复杂度
                   具体顺序由指定的Comparator来决定 或者根据键的自然顺序来判断
                  
                map整体结构
                     dictionary 
               hashtable->properties
			   abstractMap
			                 enumMap
			                 hashMap ->linkedHashMap
	          sortedMap->NavigableMap->treeMap 
                 hashmap的性能表现非常依赖于hash码的有效性
                    equals 相等 hashcode一定要相等
                    重写hashcode 也要重写equals
                linkedHashMap 和TreeMap  都能保证某种顺序
                   linkedHashMap 提供遍历顺序符合插入顺序 通过为键值对维护一个双向链表
                    构建一个空间占用敏感的资源池 希望自动将最不常访问的对象释放掉
                    treeMap 它的整体顺序由简的顺序关系决定的 通过Comparable(自然顺序) Comparator
                     有优先级的调度系统 PriorityQueue
                      put 方法 k.compareTo(t.key)
                 hashMap源码分析
                       hashmap 内部实现基本点分析
                           数组Node<k,v>[] table和链表结合组成的复合结构，数组被分成了一个个桶 通过hash值决定了键值对在这个数组上的寻址；
                           哈希值相同的键值对 以链表形式存储；如果链表大小超过阈值8 会被改造成树形结构
                         put->putval  
                            resize-> 创建初始化表格 或者表格容量不满足需要时扩容
                               门限值 等于负载因子 * 容量
                               门限以倍数进行调整
                               扩容后需要拷贝数组
                            hash 寻址忽略容量以上的高位 避免hash碰撞

                       容量和负载系数
                          容量和负载系数决定了可用的桶的数量 空桶太多会浪费空间，使用太慢会影响操作性能假如有一个桶
                          那么不能提供常数时间存的性能
                            建议；负载因子不要超过0.75数值 显示增加冲突 太小会更加频繁的扩容


                       树华 
                            putval->treeIfyBin
                            如果容量小于Min-treeify-capacity 进行简单扩容
                            容量大于Min-treeify-capacity 进行树化改造

                        为什么树华
                            链表查询是线性的     严重影响存取的性能                      
## treeMap LinkhashMap
              treeMap 是如何根据 key 进行排序的，LinkedHashMap 是如何用两种策略进行访问的。
              知识储备
                两种排序
                 实现comparable接口
                 利用外部排序器 Comparator comparator = (Comparator<DTO>) (o1, o2) -> o2.getId() - o1.getId();
             -treeMap整体架构
                TreeMap 底层的数据结构就是红黑树，和 HashMap 的红黑树结构一样。
                 不同的是，TreeMap 利用了红黑树左节点小，右节点大的性质，根据 key 进行排序，使每个元素能够插入到红黑树大小适当的位置，维护了 key 的大小关系，适用于 key 需要排序的场景。
                 因为底层使用的是平衡红黑树的结构，所以 containsKey、get、put、remove 等方法的时间复杂度都是 log(n)。
               属性
                 //比较器，如果外部有传进来 Comparator 比较器，首先用外部的
                 //如果外部比较器为空，则使用 key 自己实现的 Comparable#compareTo 方法
                 //比较手段和上面日常工作中的比较 demo 是一致的
                 private final Comparator<? super K> comparator;
                 //红黑树的根节点
                  private transient Entry<K,V> root;
                //红黑树的已有元素大小
                 private transient int size = 0;
                //树结构变化的版本号，用于迭代过程中的快速失败场景
                  private transient int modCount = 0;
                 //红黑树的节点
                  static final class Entry<K,V> implements Map.Entry<K,V> {}  
               新增节点
                 判断红黑树的节点是否为空，为空的话，新增的节点直接作为根节点
                 根据红黑树左小右大的特性，进行判断，找到应该新增节点的父节点，
                 在父节点的左边或右边插入新增节点，  
                 着色旋转，达到平衡，结束。

                 新增节点时，就是利用了红黑树左小右大的特性，从根节点不断往下查找，直到找到节点是 null 为止，节点为 null 说明到达了叶子结点；
                 查找过程中，发现 key 值已经存在，直接覆盖；
                 TreeMap 是禁止 key 是 null 值的。 

                 TreeMap 相对来说比较简单，红黑树和 HashMap 比较类似，比较关键的是通过 compare 来比较 key 的大小，然后利用红黑树左小右大的特性，为每个 key 找到自己的位置，从而维护了 key 的大小排序顺序

              -LinkedHashMap   
                 LinkedHashMap 本身是继承 HashMap 的，所以它拥有 HashMap 的所有特性，再此基础上，还提供了两大特性：
                   按照插入顺序进行访问；
                   实现了访问最少最先删除功能，其目的是把很久都没有访问的 key 自动删除。
                 -按照插入顺序访问
                    LinkedHashMap 链表结构
                     属性
                       // 链表头
                       transient LinkedHashMap.Entry<K,V> head;
                      // 链表尾
                      transient LinkedHashMap.Entry<K,V> tail;
                       // 继承 Node，为数组的每个元素增加了 before 和 after 属性
                static class Entry<K,V> extends HashMap.Node<K,V> {
                    Entry<K,V> before, after;
                 Entry(int hash, K key, V value, Node<K,V> next) {
                      super(hash, key, value, next);
                  }
                   }
                   / / 控制两种访问模式的字段，默认 false
                   // true 按照访问顺序，会把经常访问的 key 放到队尾
                  / / false 按照插入顺序提供访问
                    final boolean accessOrder;
                    LinkedHashMap 的数据结构很像是把 LinkedList 的每个元素换成了 HashMap 的 Node，像是两者的结合体，也正是因为增加了这些结构，从而能把 Map 的元素都串联起来，形成一个链表，而链表就可以保证顺序了，就可以维护元素插入进来的顺序。
                 -按照顺序新增
                    LinkedHashMap 初始化时，默认 accessOrder 为 false，就是会按照插入顺序提供访问，插入方法使用的是父类 HashMap 的 put 方法，不过覆写了 put 方法执行中调用的 newNode/newTreeNode 和 afterNodeAccess 方法。   
                    newNode/newTreeNode 方法，控制新增节点追加到链表的尾部，这样每次新节点都追加到尾部，即可保证插入顺序了
                        // 链表有数据，直接建立新增节点和上个尾节点之间的前后关系即可
                     else {
                       p.before = last;
                       last.after = p;
                    }
                 -按照顺序访问
                   LinkedHashMap 只提供了单向访问，即按照插入的顺序从头到尾进行访问，不能像 LinkedList 那样可以双向访问。
                   我们主要通过迭代器进行访问，迭代器初始化的时候，默认从头节点开始访问，在迭代的过程中，不断访问当前节点的 after 节点即可。
                   Map 对 key、value 和 entity（节点） 都提供出了迭代的方法，假设我们需要迭代 entity，就可使用 LinkedHashMap.entrySet().iterator() 这种写法直接返回 LinkedHashIterator ，LinkedHashIterator 是迭代器，我们调用迭代器的 nextNode 方法就可以得到下一个节点
               访问最少删除策略
                     LRU（Least recently used,最近最少使用），大概的意思就是经常访问的元素会被追加到队尾，这样不经常访问的数据自然就靠近队头，然后我们可以通过设置删除策略，比如当 Map 元素个数大于多少时，把头节点删除  
                       
                    // 新建 LinkedHashMap
                      LinkedHashMap<Integer, Integer> map = new LinkedHashMap<Integer, Integer>(4,0.75f,true) {
                    {
                       put(10, 10);
                       put(9, 9);
                       put(20, 20);
                       put(1, 1);
                     }

                  @Override
               // 覆写了删除策略的方法，我们设定当节点个数大于 3 时，就开始删除头节点
                   protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                      return size() > 3;
                  }
                    };

                    元素被转移到队尾
                     if (accessOrder)
                   // 这个方法把当前 key 移动到队尾
                    afterNodeAccess(e);
                   通过 afterNodeAccess 方法把当前访问节点移动到了队尾，其实不仅仅是 get 方法，执行 getOrDefault、compute、computeIfAbsent、computeIfPresent、merge 方法时，也会这么做，通过不断的把经常访问的节点移动到队尾，那么靠近队头的节点，自然就是很少被访问的元素了。

                 删除策略
                     发现队头元素被删除了，LinkedHashMap 本身是没有 put 方法实现的，调用的是 HashMap 的 put 方法，但 LinkedHashMap 实现了 put 方法中的调用 afterNodeInsertion 方法，这个方式实现了删除，
                        // removeNode 删除头节点
                       removeNode(hash(key), key, null, false, true);
## map 源码会问的问题

           - map整体数据结构类问题
              1 HashMap 底层数据结构
                 HashMap 底层是数组 + 链表 + 红黑树的数据结构，数组的主要作用是方便快速查找，时间复杂度是 O(1)，默认大小是 16，数组的下标索引是通过 key 的 hashcode 计算出来的，数组元素叫做 Node，当多个 key 的 hashcode 一致，但 key 值不同时，单个 Node 就会转化成链表，链表的查询复杂度是 O(n)，当链表的长度大于等于 8 并且数组的大小超过 64 时，链表就会转化成红黑树，红黑树的查询复杂度是 O(log(n))，简单来说，最坏的查询次数相当于红黑树的最大深度。
              2 HashMap、TreeMap、LinkedHashMap 三者有啥相同点，有啥不同点？
                 相同点：
                  三者在特定的情况下，都会使用红黑树；
                  底层的 hash 算法相同；
                  在迭代的过程中，如果 Map 的数据结构被改动，都会报 ConcurrentModificationException 的错误。
                不同点：
                   HashMap 数据结构以数组为主，查询非常快，TreeMap 数据结构以红黑树为主，利用了红黑树左小右大的特点，可以实现 key 的排序，LinkedHashMap 在 HashMap 的基础上增加了链表的结构，实现了插入顺序访问和最少访问删除两种策略;
                   由于三种 Map 底层数据结构的差别，导致了三者的使用场景的不同，TreeMap 适合需要根据 key 进行排序的场景，LinkedHashMap 适合按照插入顺序访问，或需要删除最少访问元素的场景，剩余场景我们使用 HashMap 即可，我们工作中大部分场景基本都在使用 HashMap；
                   由于三种 map 的底层数据结构的不同，导致上层包装的 api 略有差别。    
              3 map的hash算法
                  (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
                  源码中就是通过以上代码来计算 hash 的，首先计算出 key 的 hashcode，因为 key 是 Object，所以会根据 key 的不同类型进行 hashcode 的计算，接着计算 h ^ (h >>> 16) ，这么做的好处是使大多数场景下，算出来的 hash 值比较分散。   
                  key 在数组中的位置公式：tab[(n - 1) & hash]  

                  hash 值算出来之后，要计算当前 key 在数组中的索引下标位置时，可以采用取模的方式，就是索引下标位置 = hash 值 % 数组大小，这样做的好处，就是可以保证计算出来的索引下标值可以均匀的分布在数组的各个索引位置上，但取模操作对于处理器的计算是比较慢的，数学上有个公式，当 b 是 2 的幂次方时，a % b = a &（b-1），所以此处索引位置的计算公式我们可以更换为： (n-1) & hash。

                  1.1为什么不用 key % 数组大小，而是需要用 key 的 hash 值 % 数组大小。
                    如果 key 是数字，直接用 key % 数组大小是完全没有问题的，但我们的 key 还有可能是字符串，是复杂对象，这时候用 字符串或复杂对象 % 数组大小是不行的，所以需要先计算出 key 的 hash 值。

                   1.2 计算 hash 值时，为什么需要右移 16 位？
                  hash 算法是 h ^ (h >>> 16)，为了使计算出的 hash 值更分散，所以选择先将 h 无符号右移 16 位，然后再于 h 异或时，就能达到 h 的高 16 位和低 16 位都能参与计算，减少了碰撞的可能性。
                  1.3 为什么提倡数组大小是 2 的幂次方？
                  因为只有大小是 2 的幂次方时，才能使 hash 值 % n(数组大小) == (n-1) & hash 公式成立。
              4 解决hash冲突 大概有哪些办法
                   1：好的 hash 算法，细问的话复述一下上题的 hash 算法;
                   2：自动扩容，当数组大小快满的时候，采取自动扩容，可以减少 hash 冲突;
                   3：hash 冲突发生时，采用链表来解决;
                   4：hash 冲突严重时，链表会自动转化成红黑树，提高遍历速度。
          -hashMap源码细节类问题
             hashmap 是如何扩容的 
              扩容的时机：
                put 时，发现数组为空，进行初始化扩容，默认扩容大小为 16;
                put 成功后，发现现有数组大小大于扩容的门阀值时，进行扩容，扩容为老数组大小的 2 倍;
               扩容的门阀是 threshold，每次扩容时 threshold 都会被重新计算，门阀值等于数组的大小 * 影响因子（0.75）。
               新数组初始化之后，需要将老数组的值拷贝到新数组上，链表和红黑树都有自己拷贝的方法
             hash冲突时怎么办？
                 hash 冲突指的是 key 值的 hashcode 计算相同，但 key 值不同的情况。
                 如果桶中元素原本只有一个或已经是链表了，新增元素直接追加到链表尾部；
                 如果桶中元素已经是链表，并且链表个数大于等于 8 时，此时有两种情况：
                   如果此时数组大小小于 64，数组再次扩容，链表不会转化成红黑树;
                   如果数组大小大于 64 时，链表就会转化成红黑树。
                这里不仅仅判断链表个数大于等于 8，还判断了数组大小，数组容量小于 64 没有立即转化的原因，猜测主要是因为红黑树占用的空间比链表大很多，转化也比较耗时，所以数组容量小的情况下冲突严重，我们可以先尝试扩容，看看能否通过扩容来解决冲突的问题。
              
              为什么链表个数大于等于 8 时，链表要转化成红黑树了？
                 当链表个数太多了，遍历可能比较耗时，转化成红黑树，可以使遍历的时间复杂度降低，但转化成红黑树，有空间和转化耗时的成本，我们通过泊松分布公式计算，正常情况下，链表个数出现 8 的概念不到千万分之一，所以说正常情况下，链表都不会转化成红黑树，这样设计的目的，是为了防止非正常情况下，比如 hash 算法出了问题时，导致链表个数轻易大于等于 8 时，仍然能够快速遍历。
                延伸问题：红黑树什么时候转变成链表。
                  当节点的个数小于等于 6 时，红黑树会自动转化成链表，主要还是考虑红黑树的空间成本问题，当节点个数小于等于 6 时，遍历链表也很快，所以红黑树会重新变成链表。
              HashMap 在 put 时，如果数组中已经有了这个 key，我不想把 value 覆盖怎么办？取值时，如果得到的 value 是空时，想返回默认值怎么办？    
                  如果数组有了 key，但不想覆盖 value ，可以选择 putIfAbsent 方法，这个方法有个内置变量 onlyIfAbsent，内置是 true ，就不会覆盖，我们平时使用的 put 方法，内置 onlyIfAbsent 为 false，是允许覆盖的。
                 取值时，如果为空，想返回默认值，可以使用 getOrDefault 方法，方法第一参数为 key，第二个参数为你想返回的默认值，如 map.getOrDefault(“2”,“0”)，当 map 中没有 key 为 2 的值时，会默认返回 0，而不是空。
              删除不适用增强for
                 建议使用迭代器的方式进行删除，原理同 ArrayList 迭代器原理
         -其他类型
             DTO 作为 Map 的 key 时，有无需要注意的点？
                DTO 就是一个数据载体，可以看做拥有很多属性的 Java 类，我们可以对这些属性进行 get、set 操作。
                看是什么类型的 Map，如果是 HashMap 的话，一定需要覆写 equals 和 hashCode 方法，因为在 get 和 put 的时候，需要通过 equals 方法进行相等的判断；如果是 TreeMap 的话，DTO 需要实现 Comparable 接口，因为 TreeMap 会使用 Comparable 接口进行判断 key 的大小；如果是 LinkedHashMap 的话，和 HashMap 一样的。
                     
             LinkedHashMap 中的 LRU 是什么意思，是如何实现的。
               LRU ，英文全称：Least recently used，中文叫做最近最少访问，在 LinkedHashMap 中，也叫做最少访问删除策略，我们可以通过 removeEldestEntry 方法设定一定的策略，使最少被访问的元素，在适当的时机被删除，原理是在 put 方法执行的最后，LinkedHashMap 会去检查这种策略，如果满足策略，就删除头节点。
                保证头节点就是最少访问的元素的原理是：LinkedHashMap 在 get 的时候，都会把当前访问的节点，移动到链表的尾部，慢慢的，就会使头部的节点都是最少被访问的元素。

             为什么推荐 TreeMap 的元素最好都实现 Comparable 接口？但 key 是 String 的时候，我们却没有额外的工作呢？
                 因为 TreeMap 的底层就是通过排序来比较两个 key 的大小的，所以推荐 key 实现 Comparable 接口，是为了往你希望的排序顺序上发展， 而 String 本身已经实现了 Comparable 接口，所以使用 String 时，我们不需要额外的工作，不仅仅是 String ，其他包装类型也都实现了 Comparable 接口，如 Long、Double、Short 等等。   
## hashSet TreeSet源码解析
            HashSet、TreeSet 两个类是在 Map 的基础上组装起来的类      
            1 HashSet
             类注释
              底层实现基于 HashMap，所以迭代时不能保证按照插入顺序，或者其它顺序进行迭代；
              add、remove、contanins、size 等方法的耗时性能，是不会随着数据量的增加而增加的，这个主要跟 HashMap 底层的数组数据结构有关，不管数据量多大，不考虑 hash 冲突的情况下，时间复杂度都是 O (1)；
              线程不安全的，如果需要安全请自行加锁，或者使用 Collections.synchronizedSet；
              迭代过程中，如果数据结构被改变，会快速失败的，会抛出 ConcurrentModificationException 异常
             hashset 时如何组合hashMap的 
              在 Java 中，要基于基础类进行创新实现，有两种办法：
              继承基础类，覆写基础类的方法，比如说继承 HashMap , 覆写其 add 的方法；
              组合基础类，通过调用基础类的方法，来复用基础类的能力。   
             HashSet 使用的就是组合 HashMap，其优点如下：
               继承表示父子类是同一个事物，而 Set 和 Map 本来就是想表达两种事物，所以继承不妥，而且 Java 语法限制，子类只能继承一个父类，后续难以扩展。
               组合更加灵活，可以任意的组合现有的基础类，并且可以在基础类方法的基础上进行扩展、编排等，而且方法命名可以任意命名，无需和基础类的方法名称保持一致 
               如果碰到类似问题，我们的原则也是尽量多用组合，少用继承 
               组合就是把 HashMap 当作自己的一个局部变量，以下是 HashSet 的组合实现
               // 把 HashMap 组合进来，key 是 Hashset 的 key，value 是下面的 PRESENT
               private transient HashMap<E,Object> map;
               // HashMap 中的 value
               private static final Object PRESENT = new Object();
               从这两行代码中，我们可以看出两点：
                我们在使用 HashSet 时，比如 add 方法，只有一个入参，但组合的 Map 的 add 方法却有 key，value 两个入参，相对应上 Map 的 key 就是我们 add 的入参，value 就是第二行代码中的 PRESENT，此处设计非常巧妙，用一个默认值 PRESENT 来代替 Map 的 Value；
                如果 HashSet 是被共享的，当多个线程访问的时候，就会有线程安全问题，因为在后续的所有操作中，并没有加锁。
             初始化
                HashSet 的初始化比较简单，直接 new HashMap 即可
                map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
                和 16 比较大小的意思是说，如果给定 HashMap 初始容量小于 16 ，就按照 HashMap 默认的 16 初始化好了，如果大于 16，就按照给定值初始化。
                HashMap 扩容的伐值的计算公式是：Map 的容量 * 0.75f，一旦达到阀值就会扩容，此处用 (int) (c.size ()/.75f) + 1 来表示初始化的值，这样使我们期望的大小值正好比扩容的阀值还大 1，就不会扩容，符合 HashMap 扩容的公式

                往 HashMap 拷贝大集合时，如何给 HashMap 初始化大小时，完全可以借鉴这种写法：取最大值（期望的值 / 0.75 + 1，默认值 16）。
                
                // 直接使用 HashMap 的 put 方法，进行一些简单的逻辑判断
                return map.put(e, PRESENT)==null;
              hashset小结
                对组合还是继承的分析和把握；
                对复杂逻辑进行一些包装，使吐出去的接口尽量简单好用；
                组合其他 api 时，尽量多对组合的 api 多些了解，这样才能更好的使用 api；
                HashMap 初始化大小值的模版公式：取括号内两者的最大值（期望的值 / 0.75+1，默认值 16）。 

             2 treeSet
                底层组合的是 TreeMap，所以继承了 TreeMap key 能够排序的功能，迭代的时候，也可以按照 key 的排序顺序进行迭代

                复用treeMap的思路
                 1 需要迭代 TreeSet 中的元素，那应该也是像 add 那样，直接使用 HashMap 已有的迭代能力
                    // 直接使用 HashMap.keySet 的迭代能力
                     return m.keySet().iterator();
                 2  // NavigableSet 接口，定义了迭代的一些规范，和一些取值的特殊方法 interface NavigableSet<E> extends SortedSet<E> 
                     // TreeSet 实现了该方法，也就是说 TreeSet 本身已经定义了迭代的规范 
                     这种思路是 TreeSet 定义了接口的规范，TreeMap 负责去实现，实现思路和思路一是相反的。
                   TreeSet 组合 TreeMap 实现的两种思路：
                    TreeSet 直接使用 TreeMap 的某些功能，自己包装成新的 api。
                    TreeSet 定义自己想要的 api，自己定义接口规范，让 TreeMap 去实现。  
                   方案 1 和 2 的调用关系，都是 TreeSet 调用 TreeMap，但功能的实现关系完全相反，第一种是功能的定义和实现都在 TreeMap，TreeSet 只是简单的调用而已，第二种 TreeSet 把接口定义出来后，让 TreeMap 去实现内部逻辑，TreeSet 负责接口定义，TreeMap 负责具体实现，这样子的话因为接口是 TreeSet 定义的，所以实现一定是 TreeSet 最想要的，TreeSet 甚至都不用包装，可以直接把返回值吐出去都行 
                  两种复用思路的原因
                   像 add 这些简单的方法，我们直接使用的是思路 1，主要是 add 这些方法实现比较简单，没有复杂逻辑，所以 TreeSet 自己实现起来比较简单；
                   思路 2 主要适用于复杂场景，比如说迭代场景，TreeSet 的场景复杂，比如要能从头开始迭代，比如要能取第一个值，比如要能取最后一个值，再加上 TreeMap 底层结构比较复杂，TreeSet 可能并不清楚 TreeMap 底层的复杂逻辑，这时候让 TreeSet 来实现如此复杂的场景逻辑，TreeSet 就搞不定了，不如接口让 TreeSet 来定义，让 TreeMap 去负责实现，TreeMap 对底层的复杂结构非常清楚，实现起来既准确又简单。 

                   思路二 场景 dubbo 的泛化调用，DDD 中的依赖倒置
              3 面试
                TreeSet 有用过么，平时都在什么场景下使用？
                  有木有用过如实回答就好了，我们一般都是在需要把元素进行排序的时候使用 TreeSet，使用时需要我们注意元素最好实现 Comparable 接口，这样方便底层的 TreeMap 根据 key 进行排序。 
                如果我想实现根据 key 的新增顺序进行遍历怎么办？
                   要按照 key 的新增顺序进行遍历，首先想到的应该就是 LinkedHashMap，而 LinkedHashSet 正好是基于 LinkedHashMap 实现的，所以我们可以选择使用 LinkedHashSet。    
                 如果我想对 key 进行去重，有什么好的办法么？
                   我们首先想到的是 TreeSet，TreeSet 底层使用的是 TreeMap，TreeMap 在 put 的时候，如果发现 key 是相同的，会把 value 值进行覆盖，所有不会产生重复的 key ，利用这一特性，使用 TreeSet 正好可以去重。  
                 TreeSet 和 HashSet 两个 Set 的内部实现结构和原理？
                    HashSet 底层对 HashMap 的能力进行封装，比如说 add 方法，是直接使用 HashMap 的 put 方法，比较简单，但在初始化的时候，我看源码有一些感悟：说一下 HashSet 小结的四小点。
                    TreeSet 主要是对 TreeMap 底层能力进行封装复用，我发现了两种非常有意思的复用思路，重复 TreeSet 两种复用思路。  
## 集合源码作用
              集合类图
             Map
              sortedMap  navigableMap  treemap
               abstractMap hashMap linkedHashMap 
                
             iterable-> collection-> set abstractCollection List queue 
             
                    abstractList-> vetor->stack
                                -> arryaList
                                -> abstractSequeuntialList->linkedList(dequeue)   
                 set->
                     abstractSet -> hashSet->linkedSet
                     sortedSet ->navigabltSet-> treeSet
                    
                    queue->deque -> arrayDeque
                         ->abstractQueue->priorityQueue    

                以下几点：
                  每个接口做的事情非常明确，比如 Serializable，只负责序列化，Cloneable 只负责拷贝，Map 只负责定义 Map 的接口，整个图看起来虽然接口众多，但职责都很清晰；
                  复杂功能通过接口的继承来实现，比如 ArrayList 通过实现了 Serializable、Cloneable、RandomAccess、AbstractList、List 等接口，从而拥有了序列化、拷贝、对数组各种操作定义等各种功能；
                  上述类图只能看见继承的关系，组合的关系还看不出来，比如说 Set 组合封装 Map 的底层能力等
             集合中需要注意事项
                线程安全
                  如果要实现线程安全的集合，在类注释中，JDK 统一推荐我们使用 Collections.synchronized* 类， Collections 帮我们实现了 List、Set、Map 对应的线程安全的方法，
                  // 我们可以看到，List 的所有操作都使用了 synchronized 关键字，来进行加锁
				 // synchronized 是一种悲观锁，能够保证同一时刻，只能有一个线程能够获得锁     
				集合性能
				  批量新增
				     list.add(i);
				     list3.addAll(list);
				     批量新增方法性能是单个新增方法性能的 189 倍，主要原因在于批量新增，只会扩容一次，大大缩短了运行时间，而单个新增，每次到达扩容阀值时，都会进行扩容，在整个过程中就会不断的扩容，浪费了很多时间
                        // 确保容量充足，整个过程只会扩容一次
                         ensureCapacityInternal(size + numNew); 
                       // 进行数组的拷贝
                      System.arraycopy(a, 0, elementData, size, numNew);
				     HashMap 的 putAll 方法也是如此，整个新增过程只会扩容一次，大大缩短了批量新增的时间，提高了性能。
                      当碰到集合批量拷贝，批量新增场景，如何提高新增性能的时候 ，就可以从目标集合初始化方面应答。

                   批量删除
                     批量删除 ArrayList 提供了 removeAll 的方法
                       // 批量删除，removeAll 方法底层调用的是 batchRemove 方法
                       // complement 参数默认是 false,false 的意思是数组中不包含 c 中数据的节点往头移动
                       / / true 意思是数组中包含 c 中数据的节点往头移动，这个是根据你要删除数据和原数组大小的比例来决定的
                     // 如果你要删除的数据很多，选择 false 性能更好，当然 removeAll 方法默认就是 false。

                      ArrayList 在批量删除时，如果程序执行正常，只有一次 for 循环，如果程序执行异常，才会加一次拷贝，而单个 remove 方法，每次执行的时候都会进行数组的拷贝（当删除的元素正好是数组最后一个元素时除外），当数组越大，需要删除的数据越多时，批量删除的性能会越差，所以在 ArrayList 批量删除时，强烈建议使用 removeAll 方法进行删除

                 集合的一些坑
                   1 当集合的元素是自定义类时，自定义类强制实现 equals 和 hashCode 方法，并且两个都要实现。
                      在集合中，除了 TreeMap 和 TreeSet 是通过比较器比较元素大小外，其余的集合类在判断索引位置和相等时，都会使用到 equals 和 hashCode 方法，这个在之前的源码解析中，我们有说到，所以当集合的元素是自定义类时，我们强烈建议覆写 equals 和 hashCode 方法，我们可以直接使用 IDEA 工具覆写这两个方法，非常方便；
                   2 所有集合类，在 for 循环进行删除时，如果直接使用集合类的 remove 方法进行删除，都会快速失败，报 ConcurrentModificationException 的错误，所以在任意循环删除的场景下，都建议使用迭代器进行删除；   
                   3  我们把数组转化成集合时，常使用 Arrays.asList(array)，这个方法有两个坑
                      Integer[] array = new Integer[]{1,2,3,4,5,6};
                      List<Integer> list = Arrays.asList(array);
                      坑 1：数组被修改后，会直接影响到新 List 的值。
                      坑 2：不能对新 List 进行 add、remove 等操作，否则运行时会报 UnsupportedOperationException 错误。
                      Arrays.asList 方法返回的 List 并不是 java.util.ArrayList，而是自己内部的一个静态类，该静态类直接持有数组的引用，并且没有实现 add、remove 等方法，这些就是坑 1 和 2 的原因。
                   4 集合 List 转化成数组，我们通常使用 toArray 这个方法，这个方法很危险   
                     // 无法向下转化成 List<Integer>，编译都无法通过
                     // List<Integer> list2 = list.toArray();


                    // 演示数组初始化大小大于实际所需大小，也可以转化成数组
                     Integer[] array2 = new Integer[list.size()+2];
                     list.toArray(array2);

                     如果返回的数组大小和申明的数组大小一致，那么就会正常返回，否则，一个新数组就会被分配返回。
                     所以我们在使用有参 toArray 方法时，申明的数组大小一定要大于等于 List 的大小，如果小于的话，你会得到一个空数组。
## 集合在java7 java8中的不同
           通用区别
             1 所有集合都新增了forEach方法
             List、Set、Map 在 Java 8 版本中都增加了 forEach 的方法，方法的入参是 Consumer，Consumer 是一个函数式接口，可以简单理解成允许一个入参，但没有返回值的函数式接口
                 public void forEach(Consumer<? super E> action) {
                 	 // 执行循环内容，action 代表我们要干的事情
                    action.accept(elementData[i]);
              forEach 方法上打了 @Override 注解，说明该方法是被继承实现的，该方法是被定义在 Iterable 接口上的，Java 7 和 8 的 ArrayList 都实现了该接口，但我们在 Java 7 的 ArrayList 并没有发现有实现该方法，编译器也木有报错，这个主要是因为 Iterable 接口的 forEach 方法被加上了 default 关键字，这个关键字只会出现在接口类中，被该关键字修饰的方法无需强制要求子类继承，但需要自己实现默认实现，
                  不仅仅是 forEach 这一个方法是这么干的，List、Set、Map 接口中很多新增的方法都是这么干的，通过 default 关键字，可以让 Java 7 的集合子类无需实现 Java 8 中新增的方法
                  如果想在接口中新增一个方法，但又不想子类强制实现该方法时，可以给该方法加上 default 关键字，  
              2 List区别
                ArrayList    
                  ArrayList 无参初始化时，Java 7 是直接初始化 10 的大小，Java 8 去掉了这个逻辑，初始化时是空数组，在第一次 add 时才开始按照 10 进行扩容
              3 map区别
                  和 ArrayList 一样，Java 8 中 HashMap 在无参构造器中，丢弃了 Java 7 中直接把数组初始化 16  的做法，而是采用在第一次新增的时候，才开始扩容数组大小；
                  
                  hash 算法计算公式不同，Java 8 的 hash 算法更加简单，代码更加简洁；
                  
                  Java 8 的 HashMap 增加了红黑树的数据结构，这个是 Java 7 中没有的，Java 7 只有数组 + 链表的结构，Java 8 中提出了数组 + 链表 
                  + 红黑树的结构，一般 key 是 Java 的 API 时，比如说 String 这些 hashcode 实现很好的 API，很少出现链表转化成红黑树的情况，因为 String 这些 API 的 hash 算法够好了，只有当 key 是我们自定义的类，而且我们覆写的 hashcode 算法非常糟糕时，才会真正使用到红黑树，提高我们的检索速度

                  新增了一些好用的方法，比如 getOrDefault， 还有 putIfAbsent(K key, V value) 方法，意思是，如果 map 中存在 key 了，那么 value 就不会覆盖，如果不存在 key ，新增成功
                  还有 compute 方法，意思是允许我们把 key 和 value 的值进行计算后，再 put 到 map 中，为防止 key 值不存在造成未知错误，map 还提供了 computeIfPresent 方法，表示只有在 key 存在的时候，才执行计算
                   map.compute(10,(key,value) -> key * value);
                    // 为了防止 key 不存在时导致的未知异常，我们一般有两种办法
                   // 1：自己判断空指针
                    map.compute(11,(key,value) -> null == value ? null : key * value);
                  // 2：computeIfPresent 方法里面判断
                   map.computeIfPresent(11,(key,value) -> key * value);

                  linkedHashMap   
                 其他区别
                   Java 8 的 Arrays 提供了一些 parallel 开头的方法，这些方法支持并行的计算，在数据量大的时候，会充分利用 CPU ，提高计算效率，比如说 parallelSort 方法，方法底层有判断，只有数据量大于 8192 时，才会真正走并行的实现

              面试
                 java 8 在 List、Map 接口上新增了很多方法，为什么 Java 7 中这些接口的实现者不需要强制实现这些方法呢？
                    主要是因为这些新增的方法被 default 关键字修饰了，default 一旦修饰接口上的方法，我们需要在接口的方法中写默认实现，并且子类无需强制实现这些方法，所以 Java 7 接口的实现者无需感知。  
                  
                 说说 computeIfPresent 方法的使用姿势？
                   computeIfPresent 是可以对 key 和 value 进行计算后，把计算的结果重新赋值给 key，并且如果 key 不存在时，不会报空指针，会返回 null 值。
                 Java 8 集合新增了 forEach 方法，和普通的 for 循环有啥不同？
                    新增的 forEach 方法的入参是函数式的接口，比如说 Consumer 和 BiConsumer，这样子做的好处就是封装了 for 循环的代码，让使用者只需关注实现每次循环的业务逻辑，简化了重复的 for 循环代码，使代码更加简洁，普通的 for 循环，每次都需要写重复的 for 循环代码，forEach 把这种重复的计算逻辑吃掉了，使用起来更加方便。          
                 shMap 8 和 7 有啥区别？
                    HashMap 8 和 7 的差别太大了，新增了红黑树，修改了底层数据逻辑，修改了 hash 
                    算法，几乎所有底层数组变动的方法都重写了一遍，可以说 Java 8 的 HashMap 几乎重新了一遍。
## Guava中使用
           1 运用工厂模式进行初始化
               JDK 7 之前，我们新建集合类时，声明和初始化都必须写上泛型说明，像这样：List<泛型> list = new ArrayList<泛型>(); ， JDK 7 之后有所改变，我们只需要在声明处写上泛型说明，像这样：List<泛型> list = new ArrayList<>();   

               Guava 提供了更加方便的使用姿势，采用了工厂模式，把集合创建的逻辑交给了工厂，开发者无需关注工厂底层是如何创建的，只需要关心，工厂能产生什么，代码于是变成了这样：List<泛型> list = Lists.newArrayList();，Lists 就是 Guava 提供出来的，方便操作 List 
               的工具类。       
           2 Lists 
             初始化 
                 如果你清楚 List 的大小，我们也可以这样做：
                 // 可以预估 list 的大小为 20
                 List<String> list = Lists.newArrayListWithCapacity(20);
                 // 不太肯定 list 大小是多少，但期望是大小是 20 上下。
                 List<String> list = Lists.newArrayListWithExpectedSize(20);
                 Lists 在初始化的时候，还支持传迭代器的入参（只适合小数据量的迭代器的入参 
                   <E> ArrayList<E> newArrayList(Iterator<? extends E> elements)
             分组和反转排序
              Lists 还提供了两个比较实用的功能，分组和反转排序功能
                Lists.reverse(list);
                 reverse 方法底层实现非常巧妙，底层覆写了 List 原生的 get (index) 方法，会把传进来的 index 进行 (size - 1) - index 的计算，使计算得到的索引位置和 index 位置正好相反，这样当我们 get 时，数组索引位置的 index 已经是相反的位置了，达到了反转排序的效果，其实底层并没有进行反转排序，只是在计算相反的索引位置，通过计算相反的索引位置这样简单的设计，得到了反转排序的效果，很精妙。
                List<List<String>> list2 = Lists.partition(list,3);
                partition 方法的第二个参数的意思，你想让分组后的 List 包含几个元素，这个方法的底层实现其实就是 subList 方法。
                有一点需要我们注意的是这两个方法返回的 List 并不是 ArrayList，是自定义的 List，所以对于 ArrayList 的有些功能可能并不支持，使用的时候最好能看下源码，看看底层有无支持。
           3 Maps   
               初始化
                 Map<String,String> hashMap = Maps.newHashMap();
                 Map<String,String> linkedHashMap = Maps.newLinkedHashMap();
                // 这里 Map 的初始化大小公式和 HashSet 初始化公式类似，还记得 HashSet 初始化 HashMap 时，经典的计算初始大小的公式么：取最大值（期望的值 / 0.75 + 1，默认值 16），newHashMapWithExpectedSize 方法底层也是这么算的初始化大小的
                  Map<String,String> withExpectedSizeHashMap = Maps.newHashMapWithExpectedSize(20);

                difference
                 Maps 提供了一个特别有趣也很实用的方法：difference，此方法的目的是比较两个 Map 的差异，入参就是两个 Map，比较之后能够返回四种差异：
                       左边 Map 独有 key。
                       右边 Map 独有 key。
                       左右边 Map 都有 key，并且 value 相等。
                       左右边 Map 都有 key，但是 value 不等。  
                  // ImmutableMap.of 也是 Guava 提供初始化 Map 的方法，入参格式为 k1,v1,k2,v2,k3,v3……
                   Map<String,String> leftMap = ImmutableMap.of("1","1","2","2","3","3");

                   log.info("左边 map 独有 key：{}",difference.entriesOnlyOnLeft());
                   log.info("右边 map 独有 key：{}",difference.entriesOnlyOnRight());
                   log.info("左右边 map 都有 key，并且 value 相等：{}",difference.entriesInCommon());
                    log.info("左右边 map 都有 key，但 value 不等：{}",difference.entriesDiffering());

                    List 或者 Map 高效的差异排序算法，完全可以参考 Maps.difference 的内部实现，该方法只使用了一次循环，就可得到所有的相同或不同结果，这种算法在我们工作中也经常被使用。



 
