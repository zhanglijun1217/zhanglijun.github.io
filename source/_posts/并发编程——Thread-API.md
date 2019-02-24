---
title: 并发编程——Thread API
copyright: true
date: 2019-02-21 23:19:21
tags:
	- Thread API
categories:
	- 并发编程
	- 并发基础
---

这篇主要介绍Thread API，也是并发编程中的基础

<!-- more -->

## Thread一些常用API

### 守护线程

- 守护线程的概念和原理可以见：[守护线程和非守护线程](https://zhanglijun1217.github.io/blog/2018/11/26/%E5%AE%88%E6%8A%A4%E7%BA%BF%E7%A8%8B%E5%92%8C%E9%9D%9E%E5%AE%88%E6%8A%A4%E7%BA%BF%E7%A8%8B/)

- 守护线程的一个应用：

比如在做长连接的时候，需要一个心跳检查线程，这个线程就应该设置为后台线程，这样当整个连接关闭时，也会跟随连接线程消亡。

- 在构建Daemon线程时，不能依靠finally块中的内容来确保执行关闭或清理资源的逻辑，会有可能不去执行。

这里可以在一个线程中再创建一个后台线程，来验证上述的这个应用：

```java
/**
 * 这里对后台线程提出一个问题：
 * （1）当在main函数中的一个Thread里再创建一个线程，设置为后台线程，那么外边线程结束之后 里面的线程是否也会退出？ 会的 这个就长连接中的健康检查
 *
 *
 * @author 夸克
 * @date 2019/2/19 00:21
 */
public class DaemonQuestionThread {

    public static void main(String[] args) {
        Thread outerThread = new Thread(() -> {
            Thread innerThread = new Thread(() -> {
               try {
                   while (true) {
                       System.out.println("do Something for health check");
                       Thread.sleep(1_000);
                   }
               } catch (Exception e) {
                   e.printStackTrace();
               }
            });
            // 设置一个守护线程 设置的过程必须在start方法之前
            innerThread.setDaemon(true);
            innerThread.start();
        });

        try {
            Thread.sleep(1_000);

        } catch (Exception e) {
            e.printStackTrace();
        }
        outerThread.start();
        System.out.println("程序结束");
    }
}
```

### 线程id

线程id是Thread类在构造函数初始化时赋值给的Thread类中的tid字段，而赋值时其实调用的是静态的加锁方法nextThreadId()。可以看下源码：

```java
private static synchronized long nextThreadID() {
    return ++threadSeqNumber;
}
```

其实就是对Thread类中的静态字段threadSeqNumber自增。

```java
  /* For generating thread ID */
    private static long threadSeqNumber;
```

### 线程优先级

概念：

OS是采用分时间片的形式调度运行的线程，OS会分出一个个时间片，线程会分配到若干时间片，当线程的时间片用完了就会发生线程调度，并等待着下次分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而线程优先级就是决定线程需要多或者少分配一些处理器资源的线程属性。

- 是一个和操作系统有关的线程属性，有的操作系统直接回忽略用户设置线程的优先级。
- IO密集型是要设置高线程优先级的，cpu密集型是要设置低优先级的。

### join方法

- 一个线程调用join方法其实就是让被join的线程等待该线程执行完毕之后再执行。下面是java代码示例。可以看到join其实也是通过wait()方法去实现的。

```java
/**
 * 哪个线程去调用join方法 就可以在加的线程内先执行该线程完毕之后才执行外部线程
 *
 * join方法可以加时间控制
 * join方法的底层实现其实是wait
 *
 * @author 夸克
 * @date 2019/2/20 00:11
 */
public class ThreadJoinTest {

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            // t1线程
            IntStream.range(1, 1000).forEach(i -> System.out.println(Thread.currentThread().getName() + "--" + i));
        });

        t1.start();
        Thread t2 = new Thread(() -> {
            // t2线程
            IntStream.range(1, 1000).forEach(i -> System.out.println(Thread.currentThread().getName() + "--" + i));
        });

        t2.start();

        //
        try {
            // 调用了t1.join 会先输出t1 再输出t2
            t1.join();
            t2.join();// 这样写了之后 对于main线程来说 必须等到t1 和 t2线程执行完毕 才能执行main线程  但是对于t1 和 t2 是交替执行的
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("==================t1 和 t2执行完毕================");
        IntStream.range(1, 1000).forEach(i -> System.out.println(Thread.currentThread().getName() + "--" + i));

        // 如果这里调用 Thread.currentThread.join 则会出现程序无法关闭的问题。 main线程自己join了自己的情况
//        try {
//
//            Thread.currentThread().join();
//        } catch (Exception e) {
//            e.printStackTrace();
//        }
    }
}
```

- 这里再补充一个join的小demo：比如要每个线程去采集对应服务器的数据，现在有三台服务器，每台除了要记录对应服务器采集的时间外，还要在主线程中输出一共消耗了多少时间。这个就是一个join方法的简单应用。

```java
/**
 * Thread.join方法的一个小demo ：
 *  假设有四台服务器，每个线程要对每台服务器采集信息，比如不同的服务器采集需要不同的时间，
 *  这里要求主线程去记录时间的时候，必须等待每个线程采集信息完毕
 *
 * @author 夸克
 * @date 2019/2/20 00:29
 */
public class ThreadJoinDemo {

    public static void main(String[] args) {

        // 模拟三个服务器
        long begin = System.currentTimeMillis();
        Thread thread1 = new Thread(new CaptureRunnable("m1", 1_000));
        Thread thread2 = new Thread(new CaptureRunnable("m2", 2_000));
        Thread thread3 = new Thread(new CaptureRunnable("m3", 3_000));

        thread1.start();
        thread2.start();
        thread3.start();

        // 这里必须调用join 才能保证每个线程执行完毕
        try {
            thread1.join();
            thread2.join();
            thread3.join();

        } catch (Exception e) {
            e.printStackTrace();
        }

        // 因为是每个服务器 并行采集 所以这里的最长应该是t3的
        System.out.println("采集信息完毕,最长花费时间：" + (System.currentTimeMillis() - begin));
    }

}

class CaptureRunnable implements Runnable {

    private String name;
    private long expiredTime;

    public CaptureRunnable(String name, long expiredTime) {
        this.name = name;
        this.expiredTime = expiredTime;
    }
    @Override
    public void run() {
        try {
            long beginTime = System.currentTimeMillis();
            Thread.sleep(expiredTime);
            System.out.println(Thread.currentThread().getName() + "采集信息花了" + (System.currentTimeMillis() - beginTime));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### interrupt方法

- interrupt方法是Thread API中一个重要的方法。interrupt方法是Thread类提供的一个线程中断机制，这个方法的原理是给线程设置一个为true的中断标志，设置之后，会根据线程的状态有不同的结果。如果当前线程处于阻塞状态，那么将中断标志设为true后，如果是由join、wait、sleep引起的阻塞状态，那么会将线程的中断标志设置回false，并且抛出一个InterruptedException；如果打断时当前线程处于非阻塞状态，那么仅仅会将线程中的中断 标志设置为true，在之后如果线程进入了阻塞状态，也会立马抛出一个InterruptedException，且中断标志被清除，重新设置为false。

- 所以对于interrupt方法，可以知道调用了interrupt方法之后线程不一定会中断，它只是将线程的中断标志位设置为true，而是否抛出InterruptedEx是根据线程的阻塞状态相关。它更像是线程的一个协作机制，线程A需要中断B线程，那么调用B.interrupt()进行线程的协作。

我们可以看看其中的源码：

在Thread类中，有一个变量blocker来表示线程的中断标志位，对于这个字段我们可以知道它的默认值是null，之后我们的理解其实可以简单的认为当中断标志位设置了，就是true，而中断标志位被清除了，就是false了。

```java
private volatile Interruptible blocker;
```

而关于中断有三个方法：

- public void interrupt(); 

这个方法就不去赘述了，作用就是设置中断标志位为true，如果当前

- public static boolean interrupted(); 和 public boolean isInterrupted();

这两个方法前者是静态方法，后者是类实例方法，都返回了当前线程是否被中断。主要有两个区别：

（1）静态方法提供了一种访问的方式，比如初始化Thread中传入lambda表达式作为Runnable接口的方式，这样无法在当前内部类中调用线程实例的判断方法，有了前者方法就可以直接调用静态方法，获取当前线程是否被中断。

```
Thread thread = new Thread(() -> {
    while (true) {
        try {
            Thread.sleep(100);// 阻塞状态 清除中断标志位 抛出异常
        } catch (Exception e) {
            System.out.println("收到打断信号");
            // 这里就是外边的interrupt方法打断了这里的sleep
            e.printStackTrace();
        }
        System.out.println(">>" + Thread.currentThread().getName() + ".." + Thread.interrupted());
    }
});
```

（2）前者静态方法，会判断当前线程是否已经中断。**线程的中断状态 由该方法清除**。线程中断被忽略，因为在中断时不处于活动状态的线程将由此返回 false 的方法反映出来。而后者实例方法，判断线程是否已经中断。**线程的中断状态 不受该方法的影响**。线程中断被忽略，因为在中断时不处于活动状态的线程将由此返回 false 的方法反映出来。这里区别是静态方法会清除线程的中断标志，这里可以分析源码得到：

```java
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}

public boolean isInterrupted() {
    return isInterrupted(false);
}

private native boolean isInterrupted(boolean ClearInterrupted);
```

可以看到其实两个方法都是调用的native方法isInterrupted方法，但是静态方法传入的参数是true，而实例方法传入的是false，这个参数的含义很清晰就是控制是否清楚当前中断标志位。

- 这里关于interrupt方法不能真正中断线程的实现，也可以在源码中得到，同时也能解释为什么当线程处于阻塞状态时，调用interrupt()方法，会抛出InterruptedException，并且将标志位清空。

```java
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        // blocker为空时不会进入到if的判断，所以只会调用synchronized代码块之后的最后的interrupt0方法，而从注释来看这个native方法仅仅是设置interrupt标志位的
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this); // 真正执行中断线程的方法
            return;
        }
    }
    interrupt0();
}
```

这个时候当线程中的中断标志位为空时，很明显不会进入if的判断，这时只是会设置当前线程的中断标志位。而且这时也没有进入的if中的interrupt真正中断线程的方法。

而当线程阻塞时，以sleep方法为例，在调用sleep时，就会调用native方法interrupt0，这个方法会将线程标志位设置为true，并且现在blocker标志肯定不为null。所以会进入到if的判断代码块中，这时会再调用一次interrupt0方法（调用这个方法会清除当前线程的标志位），并且使当前线程退出阻塞状态（调用了真正的中断线程的方法）并且抛出InterruptedException异常。见下图

![](http://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/122016594880.jpg)

- InterruptedException异常的处理

这里要规范的处理方法有两种：

（1） 把该类异常抛给上层调用者来处理（当然，抛出去后，接收者也要考虑这个同样的问题）

（2）在 catch 该异常后，通过 interrupt() 方法恢复当前线程的中断状态，示例如下：

```java
try {
    Thread.sleep(100);// 阻塞状态 清除中断标志位 抛出异常
} catch (InterruptedException e) {
    System.out.println("收到打断信号");
    // 这里就是外边的interrupt方法打断了这里的sleep
    e.printStackTrace();
    
    // 正确的处理InterruptException的一种方式
    Thread.currentThread().interrupt();// 通过这个回复线程的中断标志位，给下面的操作处理
}
```

这样原因已经比较清楚了：出现了 InterruptedException，说明当前线程在 wait / sleep / join 时的阻塞（等待）状态下被打断，此时 JDK 的实现默认会退出阻塞，并且**清除了中断状态**。也即是讲，如果此时通过 isInterrupted() 去读取中断状态时，得到的是 false。而这与 interrupt() 的调用目的是违背的，因为 interrupt 的目的是请求和标记目标线程的中断。如果我们不去主动恢复中断状态，就会导致其他需要读取中断状态的地方 判断错误，导致一些意外情况的发生。

#### interrupt这里的参考

- [一个interrupt方法的详解](<https://www.cnblogs.com/carmanloneliness/p/3516405.html>)
- [interrupt的用途](http://www.cnblogs.com/w-wfy/p/6415005.html)
- [关于InterruptedException的思考](https://mp.weixin.qq.com/s?__biz=MzU4Nzc0MDQ1OA==&mid=2247483689&idx=1&sn=5315c42923956cebf4c253960c26fa14&chksm=fde6256cca91ac7a31f567a90528f549ea635fcbf6a2c285523ef41788f0b5f6a6f6b0d39f43&token=2022964646&lang=zh_CN#rd)

## 交流

上述代码都能在github中找到:
[](https://github.com/zhanglijun1217/juc/tree/master/src/lesson/wwj/juc/thread_api)