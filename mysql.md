# mysql binlog、文件同步
## 高性能数据库连接池HiKariCP 
   什么是数据库连接池
     
     在详细分析 HiKariCP 高性能之前，我们有必要先简单介绍一下什么是数据库连接池。本质上，数据库连接池和线程池一样，都属于池化资源，作用都是避免重量级资源的频繁创建和销毁，对于数据库连接池来说，也就是避免数据库连接频繁创建和销毁。如下图所示，服务端会在运行期持有一定数量的数据库连接，当需要执行 SQL 时，并不是直接创建一个数据库连接，而是从连接池中获取一个；当 SQL 执行完，也并不是将数据库连接真的关掉，而是将其归还到连接池中。 
   而宏观上主要是和两个数据结构有关，一个是 FastList，另一个是 ConcurrentBag。下面我们来看看它们是如何提升 HiKariCP 的性能的。
   FastList 解决了哪些性能问题
      按照规范步骤，执行完数据库操作之后，需要依次关闭 ResultSet、Statement、Connection，但是总有粗心的同学只是关闭了 Connection，而忘了关闭 ResultSet 和 Statement。为了解决这种问题，最好的办法是当关闭 Connection 时，能够自动关闭 Statement。为了达到这个目标，Connection 就需要跟踪创建的 Statement，最简单的办法就是将创建的 Statement 保存在数组 ArrayList 里，这样当关闭 Connection 的时候，就可以依次将数组中的所有 Statement 关闭。  

      HiKariCP 觉得用 ArrayList 还是太慢，当通过 conn.createStatement() 创建一个 Statement 时，需要调用 ArrayList 的 add() 方法加入到 ArrayList 中，这个是没有问题的；但是当通过 stmt.close() 关闭 Statement 的时候，需要调用 ArrayList 的 remove() 方法来将其从 ArrayList 中删除，这里是有优化余地的

      假设一个 Connection 依次创建 6 个 Statement，分别是 S1、S2、S3、S4、S5、S6，按照正常的编码习惯，关闭 Statement 的顺序一般是逆序的，关闭的顺序是：S6、S5、S4、S3、S2、S1，而 ArrayList 的 remove(Object o) 方法是顺序遍历查找，逆序删除而顺序查找，这样的查找效率就太慢了。如何优化呢？很简单，优化成逆序查找就可以了

      HiKariCP 中的 FastList 相对于 ArrayList 的一个优化点就是将 remove(Object element) 方法的查找顺序变成了逆序查找。除此之外，FastList 还有另一个优化点，是 get(int index) 方法没有对 index 参数进行越界检查，HiKariCP 能保证不会越界，所以不用每次都进行越界检查

   ConcurrentBag 解决了哪些性能问题
       如果让我们自己来实现一个数据库连接池，最简单的办法就是用两个阻塞队列来实现，一个用于保存空闲数据库连接的队列 idle，另一个用于保存忙碌数据库连接的队列 busy；获取连接时将空闲的数据库连接从 idle 队列移动到 busy 队列，而关闭连接时将数据库连接从 busy 移动到 idle。这种方案将并发问题委托给了阻塞队列，实现简单，但是性能并不是很理想。因为 Java SDK 中的阻塞队列是用锁实现的，而高并发场景下锁的争用对性能影响很大。

       HiKariCP 并没有使用 Java SDK 中的阻塞队列，而是自己实现了一个叫做 ConcurrentBag 的并发容器。ConcurrentBag 的设计最初源自 C#，它的一个核心设计是使用 ThreadLocal 避免部分并发问题，不过 HiKariCP 中的 ConcurrentBag 并没有完全参考 C# 的实现，下面我们来看看它是如何实现的。

       ConcurrentBag 中最关键的属性有 4 个，分别是：用于存储所有的数据库连接的共享队列 sharedList、线程本地存储 threadList、等待数据库连接的线程数 waiters 以及分配数据库连接的工具 handoffQueue。其中，handoffQueue 用的是 Java SDK 提供的 SynchronousQueue，SynchronousQueue 主要用于线程之间传递数据


       //用于存储所有的数据库连接CopyOnWriteArrayList<T> sharedList;
       //线程本地存储中的数据库连接ThreadLocal<List<Object>> threadList;/
       /等待数据库连接的线程数AtomicInteger waiters;
       //分配数据库连接的工具SynchronousQueue<T> handoffQueue;


       当线程池创建了一个数据库连接时，通过调用 ConcurrentBag 的 add() 方法加入到 ConcurrentBag 中，下面是 add() 方法的具体实现，逻辑很简单，就是将这个连接加入到共享队列 sharedList 中，如果此时有线程在等待数据库连接，那么就通过 handoffQueue 将这个连接分配给等待的线程。

       /将空闲连接添加到队列void add(final T bagEntry){  //加入共享队列  sharedList.add(bagEntry);  //如果有等待连接的线程，  //则通过handoffQueue直接分配给等待的线程  while (waiters.get() > 0     && bagEntry.getState() == STATE_NOT_IN_USE     && !handoffQueue.offer(bagEntry)) {      yield();  }}


       通过 ConcurrentBag 提供的 borrow() 方法，可以获取一个空闲的数据库连接，borrow() 的主要逻辑是：首先查看线程本地存储是否有空闲连接，如果有，则返回一个空闲的连接；如果线程本地存储中无空闲连接，则从共享队列中获取。如果共享队列中也没有空闲的连接，则请求线程需要等待

       需要注意的是，线程本地存储中的连接是可以被其他线程窃取的，所以需要用 CAS 方法防止重复分配。在共享队列中获取空闲连接，也采用了 CAS 方法防止重复分配

       T borrow(long timeout, final TimeUnit timeUnit){ // 先查看线程本地存储是否有空闲连接 final List list = threadList.get(); for (int i = list.size() - 1; i >= 0; i--) { final Object entry = list.remove(i); final T bagEntry = weakThreadLocals ? ((WeakReference) entry).get() : (T) entry; //线程本地存储中的连接也可以被窃取， //所以需要用CAS方法防止重复分配 if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) { return bagEntry; } } // 线程本地存储中无空闲连接，则从共享队列中获取 final int waiting = waiters.incrementAndGet(); try { for (T bagEntry : sharedList) { //如果共享队列中有空闲连接，则返回 if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) { return bagEntry; } } //共享队列中没有连接，则需要等待 timeout = timeUnit.toNanos(timeout); do { final long start = currentTime(); final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS); if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) { return bagEnt

       释放连接需要调用 ConcurrentBag 提供的 requite() 方法，该方法的逻辑很简单，首先将数据库连接状态更改为 STATE_NOT_IN_USE，之后查看是否存在等待线程，如果有，则分配给等待线程；如果没有，则将该数据库连接保存到线程本地存储里

       //释放连接void requite(final T bagEntry){ //更新连接状态 bagEntry.setState(STATE_NOT_IN_USE); //如果有等待的线程，则直接分配给线程，无需进入任何队列 for (int i = 0; waiters.get() > 0; i++) { if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) { return; } else if ((i & 0xff) == 0xff) { parkNanos(MICROSECONDS.toNanos(10)); } else { yield(); } } //如果没有等待的线程，则进入线程本地存储 final List threadLocalList = threadList.get(); if (threadLocalList.size() < 50) { threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry); }}

       FastList 适用于逆序删除场景；而 ConcurrentBag 通过 ThreadLocal 做一次预分配，避免直接竞争共享资源，非常适合池化资源的分配
## mysql 调优
    调优维度
       业务需求   复杂sql   
         存储过程 只是编译时有开销  降低网络开销
         做分页 详情标签 统计模式
         勇敢说不
       系统架构
          读写分离 高可用 分库分表  实例个数  用什么数据库
       sql及索引
          编写良好sql   创建高效的索引
       表结构 
          设计良好的表结构 
       数据库参数设置
          设置合理的数据库参数
          join buffer sort buffer
       系统配置
           资源使用策略 数据库充分利用资源
           sswap ->swappiness                  
       硬件
           机器配置
    测试数据导入  
    mysql -uroot -proot -< 导入.sql
    mysql 客户端工具 
      idea自带database插件 
      navicat  
      syslog
    发现慢sql的工具
      Skywalking
      VisualVm
      JavaMelody

      相关参数与默认值

      log_output  日志输出到哪儿 file table 也可以同时配置
      long_query_time 执行超过多久记录到慢查询日志
      log_queries_not_using_indexes  
       是否将未使用索引的sql记录到慢查询日志 生产环境建议关闭 开发环境建议打开
      log_throttle_quries_not_using_indexes 上面参数的限流
      min_examined_row_limit  扫描行数至少达到这么多才记录到慢查询日志 默认0
      log_slow_admin_statements 是否记录管理语句
      show_query_log_file 指定查询日志路径
      log_slow_slave_statements  该参数在从库设置 决定 是否记录在复制过程中超过long_query_time的sql
      如果binlog格式时row 忽略该参数 statement
      log_slow_extra 是否记录额外信息

      使用方式
       mysql.in 配置文件修改 需要重启
       全局变量 
       set GLOBAL slow_query_log='ON'; #开启慢查询
       set GLOBAL log_output='FILE,TABLE';#记录到文件和表
       set GLOBAL long_query_time=0.001; #设置完断开连接
       set GLOBAL log_queries_not_using_indexes='ON';
       show VARIABLES like '%long_query_time%'
       SELECT * from employees;
       select * from  `mysql`.slow_log#基于table的慢查询日志
       show VARIABLES like'%slow_query_log_file%';#基于文件的慢查询日志
       mysqldumpslow -s t -t 10 -a slave001-slow.log

        工具pt-query-digest Perconna-Toolkit套件 
    EXPLAIN 详解

      EXPLAIN select * from salaries WHERE from_date='1996-12-02'
      # type   联接的类型  all 全表扫描
      #possible_keys 可能的索引选择
      #key  实际选择的索引
       #key_len  索引的长度
      #rows  估计要扫描的行  越小越好 200多万
      #filtered  表实符合查询条件的数据百分比 越少越好 explain extend mysql5.7之前
      #extra using filesort  using tempoary
      #id 相同从上到下一次执行  id越大越先执行
      1 SIMPLE  salaries    ALL         2838426 10  Using where
      以tree形式展示
      explain format=tree
      select * from
        employees e left join salaries s on e.emp_no=s.emp_no
      where e.emp_no=10001
      
      可视化explain
        idea explain plan  show visual
        mysqlworkbench 

      sql性能分析
      
        expalin 分析sql

        如何更细致的分析sql的性能

        深入sql内部分析性能
      show profile
        SELECT @@have_profiling;
        select @@profiling;
        set @@profiling=1
        set profiling_history_size=100;
        show profiles;
        select * from salaries;
        show profiles;
        show profile for query 44;
        show PROFILE memory for query 44;
        show PROFILE all for query 44;
        set @@profiling=0; #关闭
      使用schema_profiling表
          set @@profiling=1  
           show profiles;
           select * from salaries;
           SELECT STATE, FORMAT(DURATION, 6) AS DURATION
           FROM INFORMATION_SCHEMA.PROFILING
           WHERE QUERY_ID = 90 ORDER BY SEQ;
           select * from performance_schema.setup_actors; 
      performance_schema     

           SELECT * from salaries;
         select * from performance_schema.setup_actors;
        SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT
       FROM performance_schema.events_statements_history_long WHERE SQL_TEXT like '%salaries%';
       
       SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration
       FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=284;  
    optimizer_trace
       5.6之后
       跟踪优化器做出的各种决策

       了解优化器的执行细节

       理解sql执行过程 进而优化sql
       开启设置
         SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
         SET optimizer_trace_offset=-30, optimizer_trace_limit=30;
         SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE limit 30

         json字段
    数据库诊断命令
      show full PROCESSLIST
      select * from  information_schema.`PROCESSLIST`


      数据库运行状态 show status;


      show variables; 查看表变量

      show table status; 查看表状态

      show index from table; 查看索引

      show ENGINE INNODB status; 查看数据库引擎的状态
  理论
## 数据库索引的类型 
       二叉树查找
       平衡二叉树搜索树（Avl树） 
        每个节点的左子树和右子树的高度不超过1
        对于n个节点 树的深度时log2n 查询的时间复杂度时O(log2n)
      b-tree  balance tree
       指针 关键字 数据（磁盘块）
      特性
        根节点的子节点个数在2<=x<m m是树的阶
        假设m=3 则根节点可以有2-3个孩子
        中间节点的子节点个数 m/2<=y<=m 
        假设m=3  中间节点至少有2个孩子 最多3个孩子
        每个中间节点包含n个关键字 女子节点个数-1 ，且按升序排序 如果中间节点有3个子节点 则里面会有2个关键字 且按升序排序
        每个节点n个关键字 n+1个指针
      b+tree  innodb使用
       区别
        b+tree 有n个子节点中含有n个关键字
        b-tree 是n个子节点有n-1关键字

        b+tree中所有叶子节点包含了全部关键字且叶子节点包含了所有关键字信息
        而b-tree不包含所有关键字信息

        b+tree非叶子节点仅仅用于索引 不保存数据记录 记录存放在叶子节点中
        b_tree非叶子节点机保存索引 也保存数据 

        b+tree范围查询 直接查询  
      InnoDb存储方式
       b+tree
       主键索引：叶子节点存储主键及数据
       非主键索引（二级索引 辅助索引） 叶子节点存储索引以及主键
      Myisam 存储方式
       B+tree
        主键与非主键索引的叶子节点都是存储指向数据块的指针
        innodb 聚簇索引
        myisam 非聚簇索引
      hash索引 keys buckets entries  时间复杂度o(1)
         尽量防止hash冲突  只支持memory引擎
        innodb引擎自适应hash索引
        show variables like ‘innodb_adaptive_hash_index’
      空间索引 R-tree索引
        存储GIS数据 基于R-Tree
      全文索引

      b+tree 
      特性
         完全匹配 index(name)=>where name ='大目'
         范围匹配 index(age)=> where age >5
         前缀匹配 index(name) =>where name like '大%'（右模糊）
     
      限制
        index(name,age,sex)
          查询条件不包括最左列 无法使用索引
           where age=5 and sex =1 无法使用索引
          跳过了索引中的列 则无法完全使用索引
           where name ='大目' and sex =32=>只能用name 这一列
         查询中有某个列的范围查询 则其右边所有列都无法使用索引
           where name ='大目' and age >32 and sex =1 =>只能用name age 两列

      hash索引
        性能高

        hash索引不是按照索引的值排序 所以没法使用排序
        不支持部分索引匹配查找
         hash(a,b)=>where a=1
         只支持等值查询（=，in） 不支持范围查询 模糊查询
         hash冲突严重性能下降越厉害
    创建索引的场景
      建议
      select 语句   频繁作为where条件的字段  （单列索引  组合索引注意最左前缀原则）    
      update/delete 语句的where条件 先查询 再操作
      分组，排序的字段
      distinct 使用的字段
      字段的值有唯一性约束 
      对于多表查询，连接字段应该创建索引 而且类型务必一致，避免隐式转换
    不建议
      where子句中用不到的字段
      表的记录非常少
      有大量重复数据 选择性低
        索引的选择性高 查询效率越好 因为可以在查找是过滤更多的行
      频繁更新的字段 如果创建索引要考虑开销  
    索引失效的场景
      -- 示例1：索引字段不独立(索引字段进行了表达式计算)
        explain
        select *
       from employees
       where emp_no + 1 = 10003;
       -- 解决方案：事先计算好表达式的值，再传过来，避免在SQLwhere条件 = 的左侧做计算
       explain
       select *
       from employees
       where emp_no = 10002;

       -- ---------------------
       -- ---------------------
     -- 示例2：索引字段不独立(索引字段是函数的参数)
        explain
        select *
        from employees
        where SUBSTRING(first_name, 1, 3) = 'Geo';
         -- 解决方案：预先计算好结果，再传过来，在where条件的左侧，不要使用函数；或者使用等价的SQL去实现
        explain
        select *
        from employees
         where first_name like 'Geo%';

         示例4：使用OR查询的部分字段没有索引
         explain
         select *
         from employees
         where first_name = 'Georgi'
         or last_name = 'Georgi';
         --解决方案：分别为first_name以及last_name字段创建索引

          示例5：字符串条件未使用''引起来
          explain
          select *
          from dept_emp
          where dept_no = 3;
          -- 解决方案：规范地编写SQL
         explain
         select *
         from dept_emp
         where dept_no = '3';

         -- 示例6：不符合最左前缀原则的查询
         -- 存在index(last_name, first_name)
          explain select *
         from employees
         where first_name = 'Facello';
          -- 解决方案：调整索引的顺序，变成index(first_name,last_name)/index(first_name)

         示例7：索引字段建议添加NOT NULL约束
         -- 单列索引无法储null值，复合索引无法储全为null的值
         -- 查询时，采用is null条件时，不能利用到索引，只能全表扫描
         -- MySQL官方建议尽量把字段定义为NOT NULL：https://dev.mysql.com/doc/refman/8.0/en/data-size.html
         explain
         select *
         from `foodie-shop-dev`.users
           where mobile is null;
         -- 解决方案：把索引字段设置成NOT NULL，甚至可以把所有字段都设置成NOT NULL并为字段设置默认值

         隐式转换导致索引失效
         -- 目前没这样的表，演示不了，同学们可以试试把de.emp_no的字段类型改成varchar
         select emp.*, d.dept_name
         from employees emp
         left join dept_emp de
                   on emp.emp_no = de.emp_no
         left join departments d
                   on de.dept_no = d.dept_no
        where de.emp_no = '100001';
        - 解决方案：在创建表的时候尽量规范一点，比如统一用int，或者bigint 
    长字段的索引调优
     -- 长字段的调优
       
       （1） select *
       from employees
        where first_name = 'Facello';
        添加新列first_name_hash
       insert into employees (emp_no, birth_date, first_name, last_name, gender, hire_date, first_name_hash)
        value (
           999999, now(),
           '大目......................',
           '大', 'M', now(),
           CRC32('大目......................')
        );
        -- first_name_hash的值应该具备以下要求：
        -- 1. 字段的长度应该比较的小，SHA1/MD5是不合适的
        -- 2. 应当尽量避免hash冲突，就目前来说，流行使用CRC32()或者FNV64()

        无法使用
         select *  
         from employees
         where first_name like 'Facello%';

         （2）-- 前缀索引
           alter table employees add key (first_name(5));
             -- 完整列的选择性：0.0042[这个字段的最大选择性了]
            select count(distinct first_name)/count(*) from employees;
            select count(DISTINCT first_name) FROM employees;
            -- 5: 0.0038   6:0.0041   7:0.0042
            select count(distinct left(first_name, 8))/count(*) from employees;
             -- 结论
            alter table employees add key (first_name(7));
         -- 局限性：无法做order by、group by；无法使用覆盖索引

         "后缀索引"：额外创建一个字段，比如说first_name_reverse，在存储的时候，把first_name的值
         -- 翻转过来再存储。比方说：Facello  ==> ollecaF存储到first_name_reverse 
    单列索引与组合索引    
       创建单列索引
         type index_meger  Extra Using intersect(to,from); Using where


         #单表查询 rows_estimation

         SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
         SET optimizer_trace_offset=-30, optimizer_trace_limit=30;
         SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE where QUERY like '%salaries%' LIMIT 30


         组合索引
          type ref
         
         单列索引 组合索引

         sql存在多个条件 多个单列索引 会使用索引合并 并求交集
         
         如果出现索引合并 往往说明索引不够合理

         如果sql暂时没有性能问题 可以不管

         组合索引要注意索引顺序 注意最左前缀原则
    覆盖索引
      对于索引 select 的字段只需要从索引能获得，从而无需到表数据里获取，这样的索引就交覆盖索引

      索引无法覆盖查询字段时
      type: ref  row: 86 extra: null 
      explain  select * from  salaries where from_date='1986-06-26' AND to_date='1987-06-26'        
      索引能覆盖查询字段时
      type: ref  row：86 extra using index

      explain  select from_date,to_date from  salaries where from_date='1986-06-26' AND to_date='1987-06-26'
       
      SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
      SET optimizer_trace_offset=-30, optimizer_trace_limit=30;
         
      SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE where QUERY like '%salaries%' LIMIT 30

      覆盖索引启发

        尽量只返回想要的字段
        使用覆盖索引
        减少网络开销 
    重复索引 荣冗余索引 未使用的索引    
        索引是有开销的
        重复索引 在相同的列上按照相同的顺序创建的索引
        create table test_table
        (
          id int not null primary key auto_increment,
          a  int not null,
          b  int not null,
           UNIQUE (id),
           INDEX (id)
         ) ENGINE = InnoDB;
        -- 发生了重复索引，改进方案：
        create table test_table
       (
        id int not null primary key auto_increment,
         a  int not null,
          b  int not null
         ) ENGINE = InnoDB;
        冗余索引
         如果已经存在索引index（A，B） 又创建了index(A) 那么index(A)就是index(A,B)的冗余索引  
         A是A,B的前缀索引

         特例
         -- index(from_date): type=ref    extra=null，使用了索引
         -- index(from_date) 某种意义上来说就相当于index(from_date, emp_no)
         -- index(from_date, to_date): type=ref    extra=Using filesort，order by子句无法使用索引
         -- index(from_date, to_date)某种意义上来说就相当于index(from_date, to_date, emp_no)
        未使用的索引
          某个索引未使用 删除
      
