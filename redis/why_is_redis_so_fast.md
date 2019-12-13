# 为什么Redis这么快

1. Redis的操作完全基于内存，类似于HashMap，查找效率是O(1)

2. 数据结构足够简单，对数据的操作也很简单，而且数据结构也是经过专门设计的，比如String就是使用SDS实现的，它的空间预分配和惰性空间释放能极大的提高性能。参考[Redis字符串与SDS](data_structure/simple_dynamic_string.md)；比如使用了跳表来实现的[Sorted Set](data_structure/sorted_set.md)，就拥有非常高的查询以及插入效率

3. Redis的单进程单线程模型避免了不必要的上下文切换以及线程竞争、也不需要考虑各种加锁解锁的操作，极大的节省了性能开销。关于多线程的性能开销，可以查看[多线程编程的挑战](../core_java/java_concurrent/concurrent_programming_challenges.md)中的上下文切换、死锁等章节

4. Redis使用了[多路I/O复用模型](../operating_system/io_models.md)，实现了非阻塞I/O。
