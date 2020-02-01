# ReentrantLock

ReentrantLock 是独占式的可重入锁，即：
1. 同一时刻只有一个线程可以获取锁
2. 当一个线程已经获取锁了之后，再来获取的时候可以直接获取到锁，而不需要重新 CAS 竞争

另外，ReentrantLock 也有公平锁以及非公平锁的概念。


```java

public class ReentrantLock implements Lock {

    /**
     * 按照惯例，JDK 里的锁肯定都是用 AQS 实现的
     * 1. 依然是 AQS 推荐的模式，定义一个 sync 的 filed，然后构造内部类继承 AQS
     * 2. 因为是独占锁，所以只需要重写 AQS 的 tryAcquire、tryRelease 方法即可
     * 3. 因为有公平非公平的概念，所以采用和 Semaphore 同样的方式，定义多个 AQS 的子类
     *    分别代表公平以及非公平锁两种模式，默认非公平
     *    同样，为了代码复用(公平 or 不公平两个 AQS 子类实现的 tryRelease 方法是可以共用的)
     *    所以多加了一个 abstract class: sync
     */
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {

        /**
         * 默认非公平，所以非公平的获取方法写在父类里
         * 这个方法需要有如下的几个功能：
         * 1. 获取到就直接返回，如果获取不到就阻塞线程并加入到队列(这部分只需要返回 false 即可，其他的步骤 AQS 会做的)
         * 2. 支持可重入，所以需要判断是否当前线程获取到了锁
         */
        final boolean nonfairTryAcquire(int acquires) {
            // 先判断同步状态是否可以获取，如果可以，直接获取同步状态
            // 否则就比较当前线程是否是已经获取同步状态的线程
            // 如果是，则返回 true；否则返回 false，AQS 会阻塞线程并加入到阻塞队列的
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(c, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                // 如果是重复获取，则增加计数
                // 但是增加计数的方法不需要 CAS，因为获取锁才能进行这个操作
                // 而当前线程已经获取到了这个锁
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        /**
         * 因为锁是可重入的，所以释放锁的方法需要对同步状态的值进行修改
         */
        protected final boolean tryRelease(final int arg) {
            if (Thread.currentThread() != getExclusiveOwnerThread()) {
                throw new IllegalArgumentException();
            }
            int c = getState() - arg;
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
        /**
         * 因为可重入锁是根据拥有锁的线程来进行判断的，而且是独占式的锁
         * 所以需要重写 isHeldExclusively 方法
         */
        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
        /**
         * 同时也提供了一些方法方便使用
         */
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }
    }


    static final class NonfairSync extends Sync {

        @Override
        protected boolean tryAcquire(final int arg) {
            // 因为默认是非公平锁，所以直接使用父类的方法即可
            return nonfairTryAcquire(arg);
        }
    }

    static final class FairSync extends Sync {

        @Override
        protected boolean tryAcquire(final int acquires) {
            // 需要重写 tryAcquire 方法，实现公平锁
            Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 1. 如果前面有线程在等待，就跳出了这个条件分支，最外层返回 false，AQS 就会
                //    阻塞当前线程，并加入到阻塞队列等待
                // 2. 如果前面没有线程在等待，就直接 CAS 尝试获取同步状态
                if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) {
                    throw new Error("Maximum lock count exceeded");
                }
                setState(nextc);
                return true;
            }
            
            return false;
        }
    }
```


```java
  /**
     * 构造函数就比较简单了，区分一下公平锁与非公平锁即可
     */
    public ReentrantLock() {
        this.sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        this.sync = fair ? new FairSync() : new NonfairSync();
    }
```

因为 ReentrantLock 实现了 Lock 接口，所以需要实现一些锁相关的方法，其实这部分相对来说简单，就是调用一下 sync 的方法而已。
```java
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
        // 这里感觉不太对，lock 直接调用了非公平的 acquire
        return sync.nonfairTryAcquire(1);
    }

    @Override
    public boolean tryLock(final long time, final TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

```