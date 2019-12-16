# List

很多语言都会自带链表结构，但是C语言是没有的，所以Redis也是自己实现了链表数据结构。

## 链表的定义

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 具体数值，这里是一个void的指针，也就是可以放任何内容
    // 这东西相当于一个泛型啊
    void *value;
} listNode;

typedef struct list {
    // 头节点
    listNode *head;
    // 尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;

```

从上面的数据结构中可以看出，Redis的List实现具有如下的特性：
- 双端链表，获取前置or后置节点的时间复杂度是O(1)
- 无环，首节点前置节点和尾节点后置节点都是null
- list结构带有头节点和尾节点，获取头节点和尾节点的时间复杂度是O(1)
- 带有长度，获取链表长度的时间复杂度是O(1)
- 链表节点的`viod*`指针使得节点可以不同类型的值


## List数据类型的应用

可以用来做任务队列，利用`lpush`、`rpush`命令实现入队，`lpop`、`rpop`实现出队列：
- 可以左进(`lpush`)右出(`rpop`)或者右进左出来实现先进先出队列
- 也可以左进左出或者右进右出来实现先进后出队列

当然可以用`brpop`、`blpop`实现阻塞的访问。





