---
title: Java线程状态
tags: ["Java线程", "多线程"]
categories: ["多线程"]
---

# Java线程状态
java线程有六种状态，在Thread类中有所体现，如下：
```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```
<!-- more -->
## NEW
```java
/**
 * Thread state for a thread which has not yet started.
 */
NEW
```
新创建并且还未启动的线程，状态为NEW。
```java
public class NewThread {

    public static void main(String[] args) {
        Thread thread = new Thread();
        System.out.println("线程状态：" + thread.getState());
    }

}
```
> 线程状态：NEW
## RUNNABLE
```java
/**
 * Thread state for a runnable thread.  A thread in the runnable
 * state is executing in the Java virtual machine but it may
 * be waiting for other resources from the operating system
 * such as processor.
 */
RUNNABLE
```
线程调用`start()`方法，进入可运行状态——RUNNABLE。RUNNABLE状态的线程在java虚拟机中已经开始执行，但是它可能还在等待来自操作系统的资源，比如处理器。为了便于理解，我们“擅自”将RUNNABLE状态拆分成两个阶段：
1. READY——就绪状态，已经在java虚拟机中执行，但是并未实际执行任务，相当于空转
```java
public class ReadyThread {

    public static void main(String[] args) {
        Thread thread = new Thread(() -> System.out.println("线程开始执行"));
        thread.start();
        System.out.println("线程状态：" + thread.getState());
    }

}
```
> 线程状态：RUNNABLE
> 线程开始执行
2. RUNNING——执行中状态，获取了CPU时间片，真正在执行任务了
```java
public class RunningThread {

    public static void main(String[] args) {
        new Thread(() -> System.out.println("线程状态：" + Thread.currentThread().getState())).start();
    }
}
```
> 线程状态：RUNNABLE

## BLOCKED
```java
/**
 * Thread state for a thread blocked waiting for a monitor lock.
 * A thread in the blocked state is waiting for a monitor lock
 * to enter a synchronized block/method or
 * reenter a synchronized block/method after calling
 * {@link Object#wait() Object.wait}.
 */
```
阻塞线程的状态为BLOCKED。此状态下的线程正在等待一个监视器锁，目的是为了：
1. 能进入一个同步块或者同步方法
```java
public class BlockedThread {

    private final Object obj = new Object(); // 监视器锁

    public void test() {
        synchronized (obj) {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        BlockedThread blockedThread = new BlockedThread();
        Thread t1 = new Thread(blockedThread::test);
        Thread t2 = new Thread(blockedThread::test);
        t1.start();
        Thread.sleep(1000); // 保证线程t1先拿到监视器锁，进入同步块
        t2.start();
        Thread.sleep(1000);
        System.out.println("线程t2状态：" + t2.getState()); // 线程t2正在等待获取监视器锁，以进入同步块
    }

}
```
> 线程t2状态：BLOCKED
2. 能（在监视器锁调用`Object#wait()`方法之后）重新进入一个同步块或者同步方法
```java
public class Waiting2BlockedThread {

    private final Object obj = new Object(); // 监视器锁

    private boolean flag = true;

    public void test() {
        synchronized (obj) {
            if (flag) {
                flag = false;
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                obj.notify();
                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Waiting2BlockedThread waiting2BlockedThread = new Waiting2BlockedThread();
        Thread t1 = new Thread(waiting2BlockedThread::test);
        Thread t2 = new Thread(waiting2BlockedThread::test);
        t1.start();
        Thread.sleep(1000); // 保证线程t1先拿到监视器锁，进入同步块
        System.out.println("线程t1状态：" + t1.getState()); // 监视器锁调用Object#wait()方法，当前锁持有线程t1释放锁并置为等待状态
        t2.start();
        Thread.sleep(2000); // 保证线程t2进入同步块，监视器锁调用Object#notify()方法，将线程t1唤醒
        System.out.println("线程t1状态：" + t1.getState()); // 唤醒之后的线程t1首先进入RUNNABLE状态去尝试获取监视器锁，发现锁被线程t2持有，于是又变为阻塞状态
    }

}
```
> 线程t1状态：WAITING
> 线程t1状态：BLOCKED

## WAITING
```java
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
 * For example, a thread that has called <tt>Object.wait()</tt>
 * on an object is waiting for another thread to call
 * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
 * that object. A thread that has called <tt>Thread.join()</tt>
 * is waiting for a specified thread to terminate.
 */
```
等待线程的状态为WAITING。等待线程等待另一个线程去执行某一个特定的操作，以此将自己唤醒。当以下方法被执行时，线程将进入等待状态：
1. Object#wait()：在线程t1中调用了监视器锁对象的wait()方法，线程t1将进入等待状态。等待另一个线程t2去调用相同的监视器锁对象的notify()/nofityAll()方法。
```java
public class Wait2Waiting {

    private final Object obj = new Object(); // 监视器锁

    private boolean flag = true;

    public void test() {
        synchronized (obj) {
            if (flag) {
                flag = false;
                try {
                    obj.wait(); // 线程进入等待状态
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                obj.notify(); // 等待线程被唤醒
                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Wait2Waiting wait2Waiting = new Wait2Waiting();
        Thread t1 = new Thread(wait2Waiting::test);
        Thread t2 = new Thread(wait2Waiting::test);
        t1.start();
        Thread.sleep(1000);
        System.out.println("线程t1状态：" + t1.getState());
        t2.start();
        Thread.sleep(2000);
        System.out.println("线程t1状态：" + t1.getState());
    }

}
```
> 线程t1状态：WAITING
> 线程t1状态：BLOCKED
2. Thread#join()：在线程t2中调用了线程t1的join()方法，那么线程t2将进入等待状态，直到线程t1死亡。
```java
public class Join2Waiting {

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        Thread t2 = new Thread(() -> {
            try {
                t1.join(); // 线程t2进入等待状态，直到线程t1死亡
                while (true) { }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        t1.start();
        Thread.sleep(1000);
        t2.start();
        Thread.sleep(1000);
        System.out.println("线程t2状态：" + t2.getState());
        Thread.sleep(4000); // 保证线程t1已经死亡，线程t2被唤醒
        System.out.println("线程t2状态：" + t2.getState());
    }

}
```
> 线程t2状态：WAITING
> 线程t2状态：RUNNABLE
3. LockSupport#park()：如果当前线程未被颁发过“许可证”，那么将会进入等待状态。
```java
public class park2Waiting {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(LockSupport::park);
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
    }

}
```

## TIMED_WAITING
```java
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
```
限时等待线程的状态为TIMED_WAITING。限时等待状态的线程当超过等待时间时会自动唤醒，也可以像等待线程一样，通过等待另一个线程执行某一个操作将自己唤醒。当以下方法被执行时，线程将进入限时等待状态：
1. Thread.sleep：当前线程睡眠指定时间。
```java
public class Sleep2TimedWaiting {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(5000);
                while (true) {}
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        Thread.sleep(5000); // 保证已经超过等待时间
        System.out.println("线程状态：" + thread.getState());
    }

}
```
> 线程状态：TIMED_WAITING
> 线程状态：RUNNABLE
2. Object#wait(long)：与Object#wait()类似，等待其他线程调用nofity()/notifyAll()方法，或者等待超时自动唤醒。
3. Thread#join(long)：与Thread#join()类似，等待其他线程死亡，或者等待超时自动唤醒。
4. LockSupport.parkNanos(long)：与LockSupport.park()类似，传参为纳秒，代表最多等待的时长。
5. LockSupport.parkUntil(long)：与LockSupport.park()类似，传参为时间戳，代表最多等待到某时刻为止。

## TERMINATED
```java
/**
 * Thread state for a terminated thread.
 * The thread has completed execution.
 */
```
当线程执行完任务之后，就会变为终止状态——TERMINATED。
```java
public class TerminatedThread {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> System.out.println("线程执行完毕！"));
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
    }

}
```
> 线程执行完毕！
> 线程状态：TERMINATED

# Java线程状态切换图
![线程状态切换图.png](线程状态切换图.png)
# 阻塞和等待的区别
我们在讨论“阻塞”和“等待”的区别之前，有必要先说明一下：这里的“阻塞”是狭义上的阻塞，特指Java中的`java.lang.Thread.State#BLOCKED`，即由于争抢锁失败而的进入的线程状态。**那什么是广义上的“阻塞”呢？一般来讲，线程阻塞泛指由于某些原因而导致线程暂停执行的现象，此时的线程不会被分配CPU时间片。如此说来，广义上的“阻塞”实际上就包括了Java线程的“阻塞”状态和“等待”状态了。并且，通常大家在讨论并发时所说的线程阻塞，也指的是广义上的“阻塞”。**我们这里讨论的“阻塞”和“等待”的区别，指的是Java中的`java.lang.Thread.State#BLOCKED`和`java.lang.Thread.State#WAITING`，两者区别如下：
* 线程进入阻塞状态是被动的，而线程进入等待状态是主动的
> 主动等待：当前线程中一定是主动调用了某个方法才会进入等待状态
> 被动阻塞：线程争抢锁失败之后被动进入阻塞状态

两者相同点如下：
* 都会暂停线程的执行，线程本身不会占用CPU时间片

# 线程状态相关API
## Object类
* Object.wait()
* Object.wait(long)
* Object.notify()
* Object.nofityAll()

## Thread类
* Thread.start()
* Thread.yield()
* Thread.join()
* Thread.join(long)
* Thread.sleep(long)

## LockSupport类
* LockSupport.park()
* LockSupport.parkNanos(long)
* LockSupport.parkUntil(long)
* LockSupport.unpark(Thread)

# Java线程中断
线程中断并不是线程的一种状态，而是线程的一个标志。
## 中断相关API
* public boolean isInterrupted() —— 查看线程的中断标志
* public void thread.interrupt() —— 设置线程中断标志
* public static boolean interrupted() —— 查看并重置线程的中断标志

```java
public class InterruptedTest {

    public static void main(String[] args) {
        System.out.println("当前线程是否中断：" + Thread.currentThread().isInterrupted());
        System.out.println("重置线程中断标志：" + Thread.interrupted());
        System.out.println("重置后......线程是否中断：" + Thread.currentThread().isInterrupted());

        System.out.println("******************************************");

        Thread.currentThread().interrupt();
        System.out.println("当前线程是否中断：" + Thread.currentThread().isInterrupted());
        System.out.println("重置线程中断标志：" + Thread.interrupted());
        System.out.println("重置后......线程是否中断：" + Thread.currentThread().isInterrupted());
    }
}
```
> 当前线程是否中断：false
> 重置线程中断标志：false
> 重置后......线程是否中断：false
> ******************************************
> 当前线程是否中断：true
> 重置线程中断标志：true
> 重置后......线程是否中断：false


## 不同状态对中断的反应
当线程处于不同的状态时，对调用interrupt()方法的反应是不同的。
### NEW——无反应
```java
public class New2Interrupted {

    public static void main(String[] args) {
        Thread thread = new Thread();
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
        thread.interrupt();
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
    }

}
```
> 线程状态：NEW
> 是否中断：false
> 线程状态：NEW
> 是否中断：false

### TERMINATED——无反应
```java
public class Terminated2Interrupted {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread();
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
        thread.interrupt();
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
    }

}
```
> 线程状态：TERMINATED
> 是否中断：false
> 线程状态：TERMINATED
> 是否中断：false

### RUNNABLE——标志置位true
```java
public class Runnable2Interrupted {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            while (true) {
                if (Thread.currentThread().isInterrupted()) { // 我们可以根据中断标志做一些事情
                    System.out.println("线程已中断......");
                    break;
                }
            }
        });
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断；" + thread.isInterrupted());
        thread.interrupt();
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断；" + thread.isInterrupted());
    }

}
```
> 线程状态：RUNNABLE
> 是否中断；false
> 线程状态：RUNNABLE
> 是否中断；true
> 线程已中断......

### BLOCKED——标志置位true
```java
public class Blocked2Interrupted {

    private void test() {
        synchronized (this) {
            while (true) {

            }
        }
    }


    public static void main(String[] args) throws InterruptedException {
        Blocked2Interrupted blocked2Interrupted = new Blocked2Interrupted();
        Thread t1 = new Thread(blocked2Interrupted::test);
        Thread t2 = new Thread(blocked2Interrupted::test);
        t1.start();
        Thread.sleep(1000);
        t2.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + t2.getState());
        System.out.println("是否中断：" + t2.isInterrupted());
        t2.interrupt();
        System.out.println("线程状态：" + t2.getState());
        System.out.println("是否中断：" + t2.isInterrupted());
    }


}
```
> 线程状态：BLOCKED
> 是否中断：false
> 线程状态：BLOCKED
> 是否中断：true

### WAITING——抛异常（LockSupport除外）
```java
public class Waiting2Interrupted {

    public static void main(String[] args) throws InterruptedException {

        /**
         * {@link LockSupport#park()}方法导致的waiting状态时，执行中断
         * 中断标志位变为true，但是线程的运行状态不受影响。
         */
        Thread thread = new Thread(LockSupport::park);
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
        thread.interrupt();
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());

    }
}
```
> 线程状态：WAITING
> 是否中断：false
> 线程状态：WAITING
> 是否中断：true

```java
public class Waiting2Interrupted {

    private void test() {
        synchronized (this) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {

        /**
         * {@link Object#wait()}方法导致的waiting状态时，执行中断会抛出异常。
         */
        Waiting2Interrupted waiting2Interrupted = new Waiting2Interrupted();
        Thread thread = new Thread(waiting2Interrupted::test);
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
        thread.interrupt();

    }
}
```
> 线程状态：WAITING
> 是否中断：false
> java.lang.InterruptedException
>   at java.lang.Object.wait(Native Method)
>   at java.lang.Object.wait(Object.java:502)
>   at thread.interrupt.Waiting2Interrupted.test(Waiting2Interrupted.java:17)
>   at java.lang.Thread.run(Thread.java:748)

```java
public class Waiting2Interrupted {

    public static void main(String[] args) throws InterruptedException {

        /**
         * {@link Thread#join()}方法导致的waiting状态时，执行中断会抛出异常。
         */
        Thread t1 = new Thread(() -> {
           while (true) {

           }
        });
        Thread t2 = new Thread(() -> {
            try {
                t1.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        t1.start();
        Thread.sleep(1000);
        t2.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + t2.getState());
        System.out.println("是否中断：" + t2.isInterrupted());
        t2.interrupt();

    }
}
```
> 线程状态：WAITING
是否中断：false
java.lang.InterruptedException
	at java.lang.Object.wait(Native Method)
	at java.lang.Thread.join(Thread.java:1252)
	at java.lang.Thread.join(Thread.java:1326)
	at thread.interrupt.Waiting2Interrupted.lambda$main$1(Waiting2Interrupted.java:60)
	at java.lang.Thread.run(Thread.java:748)

### TIMED_WAITING——抛异常（LockSupport除外）
`LockSupport.parkNanos(long)`和`LockSupport.parkUntil(long)`导致的TIMED_WAITING状态类似于`LockSupport.park()`导致的WAITING状态，调用`interrupt()`方法会将线程的标志置为true。而`Thread.sleep(long)`、`Object.wait(long)`及`Thread.join(long)`导致的TIMED_WAITING状态类似于`Object.wait()`及`Thread.join()`导致的WAITING状态，调用`interrupt()`方法会抛异常：
```java
public class Timed_Waiting2Interrupted {


    public static void main(String[] args) throws InterruptedException {

        /**
         * {@link Thread#sleep(long)}方法导致的timed_waiting状态时，执行中断会抛出异常。
         * 同理{@link Object#wait(long)}与{@link Thread#join(long)}结果一样。
         */
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread.start();
        Thread.sleep(1000);
        System.out.println("线程状态：" + thread.getState());
        System.out.println("是否中断：" + thread.isInterrupted());
        thread.interrupt();

        /**
         * {@link LockSupport#parkUntil(long)}方法导致的waiting状态时，执行中断
         * 中断标志位变为true，但是线程的运行状态不受影响。
         */
//        Thread thread = new Thread(() -> LockSupport.parkUntil(System.currentTimeMillis() + 10000));
//        thread.start();
//        Thread.sleep(1000);
//        System.out.println("线程状态：" + thread.getState());
//        System.out.println("是否中断：" + thread.isInterrupted());
//        thread.interrupt();
//        System.out.println("线程状态：" + thread.getState());
//        System.out.println("是否中断：" + thread.isInterrupted());

        /**
         * {@link LockSupport#parkNanos(long)}方法导致的waiting状态时，执行中断
         * 中断标志位变为true，但是线程的运行状态不受影响。
         */
//        Thread thread = new Thread(() -> LockSupport.parkNanos(10000000000L));
//        thread.start();
//        Thread.sleep(1000);
//        System.out.println("线程状态：" + thread.getState());
//        System.out.println("是否中断：" + thread.isInterrupted());
//        thread.interrupt();
//        System.out.println("线程状态：" + thread.getState());
//        System.out.println("是否中断：" + thread.isInterrupted());

    }
}
```
> 线程状态：TIMED_WAITING
是否中断：false
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at thread.interrupt.Timed_Waiting2Interrupted.lambda$main$0(Timed_Waiting2Interrupted.java:22)
	at java.lang.Thread.run(Thread.java:748)
