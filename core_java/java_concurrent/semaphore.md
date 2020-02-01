# Semaphore

JDK提供了Semaphore(信号量)的概念，它维护了一组许可。
- 每次调用`acquire`方法获取许可的时候，如果能直接获取到就返回；如果获取不到就阻塞，直到获取到许可为止
- 每次调用`release`方法就会释放一个许可，会唤醒阻塞队列中的线程

Semaphore基于AQS实现，很明显，许可Permit刚好对应AQS里的state整型变量的值。

另外，Semaphore还提供公平获取同步状态以及非公平获取同步状态的概念。
- 公平获取同步状态：是指已经在有线程阻塞等待Permit的情况下，当新的线程来获取Permit的时候，直接加入到队列
- 非公平的获取同步状态：是指已经在有线程阻塞等待Permit的情况下，当新的线程来获取Permit的时候，先尝试直接获取Permit，如果成功则直接返回，如果失败则加入到队列


接下来看一下源码分析：



```java
public class Semaphore {
        /**
     * Semephore 是基于 AQS 的，并且有公平与非公平的概念，所以
     * 1. 基于 AQS 的使用方式，定义一个 sync 的 field，然后构造内部类继承 AQS
     * 2. 因为 Permit 的数量可能由多个，也就是允许多个线程同时获取 Permit，所以是共享式的，应该重写 AQS 的
     *    tryAcquireShared、 tryReleaseShared 方法
     * 3. 公平与非公平的概念，区别在于 acquire 的时候，
     *    是直接尝试获取 Permit 还是先查看是否有线程再等待然后再决定是否直接获取 Permit
     *    从实现上来说，这两者的区别就是实现 AQS 的 tryAcquireShared 方法时的逻辑不一样
     *    所以其实在 Semaphore 里，需要实现两个基于 AQS 的子类，分别表示公平以及不公平获取 Permit 这两种方式
     *    这里为了代码复用(公平 or 不公平两个 AQS 子类实现的 tryReleaseShared 方法是可以共用的)
     *    所以多加了一个 abstract class: sync
     */
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {

        /**
         * Permit 的数量对应 AQS 里整型变量的值，所以构造函数就很好写了
         */
        Sync(int permits) {
            setState(permits);
        }

        /**
         * 因为默认是非公平的，所以非公平的获取 Permit 的方法写在这里
         * 然后因为使用了 AQS， 需要重写的 tryAcquireShared 方法写在具体的子类里
         * 可以查看 FairSync 和 NonFairSync 这两个类
         */
        final int nonFairTryAcquireShared(int permits) {
            // 加入等待队列、阻塞线程的操作在 AQS 里会做的，只要返回的值小于 0 即可
            // AQS 规定了这个方法的实现必须是非阻塞的
            // 这个方法有两种情况需要考虑：
            // 1. Permit 足够的情况，因为是非公平的，所以尝试使用自旋 CAS 获取同步状态
            //    直到获取成功或者 Permit 小于 0 为止
            // 2. Permit 不够，就直接返回即可
            for (; ; ) {
                int available = getState();
                int remaining = available - permits;
                if (remaining < 0 || compareAndSetState(available, remaining)) {
                    return remaining;
                }
            }
        }

        /**
         * 如上所说，为了代码复用，公平与非公平的的 tryReleaseShared 方法实现是一样的
         * 所以放在父类里实现
         * release 方法就比较简单了，自旋式的释放就好了
         */
        @Override
        protected boolean tryReleaseShared(final int arg) {
            for (; ; ) {
                int current = getState();
                int next = current + arg;
                if (next < current) {
                    // 处理溢出的情况
                    throw new Error("Maximum permit count exceeded");
                }
                if (compareAndSetState(current, next)) {
                    return true;
                }
            }
        }
        /**
         * Semaphore 还提供了两个操作 Permit 的方法，以方便使用
         * 一个是直接清空 Permit，一个是减少 Permit
         */
        final void reducePermits(int reductions) {

            for (; ; ) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next)) {
                    return;
                }
            }
        }
        
        final int drainPermits() {
            for (; ; ) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    static class FairSync extends Sync {
        FairSync(final int permits) {
            super(permits);
        }

        @Override
        protected int tryAcquireShared(final int arg) {
            for (; ; ) {
                // 公平的获取锁最简单的实现，就是不管怎么样，都直接加入到队列中
                // 但是这里还是判断了一下是否存在前驱节点，当存在前驱节点时才加入到队列中
                // 不存在时就尝试直接获取同步状态
                if (hasQueuedPredecessors()) {
                    // 返回负值，AQS 会把阻塞当前线程并加入到队列的
                    return -1;
                }
                // 当队列里没有等待的线程，就可以尝试直接获取 Permit 了
                int available = getState();
                int remaining = available - arg;
                if (remaining < 0 || compareAndSetState(available, remaining)) {
                    return remaining;
                }
            }
        }
    }

    static class NonFairSync extends Sync {
        NonFairSync(final int permits) {
            super(permits);
        }

        @Override
        protected int tryAcquireShared(final int arg) {
            // 那么这个就很简单了，直接调用父类的实现即可
            return nonFairTryAcquireShared(arg);
        }
    }
}
```
```java
public class Semaphore {
     /**
     * 对外的构造函数就比较好写了，Semaphore 提供两个概念，
     * 一是许可Permit
     * 二是公平与非公平的获取Permit，默认非公平的获取Permit
     * 所以构造函数如下：
     */
    public Semaphore(int permits) {
        // 默认非公平
        this.sync = new NonFairSync(permits);
    }

    public Semaphore(int permits, boolean fair) {
        this.sync = fair ? new FairSync(permits) : new NonFairSync(permits);
    }

}
```

Semaphore还有一些对外提供的工具方法，就不分析了，类似于`tryAcquire`之类的其实都是调用`sync`来完成的。