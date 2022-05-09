# Redis数据结构

Redis数据结构可以从两个角度来看：
- 从用户使用的角度看，Redis暴露给用户使用的数据类型可以分为String、List、Hash、Set、SortedSet
- 从Redis底层实现来看，可以分为SDS、dict、ziplist、skiplist、intset、quicklist这几种，Redis暴露给用户使用的数据类型都是由这些数据结构实现的