# 为什么Redis这么快

1. Redis的操作完全基于内存，类似于HashMap，查找效率是O(1)

2. 数据结构足够简单，对数据的操作也很简单，而且数据结构也是经过专门设计的，比如String就是使用SDS实现的，它的空间预分配和惰性空间释放能极大的提高性能。参考[Redis字符串与SDS](data_structure/simple_dynamic_string.md)；比如使用了跳表来实现的[Sorted Set](data_structure/sorted_set.md)，就拥有非常高的查询以及插入效率

3. Redis的单进程单线程模型避免了不必要的上下文切换以及线程竞争、也不需要考虑各种加锁解锁的操作，极大的节省了性能开销。关于多线程的性能开销，可以查看[多线程编程的挑战](../core_java/java_concurrent/concurrent_programming_challenges.md)中的上下文切换、死锁等章节

4. Redis使用了[I/O多路复用模型](../operating_system/io_models.md)，实现了非阻塞I/O。

## Redis单线程模型

Redis的核心是文件事件处理器(File Event Handler)，它的File Event Handler由四部分组成：套接字、I/O多路复用程序、文件事件分派器以及事件处理器。
具体的运行过程是：采用非阻塞的I/O多路复用模型，监听多个socket，同步、有序的将产生事件的socket放到一个FIFO内存队列当中，然后文件时间分派器也总是以同步、有序的方式将事件分派给事件处理器进行处理。
因为这整个过程都是单线程的，或者说Redis的文件事件处理器就是单线程的，所以Redis被称为单线程模型。

[这篇文章里](https://segmentfault.com/a/1190000020014518?utm_source=tag-newest)的图很清楚的展示了文件时间处理器的工作流程：
![](https://segmentfault.com/img/bVbv8Qx?w=1240&h=813)



