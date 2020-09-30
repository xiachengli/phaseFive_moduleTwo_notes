GuavaCache、Tair、EVCache、Aerospike比较

|        | GuavaCache          | Tair       | EVCache     | Aerospike              |
| ------ | ------------------- | ---------- | ----------- | ---------------------- |
| 类别   | 本地缓存（JVM缓存） | 分布式缓存 | 分布式缓存  | 分布式NoSQl数据库      |
| 应用   | 高并发本地缓存      | 阿里、美团 | Netﬂix、AWS | 互联网广告行业（国外） |
| 性能   | 高                  | 较高       | 很高        | 较高                   |
| 持久化 | 无                  | 有         | 有          | 有                     |
| 集群   | 无                  | 有         | 有          | 有                     |

---

#### Guava Cache本地缓存

##### Guava Cache简介

Guava Cache是一套完善的本地缓存机制，设计来源于CurrentHashMap，可以按照多种策略来清理过期数据且保持很高的并发读写性能

###### 应用场景

- 对性能有非常高的要求 
- 不经常变化 
- 占用内存不大 
- 有访问整个集合的需求 
- 数据允许不实时一致

###### 优势

- 缓存过期和淘汰机制 

  可以设置Key的过期时间，包括访问过期和创建过期 

- 并发处理能力 

  GuavaCache类似CurrentHashMap，是线程安全的

  提供了设置并发级别的api，使得缓存支持并发的写入和读取 

  采用分离锁机制，分离锁能够减小锁力度，提升并发能力 

  分离锁是分拆锁定，把一个集合看分成若干partition, 每个partiton一把锁。ConcurrentHashMap 就是分了16个区域，这16个区域之间是可以并发的。GuavaCache采用Segment做分区

- 更新锁定 

  一般情况下，在缓存中查询某个key，如果不存在，则查源数据，并回填缓存

  在高并发下会出现，多次查源并重复回填缓存，可能会造成源的宕机（DB），性能下降 GuavaCache可以在CacheLoader的load方法中加以控制，对同一个key，只让一个请求去读源并 回填缓存，其他请求阻塞等待

- 集成数据源 

  一般我们在业务中操作缓存，都会操作缓存和数据源两部分 而GuavaCache的get可以集成数据源，在从缓存中读取不到时可以从数据源中读取数据并回填缓存 

- 监控缓存加载/命中情况 

GuavaCache有两种创建方式： CacheLoader和Callable callback

##### 缓存数据删除

GuavaCache的数据删除分为：被动删除和主动删除 

- 被动删除

  - 基于数据大小的删除 （LRU+FIFO ）

  - 基于过期时间的删除 

    隔多长时间后没被访问的key被删除.expireAfterAccess(3, TimeUnit.SECONDS) ；写入多长时间后删除.expireAfterWrite(3, TimeUnit.SECONDS) 

  - 基于引用的删除 

    可以通过weakKeys和weakValues方法指定Cache只保存对缓存记录key和value的弱引用。这样当没有 其他强引用指向key和value时，key和value对象就会被垃圾回收器回收

- 主动删除

  - 单独删除

    ```
     //将key=1 删除 
     cache.invalidate("1");
    ```

  - 批量删除

    ```
    //将key=1和2的删除
    cache.invalidateAll(Arrays.asList("1","2"));
    ```

  - 清空所有数据

    ```
    //清空缓存 
    cache.invalidateAll();
    ```

##### Guava Cache原理

###### 数据结构

![image-20200929152938360](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200929152938360.png)

- LocalCache为Guava Cache的核心类，包含一个Segment数组

- Segement数组的长度决定了cache的并发数 

- 每一个Segment使用了单独的锁，每个Segment继承了ReentrantLock，对Segment的写操作需要先拿到锁 

- 每个Segment由一个table和5个队列组成 

  - table

    AtomicReferenceArray<ReferenceEntry<K, V>> table：AtomicReferenceArray可以用原子方式

    更新其元素的对象引用数组 

    - ReferenceEntry<k,v> 

      ReferenceEntry是Guava Cache中对一个键值对节点的抽象，每个ReferenceEntry数组项都是一 条ReferenceEntry链。并且一个ReferenceEntry包含key、hash、valueReference、next字段 （单链） 

  - 队列

    - ReferenceQueue  keyReferenceQueue ：已经被GC，需要内部清理的键引用队列
    - ReferenceQueue  valueReferenceQueue ：已经被GC，需要内部清理的值引用队列
    - ConcurrentlinkedQueue<ReferenceEntry<k,v>> recencyQueue ：LRU队列，当segment上达到临界值发生写操作时该队列会移除数据
    - Queue<ReferenceEntry<K, V>> writeQueue：写队列，按照写入时间进行排序的元素队列，写入一个元素时会把它加入到队列尾部
    - Queue<ReferenceEntry<K, V>> accessQueue：访问队列，按照访问时间进行排序的队列，访问一个元素时会把它加入到队列尾部



###### 回收机制

Guava Cache提供了三种基本的缓存回收方式：

- 基于容量回收 

  在缓存项的数目达到限定值之前，采用LRU的回收方式 

- 定时回收 

  expireAfterAccess：在给定时间内没有被访问，则回收。回收顺序和基于大小回收一 样（LRU） expireAfterWrite：在给定时间内没有被写访问（创建或覆盖），则回收 

- 基于引用回收

  通过使用弱引用的键、或弱引用的值、或软引用的值，Guava Cache可以垃圾回收 

惰性删除 ：如get()或者put()的时候，判断缓存是否过期 

###### Segment定位

hash(key)定位到指定的segment，然后再找到segment中的Entry链数组，通过key的hash定位到某个Entry节点

```
 V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {     
 // 注意，key不可为空        
 int hash = hash(checkNotNull(key));       
 // 通过hash定位到segment数组的某个Segment元素，然后调用其get方法        
 return segmentFor(hash).get(key, hash, loader);    
 }

```

```
 V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
 checkNotNull(key);      
 checkNotNull(loader);      
 try {        
 if (count != 0) { // read-volatile          // 内部也是通过找Entry链数组定位到某个Entry节点          ReferenceEntry<K, V> e = getEntry(key, hash);
 ......
```

##### 实战

###### 并发操作

GuavaCache通过设置 concurrencyLevel 使得缓存支持并发的写入和读取，concurrencyLevel=Segment数组的长度 。同ConcurrentHashMap类似，Guava cache的并发也是通过分离锁实现

LoadingCache采用了类似ConcurrentHashMap的方式，将映射表分为多个segment。segment之间 可以并发访问，这样可以大大提高并发的效率，使得并发冲突的可能性降低了

```
LoadingCache<String,Object> cache = CacheBuilder.newBuilder() // 最大3个       同时支持CPU核数线程写缓存                .maximumSize(3).concurrencyLevel(Runtime.getRuntime().availableProcessors()).bui ld();

```

###### 更新锁定

GuavaCache提供了一个refreshAfterWrite定时刷新数据的配置项，如果经过一定时间没有更新或覆盖，则会在下一次获取该值的时候，会在后台异步去刷新缓存，刷新时只有一个请求回源取数据，其他请求会阻塞（block）在一个固定时间段，如果在该时间段内没 有获得新值则返回旧值

```
LoadingCache<String,Object> cache = CacheBuilder.newBuilder()                // 最大3个        同时支持CPU核数线程写缓存             .maximumSize(3).concurrencyLevel(Runtime.getRuntime().availableProcessors()).    //3秒内阻塞会返回旧数据    refreshAfterWrite(3,TimeUnit.SECONDS).build();

```

###### 动态加载

动态加载行为发生在获取不到数据或者是数据已经过期的时间点，Guava动态加载使用回调模式。用户自定义加载方式，然后Guava cache在需要加载新数据时会回调用户的自定义加载方式

```
segmentFor(hash).get(key, hash, loader)
```

loader即为用户自定义的数据加载方式，当某一线程get不到数据会去回调该自定义加载方式去加载数据 

###### 疑难问题

1. GuavaCache会内存溢出吗？

   会。当我们设置缓存永不过期（或者很长），缓存的对象不限个数（或者很大）时

   解决方案：：缓存时间设置相对小些，使用弱引用方式存储对象

2.  GuavaCache缓存到期就会立即清除吗？

   不会。

   惰性删除：在get()或者put()的时候，判断缓存是否过期

3. GuavaCache如何找出最久未使用的数据 ？

   借助accessQueue队列

---

####  EVCache分布式缓存

##### EVCache简介

EVCache是一个开源、快速的分布式缓存 ，基于Memcached的内存存储和Spymemcached客户端实现的

E：Ephemeral：数据存储是短暂的，有自身的存活时间

V：Volatile：数据可以在任何时候消失

Cache：内存级键值对存储 

![image-20200930000830129](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930000830129.png)

Rend服务：是一个代理服务，用GO语言编写，能够高性能的处理并发

Memcached：基于内存的键值对缓存服务器

Mnemonic：基于硬盘（SSD）的嵌入式键值对存储服务器，封装了RocksDB(是一种SSD的技术) 

##### EVCache应用

EVCache典型地适合对强一致性没有必须要求的场合

典型用例：Netﬂix向用户推荐用户感兴趣的电影

![image-20200930001730908](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930001730908.png)

典型部署：EVCache 是线性扩展的，可以在一分钟之内完成扩容，在几分钟之内完成负载均衡和缓存预热

单节点部署：

![image-20200930001830700](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930001830700.png)

- 集群启动时，EVCache向服务注册中心（Zookeeper、Eureka）注册各个实例 
- 在web应用启动时，通过EVCache的客户端(EVCache Client)查询命名服务中的EVCache服务器列 表，并建立连接
- web应用通过EVCache的客户端操作数据，客户端通过key使用一致性hash算法，将数据分片到集群

多可用区部署：

![image-20200930001947323](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930001947323.png)

##### EVCache原理

EVCache的内存存储是基于Memcached实现的；EVCache的客户端是基于Spymemcached实现的 

###### Memcached简介 

Memcached是danga（丹加）开发的一套分布式内存对象缓存系统

- Memcached是C/S模式的
- 基于libevent的事件处理 
- Memcached是多线程的

###### Slab Allocation机制

传统的内存分配是通过对所有记录简单地进行malloc（动态内存分配）和free（释放） 来进行的。是以 Page（M）为存储单位的(BuddySystem)。这种方式会导致内存碎片，加重操作系统内存管理器的负担

memcached采用Slab（块） Allocation的方式分配和管理内存 

slab是Linux操作系统的一种内存分配机制 

slab分配器分配内存以Byte为单位，专为小内存分配而生

Slab Allocation的原理： 

将分配的内存分割成各种大小的块(Chunk)，并把尺寸相同的块分成组(Slab Class)，Memcached根据收到的数据大小，选择最合适的slabClass进行存储

![image-20200930003103081](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930003103081.png)

注：块越小存的数据越多，块越来越大，是由growth factor决定的(1.25) 

###### 数据Item

Item就是我们要存储的数据。是以双向链表的形式存储的

```
/** * Structure for storing items within memcached. */ typedef struct _stritem {    struct _stritem *next;  //next即后向指针    struct _stritem *prev;  //prev为前向指针    struct _stritem *h_next;    /* hash chain next */    rel_time_t      time;       /* least recent access */    rel_time_t      exptime;    /* expire time */    int             nbytes;     /* size of data */    unsigned short  refcount;    uint8_t         nsuffix;    /* length of flags-and-length string */    uint8_t         it_flags;   /* ITEM_* above */    uint8_t         slabs_clsid;/* which slab class we're in */    uint8_t         nkey;       /* key length, w/terminating null and padding */
    union {        uint64_t cas;        char end;    } data[];   } item;
```

###### 缓存过期机制

Memcached有两种过期机制：Lazy Expiration（惰性过期）和LRU 

Lazy Expiration

Memcached在get数据时，会查看exptime，根据当前时间计算是否过期(now-exptime>0)，如果过期 则删除该数据

LRU

当Memcached使用内存大于设置的最大内存(-m 启动指定 默认64M)使用时，Memcached会启动 LRU算法淘汰旧的数据项

使用slabs_alloc函数申请内存失败时，就开始淘汰数据了

淘汰规则：

- 从数据项列表尾部开始遍历，在列表中查找一个引用计数器(refcount)为0的item，把 此item释放掉
- 如果在item列表找不到计数器为0的item，就查找一个3小时没有访问过的item(now-time>3H)。把 他释放，如果还是找不到，就返回NULL（申请内存失败）
- 当内存不足时，memcached会把访问比较少或者一段时间没有访问的item淘汰，以便腾出内存空 间存放新的item

###### Spymemcached设计思想 

spymemcached 是一个 memcached的客户端， 使用NIO实现，有以下特性：

- Memcached协议支持Text和Binary(二进制) 
- 异步通信：使用NIO，采用callback 
- 集群：支持sharding机制
- 自动恢复：断网重连
- failover：可扩展容错，支持故障转移
- 支持批量get、支持jdk序列化 

整体设计：

![image-20200930003741357](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930003741357.png)

- API接口：提供同步或异步接口调用，异步接口返回Future 
-  任务封装：将访问的操作及callback封装成Task
- 路由分区：通过Sharding策略，选择Key对应的连接（connection） 
-  将Task放到对应连接的队列中 
- Selector异步获取队列中的Task，进行协议的封装和发送相应Memcached
-  收到Memcached返回的包，会找到对应的Task，回调callback，进行协议解析并通知业务线程

API接口设计：对外API接口有两种，同步和异步。同步接口：比如set、get等，异步接口：asyncGet、asyncIncr等

线程设计：SpyMemcached有两类线程：业务线程和selector线程

- 业务线程
  - 封装请求task、对象序列化、封装发送的协议、并将task放到对应连接的队列中 
  - 对收到的数据进行反序列化为对象
- Selector线程
  - 读取连接中的队列，将队列中的task的数据发送到memcached 
  - 读取Memcached返回的数据，解析协议并通知业务线程处理 
  - 对失败的节点进行自动重连 

###### Sharding机制

![image-20200930004149244](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930004149244.png)

Spymemcached默认的hash算法有： 

- 传统hash ：hash(key)%节点数
- 一致性hash

容错：key路由到服务节点，服务节点宕机，有两种方式处理

- 自动重连

  失败的节点从正常的队列中摘除，添加到重连队列中，Selector线程定期对该节点进行重连

- failover处理

  - Redistribute(推荐) 

    遇到失败节点后选择下一个节点，直到选到正常的节点 

  - Retry

    继续访问

  - Cancel

    抛异常退出访问 

---

#### Tair分布式缓存

##### Tair简介

- Tair(Taobao Pair)是淘宝开发的分布式Key-Value存储引擎
- 服务器端自动负载均衡
- 分为持久化和非持久化两种方式存储
  - 非持久化：分布式缓存使用 Memcached（mdb）、Redis（rdb） 
  - 持久化：SQL-DB使用FireBird（fdb）；NoSQL-DB：使用Kyoto Cabinet（kdb）、LevelDB（ldb） 

Tair采用可插拔存储引擎设计，以上这些存储引擎可以很方便的替换，还可以引入新的存储引擎比如： MySQL

###### 整体架构分析

![image-20200930004735625](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930004735625.png)

一个Tair集群主要包括client、Conﬁg server和Dataserver 三个部分

- Client在初始化时，从Conﬁg server处获取数据的分布信息，根据分布信息和相应的Data server交互
- Conﬁg Server通过和Data Server的心跳（HeartBeat）维护集群中可用的节点，并根据可用的节点， 构建数据的在集群中的分布信息
- Data server负责数据的存储，并按照Conﬁg server的指示完成数据的复制和迁移工作

###### config Server

管理所有的data server, 维护data server的状态信息、用户配置的桶数量、副本数、机房信息、数据分布的对照表、协调数据迁移、管理进度，将数据迁移到负载较小的节点上

Client和ConﬁgServer的交互主要是为了获取数据分布的对照表，当client获取到对照表后，会缓存这张表，然后通过查这张表决定数据存储的节点，所以不需要和conﬁgserver交互，这使得Tair对外的服 务不依conﬁgserver，所以它不是传统意义上的中心节点 

Conﬁg server维护的对照表有版本概念，由于集群变动或管理触发，构建新的对照表后，对照表的版本号递增，并通过Data server的心跳，将新表同步给数据节点

客户端和Data server交互时，Dataserver每次都把自己缓存的对照表版本号放入response结构中，返回给客户端，客户端将Data server的对照表版本号和自己缓存的对照表版本号比较，如果不相同，会主动和Conﬁg server通信，请求新的对照表

###### Data Server

Data server负责数据的物理存储，并根据Conﬁgserver构建的对照表完成数据的复制和迁移工作

Data server具备抽象的存储引擎层，可以很方便地添加新存储引擎。Data server还有一个插件容器， 可以动态地加载/卸载插件

![image-20200930005512988](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930005512988.png)

##### Tair高可用和负载均衡

Tair的高可用和负载均衡，主要通过对照表和数据迁移两大功能进行支撑

对照表将数据分为若干个桶，并根据机器数量、机器位置进行负载均衡和副本放置，确保数据分布均匀，并且在多机房有数据副本

在集群发生变化时，会重新计算对照表，并进行数据迁移

###### 对照表

在Tair系统中，采用对照表将数据均衡的分布在DataServer上；在Tair系统中，采用对照表将数据均衡的分布在DataServer上；Tair基于一致性Hash算法存储数据，根据配置建立固定数量的桶（bucket）

桶是负载均衡和数据迁移的基本单位

conﬁg server 根据一定的策略把每个桶指派到不同的data server上，因为数据按照key做hash算法，所以可以认为每个桶中的数据基本是平衡的， 保证了桶分布的均衡性, 就保证了数据分布的均衡性 

对照表初始化：

第一次启动ConﬁgServer时，会根据在线的DataServer生成对照。ConﬁgServer会启动一个线程（table_builder_thread）,该线程会每秒检查一次是否需要重新构造对照表

Tair 提供了两种生成对照表的策略：

- 负载均衡优先

  conﬁg server会尽量的把桶均匀的分布到各个data server上
  一个桶的备份数据不能在同一台主机上

- 位置安全优先

  一般我们通过控制 _pos_mask（Tair的一个配置项） 来使得不同的机房具有不同的位置信息
  一个桶的备份数据不能都位于相同的一个位置（不在同一个机房）

###### 数据迁移

当DataServer发生故障时， conﬁg server负责重新计算一张新的桶在data server上的分布表,将原来由故障机器服务的桶的访问重新指派到其它的data server中，数据迁移的具体流程如下：

- 设置当前正在迁移的桶ID 
- DataServer写入桶数据时，会写入redolog 
- migrate_manager迁移内存中的桶数据
- migrate_manager迁移redolog数据
- redolog数据迁移完成后，将桶标记为迁移完成、
- 将信息发送给ConﬁgServer用于同步对照表 

##### Tair存储引擎

- |      | 数据存储      | 持久化 | 数据格式应用场景      |                       |
  | ---- | ------------- | ------ | --------------------- | --------------------- |
  | mdb  | memcached     | 非     | <k,v>存储，prefix操作 | 缓存使用，session分离 |
  | fdb  | fireBird      | 是     | SQL，Btree            | 快速访问较小的数据    |
  | kdb  | kyoto Cabinet | 是     | Hash,B+Tree           | 简单临时存储          |
  | ldb  | levelDB       | 是     | <k,v>，prefix,batch   | 大数据量高频度的存储  |
  | rdb  | redis         | fe非   | list，set，hash       | 复杂数据结构缓存      |

##### Tair API

get：获取key的值

put：设置key,value

incr：对key的值自增

decr：自减

batchGet：批量获得

expire：设置key的过期时间

prefixPut：根据前缀设置，按照prefix计算hash，同一个prefix会存在同一个hash表中，形成链表

prefixGet：根据前缀读取

###### version

Tair中的每个数据都包含版本号，版本号在每次更新后都会递增。这个特性有助于防止由于数据的并发 更新导致的问题，类似于乐观锁

###### Tair实现乐观锁 

trylock（获取版本）–> transaction –> unlock(版本检查) — 正常—> version+1 commit — 异常—> rollback 

- 只要能获取版本，就认为trylock成功
- 版本检查则更新version，一般是加1；否则，异常处理，version不变，并开启回滚 
- 多事务同时获得乐观锁时，unlock只能有一个成功，其余的都应该失败

version 初始化要大于1   

###### Tair实现分布式锁

利用 Tair 的 version 特性可以实现分布式锁，由于 LDB 具有持久化功能，当服务有出现宕机的情况， 也不会因此出现锁丢失或者锁不可释放的情况

如果KEY不存在的话，传入一个固定的初始化VERSION（需要大于1），Tair会在保存这个缓存的同时设 置这个缓存的VERSION为你传入的 VERSION+1 

KEY如果已经存在，Tair会校验你传入的VERSION是否等于现在这个缓存的VERSION，如果相等则允许 修改，否则将失败

---

#### Aerospike分布式NoSQL数据库

##### Aerospike简介

Aerospike（简称AS）是一个分布式，可扩展的键值存储的NoSQL数据库

- T级别大数据高并发的结构化数据存储 
- 读写操作达微妙级，99%的响应可在1毫秒内实现
- 采用混合架构，索引存储在内存中，而数据可存储在机械硬盘(HDD)或固态硬盘(SSD) 上（也可存储在 内存）
- AS内部在访问SSD屏蔽了文件系统层级，直接访问地址，保证了数据的读取速度
- AS同时支持二级索引与Client聚合，支持简单的sql操作（aql），相比于其他nosql数据库，有一定优 势

##### Aerospike架构

![image-20200930014410481](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930014410481.png)

- Client层

  对Aerospike Server中的数据进行访问。包括CRUD、批量操作和基于二级索引的查询 

  追踪节点感知数据存储位置，当节点启动或停止时立即感知集群配置变化

  具有高效性、稳定性和内部的连接池

- Distribution层

  负责管理集群内部数据的平衡分布、备份、容错和不同集群之间的数据同步。主要包含三个模块

  - Cluster Management Module 

    用于追踪集群节点。关键算法是确定哪些节点是集群的一部分的Paxos-like一致投票过程

    Aerospike实现专门的心跳检测（主动与被动），用于监控节点间的连通性。 

  - Data Migration Module

    当有节点添加或移除时，该模块保证数据的重新分布，按照系统配置的复制因子确保每个数据块跨 节点和跨数据中心复制

  - Transaction Processing Module 

    确保读写的一致性与隔离性，，写操作先写副再写主库。该模块包括： 

    - Sync/Async Replication（同步/异步复制）：为保证写一致性，在提交数据之前向所有副本传播更 新并将结果返回客户端
    - Proxy （代理）：集群重配置期间客户端可能出现短暂过期，透明代理请求到其他节点
    - Duplicate Resolution（副本解析）：当集群从活动分区恢复时，解决不同数据副本之间的冲突

- Data层

  负责数据的存储，Aerospike 属于弱语法的key-value数据库。数据存储模式如下：

  ![image-20200930014750969](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930014750969.png)

| Aerosipike | MySQL    |
| ---------- | -------- |
| namespace  | database |
| set        | table    |
| bin        | column   |
| record     | row      |
| key        | pk       |

索引：Aerospike Index包含主索引（Primary Index）和二级索引（Second Index），索引存储在内存 中，并不保存在硬盘里

- 一级索引（Primary-Index） 

  一级索引位于主键(key)，采用hash表与红黑树的混合

- 二级索引（Secondary-Index） 

  二级索引位于非主键上，这允许一对多关系。采用哈希表和B-tree的混合 

磁盘：Aerospike可以直接访问硬盘中的raw blocks（原始块）直接寻址，并特别优化了Aerospike的最 小化读、大块写和并行SSD来增加响应速度和吞吐量

##### Aerospike集群

###### 集群管理

集群处理节点成员身份，并确保当前成员和所有集群中的节点保持一致。包括：集群视图、节点发现和视图改变 

- 集群视图

  每个Aerospike节点都会自动分配唯一的节点标识符，是由MAC地址和监听端口组成的。包括： cluster_key：是一个随机生成的8字节值，用于标识集群视图的实例。
  succession_list:是作为集群一部分的唯一节点标识符集合。 

- 节点发现

  节点间通过心跳消息来检测节点的有效或失效

  集群中的每个节点维护一个邻接表，是最近向该节点发送心跳消息的其他节点列表

  如果心跳超时，则表示该节点失效，从邻接表中删除
  Aerospike还采用其他信息作为备用二次心跳机制
  集群中的每个节点通过计算平均消息丢失来评估每个相邻节点的运行状态评分 

- 视图改变

  领接表一旦发生改变，就会触发运行一个Paxos共识算法来确定一个新的集群视图

  Aerospike把邻接表中节点标识符最高的节点当Proposer，并承担Proposal的角色 

  Proposal提成新的集群视图，如果被接受，则节点开始重新分配数据（Rebalen复制因子（ replication factor）是一个配置参数，不能超过集群节点数。副本越多可靠性越高。ce） 

###### Aerospike数据分布 

Aerospike有智能分区算法，即把用户输入的key在内部根据RIPEMD-160算法，重新hash出一个key并 取前20位，然后相对均衡的把数据分布到各个节点之上。并且满足namespace配置文件的配置（例如 保存多少个备份、是存在磁盘还是存在内存中

每个Digest Space（namespace）被分为4096个不重叠的分区（partitions），它是Aerospike数据存 储的最小单位

![image-20200930015422952](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200930015422952.png)

如上，一个4个节点的集群，每个节点存储1/4数据的主节点，同时也存储1/4数据的副本。如果节点1不 可访问，节点1的副本将被拷贝到其他节点上

复制因子（ replication factor）是一个配置参数，不能超过集群节点数

副本越多可靠性越高，作为必须经过所有数据副本的写请求也越高。实践中，大部分部署使用的数据因子为2（一份主数据和 一个副本）。 同步复制保证即时一致性，没有数据丢失。在提交数据并返回结果给客户端之前，写事务 被传播到所有副本

主成功同时备成功后，客户端认为是成功

在集群重新配置期间，当Aerospike智能终端发送请求到那些短暂过时的错误节点时，Aerospike智能集 群会透明的代理请求至正确的节点

###### Aerospike集群配置和部署 
有两种方式可以搭建集群：Multicast组播方式（UDP）和Mesh网格方式（TCP） 

- Multicast组播方式（UDP）

  udp是不可靠协议，所以有可能会造成节点脱落，而且网络有可能不支持组播 

  ```
  heartbeat {        mode multicast        multicast-group 239.1.139.1        port 3000        address 192.168.127.131        interval 150        timeout 10    }
  ```

  

- Mesh网格方式（TCP）

```
heartbeat {        mode mesh        # add current node address here                address 192.168.127.131        port 3000        # add all cluster node address here        mesh-seed-address-port 192.168.127.131 3002        mesh-seed-address-port 192.168.127.128 3002                interval 150        timeout 10    }

```

集群部署：

在192.168.127.128上安装Aerospike后，修改配置文件/etc/aerospike/aerospike.conf

```
vim /etc/aerospike/aerospike.conf
service {        user root        group root        paxos-single-replica-limit 1 # Number of nodes where the replica count is automatically reduced to 1.        pidfile /var/run/aerospike/asd.pid        proto-fd-max 15000 }
logging {        # Log file must be an absolute path.        file /var/log/aerospike/aerospike.log {                context any info        } }
network {        service {                address any                port 3000                access-address 192.168.127.128 3002        }
        heartbeat {                mode mesh                address 192.168.127.128                port 3002                #all cluster                mesh-seed-address-port 192.168.127.128 3002                mesh-seed-address-port 192.168.127.131 3002                # To use unicast-mesh heartbeats, remove the 3 lines above, and see                # aerospike_mesh.conf for alternative.
                interval 150                timeout 10  }
        fabric {                address any                port 3001        }
        info {                address any                port 3003        } }
namespace test {        replication-factor 2        memory-size 256M
        storage-engine memory }
namespace bar {        replication-factor 2        memory-size 256M
        storage-engine memory
}

```

集群访问：

```
Host[] hosts = new Host[]{                new Host("192.168.127.128", 3000),                new Host("192.168.127.131", 3000)        };        ClientPolicy policy = new ClientPolicy();
        AerospikeClient client = new AerospikeClient(policy, hosts);
        //写策略        WritePolicy wp = new WritePolicy();        //超时时间        wp.setTimeout(500);
   
   
        Key key1 = new Key("test", "SUser", "11");        Bin bin11 = new Bin("name", "zhangfei-c");        Bin bin12 = new Bin("age", 25);        Bin bin13 = new Bin("sex", "M-c");        client.put(wp, key1, bin11, bin12, bin13);
        Key key2 = new Key("test", "SUser", "22");        Bin bin21 = new Bin("name", "zhaoyun-c");        Bin bin22 = new Bin("age", 24);        Bin bin23 = new Bin("sex", "M-c");        client.put(wp, key2, bin21, bin22, bin23);
        Record r1 = client.get(wp, key1, "name", "age", "sex");        System.out.println(r1);
```

