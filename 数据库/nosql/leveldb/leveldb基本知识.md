

## 数据结构

LevelDb 的数据结构分类基本分为以下几类

- **log（磁盘）**

- **memtable（内存）**
- **immutable memtable（内存）**
- **sstable（磁盘）**

log：用于记录操作，数据库断电重启后可以通过Log恢复状态

memtable：基于跳表实现，在内存活跃的表

immutable memtable：当memtable的大小达到一个上限后就会转为immutable memtable

sstable：immutable memtable到一定大小后dump到磁盘成为sstable，sstable有多层，分别level0~leveln，其中先dump到level0，level0到一定大小后压缩成level1，以此类推。level0层的sstable文件较特殊，文件间无序，文件内有序，因此可能有重复key的存在。



### Table的文件格式

Table文件格式如图

![image-20190826234937583](../resource/sstable文件结构.png)



Table文件格式是划分为多个连续的Block，每个Block大小固定，末尾用1个字节记录该Block压缩格式，4个字节的CRC校验码，其余空间记录1或多个Record，Record有四种类型：

- **First**：表示该Record的起始部分在该Block内
- **Middle**：表示该Record的中间一部分在该Block内
- **Las**t：表示该Record的最后一部分在该Block内
- **Full**：表示该Record完整的在Block内

多个Record会组成一个Record Group存在Block内，每个Record Group会取出一个**公共的前缀**，而在该组中的**其它字符串只保存非公共的字符串做为key**即可，由于sstable保存KV数据是严格按照key的顺序来排序的，所以这样能节省出保存key数据的空间来。如图：



![image-20190826235426909](../resource/block共享前缀.png)

因此可得到Block文件的详细结构，其中**restart point记录每个record group的偏移量**：

![img](../resource/block详细格式.png)





## Iterator

由于LevelDb有多种数据结构和应用场景，因此也有多种Iterator，主要分为两类：**基本Iterator**和**组合Iterator**。

基本Iterator分为三类：

- MemTable Iterator：memtable的迭代器
- Block Iterator：sstable中block的迭代器
- LevelFileNumIterator：一组有序sstable文件的迭代器（不会在level0中使用）

组合Iterator：

- TwoLevelIterator，有以下两类：

    - Table Iterator：实现对于单个sstable文件的迭代。Table Iterator先通过index iterator找到对应的block，再使用block iterator遍历数据。level 0层各文件都有一个对应维护的 Table Iterator。

    - Concatenating Iterator：组合了LevelFileNumIterator以及Table Iterator，用于在某一层内的sstable文件中查询数据。Concatenating Iterator先通过LevelFielNumIterator找到对应的sstable文件，再使用table iterator遍历该sstable。

        ![image-20190829195115057](../resource/Two-level迭代器.png)

- MergingIterator：用于合并流程的迭代器。



## 数据写入顺序

log --> memtable --> immutable memtable --> sstable

一次写KV操作需要一次磁盘的顺序写（写到log文件的最后）和一次内存写（写入metable），当KV数据到达sstable之后，旧的log文件会被删除并且产生新的log文件。需要注意的是，level0是由Imuabletable memtable直接dump到磁盘中的，level1是由level0经过compaction获得，level2是由level1经过compaction获得，以此类推。



## 数据合并

合并有两种类型：

minor compaction：将内存数据落盘到磁盘的过程

major compaction：合并sstable的过程

**思考：level0有多个key存在的时候怎么确定哪个key更新，通过seq？**

从键值中decode出来sequence值，对比sequence值，对sequence值进行降序比较。由于这个值是单调递增的，因此越新的数据sequence值越大。

参考资料

https://zhuanlan.zhihu.com/p/58385815