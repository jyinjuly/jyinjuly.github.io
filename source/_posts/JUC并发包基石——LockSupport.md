---
title: JUC并发包基石——LockSupport
tags: ["juc并发包", "多线程"]
categories: ["多线程"]
---
LockSupport是java.util.concurrent包中一个工具类，它为构建锁和同步器提供了基本的线程阻塞/唤醒原语。JDK中我们熟悉的AQS同步器就使用了它来控制线程的阻塞和唤醒，当然还有其他的锁或同步器也会使用它。因此，LockSupport可以说是juc并发包的基石。
<!-- more -->
# LockSupport核心方法
我们知道，LockSupport最核心的功能就是阻塞/唤醒线程，对应方法如下：
* park()：阻塞当前线程，直到获取到可用许可。
* parkNanos(long)：阻塞当前线程，直到获取到可用许可。参数为最大等待时间，一旦超过最大等待时间也会解除阻塞。
* parkUntil(long)：阻塞当前线程，直到获取到可用许可。参数为最后期限时间，一旦超过最后期限时间也会解除阻塞。
* park(Object)：与park()方法同义，但指定了阻塞对象。
* parkNanos(Object,long)：与parkNanos(long)同义，但指定了阻塞对象。
* parkUntil(Object,long)：与parkUntil(long)同义，但指定了阻塞对象。
* unpark(Thread)：将指定线程的许可置为可用，也就相当于唤醒了该线程。

![LockSupport.png](LockSupport.png)

# LockSupport许可机制
在介绍核心方法时出现了一个高频的词——许可，那么什么是许可呢？其实很容易理解，线程成功拿到许可则能够往下执行，否则将进入阻塞状态。对于LockSupport使用的许可可看成是一种二元信号，该信号分有许可和无许可两种状态。每个线程都对应一个信号变量，当线程调用park时其实就是去获取许可，如果能成功获取到许可则能够往下执行，否则则阻塞直到成功获取许可为止。而当线程调用unpark时则是释放许可，供线程去获取。如下图所示，首先在线程t1和线程t2中调用park()，二者都进入阻塞状态。然后调用unpark(t1)，为线程t1颁发许可。那么线程t1将解除阻塞，恢复执行，同时将许可置为不可用（或者说消费许可）。
![park.png](park.png)
> park()：尝试消费许可，如果消费失败，则阻塞
> unpark(thread)：为指定线程颁发许可

## park与unpark无顺序
按照我们通常的理解，线程应该是先阻塞(park)，然后再去唤醒(unpark)，如下：
```java
public class NewThread {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            LockSupport.park();
            System.out.println("park before unpark");
        });
        thread.start();
        Thread.sleep(2000L);
        LockSupport.unpark(thread);
    }

}
```
> park before unpark

但是LockSupport同样支持先颁发许可(unpark)，然后再去消费(park)，如下：
```java
public class NewThread {

    public static void main(String[] args) {
        Thread thread = Thread.currentThread();
        LockSupport.unpark(thread);
        LockSupport.park();
        System.out.println("unpark before park");
    }

}
```
> unpark before park

## 多次unpark相当于一次
我们不能为同一个线程颁发多个许可，多次颁发的结果也只是让线程拥有一个许可。
```java
public class NewThread {

    public static void main(String[] args) {
        Thread thread = Thread.currentThread();
        LockSupport.unpark(thread); // 颁发许可
        LockSupport.park(); // 消费许可

        LockSupport.park(); // 此时无可用许可，则阻塞
        System.out.println("no avail permit");
    }

}
```

# 与wait和notify/notifyAll异同
在没有LockSupport之前，线程的阻塞和唤醒都是通过wait和notify/notifyAll实现的，那么两种实现有什么区别呢？如下：
1. wait和notify/notifyAll只能在同步方法/同步块中使用，而LockSupport可以在任何位置使用。
> LockSupport线程间不再需要维护同一个锁对象，实现了线程间的解耦
2. wait和notify/notifyAll必须得先wait阻塞，然后才能notify唤醒，而LockSupport则是无顺序的。
> unpark函数可以先于park调用，所以不需要担心线程间的执行的先后顺序