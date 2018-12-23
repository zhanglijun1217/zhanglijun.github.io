---
title: 并发编程——ThreadPoolExecutor源码分析（一）
copyright: true
date: 2018-10-04 18:26:46
tags:
	- 并发编程
	- ThreadPoolExecutor
categories:
	- 并发编程
	- 并发基础
---

## 前言

线程池是并发编程中最重要的应用之一，使用线程池可以防止大量的创建和销毁线程的过程，可以节省很多的内存空间，提高程序的响应率和cpu的利用率，并且也可以对线程进行统一管理和监控。这里将分几篇文章介绍一下线程池的源码分析。本篇是分析ThreadPoolExecutor中的ctl变量。并且去写了线程中的

<!-- more -->

## ctl变量

### 源码中的解释

ThreadPoolExecutor中有个字段是ctl，具体来说是对线程池的运行状态和池子中的有效线程数量的控制的一个字段变量，我们可以看看源码中的解释：

```java
/**
 * The main pool control state, ctl, is an atomic integer packing
 * two conceptual fields
 *   workerCount, indicating the effective number of threads
 *   runState,    indicating whether running, shutting down etc
 *
 * In order to pack them into one int, we limit workerCount to
 * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
 * billion) otherwise representable. If this is ever an issue in
 * the future, the variable can be changed to be an AtomicLong,
 * and the shift/mask constants below adjusted. But until the need
 * arises, this code is a bit faster and simpler using an int.
```

可以看到ctl是AtomicInteger对象，里面的操作都是基于CAS的原子操作。一个ctl变量包含两部分信息，因为int类型的变量是32位的，所以高3位表示线程池的运行状态（runState），低29位表示线程池中有效线程数（workerCount）。

所以，当我们知道了线程池中的运行状态和有效线程数，就可以通过ctlOf方法计算出ctl的值：

```java
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

反过来，我们也可以通过ctl计算出runState和workerCount的值：

```java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

其中CAPACITY是(2^29)-1， 也就是高三位是0，低29位是1的一个int类型的数字常量。这个CAPACITY表示线程池中有效线程的上限值。这个值的计算过程：

```java
private static final int COUNT_BITS = Integer.SIZE - 3; // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

### 线程池的状态

线程池中有五种状态：

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

其中COUNT_BITS的值是29，在后边的源码分析过程中，我们其实只需要知道这几个状态是逐渐递增的即可。比如说在源码中看到 rs < SHUTDOWN 其实就是表示此时线程池的运行状态是RUNNING。

五种状态的解释：

（1）RUNNING：运行状态，能接受提交新的任务，并且也能处理阻塞队列中的任务。

（2）SHUTDOWN：关闭状态，不再接受新提交的任务，但是可以处理阻塞在任务队列中已保存的任务。在线程池处于RUNNING状态的时候，调用shutdown方法线程池即变为此状态。

（3）STOP：停止状态，不再接受新提交的任务，也不会处理阻塞队列中已保存的任务，并且会中断正在处理的任务，在线程池处于RUNNING状态或者SHUTDOWN状态的时候，调用shutdownNow方法线程池即进入该状态。

（4）TIDYING：清理状态，所有的任务都已经终止，workerCount有效线程数量为0，线程池进入该状态后调用terminated方法可以使线程池进入Terminated状态。当线程池处于SHUTDOWN状态时，如果此后线程池中没有线程并且阻塞队列中没有要执行的任务，就会进入到这个状态；当线程池处于STOP状态时，如果此时线程池中没有线程了，线程池会进入该状态。
（5）TERMINATED：terminated()方法执行之后会进入该状态。

### 其他字段参数

```java

//线程池控制器
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//任务队列
private final BlockingQueue<Runnable> workQueue;
//全局锁
private final ReentrantLock mainLock = new ReentrantLock();
//工作线程集合
private final HashSet<Worker> workers = new HashSet<Worker>();
//终止条件 - 用于等待任务完成后才终止线程池
private final Condition termination = mainLock.newCondition();
//曾创建过的最大线程数
private int largestPoolSize;
//线程池已完成总任务数
private long completedTaskCount;
//工作线程创建工厂
private volatile ThreadFactory threadFactory;
//饱和拒绝策略执行器
private volatile RejectedExecutionHandler handler;
//工作线程活动保持时间(超时后会被回收) - 纳秒
private volatile long keepAliveTime;
/**
 * 允许核心工作线程响应超时回收
 * false：核心工作线程即使空闲超时依旧存活
 * true：核心工作线程一旦超过keepAliveTime仍然空闲就被回收
 */
private volatile boolean allowCoreThreadTimeOut;
//核心工作线程数
private volatile int corePoolSize;
//最大工作线程数
private volatile int maximumPoolSize;
//默认饱和策略执行器 - AbortPolicy -> 直接抛出异常
private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```

## 下一章

下一章将会去写一下ThreadPoolExecutor的构造函数和核心参数。