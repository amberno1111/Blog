# Redis的过期策略以及内存淘汰机制

过期策略指的是存储在Redis中的key失效了之后，删除这些key的策略。

内存淘汰机制是指Redis在内存达到某个阈值之后，删除一些key的策略。


## 过期策略

Redis使用**定期删除**和**惰性删除**配合的方式来删除失效的key。
- 定期删除：Redis默认每隔100ms就随机抽取一些设置了过期时间的key，检测这些key是否已经过期，如果过期，就删除
- 惰性删除：Redis不主动删除，而是在客户端请求获取某个key的时候，Redis会检测一下这个key是否过期，如果已经过期，就直接删除，不返回给客户端


## 内存淘汰机制

Redis在内存使用达到配置的阈值的时候，就会触发内存淘汰机制，选取一些key来进行删除，主要有一下这些策略：
- noeviction：默认策略，不删除，内存满时直接报错
- allkeys-lru：从所有的key中删除最少使用的key
- allkeys-random：从所有的key中随机删除一些key
- volatile-lru：从设置了过期时间的key中，选取最少使用的删除
- volatile-random：从设置了过期时间的key中，随机删除一些key
- volatile-ttl：从设置了过期时间的key中，删除有更早过期时间的key


