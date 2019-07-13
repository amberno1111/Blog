# Java中的线程

线程是现代操作系统调度的最小单位，是轻量化的进程，它和进程的区别是：
- 线程之间能够通过访问共享的内存变量来交换数据，进程之间只能进行通信来交换数据
- 线程有独立的程序计数器、堆栈和局部变量，进程有独立的内存地址空间

为什么要使用多线程：
- 充分利用多核的计算能力
- 并行执行任务可以提高响应时间

## 一些零碎的基础概念

### 线程优先级

现代操作系统通过给线程分配CPU时间片来进行线程调度，当一个线程用完了操作系统给它分配的时间，就会发生线程调度，这个线程只能等待下次分配才能继续运行。操作系统给每个线程分配的时间片可以是不相等的，可以通过**线程的优先级**来控制。线程的优先级越高，所分配到的时间片就会越多，表示操作系统希望这个线程能占用更多的CPU执行时间。

在Java里，使用`Thread`里的`priority`表示线程的优先级，默认优先级是5，优先级范围是1~10。**但是，即使你在Java里设置了线程优先级，操作系统也可能完全不care这个优先级，它可能会忽略对线程优先级的设定，比如Ubuntu就是这么做的，所以程序的正确性不要依赖线程优先级来实现。**

### 线程的状态

Java线程总共有6个状态，可以在`Thread`类里的一个枚举中看到：
- NEW：表示新建线程，`new`了一个`Thread`对象，但还没有执行`Thread.start()`的状态
- RUNNABLE：线程执行了`start()`方法之后的状态，但是这并不意味着已经开始运行了，只是线程处于就绪状态，要等到系统调度这个线程真正获取到CPU控制权才会开始运行
- WAITING：等待状态，需要其他线程来做一些操作(通知或中断)，一般是调用了`wait()`、`join()`、`LockSupport.parn()`等方法导致的；对应的等到`notify()`、`notifyAll()`、`LockSupport.unparn()`等方法调用的时候可以从WAITING状态唤醒进入RUNNABLE状态
- BLOCKED：阻塞状态，等待获取锁
- TIMED_WAITING：超时等待，过了时间可以自己返回，一般是调用了`Thread.sleep(long)`、`Object.wait(long)`、`Thread.join(long)`、`LockSupport.parkNanos()`、`LockSupport.parkUntil()`之类的方法；对应的等到`notify()`、`notifyAll()`、`LockSupport.unparn(Thread)`等方法的时候可以被唤醒进入`RUNNABLE`状态
- TERMINATED：线程终止，表示当前线程已经执行完毕

```java
public class Thread {
     public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called {@code Object.wait()}
         * on an object is waiting for another thread to call
         * {@code Object.notify()} or {@code Object.notifyAll()} on
         * that object. A thread that has called {@code Thread.join()}
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
}
```

### Daemon线程

`Deamon`线程是一种支持型线程，它主要被用于程序中后台调度以及支持性工作。这意味着，当一个Java虚拟机中不存在非`Deamon`线程的时候，Java虚拟机将会退出。

另外`Deamon`线程被用作完成支持性工作，但是Java虚拟机退出的时候，并不一定会执行`Deamon`线程当中的`finally`块。所以，在构建`Deamon`线程时，不能依靠`finally`块来确保执行关闭或者清理资源。

## 线程的启动和终止

### 构造线程对象

运行线程之前，需要构造一个线程对象。一个新的线程对象是由其parent线程来进型空间分配的，而child线程继承了parent线程是否为`Deamon`、优先级和加载资源的`contextClassLoader`以及可继承的`ThreadLocal`，同时还可以分配一个唯一的ID来标识这个child线程。具体可以如下代码，来自于JDK12的`Thread.class`：
```java
public class Thread {
    // 虽然这里是一个 private 方法，但实际上所有 new Thread()的线程都会跑到这个private方法里
    private Thread(ThreadGroup g, Runnable target, String name,
                   long stackSize, AccessControlContext acc,
                   boolean inheritThreadLocals) {
        if (name == null ) {
            throw new NullPointerException("name cannot be null");
        }
        this.name = name;
        // parent 就是当前正在创建 child 的线程
        Thread parent = currentThread();
        // .... 这中间省略一下安全性相关的代码
        this.group = g;
        // deamon、contextClassLoader、可继承的 ThreadLocal、优先级都继承于 parent 线程
        this.deamon = parent.isDeamon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        this.tid = nextThreadID();
    }
}
```


### 启动线程

线程的启动只需要`start()`就可以了，但并不是线程调用了`start()`方法之后就马上执行的，parent线程调用这个方法只是告诉Java虚拟机等到线程规划器空闲，就应该启动调用`start()`方法的线程。


### 终止线程

那么应该如何终止线程呢？
- JDK提供了`stop()`方法，但这是个过期的方法，它和`suspend()`、`resume()`往往是一起使用的，但是这三个方法都存在缺陷，可能无法正常的释放锁导致死锁，或者在终止线程的时候不能正常的释放资源。这些问题都会导致程序运行出现问题。
- 通过中断或者标识位检测的方法来安全的终止线程


#### 线程的中断

线程的中断可以理解为线程的一个标识位属性，它表示一个线程在运行中是否被其他线程进行了中断操作。被中断的线程通过检测自身是否被中断来进型响应，其他线程则通过调用该线程的`interrupt()`方法来对线程进行中断。

线程可以通过调用`isInterrupted()`方法来判断是否被中断，可以通过`Thread.interrupted()`方法对中断标识位进行复位。但是如果当前线程已经处于TERMINATED状态，那么及时这个线程曾经被中断过，`isInterrupted()`依然会返回false。

**线程的中断虽然能用于终止线程，但是并不能100%保证停止线程也没法立刻停止线程，因为线程的中断需要被中断线程本身通过检测中断标识位并进行响应才能发生，而当一个线程在执行CPU密集型的计算的时候，往往是不会进行中断标识位的检测的，也就不会进行中断操作了**。

通过标志位终止线程的方法和中断其实差不多，只不过是程序员自己给线程设置了一个类似于中断的标识位而已。

## 线程间通信

Java的线程间通信有很多的方法，比如前面提到过的`volatile`与`synchronized`关键字，比如JDK提供的等待/通知机制等等。

### `volatile`和`synchronized`

Java支持多个线程同时访问一个对象或者对象的成员变量，并且允许每个线程可以拥有这个变量的拷贝，这样做的目的是提高程序的运行速度(这种优化方式类似于处理器的高速缓存)。所以每个线程不一定能看到这个变量的最新状态，而Java内存模型提供了`volatil`和`synchronized`关键字来避免出现这种情况，只要正确的使用这两个关键字进行同步，就可以确保每个线程都能看到这个变量的最新状态。

`volatile`可以用来修饰成员变量，任意的线程来读这个成员变量的数据都需要从共享内存中读取；任意的线程写这个成员变量都需要把最新的值刷新回共享内存。但是`volatile`关键字也不能滥用，不然会影响程序的运行效率。

`synchronized`关键字可以用来修饰方法或者代码块，它能够确保多线程环境下，有且只有一个线程能够处于被修饰的方法或者代码块中，它保证了线程对变量读写的可见性与排他性。

每个对象都有监视器，当这个对象被`synchronized`关键字修饰时，线程只有获取这个对象的监视器才能进入这个对象的同步方法或者同步代码块，没有获取到这个对象的监视器的线程将会进入BLOCKED状态。同步代码块是通过监视器的`monitorenter`和`monitorexit`指令来实现的，而同步方法则是通过方法修饰符上的`ACC_SYNCHRONIZED`变量的值来实现的。

### 等待/通知机制

现实中我们经常遇到这样的场景：线程A修改了一个flag标记，需要线程B感知到这个flag的值得变化，然后执行一系列操作。这种模式就是经典的**生产者-消费者模式**。线程A是生产者，线程B是消费者。

我们手动实现这个模式的时候，一般是这么做的：
- 启动两个线程，线程A和线程B，假设线程A是生产者，线程B是消费者
- 定义一个`volatile`对象`flag`，初始值为`false`，线程A可能在某个时间节点把`flag`的值设定为`true`
- 线程B里有一个循环，不断的对`flag`的值进行判断，直到`flag==true`的时候再执行对应的逻辑，代码类似于这样：
```java
while (!flag) {
    // 这里加 sleep 的目的是，防止线程 B 一直消耗处理器资源
    Thread.sleep(100);
}
// do something
doSomething();
```

但这种方法有一些缺陷：
- 加了`sleep`会导致任务处理的不及时
- 不加`sleep`又会导致处理器计算资源被浪费

**Java内置的等待/通知机制可以比较优雅的满足这种场景并且避免缺陷**。等待/通知的相关方法是所有的对象都具备的，因为被定义在了`Object`类当中，主要有以下这些方法：
- `wait()`：调用该方法的线程进入WAITING状态，只有等待另外线程的通知或者被中断才会返回。另外，调用`wait()`方法之后会释放锁
- `wait(long)`：作用和`wait()`方法基本一样，不过加了一个超时时间，只要超过这个时间就返回
- `wait(long, int)`：对于超时时间的控制更加精细，可以达到纳秒级别
- `notify()`：通知在一个对象上等待的线程，使其从`wait()`方法上返回，而返回的前提是该线程获取到了对象的锁
- `notifyAll()`：通知所有等待在这个对象上的线程

**等待/通知机制：线程A调用了对象O的`wait()`方法，进入了WAITING状态，并且释放了锁；然后线程B调用了对象O的`notify()`或者`notifyAll()`方法，线程A收到通知后从对象O的`wait()`方法返回，然后进行后续的操作。上述两个线程基本上是通过对象O来进行交互的**。

来看一个实现：
```java
public class WaitNofityDemo {
    private static boolean flag = true;
    private static Object lock = new Object();

    public statice void main(String[] args) throws Exception {
        final Thread wait = new Thread(new Wait(), "waitThread");
        wait.start();
        Thread.sleep(1);
        final Thread notify = new Thread(new Nofity(), "notifyThread");
        notify.start();
    }

    static class Wait implements Runnable {
        public void run() {
            // 首先要获取锁
            synchronized(lock) {
                while (flag) {
                    System.out.println(Thread.currentThread() + "flag is true");
                    // 进入等待，同时会释放 lock 对象的锁
                    lock.wait();
                }
                // 当 flag 被 Nofity 线程改变之后，wait 线程被唤醒，然后就执行到这里了
                System.out.println(Thread.currentThread() + "flag is false");
            }
        }
    }

    static class Notify implements Runnable {
        public void run() {
            // 获取锁，从 notify 线程可以获取 lock 的锁来看，wait 线程在调用了 lock 的 wait 方法之后肯定释放了锁
            synchronized(lock) {
                System.out.println(Thread.currentThread() + "flag is true. start logic");
                // 唤醒等待在 lock 对象上的所有线程，当然也包括 wait 线程
                // notifyAll 的时候不会释放 lock 的锁
                lock.notifyAll();
                flag = false;
                System.out.println(Thread.currentThread() +"flag is false");
            }
            System.out.println(Thread.currentThread() + "end");
        }
    }

}
```

使用等待/通知机制有一些细节需要了解：
- 使用之前要对调用对象加锁
- 调用对象的`wait()`方法会释放锁，但是调用`notify()`或者`notifyAll()`方法不会释放锁
- `notify()`方法是将等待队列中的一个线程从等待队列中移动到同步队列；`notifyAll()`方法是讲等待队列中的所有线程移动到同步队列，被移动的线程状态从WAITING状态变成BLOCKED
- 从`wait()`返回的前提是获取了对象的锁

**等待/通知机制是依托于同步机制的，其目的是保证等待线程从`wait()`方法返回时能够感知到通知线程对变量做出的修改**。

### 管道输入/输出流

管道输入/输出流是专门用于线程之间传输数据的流，传输的媒介是内存。在JDK中有4中具体实现：`PipedOutputStream`、`PipedInputStream`、`PipedReader`、`PipedWriter`。前面两种面向字节，后面两种面向字符。

来看一个例子，通过main线程写入，另一个线程输出：
```java
public class PipedDemo {

    public static void main(String[] args) throws IOException {
        final PipedWriter writer = new PipedWriter();
        final PipedReader reader = new PipedReader();
        // 必须把输入和输出连接，不然会抛异常
        writer.connect(reader);
        final Thread thread = new Thread(new Reader(reader), "Reader");
        thread.start();
        // 接收一个来自于控制台输入的字符
        int receive = System.in.read();
        writer.write(receive);
    }

    static class Reader implements Runnable {
        private PipedReader reader;

        public Reader(final PipedReader reader) {
            this.reader = reader;
        }

        @Override
        public void run() {
            try {
                int receive = reader.read();
                System.out.println((char) receive + " ---- " + Thread.currentThread());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
}
```
运行这段代码，当你从控制台输入一个字符时，main线程负责把这个字符写入流，然后新建的名为Reader的线程会把这个字符输出。

### `Thread.join()`

如果线程A调用了线程B的`join()`方法，其程序语义是：线程A必须等待线程B的返回，然后才能继续执行。当然也有`join(long)`和`join(long, int)`这种带有超时特性的方法。

`Thread.join()`实际上就是通过等待/通知机制实现的，来看一下JDK12中的源代码：
```java
public class Thread {
    // 对当前线程对象加锁
    public final synchronized void join(final long millis) {
        if (millis > 0) {
            if (isAlive()) {
                final long startTime = System.nanoTime();
                long delay = millis;
                do {
                    // 如果条件不满足则 wait，否则就直接返回了
                    wait(millis);
                } while (isAlive() && (delay = millis -
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
            }
        } else {
            // .....
        }
    }
}
```
