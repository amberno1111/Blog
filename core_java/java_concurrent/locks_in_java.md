# Java中的锁
锁被用作控制多线程访问共享资源的方式，一般来说，锁能够防止多个线程同时写共享资源，也能够控制允许一个或多个线程同时读共享资源。
在Java中，`Lock`接口提供的API如下：

```java
// 加锁
void lock();
// 可中断的加锁
// 多提几句，持有锁的线程被中断之后，会释放锁；但是如果线程Sleep，是不会释放锁的；另外，Object.wait()也会释放锁
void lockInterruptibly();
// 尝试非阻塞的获取锁
boolean tryLock();
// 带有超时功能的非阻塞获取锁
boolean tryLock(long time, TimeUnit unit);
// 释放锁
void unlock();
// 返回绑定到这个Lock对象实例的一个Condition对象
// 在等待这个condition之前，当前的线程必须持有与这个condition绑定的锁
// 调用这个condition对象的`await()`方法将会自动释放锁，而在等待返回之前则会重新获取锁
Condition newCondition();
```

所以可以看出，锁的API就分为三类：
- 加锁
- 解锁
- 获取`condition`



JDK中`Lock`接口有如下的一些实现类：
- `ReentrantLock`
- `ReentrantReadWriteLock`
- `StampedLock`

但是，在一个现实的多线程的场景里，想要获取锁的线程可能会很多，所以对于这些`Lock`的实现类来说，它们要做的，除了加锁、解锁这些功能之外，还需要考虑**如何管理线程的排队，如何暂停以及唤醒线程等工作**。

上面的这些`Lock`实现类其实在内部都是依赖了JDK提供的一个模版类`AbstractQueuedSynchronizer`来实现这些功能的：
- 实现加锁解锁的一些功能，虽然AQS并不实现`Lock`接口，但是它所提供的功能必须要能够实现`Lock`API规定的一些功能，否则其他的`Lock`实现类就没法使用AQS来实现锁的功能了
- 使用一个CLH队列管理线程的排队
- 借助`LockSupport`实现线程的等待以及唤醒

所以后文会先分析AQS的实现，然后再分析具体的`Lock`接口实现类。


# `AbstractQueuedSynchronizer`解析





## AQS提供的API如何实现加锁解锁的功能

AQS如何提供锁的功能：
- 首先是使用一个整型变量来表示锁的状态，并且在内部提供来一些修改这个整型变量状态的方法


```java
 public class AbstractQueuedSynchronizer {

    /**
     * AQS 使用一个整型变量来表示锁的状态，可以通过修改 state 的值来表示锁的状态的变更
     * 1. 对于可重入锁，state 的值表示当前可以获取的锁的数量
     * 2. 对于独占锁，state 等于 0 的时候就是锁已经被获取
     */
    private volatile int state;

    /**
     * AQS 内部还提供了一些修改 state 的方法，但是这些方法基本上都是 protected final 的
     * 1. 模版方法的设计模式保证了安全性，修改 state 的操作只能过继承 AQS 进行修改，且一些重要方法不可重写
     * 2. 模版方法的设计模式同时也提高了 AQS 的易用性
     */
    protected final int getState(){
        return this.state;
    }
    protected final void setState(int newState) {
        // 非线程安全的状态修改方法，需要慎用
        this.state = newState;
    }
    protected final boolean compareAndSetState(int expect, int update) {
        // CAS 在存在多线程竞争的情况下，一定要使用这个方法来修改状态
        // 同时也可以基本确定，获取锁的方法肯定是通过这个 CAS 方法来修改状态的
        // 如果返回 true 就是加锁成功，否则加锁失败
        return STATE.compareAndSet(this, expect, update);
    }

    /**
     * 通过 CAS 设置尾节点
     */
    private final boolean compareAndSetTail(Node expect, Node update) {
        return TAIL.compareAndSet(this, expect, update);
    }

    private static final VarHandle STATE;
    private static final VarHandle HEAD;
    private static final VarHandle TAIL;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            STATE = l.findVarHandle(AbstractQueuedSynchronizer.class,
                    "state", int.class);
            HEAD = l.findVarHandle(AbstractQueuedSynchronizer.class,
                    "head", Node.class);
            TAIL = l.findVarHandle(AbstractQueuedSynchronizer.class,
                    "tail", Node.class);
        } catch (final ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```
- 然后提供了一些对应`Lock`API中规定的加锁解锁的方法，这些方法都是向外暴露的
AQS除了提供阻塞、非阻塞的获取锁的功能之外，还提供了独占式、共享式的获取、释放锁的API。
区分独占和共享的目的是为了提供锁的可重入功能。

这些API可以分为两类：
- 独占式
- 共享式
每一类都有对应`Lock`API中所需要的获取锁、可中断的获取锁、非阻塞的获取锁、支持超时非阻塞的获取锁、释放锁这些API，如下所示：

```java

    /**
     * 阻塞、独占式的获取同步状态的方法，同时也不支持中断
     * 对应 Lock API 的 lock 方法
     */
    public final void acquire(int arg) {

    }

    /**
     * 阻塞、独占式的获取同步状态的方法，支持中断
     * 对应 Lock API 的 lockInterruptibly()
     */
    public final void acquireInterruptibly(int arg) {

    }

    /**
     * 非阻塞，独占式的获取同步状态的方法，支持中断
     * 支持超时获取同步状态
     * 对应 Lock API 的 tryLock(long time, TimeUnit unit)、tryLock()
     */
    public final boolean tryAcquireNanos(int arg, long nanosTimeout) {

    }

    /**
     * 独占式的释放锁，注意释放锁并没有中断 or 超时 or 阻塞之说
     * 对应 Lock API 的 unlock()
     */
    public final boolean release(int arg) {

    }


    /**
     * 共享、阻塞的获取同步状态，不支持中断
     * 对应 Lock API 的 lock 方法
     */
    public final void acquireShared(int arg) {

    }

    /**
     * 共享、阻塞的获取同步状态，支持中断
     * 对应 Lock API 的 lockInterruptibly()
     */
    public final void acquireSharedInterruptibly(int arg) {

    }

    /**
     * 共享、非阻塞获取同步状态，支持中断
     * 支持超时获取同步状态
     * 对应 Lock API 的 tryLock(long time, TimeUnit unit)、tryLock()
     */
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) {

    }

    /**
     * 共享式的释放锁
     * 对应 Lock API 的 unlock()
     */
    public final boolean releaseShared(int arg) {

    }


```
- AQS使用了模版方法的设计模式，所以提供了一些需要重写的方法，这些方法非常简单，主要分为三类：独占式获取、释放同步状态的方法；共享式获取、释放同步状态的方法；判断同步状态是否被当前线程独占的方法；
```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
```



## AQS提供队列管理的功能

在多线程并发的场景下，肯定是需要有一个线程排队获取锁的机制的，AQS使用了一个CLH队列(FIFO)来做管理。CLH队列本身是使用链表来表示的。

### 数据结构定义

首先我们来看关于CLH的数据结构定义：

```java
public class AbstractQueuedSynchronizer {

    /**
     * CLH 队列就是用链表来表示的，所以必须有链表的节点数据结构，这里用 Node 来表示
     * 1. 为了形成链表，Node 这个数据结构必须有指向前一个节点和后一个节点的指针
     * 2. 入队和出队的时候，不可避免的要去修改前一个或者后一个节点的指针，
     *    所以提供了 prev 和 next 修改的 CAS 方法
     * 3. 为了表示当前的节点代表的是哪个线程，所以增加了 thread 字段
     *    关于 thread 字段为什么是 volatile 的，后面会有解释
     * 4. 为了区分独占式和共享式 Node，使用了 nextWaiter 字段，并且它的值使用静态变量来表示
     *    如果是共享式的 Node，则新建了一个 Node 变量
     *    如果是独占式的 Node，则为 null
     *    这个字段不是 volatile 修饰的，因为这个字段是不会修改的
     *
     */
    static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;

        volatile Node prev;
        volatile Node next;
        volatile Thread thread;

        Node nextWaiter;
        /** Constructor used by addWaiter. 构造方法 */
        Node(Node nextWaiter) {
            this.nextWaiter = nextWaiter;
            THREAD.set(this, Thread.currentThread());
        }
        final boolean isShared() {
            return this.nextWaiter == SHARED;
        }

        /**
         * 会有多个线程竞争入队，都企图把自己代表的节点设置为尾节点，同时就需要设置前一个节点的 next
         * 节点为自己代表的节点
         */
        final boolean compareAndSetNext(Node expect, Node update) {
            return NEXT.compareAndSet(this, expect, update);
        }

        /**
         * 返回当前节点 prev
         * 但是并不是直接返回 this.prev
         * 因为那是个 volatile 修饰的对象，直接返回给外层调用方，可能会在某个时期被另一个线程
         * 把 prev 的引用改掉导致程序运行出错，所以这里用一个新的 Node 对象指向 prev 之前指向的对象
         * 然后再返回
         */
        final Node predecessor() {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        /**
         * set 前一个节点指针的操作，基本上是不会存在竞争的，所以不需要 CAS
         */
        final void setPrevRelaxed(Node p) {
            PREV.set(this, p);
        }

        private static final VarHandle NEXT;
        private static final VarHandle PREV;
        private static final VarHandle THREAD;
        static {
            try {
                MethodHandles.Lookup l = MethodHandles.lookup();
                NEXT = l.findVarHandle(Node.class, "next", Node.class);
                PREV = l.findVarHandle(Node.class, "prev", Node.class);
                THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }


    }

    /**
     * 表示一个链表队列，只需要头节点、尾节点即可
     * 都是 volatile 修饰的，因为还是存在多线程竞争设置头节点和尾节点的情况的
     * 1. 对于头节点来说，只有在队列为空/初始化的时候，才会存在多个线程并发修改头节点的情况，
     *    每一个线程把自己的节点入队的时候，都会尝试把头节点改成自己代表的节点的
     * 2. 对于尾节点来说，情况就非常简单了，当多个线程尝试获取锁时，
     *    肯定是同时会有多个线程试图将自己代表的节点设置为尾节点的(这个操作必须是 CAS 的)
     */
    private volatile Node head;
    private volatile Node tail;
}
```


### 入队、出队的操作以及节点管理

然后再来看节点入队和出队的操作：

```java
    /**
     * 当某个线程无法获取锁的时候，就需要入队操作
     * 而这个操作是会存在多个线程竞争的，因为可能会存在多个线程同时获取锁失败需要入队的情况
     * 入队操作要做的事情是：
     * 1. 构建一个属于当前线程的 Node，并且把这个 Node 的 prev 设置为尾节点
     * 2. 通过 CAS 把这个 Node 设置为队列的尾节点，如果失败，则通过自旋式的重试，直到成功为止
     *    如果成功，还需要把之前的尾节点的 next 设置为当前的 Node
     * 3. 如果没有初始化，则需要初始化队列
     */
    private Node enq(Node node) {
        for (; ; ) {
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return oldTail;
                }
            } else {
                initializeSyncQueue();
            }
        }

    }
    /**
     * 入队操作
     * 因为需要支持独占、共享式的 Node，所以增加一个 addWaiter 方法
     * 支持创建独占、共享式的 Node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);
        // 实际上基本上和 enq() 方法差不多，感觉是可以复用 enq() 方法的
        // 但是不知道为啥 JDK 里选择里 copy 这段代码
        for (; ; ) {
            Node oldTail = this.tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return oldTail;
                }
            } else  {
                initializeSyncQueue();
            }
        }
    }

    /**
     * 初始化操作就是把 tail 和 head 指向一个 dummy 节点
     * 当然这个时候可以使用一下 CAS 操作，如果失败了，说明已经被其他线程初始化了
     */
    private void initializeSyncQueue() {
        Node h;
        if (HEAD.compareAndSet(this, null, (h = new Node()))) {
            tail = h;
        }
    }

    /**
     * 出队操作就相对简单，首节点的线程在释放同步状态时，会唤醒后续节点
     * 而后续节点获取到同步状态时，就将自己代表的节点设置为首节点
     * 由于只有一个线程能获取同步状态，所以设置头节点时不需要 CAS 操作
     */
    private void setHead(Node node) {
        head = node;
        // 这边设置为 null 是为了帮助 GC
        node.prev = null;
        node.thread = null;
    }
```

### 节点的状态








### 独占式同步状态的获取和释放

```java
    /**
     * 阻塞、独占式的获取同步状态的方法，同时也不支持中断
     * 对应 Lock API 的 lock 方法
     */
    public final void acquire(int arg) {
        // 独占式的同步状态获取，会先使用 tryAcquire() 方法尝试获取同步状态，
        // tryAcquire()方法是非阻塞的，如果获取成功，就直接返回了
        // 如果获取失败，则 addWaiter() 自旋式的加入到等待队列里
        // 然后通过 acquireQueued 自旋的获取同步状态直到成功为止
        // 如果失败，尝试自我中断，这是个兜底策略
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    /**
     * 当 node 已经在队列里之后，独占式的获取同步状态，并且不支持中断
     *
     */
    final boolean acquireQueued(final Node node, int arg) {
        // 因为不支持中断，所以设置为 false
        boolean interrupted = false;
        try {
            for (; ; ) {
                // 检查当前节点的前驱节点是否头节点，只有这样才能尝试获取同步状态，原因有二
                // 1. 为了维护 FIFO 原则，不能让后面的节点获取同步状态
                // 2. 头节点是成功获取到同步状态的节点，而头节点释放了同步状态之后，
                //    将会唤醒后续节点的线程，后续节点线程被唤醒之后需要检查自己的前驱节点是否头节
                //    因为不能保证只唤醒一个线程
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null;
                    return interrupted;
                }
                // 如果不是头节点或者获取同步状态失败，就需要检查是否需要 park
                // 对于独占式的获取同步状态的流程来说，这个条件肯定是 true
                if (shouldParkAfterFailedAcquire(p, node))
                    // 直接调用 LockSupport 让线程进入等待状态
                    // 虽然 parkAndCHeckInterrupt 这个方法可能会返回 true
                    // 但是 JVM 层面不会抛出异常，所以也算作不会主动中断
                    interrupted |= parkAndCHeckInterrupt();
            }

        } catch (Throwable t) {
            // 如果抛出异常了，则需要先把这个 node 从队列中移除
            // 然后再尝试中断这个线程
            // 最后抛出异常
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }


    }

    /**
     * 取消一个线程的 acquire 操作，其实就相当于把 Node 从队列中直接删除
     */
    private void cancelAcquire(Node node) {
        // 这一段等后面看 Node 的状态的时候再详细分析

    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 这一段等后面看 Node 的状态的时候再详细分析
    }

    private final boolean parkAndCHeckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

    private void unparkSuccessor(Node node) {

    }
    /**
     * 独占式的释放锁，注意释放锁并没有中断 or 超时 or 阻塞之说
     * 对应 Lock API 的 unlock()
     */
    public final boolean release(int arg) {
        // 释放锁就比较简单了，需要唤醒后续的线程
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null) {
                unparkSuccessor(h);
            }
            return true;
        }
        return false;
    }
```
