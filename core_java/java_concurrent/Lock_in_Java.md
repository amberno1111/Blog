# Java中的锁

锁被用来控制多线程访问共享资源的方式，一般来说一个锁能够防止多个线程同时访问共享资源，但是也有些锁能够支持多个线程并发的访问资源，比如读写锁。

Java本身提供了`synchronized`来作为锁使用，但是`synchronized`的功能不够强大，它不支持可操作的锁获取与释放，也不支持超时获取锁等功能，在实际应用中有比较多的局限性。

所以JDK还提供了`Lock`API，用于支持可中断的获取、释放锁，超时获取锁等等新特性。来看一个简单的例子：
```java
Lock lock = new ReentrantLock();
// 不在try里加锁，为了避免获取到锁之后，如果try里发生异常，锁会被无故释放
lock.lock();
try {
    // do something
} finally {
    // finally 块能保证锁肯定会被释放
    lock.unlock();
}
```

相比于`synchronized`，`Lock`API具有以下新特性：
- 尝试非阻塞的获取锁：当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，就成功持有锁
- 支持可中断的获取锁：`Lock`API支持获取锁的线程可以响应中断，响应中断之后，中断异常将会抛出，锁会被释放
- 支持超时获取锁：在指定时间截止前获取锁，如果时间到还没获取到锁，就直接返回

`Lock`API提供以下接口：
- 获取锁的方法：`lock()`，`tryLock()`，`tryLock(long time, TimeUnit timeUint)`，`lockInterruptibly()`
- 释放锁的方法：`release()`、`tryRelease()`
- `newCondition()`：获取等待通知组件，该组件与当前的锁绑定，只有当前线程获取了锁，才能调用该组件的`wait()`方法，调用后，该线程将释放锁。

## 队列同步器

上面提到的`Lock`接口的实现基本都是通过聚合了一个队列同步器(AbstractQueuedSynchronizer)来实现的，AQS使用了一个同步int变量来表示同步状态，用一个FIFO队列来完成资源获取线程的排队工作。

AQS的使用方式是继承，它提供了比较多的模板方法，也提供了一些可覆写的方法。

### 可覆写的方法

- `tryAcquire(int arg)`：独占式获取锁的方法，实现该方法需要判断同步状态是否符合预期，并且使用CAS设置同步状态
- `tryRelease(int arg)`：独占式释放锁的方法
- `tryAcquireShared(int arg)`：共享式获取锁的方法，返回大于等于0的值，则表示获取成功，反之获取失败
- `tryReleaseShared(int arg)`：共享式释放锁状态
- `isHeldExclusively()`：表示同步器是否在独占模式下被线程占用，一般用来判断同步状态是否被当前线程独占

### 模板方法

- `acquire(int arg)`
- `acquireInterruptibly(int arg)`
- `tryAcquireNanos(int arg, long nanos)`
- `acquireShared(int arg)`
- `acquireSharedInterruptibly(int arg)`
- `tryAcquireSharedNanos(int arg)`
- `release(int arg)`
- `releaseShared(int arg)`
- `Collection<Thread> getQueuedThreads()`

模板方法也基本分为三类：
- 独占式的获取和释放锁
- 共享式的获取和释放锁
- 查询队列中的等待线程情况

### Mutex独占锁

然后我们可以尝试使用AQS来实现一个独占锁，独占锁就是一个时刻只有一个线程能占有锁，代码示例如下：
```java
public class Mutex implements Lock {

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            // 先使用CAS获取锁
            if (compareAndSetState(0, 1)) {
                // 如果成功了则设置为当前线程独占锁，并返回 true
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            } 
            // 因为 tryAcquire 是尝试非阻塞的获取锁，所以调用后立刻返回
            // 如果想要阻塞的获取锁，可以使用父类的 acquire() 方法，它会把线程放到同步队列里等待
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            // 使用 state 判断是否处于同步状态
            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            // 判断是否处于独占同步状态，state = 1 表示当前线程获取了锁
            return getState() == 1;
        }
        
        Condition newCondition() {
            return new ConditionObject();
        }
        
    }


    private static Sync sync = new Sync();
    
    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquire(1);
    }

    @Override
    public void unlock() {
        // 可以看一下父类的 release() 方法，最终会调用 tryRelease() 方法的
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

### AQS的实现分析

#### 同步队列

AQS依赖内部的一个FIFO队列来完成同步状态的管理，当前线程获取同步状态失败时，会将当前线程以及等待状态信息构造为一个节点加入到FIFO队列当中，同时会阻塞当前线程；当同步状态被释放时，会唤醒队列头节点的线程，使其尝试再次获取同步状态。

AQS中队列节点的源码如下：
```java
static final class Node {
    // 表示当前的等待状态
    // 1. CANCELLED 线程可能在等待获取锁的过程中超时或者被中断，需要从同步队列中取消等待，节点进入该状态后不会变化
    // 2. SIGNAL 表示后续节点处于等待状态，当前节点释放同步状态之后，必须唤醒后续节点
    // 3. CONDITION 节点在等待队列中，节点等待在condition上，当其他线程对Condition调用了signal()方法之后，该节点的线程将会从等待队列中加入到同步队列中
    // 4. PROPAGATE 表示下一次共享式同步状态会被无条件的传播下去
    // 5. INITIAL 初始状态 
    volatile int waitStatus;
    // 保存了前一个和后一个节点的引用
    volatile Node prev;
    volatile Node next;
    // 保存了线程信息
    volatile Thread thread;
    // 等待队列中的后续节点
    Node nextWaiter;
}
```

当有了节点之后，队列的数据结构就可以构建了，AQS的数据结构源码如下：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer {
        // 队列的头节点和尾节点
        private transient volatile Node head;
        private transient volatile Node tail;
        // 表示同步状态的变量
        private volatile int state;

        // 我们看到AbstractQueuedSynchronizer实际上是继承了 AbstractOwnableSynchronizer的
        // 这里列一下来自父类的 filed
        // 表示占用当前同步状态的线程
        private transient Thread exclusiveOwnerThread;

    }
```

然后我们来看一下入队操作，因为可能存在很多线程同时竞争同步状态的场景，所以就会有很多线程一起进行入队操作，这时候需要注意线程安全问题，AQS提供了使用CAS进行入队操作的方法，如下：
```java
private static final VarHandle TAIL;
// 看起来这个方法就是直接调用了底层的 native 函数进行 CAS 操作
private final boolean compareAndSetTail(Node expect, Node update) {
    return TAIL.compareAndSet(this, expect, update);
}

```

出队操作就相对来说比较简单了，当队列首节点的线程在释放同步状态时，将会唤醒后续节点，而后续节点在获取同步状态之后，就会把自己设置为队列的首节点。因为同步状态已经保证了只有一个线程能获取到，所以这个设置首节点的操作不需要使用CAS来保证。

#### 独占式同步状态的获取与释放

AQS的`acquire()`方法可以获取同步状态，该状态对中断不敏感，也就是说线程获取同步状态失败进入同步队列之后，后续对线程进行中断操作时，线程不会从同步队列中移除。这个方法的逻辑是这样的：
- 首先尝试非阻塞的获取同步状态，如果成功则直接返回，如果失败就进行下一步
- 然后自旋式的创建以及设置尾节点，直到成功的把创建好尾节点并加入到同步队列
- 再然后自旋式的在同步队列中等待获取同步状态，直到获取成功才返回

来看一下源代码：
```java
public final void acquire(int arg) {
    // tryAcquire(arg) 用于获取同步状态，如果获取成功，就直接返回了
    // 如果获取不成功，就尝试加入到队列
    // tryAcquire(arg) 这里就不解释了，这个方法需要自己实现的
    // addWatier(Node.EXECLUSIVE)是构造节点，并把节点设置为尾节点
    // acquireQueued()是获取同步状态，会一直自旋直到获取同步状态成功而已
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}
// 来看一下 addWaiter(Node.EXCLUSIVE)
// Node.EXCLUSIVE 表示一个独占式的节点
private Node addWaiter(Node mode) {
    // 创建 Node 的时候会设置 nextWaiter 以及需要存储在这个节点中的线程
    Node node = new Node(mode);
    // 这里是一个死循环，直到设置尾节点成功为止
    for(;;) {
        Node oldTail = this.tail;
        // 判断是否为 null，是因为刚开始尾节点就是 null，需要初始化 
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            // 使用 CAS 设置尾节点
            if (compareAndSetTail(oldTail, node)) {
                // 尾节点设置成功之后，在把之前尾节点的 next 指向当前的节点
                oldTail.next = node;
                return node;
            }
        } else {
            // 初始化首节点和尾节点
            initializeSyncQueue();
        }
    }
}

private final void initializeSyncQueue() {
    Node h;
    // HEAD 是一个静态变量，VarHandler 类型，是为了调用 native 方法而封装的
    if (HEAD.compareAndSet(this, null, (h = new Node()))) {
        // 初始化的时候头节点和尾节点是一样的
        tail = h;
    }
}

// 然后是acquireQueued()
final boolean acquireQueued(final Node node, int arg) {
    // 不支持中断
    boolean interrupted = false;
    try {
        // 死循环自旋
        for (;;) {
            // 获取前面的前驱节点，如果前驱节点是 null，会直接抛 NPE
            final Node p = node.predecessor();
            // 只有前驱节点是头节点时才能尝试获取同步状态，有两个原因
            // 1. 头节点是成功获取到同步状态的节点，而头节点释放了同步状态之后，将会唤醒后续节点的线程，后续节点线程被唤醒之后需要检查自己的前驱节点是否头节点
            // 2. 维护队列的 FIFO 原则，不能让后面的节点先获取同步状态
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                // help GC
                p.next = null;
                return interrupted;
            }
            // 如果前驱节点不是头节点，判断线程是否需要进入等待状态
            if (shouldshouldParkAfterFailedAcquire(p, node)) {
                // 进入等待状态，等待状态可以设置中断
                interrupted |= parkAndCheckInterrupt();
            }
        }

    } catch (Throwable t) {
        // 如果出现异常，就取消这个node竞争同步状态
        cancelAcquire(node);
        if (interrupted){
            selfInterrupt();
        }
        throw t;
    }
}

// 从名字就可以看出来了，当获取同步状态失败的时候，判断线程需要进入等待状态
// 如果是则返回true，如果不是则返回false
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) {
        // 如果这个线程是需要等待锁的释放的通知的，那就可以放心的让他进入等待状态
        return true;
    }
    if (ws > 0) {
        // ws > 0 就是取消任务
        do {
            // 判断一下前驱节点的任务是不是被取消了，如果是，那这个节点的线程就不需要等待了
            // 直接把前驱节点的 next 改成当前节点
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 这时候的状态要么是 INITIAL 要么是 PROPAGATE
        // 不可能是 CONDITION 的原因是，这里的输入肯定是已经在同步队列中的节点了
        // 而同步队列中的节点不可能是 CONDITION 状态的
        // 如果头节点是 INITIAL 或者 PROPAGATE 状态的节点，就说明头节点是在等待唤醒(有可能是之前的节点唤醒失败了什么的)
        // 需要把这个头节点设置为 SIGNAL 
        // 这样才能够去获取同步状态并执行
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // park 就是使得当前线程等待
    LockSupport.park(this);
    return Thread.interrupted();
}


```

独占式的释放就简单很多了：
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // 释放成功了之后，先对头节点做判断，
        // 如果是 null 或者未初始化状态，就没必要操作后续节点了
        // 否则去唤醒后续节点
        Node h = head;
        if (h != null && h.waitStatus != 0) {
            // 唤醒后续节点
            unparkSuccessor(h);
        }
        return ture;
    }
    return false;
}
// 名字上就可以看出来是唤醒后续节点了
private void unparkSuccessor(Node node) {
    // 先判断当前节点的状态
    int ws = node.waitStatus;
    if (ws < 0) {

    }
    Node s = node.next;
    // 如果已经被取消了，或者已经没有后续节点了
    // 就开始寻找后续节点，直到找到后续节点为止
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = this.tail; p != node && p != null; p = p.prev) {
            if (p.waitStatus <=0 ) {
                s = p;
            }
        }
    }
    // 唤醒
    if (s != null) {
        LockSupport.unpark(s.thread);
    }
}

```

在获取独占式同步状态时，AQS维护了一个FIFO队里，获取失败的线程都会进入这个队列，然后自旋直至成功的获取同步状态；移出队列(或停止自旋)的条件是前驱节点为头节点并且成功的获取了同步状态。在释放同步状态时，AQS释放同步状态，并唤醒头节点的后续节点。

#### 共享式同步状态的获取与释放

共享式的同步状态获取指的是同一时刻可以有多个线程获取到同步状态。通过AQS的`acquireShared()`可以共享的获取同步状态：

```java
public final void acquireShared(int arg) {
    // 直接使用 tryAcquireShared 获取同步状态
    // 如果返回值小于0，则表示无法获取到同步状态，需要尝试自旋获取同步状态
    if (tryAcquireShared(arg) < 0) {
        // 自旋获取同步状态
        doAcquireShared(arg);
    }
}

private void doAcquireShared(int arg) {
    // addWaiter 前面已经看过了，就是新建 node 并且自旋式的 set 尾节点
    final Node node = addWaiter(Node.SHARED);
    // 不支持中断
    boolean interrupted = false;
    try {
        // 这里同样是在队列里自旋式的获取同步状态
        for(;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r > 0) {
                    setHeadAndPropagate(node, r);
                    // help GC
                    p.next = null;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node)) {
                interrupted |= parkAndCheckInterrupted();
            }
        }
    } catch (Throable t) {
        cancelAcquire(node);
        throw t;
    } finally {
        if (interrupted) {
            selfInterrupted();
        }
    }
    
}


```


## 重入锁


## 读写锁


## Condition接口


