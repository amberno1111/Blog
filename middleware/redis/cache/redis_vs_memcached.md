# Redis vs Memcached

需要缓存的时候会做一些技术选型，目前比较常用的就Redis和Memcached两种。

功能支持：
- Redis支持更多的数据结构；Memcached只支持字符串
- Redis可以通过AOF和RDB持久化数据，挂了可以恢复数据；Memcached服务挂了数据就全丢了
- Redis还支持事务等功能；Memcached专注于存储key-value类型的数据
- Redis支持集群，具有高可用、可扩展性；Memcached不支持，需要自己二次开发
- Redis单线程；Memcached多线程，可以支持多核

性能：
- 存储小数据时Redis的性能更好，存储100KB以上的数据时Memcached性能更好
- 只是在缓存(key-value)这个场景下，Memcached内存使用率更高；如果Redis使用了Hash，则Redis内存使用率更高

应用场景：
- 在分布式缓存的场景下，虽然Memcached在内存使用率以及大数据量存储更高效这两个方面具有优势，但是考虑到丢数据之后的快速恢复、缓存的集群高可用性等等因素，还是Redis更合适一些
- 在消息中间件的场景下，肯定也是选Redis而不是Memcached，当然Redis也只能作为简单的消息中间件使用
- 在缓存SQL语句、用户临时性数据、session等场景下，Memcached会更合适，因为Memcached在这些场景下的性能更好





