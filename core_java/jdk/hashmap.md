# HashMap

HashMap大概是平时最常用的数据结构了，**它的底层实现是数组+单向链表+红黑树**。

## 哈希表

HashMap是基于哈希表的思想来实现的，所以先来了解下什么是哈希表？

在讨论哈希表之前，先来了解下其他数据结构在插入、删除、查找方面的性能：
- 数组：采用一段连续的内存空间来存储数据
    - 根据数组下标查找数据，时间复杂度O(1)；根据给定值查找数据，需要遍历数组，时间复杂度O(N)；如果是有序数组，可以用二分查找等方式，把时间复杂度降低到O(logN)
    - 插入和删除操作，涉及到数组元素的移动，时间复杂度都是O(N)
- 链表：采用不连续的节点来存储数据
    - 查找操作需要遍历整个链表，时间复杂度O(N)
    - 插入和删除操作，只需要改变节数据中指向前驱节点和后置节点的指针，时间复杂度O(1)
- 二叉树：以树形的结构排列节点来存储数据，查找、插入、删除操作的时间复杂度都是O(logN)

**哈希表的性能比上述几种数据结构都要高一些，它的查找、插入、删除操作的时间复杂度都是O(1)**。哈希表非常好的利用了数组的通过下标查找效率非常高的特性，它的实现就是利用一个数组和一个哈希函数，向哈希表中插入数据时，通过哈希函数计算出这个数据可以放到数组的哪个下标的位置上，然后把数据存入到数组中的这个目标位置。查找和删除操作同理，所以他们的时间复杂度都是O(1)。

但是哈希表也不是没有缺点，那就是可能会出现**哈希冲突**，也就是可能会出现几个数据，经过哈希函数的计算之后得到的结果(数组下标的值)是相同的，这时候就出现了冲突。一般来说，我们可以通过**链地址法、开放定址法(发现冲突、继续寻找下一块未被占用的内存)、再散列函数法等等算法来解决哈希冲突的问题。**

### HashMap如何解决哈希冲突

我们看到HashMap底层是数组+链表+红黑树实现的，也就是说它是使用链地址法来解决哈希冲突的，具体做法如下：
- HashMap的数组里存的数据结构是`Node`，这个数据结构里有指向下一个`Node`的指针
- 对于查找操作，hash寻址的时候，如果找到的`Node`里存储的指向下一个`Node`的指针为``null`，那就意味着没有哈希冲突，直接返回这个`Node`即可，时间复杂度O(1)；如果不是`null`，就只能遍历这个链表，然后依靠key对象的`equals()`方法逐一比对查找符合要求的`Node`，时间复杂度O(N)
- 对于插入和删除操作，基本上同查找操作；不过插入操作有个链表头插入和尾插入的优化，可以看后面的内容


## Fileds 

先来看一下HashMap的一些属性字段：

```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable {
    // 默认初始容量为 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 当节点数量大于 8 就会转成红黑树存储
    static final int TREEIFY_THRESHOLD = 8;
    // 当节点数量小于 6 就会转成链表存储
    static final int UNTREEIFY_THRESHOLD = 6;
    // 红黑树的最小容量是 64
    static final int MIN_TREEIFY_CAPACITY = 64;

    // 后面这些是HashMap的属性
    // Node是Map.Entry的实现类
    // 这里存储的Node数组容量是2次幂
    // 每一个Node本质上就是一个单向链表
    transient Node<K,V>[] table;
    // 因为for keySet() values()的存在
    // 所以这里cache了entrySet便于使用
    transient Set<Map.Entry<K,V>> entrySet;
    // HashMap当前的容量
    transient int size;
    // HashMap被改变的次数，用于快速失败
    transient int modCount;
    // 下一次HashMap扩容的大小
    int threshold;
    // 扩容用的负载因子
    final float loadFactor;
}

```

基本上可以看到HashMap里的字段都是紧扣**HashMap是通过数组+链表+红黑树实现的**这个主题而设置的。

### Node单向链表的实现

上面我们说HashMap使用了单向链表来避免哈希冲突，来看一下`Node`的数据结构：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    // 存储哈希值
    final int hash;
    final K key;
    V value;
    // 链表中的下一个节点
    Node<K,V> next;
}
```

### 从Filed可以看出的一些HashMap的特性

- 默认初始容量16，是2次幂，(其实HashMap的容量都是2次幂，扩容和缩容都是按照这个规则)
- 扩容机制根据`LoadFactor`和`Capacity`决定
- 由于哈希冲突的存在，可能会出现一个链表很长的情况，这时候O(N)的查找效率不是很高，因此HashMap做了一个优化，当链表的节点超过8时，就把链表转成红黑树实现，这样可以把查找效率提升到O(logN)；但是因为红黑树占用空间略大所以当树中的节点数少于6个时，再改成链表实现以优化空间占用。

### 为什么负载因子默认是0.75

这是个面试中经常问的问题。使用0.75作为默认的负载因子，是综合了查找效率与空间复杂度的一个平衡的选择。
- 负载因子过高，例如为1，虽然减少了空间开销，提高了空间利用率，但同时也增加了查询时间成本
- 如果负载因子过低，例如0.5，虽然可以减少查询时间成本，但是空间利用率很低，同时提高了rehash操作的次数
- 在设置初始容量时应该考虑到映射中所需的条目数及其负载因子，以便最大限度地减少rehash操作次数，所以，一般在使用HashMap时建议根据预估值设置初始容量，减少扩容操作



## Constructors

来看下HashMap的构造方法：
```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable {
    // 无参构造函数没啥意思，就是初始容量 16，默认负载因子是 0.75
    // 不过比较有意思的是，初始容量并不是这个时候赋值的，相当于做了一个懒加载的优化
    // 而是在第一次 put 操作的时候做的，可以看后面的内容
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    // 这个构造方法赋值了负载因子以及初始容量
    // 可以从 tableSizeFor() 这个方法里看出来容量肯定是2次幂
    // tableSizeFor()方法返回一个比给定整数大且最接近的2的幂次方整数，如给定10，返回2的4次方16
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    // 直接用原有的Map构建一个新的Map，核心是 putMapEntries 方法
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        // 这里依然是一个懒加载的思路，如果传进来是空的map，就啥也不做
        if (s > 0) {
            // 这边是需要判断table是否已经初始化的，如果没有，那就需要重新执行初始化
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            } else {
                // Because of linked-list bucket constraints, we cannot
                // expand all at once, but can reduce total resize
                // effort by repeated doubling now vs later
                while (s > threshold && table.length < MAXIMUM_CAPACITY)
                    resize();
            }
            // 开始遍历map，插入数据
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
}
```

## Methods


### HashMap的hash算法

从上面可以看到，HashMap的构造函数调用了`pubEntries`方法，然后`pubEntries`又调用了`hash()`，这个就是HashMap里使用的hash算法。

```java
// 看起来还是很简单的，就是通过key的hashCode()的高16位异或低16位实现的
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

求解了Hash值之后，如何寻址到数组的下标呢？也是一个位操作`(n - 1) & hash`，其中`n`表示数组的容量。


### HashMap的扩容操作

扩容操作都在`resize()`方法里，主要分为两步：
- 扩容：创建一个新的`Node`空数组，长度是原数组的2倍
- ReHash：遍历原数组，把所有的数据重新Hash到新数组；ReHash的步骤是必须的，因为寻址是通过`(n - 1) & hash`来实现的，也就是说，数据存储的位置实际上跟整个数组的容量有关。当扩容之后，数据存储在数组中的下标就会发生变化，所以需要ReHash，而不是原样直接复制过去

来看一下`resize()`方法：
```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable {
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 1. oldCap > 0，即resize函数在size > threshold时被调用
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 如果旧的HashMap已经超过最大容量，则直接扩容至Integer.MAX_VALUE，后续就不会再继续扩容了
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 容量 newCap 和 阈值 newThr 都直接翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        // 2. oldCap <= 0 & oldThr > 0，表示用户创建了一个HashMap
        // 但是这个HashMap里没有值，所以无需扩容，直接赋值旧的容量阈值即可
        // 使用的构造函数为HashMap(int initialCapacity, float loadFactor) 或 HashMap(int initialCapacity)
        // 或 HashMap(Map<? extends K, ? extends V> m)，导致 oldTab 为 null，oldCap 为0， oldThr 为用户指定的 HashMap的初始容量。
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        // resize（）函数在table为空被调用。oldCap 小于等于 0 且 oldThr 等于0，用户调用 HashMap()构造函数创建的　HashMap，所有值均采用默认值，oldTab（Table）表为空，oldCap为0，oldThr等于0，
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        // 开始初始化 table，进行reHash操作
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
}
```



## 尾插入的优化

我们可以看到，上面ReHash的过程，如果往链表中加入新的节点，都是加在链表尾部的。而在JDK1.7里，是加在链表头部的，所以JDK1.7的HashMap会有个在多线程环境下出现死循环的bug。

具体原理是这样的，首先假设现在有A->B->C这样的一个链表，然后在多线程环境下，一个线程对HashMap扩容了，进行了ReHash操作，这时候变成B->A，然后C被移动到其他地方。另外一个线程也同时进行了ReHash操作，但是发现容量已经足够了所以不扩容或者即使扩容了之后，B和A的Hash还落在同一个数组下表上，然后ReHash之后，就变成了A->B，但是ReHash又不会继续去修改B的的next指针，所以就出现了A->B->A的环状列表，就出现了死循环。

JDK1.8之后已经解决了这个问题，使用了尾部插入，尾部插入可以保证不去修改链表元素的顺序，所以不会出现死循环。**但是！HashMap依然不是线程安全的，因为各种修改数据的操作都没有加锁，在多线程环境下肯定是会有问题的。推荐使用ConcurrentHashMap**。











