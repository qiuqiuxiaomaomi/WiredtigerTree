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