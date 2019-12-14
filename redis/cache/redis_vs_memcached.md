# Redis vs Memcached

需要缓存的时候会做一些技术选型，目前比较常用的就Redis和Memcached两种。

功能支持方面：
- Redis支持更多的数据结构；Memcached只支持简单字符串
- Redis可以通过AOF和RDB持久化数据，挂了可以恢复数据；Memcached服务挂了数据就全丢了
- Redis还支持事务等功能；Memcached专注于存储key-value类型的数据
- Redis支持集群，具有高可用、可扩展性；Memcached不支持，需要自己二次开发

性能方面：
- 存储小数据时Redis的性能更好，存储100KB以上的数据时Memcached性能更好
- 只是在缓存(key-value)这个场景下，Memcached内存使用率更高；如果Redis使用了Hash，则Redis内存使用率更高。

