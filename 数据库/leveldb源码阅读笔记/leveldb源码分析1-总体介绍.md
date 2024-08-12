## **总体介绍**

### **LSM-Tree**
Log Structured Merge Tree，简称 LSM-Tree。LSM-Tree 通过将磁盘的随机写转化为顺序写来提高写性能 ，而付出的代价就是牺牲部分读性能、写放大。

### **如何优化写性能？**
Append only：所有写操作都是将数据添加到文件末尾。
缺点是不支持有序遍历、需要垃圾回收

### ** 如何优化读性能**
有序和哈希

### **读写性能的权衡**
如何获得接近 append only 的写性能，而又能拥有不错的读性能呢？以 LevelDB为代表的 LSM-Tree 存储引擎给出了一个参考答案。LevelDB 的写操作（Put/Delete/Write）主要由两步组成：
1. 写日志（WAL，顺序写）。
2. 写 `MemTable`（内存中的 `SkipList`）。

所以，正常情况下，LevelDB 的写速度非常快。
内存中的 `MemTable` 写满后，会转换为 Immutable MemTable，然后被后台线程 compact 成按 key 有序存储的 `SSTable`（顺序写）。 SSTable 按照数据从新到旧被组织成多个层次（上层新下层旧），点查询（Get）的时候从上往下一层层查找（先查找内存中的memtable，如果没找到再往下按层查找sstable），上层的数据比下层的数据要新，所以查找到的数据也是最新的。 LevelDB 的读操作可能会有多次磁盘 IO（LevelDB 通过 table cache、block cache 和 bloom filter 等优化措施来减少读操作的 I/O 次数）。 后台线程的定期 compaction 负责回收过期数据和维护每一层数据的有序性。在数据局部有序的基础上，LevelDB 实现了数据的（全局）有序遍历。



### **简单介绍**
1. MemTable：内存数据结构，具体实现是 `SkipList`。 接受用户的读写请求，新的数据会先在这里写入。
2. Immutable MemTable：当 MemTable 的大小达到设定的阈值后，会被转换成 Immutable MemTable，只接受读操作，不再接受写操作，然后由后台线程 flush 到磁盘上 —— 这个过程称为 minor compaction。
3. Log：数据写入 MemTable 之前会先写日志，用于防止宕机导致 MemTable 的数据丢失。一个日志文件对应到一个 MemTable。
4. SSTable：Sorted String Table。分为 level-0 到 level-n 多层，每一层包含多个 SSTable，文件内数据有序。除了 level-0 之外，每一层内部的 SSTable 的 key 范围都不相交。
5. Manifest：Manifest 文件中记录 SSTable 在不同 level 的信息，包括每一层由哪些 SSTable，每个 SSTable 的文件大小、最大 key、最小 key 等信息。
6. Current：重启时，LevelDB 会重新生成 Manifest，所以 Manifest 文件可能同时存在多个，Current 记录的是当前使用的 Manifest 文件名。
7. TableCache：TableCache 用于缓存 SSTable 的文件描述符、索引和 filter。
8. BlockCache：SSTable 的数据是被组织成一个个 block。BlockCache 用于缓存这些 block（解压后）的数据。