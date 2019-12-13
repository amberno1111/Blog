# Redis

Redis是一个基于内存的key-value存储系统，可以用来做数据库、缓存、消息中间件等。

Redis是单进程单线程模型，采用队列模式将并发访问变为串行访问，但是因为它是基于内存的，并且采用了I/O多路复用等技术，所以单线程的Redis性能依然是非常高的。

Redis支持的数据结构比较丰富，比较常用的基础数据类型有五种：String、Hash、List、Set、SortedSet，更高阶一点的有HyperLogLog、Pub/Sub之类的。

Redis内置了复制(Replication)、Lua脚本(Lua Scripting)、LRU事件驱动(LRU Eviction)、事务(Transactions)和不同级别的磁盘持久化(Persistence)，并且通过哨兵(Sentinel)和自动分区(Cluster)保证高可用性。


