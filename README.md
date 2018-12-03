# WiredtigerTree
Wiredtiger存储引擎技术研究


![](https://i.imgur.com/W4rPg4b.png)

<pre>
Mongodb3.2版本已经将WiredTiger设置为了默认的存储引擎，WiredTiger的写操作会先写入Cache,
并持久化到WAL（Write ahead log），每60s或log文件达到2G时会做一次Checkpoint，将当前的数
据持久化，产生一个新的快照。WiredTiger连接初始化时，首先将数据恢复至最新的快照状态，然后根
据WAL恢复数据，以保证存储可靠性。
</pre>

![](https://i.imgur.com/mZQ2fmC.png)

![](https://i.imgur.com/xP7xFVy.png)

<pre>
WiredTiger的Cache
      WiredTiger的Cache采用BTree的方式组织，每个BTree节点为一个page，root page是BTree
  的根节点，internal page是BTree的中间节点，leaf page是真正存储数据的叶子节点。

      Wiredtiger采用Copy on write的方式管理修改操作（insert、update、delete），修改操
  作会先缓存在cache里，持久化时，修改操作不会在原来的leaf page上进行，而是写入新分配
  的page，每次checkpoint都会产生一个新的root page。

      Checkpoint时，wiredtiger需要将btree修改过的PAGE都进行持久化存储，每个btree对应
  磁盘上一个物理文件，btree的每个PAGE以文件里的extent形式（由文件offset + size标识）
  存储，一个Checkpoit包含如下元数据：

     1）root page地址，地址由文件offset，size及内容的checksum组成
     2) alloc extent list地址，存储从上次checkpoint起新分配的extent列表
     3) discard extent list地址，存储从上次checkpoint起丢弃的extent列表
     5) available extent list地址，存储可分配的extent列表，只有最新的checkpoint包含该列表
     6) file size 如需恢复到该checkpoint的状态，将文件truncate到file size即
</pre>

<pre>
WiredTiger与原来的MMAPV1引擎比对
      性能与并发：
          在大多数工作负载下，WiredTiger的性能要比MMAPV1存储引擎高很多。
          WiredTiger引擎为现代多核系统量身定制，更好的发挥多核系统的处理能力。MMAPV1引
          擎使用表级锁，因此，当某个单表上有并发的操作，吞吐将受到限制。WiredTiger使用
          文档级锁，由此带来并发及吞吐的提高。对于典型的应用，切到WiredTiger引擎，可以带
          来5-10倍的性能提升。

      压缩与加密
          MMAPV1引擎要求数据在内存和磁盘的形式一致，因此，它并不支持压缩和加密，WiredTiger并没有这层限制，可以更好的支持。

      索引前缀压缩
          WiredTiger存储索引时使用前缀压缩，相同的前缀只存一次。由此带来的效果是：索引更小
          了，对物理内存使用也更少了。
</pre>

<pre>
WiredTiger引擎调优
 
      调优Cache Size

         WiredTiger最重要的调优参数就是Cache规模。默认，MongoDB从3.x会保留可用物理内存的
      50%作为数据cache，虽然默认的设置可以应对大部分的应用，通过调节为特定应用找到最佳配置
      还是非常值得的.cache的规模必须足够大，以便保存应用整个工作集。

      除了这个cache， MongoDB在做诸如聚合，排序，连接管理等操作时需要额外的内存。因此，必
      须确保有足够的内存可供使用，否则,MongoDB进程有可能被OOM Killer杀死的风险。

        db.serverStatus().wiredTiger.cache

      1）如果脏数据的占比很高，可以适当调大cache的大小
      2）如果应用是重读的，bytes read into cache，如果这个指标比较高，调大cache规模很有
         可能可以提升读性能。


      控制Read/Wirte Tickets

         WiredTiger使用tickets来控制可以同时被存储引擎处理的读/写操作数。默认值是128，在
      大部分情况下表现良好。如果这个值经常掉到0，所有后续操作将会被排队等待。例如观察到读
      tickets下降，系统可能有大量长连接耗时的操作。

         以下命令查看tickets使用情况：
             db.serverStatus().wiredTiger.concurrentTransactions
</pre>

<pre>
MongoDB磁盘空间释放
      mognodb 在删除数据的情况下不释放占用的磁盘空间，即使drop collection也不行，除非drop database

      可以使用命令
          db.runCommand({closeAllDatabases:1})
</pre>

<pre>
BSON数据的特点
      1）效率，BSON是为效率而设计的，它只需要很少的存储空间，即使在最坏的情况下，BSON格式
              也比JSON格式最好的情况下存储效率高。
      2）传输性 在某些情况下，BSON会牺牲额外的空间让数据的传输更加方便。比如字符串的传输
              前缀会标识字符串的长度，而不是在字符串的末尾打上结束的标记，有利于MongoDB
              修改传输的数据。
      3）性能  最后，BSON格式的编码和解码都是非常快的，它使用C风格的数据表现形式，这样在各
              种语言中都可以高效使用。
</pre>

<pre>
写入协议：
      Client端访问Server端使用了轻量级的TCP/IP写入协议。它其实是在BSON数据上面做了一层封
   装，比如说，写入数据的命令中包含了1个20字节的消息头（由消息的长度和写入命令标识符），
   需要写入的Collection名称和需要写入的数据。
</pre>

<pre>
数据文件：
      如果数据库名字是Material，那么由：
          1）Material.ns
          2) Material.0, Material.1,... 数据文件
   
      MongoDB会使用预分配的方式来保证写入的性能的稳定。并且每个预分配文件都用0来填充。这会让
      MongoDB始终保持额外的空间和空余的数据文件，从而避免了数据增长过快导致的分配磁盘空间
      引起的阻塞。
</pre>

<pre>
MongoDB使用内存映射存储殷勤
      1：它会把数据文件映射到内存中，如果是读操作，内存中的数据起到缓存的作用，内存还可以把
         随机的写操作转换成顺序的写操作，总之可以大幅度提升性能。MongoDB并不干涉内存管理工
         作，而是把这些工作留给操作系统管理，这样做的好处是简化了MongoDB的工作，但坏处是
         你没办法很方便的控制MongoDB占用多大的内存，
</pre>

<pre>
适用于以下场景：

  a.网站数据：mongo非常适合实时的插入，更新与查询，并具备网站实时数据存储所需的复制及高度伸缩性。

  b.缓存：由于性能很高，mongo也适合作为信息基础设施的缓存层。在系统重启之后，由mongo搭建的持久化缓存可以避免下层的数据源过载。

  c.大尺寸、低价值的数据：使用传统的关系数据库存储一些数据时可能会比较贵，在此之前，很多程序员往往会选择传统的文件进行存储。

  d.高伸缩性的场景：mongo非常适合由数十或者数百台服务器组成的数据库。

  e.用于对象及JSON数据的存储：mongo的BSON数据格式非常适合文档格式化的存储及查询。
</pre>

