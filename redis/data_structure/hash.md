# Hash


Redis里的Hash就是一个哈希表，类似于[Java里的HashMap](../../core_java/jdk/hashmap.md)，但是Redis里的Hash只能存储key-value都是String类型的数据，每个Hash可以存储`2^32 - 1`个键值对。

从使用者的角度看，Redis暴露给外部的数据结构是[Hashes](https://redis.io/topics/data-types-intro)。但在Redis的内部实现中，Hash是使用dict数据结构来实现的，dict称为字典，其底层实现是哈希表。

字典经常作为一种数据结构内置在编程语言中，但是C语言并没有，所以Redis自己实现了字典这个数据结构。

## dict的定义






