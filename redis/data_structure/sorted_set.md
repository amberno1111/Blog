# Redis Sorted Set

Redis里的Sorted Set是个复合结构，由一个hash和一个skiplist组成，其中hash用来保存value和score的对应关系，skiplist用于给score排序。

## Skiplist

因为Redis的应用场景，SortedSet会有很多插入、删除、更新的操作，为了保证这些操作的高效，只能使用链表来实现，链表可以保证插入、删除、更新操作的时间复杂度都在O(1)。

但是，即使是排好序的链表，查找某个特定值的操作的时间复杂度也是O(N)，因为只能通过遍历整个链表来查找。链表的特性使得二分查找在这个场景下毫无用武之地。

跳跃表借鉴了二分查找的思想，使链表的查询效率提升到了O(logN)，跳跃表的结构如下图所示：
![skiplist](https://user-images.githubusercontent.com/16413289/70847659-bb599680-1ea1-11ea-957f-8a3777df1ce3.jpeg)

跳跃表的主要思想是将链表与二分查找相结合，它维护了一个多层级的链表结构，每一层都可以看成是一条链表，跳表有如下的特点：
- 一个跳表应该有几个层（level）组成
- 跳表的第0层包含所有的元素，且节点值是有序的
- 每一层都是一个有序的链表，层数越高应越稀疏，这样在高层次中能'跳过’许多的不符合条件的数据
- 如果元素x出现在第i层，则所有比i小的层都包含x
- 每个节点包含key及其对应的value和一个指向该节点第n层的下个节点的指针数组，x->next[level]表示第level层的x的下一个节点

## zskiplist

来看下Redis是怎么实现跳表的：

```c
// 跳跃表的节点数据结构
typedef struct zskiplistNode {
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;

// 跳跃表结构
typedef struct zskiplist {
    // 头结点和尾节点
    struct zskiplistNode *header, *tail;
    // 节点的数量
    unsigned long length;
    // 层数
    int level;
} zskiplist;

```




