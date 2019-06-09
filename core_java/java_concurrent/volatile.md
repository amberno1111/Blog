# volatile

## Abstract

在Java并发编程中，`synchronized`和`volatile`关键字都扮演者很重要的角色，`volatile`是轻量级的`synchronized`，它能够保证在并发环境下**共享变量的可见性**，即当一个线程在修改一个变量时，其他线程也能读到这个修改的值。

`volatile`不会引起上下文的切换和线程调度，在某些场景下使用得当的话可以达到和`synchronized`一样的效果而且更轻量。

ps：在阅读本篇文章之前，最好了解一下[CPU缓存一致性协议](MESI.md)。

## 实现原理

在Java中使用`volatile`关键字修饰一个共享变量，当Java代码在使用JIT编译器生成汇编指令的时候，遇到对这个共享变量进行写操作的代码，JIT就会多增加一行汇编代码，类似这样`lock add1 $0x0, (%esp);`，这其实就是我们在[CPU缓存一致性协议](MESI.md)中所提到的`LOCK`前缀指令。

JIT多增加的这一行汇编代码会做两件事情：
- 把当前处理器缓存行的数据写回到系统内存
- 这个写回到内存的操作会使得在其他CPU里缓存了该内存地址的数据无效

所以`volatile`的实现原理就很明显了：
当对声明了`volatile`的变量进行写操作时，JVM会向CPU发送一条`lock`前缀的指令，将这个变量在缓存行的数据写回到内存。但是就算写回到内存，其他处理器的缓存里的值还是旧的，再执行操作就会有问题。所以这里就利用到了CPU缓存一致性协议，CPU会主动嗅探到这个变量值的改变，并且下次执行的时候会自动从内存中再把这个变量的值读取到缓存中才执行操作。

## `volatile`的使用优化

**TODO:我其实只在JDK7里看到这样的代码，在JKD8里并没有`Padded Atomic Reference`这个内部类，应该是改过了。之后需要研究下JDK8的代码然后补上这部分。**


Java中有个由链表组成的无界阻塞队列`LinkedTransferQueue`，它使用了一种**追加字节的方式来优化队列出队和入队的性能**，这其实就是利用了处理器缓存的一个例子。

代码是这样的：
```java
private transient final PaddedAtomicReference<QNode> head;
private transient final PaddedAtomicReference<QNode> tail;

static final class PaddedAtomicReference<T> extends AtomicReference <T> {
    // 这里多声明了15个变量，每个变量引用占4字节，总共60字节
    Object p0, p1, p2, p3, p4, p5, p6, p7, p8, p9, pa, pb, pc, pd, pe;
    PaddedAtomicReference(T r) {
        super(r)
    }
}

```

它使用了一个内部类来定义头节点head和尾节点tail，然后这个内部类只干了一件事，就是把共享变量追加到64字节。一个对象的引用占4个字节，它一共追加了15个变量，再加上父类value的引用，总共占用了64个字节。

### 为什么追加字节可以提升出队入队的效率

对于现代的很多CPU而言，比如Intel的Core系列，它们的告诉缓存L1、L2、L3的每一个缓存行都是64字节宽，不支持部分填充缓存行。也就意味着如果队列的头节点和尾节点不足64位宽的话，CPU会将它们都读到同一个缓存行中，**当CPU开始操作头节点时，这个缓存行的数据会被锁定，这时候其实尾节点也被锁定了，在缓存一致性机制的作用下，其他CPU无法写尾节点**。但是在队列的这个场景中，头节点和尾节点可以同时写，并且队列的头节点和尾节点是会被程序高频率修改的部分，因此在多处理器环境下会严重影响队列的性能。

追加到64字节之后，头节点和尾节点不在同一个缓存行，也就不会被同时锁定，这就能大大的提高队列元素出队和入队的效率。



