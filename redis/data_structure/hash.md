# Hash


Redis里的Hash就是一个哈希表，类似于[Java里的HashMap](../../core_java/jdk/hashmap.md)，但是Redis里的Hash只能存储key-value都是String类型的数据，每个Hash可以存储`2^32 - 1`个键值对。

从使用者的角度看，Redis暴露给外部的数据结构是[Hashes](https://redis.io/topics/data-types-intro)。但在Redis的内部实现中，Hash是使用dict数据结构来实现的，dict称为字典，其底层实现是哈希表。

字典经常作为一种数据结构内置在编程语言中，但是C语言并没有，所以Redis自己实现了字典这个数据结构。

## 哈希表的定义

```c
typedef struct dictht {
    // entry数组
    dictEntry **table;
    // 总大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 已有节点的数量
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;
    // value
    union {
        void *val;
        unit64_t u64;
        int64_t s64;
    }
    // 指向下一个Entry的指针
    struct dictEntry *next;
} dictEntry;

```

看起来这个实现和Java的HashMap基本一致，这里就不分析具体实现了。Redis利用这个哈希表实现了字典dict，实际上这个字典就是对哈希表的一个封装。

## 字典dict的定义

```c
typedef struct dict {
    // type和private属性是针对不同类型的键值对，为创建多态字典而设置的
    dictType *type;
    void *private;
    // 哈希表数组，这是一个数组，每个元素都一个dictht哈希表
    // 一般情况下只使用ht[0]的哈希表
    // h[1]哈希表只会在ReHash的时候使用
    dictht ht[2];
    // rehash 索引
    int trehashindex;
}

```


## 渐进式ReHash

Redis的Hash和Java的HashMap不同的地方在于，Redis实现了渐进式ReHash。想象一下Redis的使用场景：**服务端Redis的Hash里可能存着上千万的键值对，如果一次性对这些键值对进行ReHash，庞大的计算量可能会导致Redis在一段时间内停止服务**。所以Redis采用了渐进式ReHash的策略，分多次、渐进式的把数据ReHash到新的Entry数组里。

渐进式ReHash是一个典型的分而治之、以空间换时间的操作：
- 首先给ht[1]分配空间，这样字典就同时拥有了ht[0]和ht[1]两个哈希表
- 在字典中维持一个索引计数器变量rehashindex，并将它的值设置为0，表示rehash工作正式开始
- 在rehash期间，每次对字典执行添加、删除、查找、更新操作时，程序除了执行指定的操作之外，还会顺带将ht[0]哈希表在rehashindex索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，rehashindex属性的值加一
- 随着字典操作的不断执行，最终会在某个时间点上，ht[0]所有的键值对都会被rehash至ht[1]，这时候把rehashindex值设置为-1，表示rehash完成










