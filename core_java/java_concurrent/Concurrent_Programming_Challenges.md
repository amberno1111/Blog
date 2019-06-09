# 并发编程的挑战


并发编程的目的是为了让程序运行的更快，能够更好的利用CPU进行计算。但是，并不是开多线程就肯定可以提升程序的性能的。当我们在进行并发编程的时候，往往会遇到很多的挑战，比如锁、上下文切换等等问题。如果解决不好这些问题，多线程反而可能会比单线程更慢。本文首先分析了为什么我们需要并发编程，然后介绍了几种常见的并发编程的挑战，并且略微提及一些解决方案。

## 为什么我们需要并发编程

最主要的原因是**摩尔定律的失效**，这导致计算机硬件的单机计算性能的提升越来越有限。现在主流的处理器提升性能的主要手段不再是提升CPU的主频，而是通过多核集成来提升计算能力。这就导致操作系统，甚至是我们的应用程序都需要**为了充分利用多核处理器的计算能力**而进行并发编程。


## 并发编程有哪些常见的挑战

并发编程比普通的串行编程要复杂的多，挑战也难的多，主要有以下这几种。

### 上下文切换

操作系统提供了**即使是单核处理器也能支持多线程执行代码**的能力，它通过给每个运行的线程分配**CPU时间片**来实现这个能力。因为只有一个CPU，并且在当前时间只有一个线程能占有CPU的计算资源，所以操作系统会不断的切换线程，把CPU的计算资源分散给所有正在运行的线程。这种机制让我们看起来线程是在并发运行的。**每个CPU时间片从几毫秒到几十毫秒不等。**

操作系统在做这个CPU时间片分配算法的时候，每次切换，都需要把当前正在占用CPU的线程的运行上下文信息都保留下来，然后从内存里读取下一个需要占用CPU的线程的上下文信息，然后把CPU控制权转交给这个线程。**从一个线程的上下文信息保存到下一个线程的上下文信息加载完成的这个过程，就是一次上下文切换**。

#### 上下文切换时到底在做什么事

首先我们要明白上下文的内容是什么？
- 上下文的内容其实就是**某一个时间点，CPU寄存器和程序计数器的内容**。
- 寄存器是CPU内部的数量较少但是速度很快的内存(与之对应的就是CPU外部相对较慢的RAM主内存)。寄存器通过对常用值(通常是运算的中间值)的快速访问来提高计算机程序运行的速度。
- 程序计数器是一个专用的寄存器，用于保存指令序列中CPU正在执行的位置，存的值为正在执行的指令的位置或者下一条将要被执行的指令的位置，具体依赖于特定的系统。

上下文切换时，操作系统做的事情是：
- 挂起一个进程/线程，将这个进程/线程在CPU种的上下文储存在内存中
- 恢复一个进程/线程，在内存中检索下一个进程的上下文并将其在CPU的寄存器中恢复
- 跳转到程序计数器所指向的位置(即跳转到进程/线程被中断的代码行)，以恢复该进程/线程


#### 上下文切换的损耗

上下文切换带来的损耗分为两种：
- 直接损耗：CPU寄存器需要保存和加载，系统调度器的代码需要执行，CPU的一些流水线指令执行能功能被打断
- 间接损耗：多核的cache可以共享数据，这个主要看线程工作区操作数据的大小

所以上下文切换造成的性能损耗就是系统不断的copy数据导致的。这也使得并发编程会遇到一个问题，就是当并发量到一定程度的时候，系统花在线程上下文切换的时间有可能会比线程真正工作的时间更长。这种情况下，多线程并发的程序性能可能还不如单线程。

#### 如何减少上下文切换

- 无锁并发：并发编程中处理数据的时候，往往需要加锁，但我们可以使用一些方法来避免加锁。比如将数据的ID按照Hash取模分段，不同的线程处理不同段的数据；或者使用函数式编程等等
- CAS算法：Java的Atomic包使用CAS算法来更新数据，不需要加锁
- 使用最少线程：避免创建不需要的线程
- 使用协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换


### 死锁

锁在并发编程里的使用非常多，但是很容易出现**死锁**，死锁会让程序一直卡住无法继续往下执行，只能靠重启来解决。出现死锁的原因很简单：
- 当前线程拥有其他线程需要的资源
- 当前线程等到其他线程已经拥有的资源
- 这些线程都不放弃自己拥有的资源

来个简单的例子：
```java
public class DeadlockDemo {
    private static String A = "A";
    private static String B = "B";

    public void deadlock() {
        final Thread threadA = new Thread(
            new Runnable() {
                @Override
                public void run() {
                    synchronized (A) {
                        System.out.println("A");
                        synchronized (B) {
                            System.out.println("B");
                        }
                    }
                }
            }
        );
        final Thread threadB = new Thread(
            new Runnable() {
                @Override
                public void run() {
                    synchronized (B) {
                        System.out.println("B");
                        synchronized (A) {
                            System.out.println("A");
                        }
                    }
                }
            }
        );
        threadA.start();
        threadB.start();
    }

}

```

因为线程在系统调度下是交错执行的，就很有可能出现这种情况：
- 线程A拿到锁A
- 线程B拿到锁B
- 然后两个线程要继续执行的话，都需要拿到另外一个锁，两个线程都不主动放弃锁的情况下，就无法继续往下执行了


#### 动态的锁顺序死锁

上面的例子很简单，我们写代码的时候当然不会犯这么低级的错误，那么我们可以来看一下这个例子：
```java
public class transferMoney(final Account fromAccount,
    final Account toAccount, final double amount) {
        synchronized (fromAccount) {
            synchronized (toAccount) {
                // do something 
            }
        }
}

```

来看一下这个例子，在多线程的环境下，是有可能发生死锁的：
- 线程1执行从A账户向B账户转钱的操作，并且当前获取了A账户的锁
- 线程2执行从B账户向A账户转钱的操作，并且当前获取了B账户的锁
- 线程1在等待B账户的锁释放，线程2在等待A账户的锁释放，于是就陷入了死锁


#### 协作对象之间发生死锁

还有一种情况就是协作对象之间发生死锁，来看一个例子：

```java
public class A {
    private B b;

    public synchronized void operate() {
        b.setSomeThing();
    }

    public 
}
public class B {
    private A a;

    public synchronized void setSomeThing() {
        a.operate();

    }
}

```

- 线程1调用了A的operate方法
- 线程2调用了B的setSomeThing方法
- 然后operate和setSomeThing方法里其实还嵌套了另一个锁，两个线程都在等待对方持有的锁，并且也不释放自己已经持有的锁，于是程序就卡住了


#### 如何避免死锁

- 固定加锁的顺序：针对锁顺序死锁
- 开放调用：针对对象协作造成的死锁
- 使用定时锁--> `tryLock()`：如果获取锁超时，则直接抛出异常

首先是一个固定加锁顺序解决锁顺序死锁的例子：

```java
public class transferMoney(final Account fromAccount,
    final Account toAccount, final double amount) {
        int fromHash = System.identityHashCode(fromAcct);
        int toHash = System.identityHashCode(toAcct);
        // 根据hash值来上锁
        if (fromHash < toHash) {
            synchronized (fromAcct) {
                synchronized (toAcct) {
                    new Helper().transfer();
                }
            }

        } else if (fromHash > toHash) {
            synchronized (toAcct) {
                synchronized (fromAcct) {
                    new Helper().transfer();
                }
            }
        } else {// 额外的锁、避免两个对象hash值相等的情况(即使很少)
            synchronized (tieLock) {
                synchronized (fromAcct) {
                    synchronized (toAcct) {
                        new Helper().transfer();
                    }
                }
            }
        }
```
这样的话，永远都需要先获取hash值小的那个账户的锁才能继续操作，就不会存在死锁问题了。

开放调用避免死锁也简单，就是不要在方法上加锁。因为在协作对象之间发生死锁的场景，都是因为**调用某个方法时需要持有锁，然后这个方法内部也调用了其他带锁的方法**。


### 资源限制

资源限制指的是在进行并发编程时，程序的执行受限于计算机硬件资源或者软件资源。比如下载时候带宽只有1M，单线程下载0.5M/s，这时候最多只能增加到2个线程就把带宽跑满了，你再增加多的线程，总体速度也只能时1M/s。

资源限制也会引发一些问题，比如上面的例子中，如果你把线程数开到100个，可能会导致很多时间浪费在线程切换上，导致连0.5M/s的总体速度都达不到。




