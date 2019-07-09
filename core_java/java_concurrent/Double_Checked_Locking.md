# 双重检查锁定与延迟初始化

在日常的开发中，我们会遇到一些需要延迟初始化的场景。然后很多时候可能会选择双重检查锁定的方法，这个方法非常的常见，但实际上这是一个存在线程安全问题的用法。

## 双重锁定的由来

先来看一个例子：
```java
public class DoubleCheckedLockingDemo {
    private static Instance instance;

    public static Instance getInstance() {
        if (instance == null) {
            instance = new Instance();
        }
        return instance;
    }
```

这很明显是一个没有正确同步的代码块，当两个线程都执行`getInstances`方法的时候就会出现问题。比如一个线程执行`instance = new Instance();`的时候，另一个线程正在执行`if (instance == null)`这个判断。

如果要正确同步，可以加锁：
```java
public class DoubleCheckedLockingDemo {
    private static Instance instance;

    public synchronized static Instance getInstance() {
        if (instance == null) {
            instance = new Instance();
        }
        return instance;
    }
}
```

但是，在早期的JVM里，也就是偏向锁、轻量级锁出现之前，`synchronized`是完完全全的重量级锁，非常的耗费性能，即使是无竞争的状态，也是一样的耗费性能。所以人们就想出了**双重检查锁定**的方法来替代直接加锁，代码如下：
```java
public class DoubleCheckedLockingDemo {
    private static Instance instance;

    public static Instance getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLockingDemo.class) {
                if (instance == null) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

这么做的好处是，看起来保证了线程安全，只有获取锁的那一个线程可以对`instance`赋值，如果当前存在竞争，后续拿到锁的线程再判断一次`instance == null`就可以避免对`instance`的再次初始化操作。

**但是事情没有这么简单，因为存在指令重排序！！！**

实际上`instance = new Instance();`这个操作可以划分为三条指令：
```c
// 操作1：分配内存
memory = allocate();
// 操作2：初始化对象
ctorInstance(memory);
// 操作3：设置 instance 指针指向刚分配好的内存地址
instance = memory;
```
在一些JIT编译器上，会对操作3和操作2进行重排序，因为这个重排序并不会改变程序的语义和运行结果。但是，在多线程环境下，存在这样的场景：上述重排序后，线程A获取到锁，然后执行了操作3，还没完成操作2(假设操作2比较耗时)；线程B开始了第一个`if(instance == null)`的判断，这个时候得到的结果是`false`因为`instance`已经比指向了分配好的内存地址，所以线程B直接取到了没有初始化好的`instance`，这时候就会出现问题。


## 解决方案

从上面的分析来看，我们应该有两种方法来解决这个问题：
- 不允许操作2和操作3的重排序
- 允许操作2和操作3的重排序，但是不允许线程B看到线程A的这个重排序操作

这两个解决方案分别对应`volatile`和锁。

### 基于`volatile`的解决方案

```java
public class DoubleCheckedLockingDemo {
    private static volatile Instance instance;

    public static Instance getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLockingDemo.class) {
                if (instance == null) {
                    // volatile 变量的写一定 happens-before 后续所有的读
                    // 所以 JMM 会禁止这种情况下的重排序
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

### 基于锁的解决方案

JVM在类的初始化的期间，会获取一个锁，这个锁可以同步多个线程对同一个类的初始化。我们可以利用这个特性来实现线程安全的延迟初始化方案。

```java
public class DoubleCheckedLockingDemo {
    private static class InstanceHolder {
        // JVM 可以保证这里的初始化操作是线程安全的 
        public static Instance instance = new Instance();
    }

    public static Instance getInstance() {
        // 这里可以实现 instance 的初始化
        return InstanceHolder.instance;
    }
}
```


























