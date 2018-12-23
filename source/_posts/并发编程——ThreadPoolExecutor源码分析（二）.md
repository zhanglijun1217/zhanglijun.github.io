---
title: 并发编程——ThreadPoolExecutor源码分析（二）
copyright: true
date: 2018-10-11 23:24:26
tags:
	- 并发编程
	- ThreadPoolExecutor
categories:
	- 并发编程
	- 并发基础
---

## 前言

在上一篇中，我们分析了ThreadPoolExecutor中关键变量ctl，这篇我们继续来看ThreadPoolExecutor中的构造函数及其参数。其中参数的相关解释来源于源码中的相关注释。

<!-- more -->

## 构造函数

我们可以看到ThreadPoolExecutor有四个构造函数：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2018120300175155.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

他们其实都是调用其中的全参数的构造函数，只不过有一些参数是使用了默认提供的参数。我们可以看一下构造函数：

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

可以看到构造函数现对于参数去进行了校验：
（1）corePoolSize必须大于等于0，maximumPoolSize必须大于0，maximumPoolSize必须大于等于corePoolSize，keepAliveTime如果传入了必须大于0。

（2）workQueue、threadFactory、handler不能为空

（3）其中有acc字段的设置是为了设置安全管理器，我们可以自定义我们的安全管理器，否则为从上下文中去拿。可以参考博客：[jvm中的安全管理器](https://blog.csdn.net/yfqnihao/article/details/8262858)

### 构造函数中的参数

#### 1.corePoolSize

corePoolSize参数表示线程池中一直存活的线程的最小数量，这些一直存活的线程被称为核心线程，默认情况下，核心线程的最小数量都是整数，除非是调用了allowCoreThreadTimeout()方法并且传入了true，即允许核心线程数在空闲状态下超时而停止（terminated状态），此时如果所有的核心线程先后都因为超时停止，那么线程池中核心线程数会变为0。默认情况下，核心线程是按照需要创建并启动的，也就是说只有当线程池接收到我们提交的任务后，它才会去创建并启动一定的核心线程去执行这些任务。如果没有接收到相关任务，就不会去主动创建核心线程，这种默认的核心线程创建启动方式变主动为被动，类似于观察者模式，有利于降低系统资源的消耗。当然，也可以通过设置preStartCoreThread()或者preStartAllCoreThreads()方法来改变这一机制，使得在新任务还未提交到线程池的时候，线程池就已经创建并启动了一个或所有线程，并让这些核心线程在池中等待任务的到来。

#### 2.maximumPoolSize

maximumPoolSize表示线程池中能容量线程的最大数量，这个值不能超过常量CAPACITY的数值大小，上一篇中也提到了常量CAPACITY的计算方式，这里不去赘述。但是注意一点，**当我们提供的工作队列是一个无界的队列，那么这里提供的maximumPoolSize将毫无意义。**

当我们通过execute方法提交一个任务的时候：

（1）如果线程池处于运行状态（RUNNING）的线程数量小于核心线程数（corePoolSize），那么即使有一些非核心线程处于空闲状态，系统也倾向于新建一个线程来处理这个任务。

（2）如果线程池处于运行状态（RUNNING）的线程数量大于核心线程数（corePoolSize），但又小于maximumPoolSize，那么系统会去判断线程池内部的阻塞队列是否有空位子，如果有空位子，系统会将该任务先存入阻塞队列，如果发现队列中已没有空位子（即队列已满），系统会创建一个新的线程来执行任务。

如果将线程池中的corePoolSize和maximumPoolSize设置为相同的数（也就是说线程池中所有线程都是核心线程），那么该线程池就是一个固定容量的池子。如果将线程池的maximumPoolSize设置为一个非常大的数值（例如Integer.MAX_VALUE），那么相当于允许线程池自己在不同时段调整参与并发的总任务数。通常情况下，都是通过构造函数去初始化corePoolSize和maximumPoolSize，也可以通过set方法调整这两个参数的大小。

#### 3.keepAliveTime & unit

keepAliveTime表示空闲线程处于等待的超时时间，超过该时间后该线程会停止工作。当线程池中总线程数量大于corePoolSize并且allowCoreThreadTimeOut为false时，这些多出来的非核心线程一旦进入空闲等待的状态，就开始计算各自的等待时间，并且这里设定的keepAliveTime的数值作为他们的超时时间，一旦某个非核心线程的等待时间到达了超时时间，该线程就会停止工作（terminated）。而如果不去设置allowCoreThreadTimeout为true，核心线程及时处于空闲状态等待了keepAliveTime，也依然可以继续处于空闲状态等待。

**比较好的应用实践：**

**如果要执行的任务相对较多，并且每个任务执行的时间都比较短，那么可以为keepAliveTime参数设置一个相对较大的值，以提高线程的利用率；如果要执行的任务比较少，线程池使用率比较低，那么可以先将该参数设置为一个较小的参数值，通过超时停机的机制来降低系统资源的开销。**

注意一点：构造函数中的参数keepAliveTime和unit这个参数和ThreadPoolExecutor中的keepAliveTime字段的值不一定相等，字段被设置为long型的值，且定义为纳秒的单位，构造函数中的参数还有unit单位，应该是keepAliveTime和unit计算的结果换算为纳秒才和类中的字段是一样的值。

keepAliveTime在构造函数中的类型是long型，这样保证了这个值不会太短。

#### 4.workQueue

构造函数中的workQueue是一个BlockIngQueue（阻塞队列）的实例。传入的泛型参数是Runnable，也就是说，workQueue是一个内部元素为Runnable（各种任务，通常是异步的任务）的阻塞队列。阻塞队列是一种类似于”生产者-消费者“模式的队列，当队列已满时如果继续向队列中插入元素，该插入操作将被阻塞一直处于等待状态，直到队列中有元素被移除，才能进行插入操作；当队列为空时如果继续执行元素的删除或者获取操作，也会被阻塞进入等待队列中有新的元素之后才能执行。

workQueue是一个用于保存等待执行的任务阻塞队列，当提交一个新的任务到线程池后，线程池会根据当前池子正在运行的线程数量来判断对这个任务的处理方式：

（1）如果线程池中正在运行的线程数少于核心线程数，那么线程池总是倾向于新建一个线程来执行该任务。

（2）如果线程池中正在运行的线程数不少于核心线程数，那么线程池把该任务提交到workQueue中让其先等待

（3）如果线程池中正在运行的线程数不少于核心线程数，并且线程池中的阻塞队列也满了使得该任务入队失败，那么线程池会去判断当前池子中运行的线程数是否已经等于了该线程池允许运行的最大线程数。如果发现已经等于，说明池子已满，那么就会执行拒绝策略；如果发现运行的线程数小于池子允许的最大线程数，那么会创建一个线程（这个线程是非核心线程）来执行该任务。

这其中，队列对于提交的任务一般有三种策略：
**（1）直接切换**

常用的队列是SynchronousQueue（同步队列）,这个队列内部不会存储元素，每一次插入操作都会先进入阻塞状态，一直等到另一个线程执行了队列的删除操作，然后该插入操作才会执行。当提交一个任务到包含这种SynchronousQueue队列的线程池后，线程池会去检测是否有可用的线程来执行任务，如果没有则创建一个新的线程来执行任务而不是将任务存储在任务队列中。”直接切换“的意思是：处理方式由”将该任务暂时存储在阻塞队列中“直接切换为”新建一个线程来处理任务“。这种执行策略适合处理多个有相互依赖关系的任务，因为该策略可以避免这些任务因一个没有及时处理而导致依赖于该任务的其他任务也不能及时处理而造成的锁定结果。因为这种策略的目的是要让几乎每一个新提交的任务都能立即得到处理，所以这种策略通常配合maximumPoolSize是无边际（Integer.MAX_VALUE）的。我们知道的静态工厂方法Executors.newCachedThreadPool()就是使用了这种直接切换的队列。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001759950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

**（2）使用无界队列**

不预设队列的容量，队列将使用Integer.MAX_VALUE作为默认容量，例如：基于链表的阻塞队列 LinkedBlockingQueue。使用无界队列使得线程池中能创建的最大线程数等于核心线程数，这样的线程池的maxmumPoolSize的数值将不起任何作用。如果向线程池中提交一个新任务时发现所有的核心线程都处于运行状态，那么该任务将被放入无界队列中等待处理。当要处理的多个任务之间没有相互依赖关系的时候，就适合用这种队列策略来处理这些任务。静态工厂方法Executors.newFixedThreadPool()就使用了这个队列。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001810958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

**（3）使用有界队列**

例如使用基于数组的阻塞队列 ArrayBlockingQueue。当要求线程池的最大线程数maximumPoolSize要限定在某个值以内的时候，线程池使用有界队列能降低资源的消耗，但这也使得线程池对线程的调控变得更加困难。因为队列容量和线程池容量都是有限的值，要想使线程处理任务的吞吐量在一个相对合理的范围内，同时又能使线程调度的难度相对较低，并且又尽可能节省系统资源的消耗，那么需要合理的调配这两个值。通常来说，设置较大的队列容量和较小的线程池容量，能够降低系统的资源的消耗（包括CPU的使用率，操作系统的消耗，上下文环境的切换的开销等），但是会降低系统吞吐率。如果发现提交的任务经常频繁的发生阻塞的情况，那么你可以考虑增大线程池的容量，可以通过setMaximumPoolSize()方法来重新设定线程池的容量。而设置较小的队列量时，通常需要将线程池的容量设置大一点，这种情况下，cpu的使用率会比较高，但是如果设置线程池的容量过大的时候，线程调度成了问题，反而使得吞吐率比较低。

#### 5.threadFactory

线程工厂，用于创建线程。默认使用Executors.defaultThreadFactory()方法创建线程工厂：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203002105597.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

当然我们也可以自己实现ThreadFactory接口去实现我们自己的线程工厂。下边就是可以根据不同的namePrefix去获取单例线程的线程工厂：

```java
/**
 * 自定义线程工厂类
 */
private static class MyThreadFactory implements ThreadFactory {
    /**
     * namePrefix --> 线程名字中的计数
     */
    private static Map<String, AtomicInteger> THREAD_ID_TABLE = new ConcurrentHashMap<>();
    /**
     * 线程名称前缀
     */
    private String namePrefix;
    /**
     * 是否后台线程
     */
    private boolean isDamon;
    public MyThreadFactory(String namePrefix) {
        this(namePrefix, true);
    }

    public MyThreadFactory(String namePrefix, boolean isDamon) {
        this.namePrefix = namePrefix;
        this.isDamon = isDamon;
    }


    @Override
    public Thread newThread(Runnable r) {
        String threadName = namePrefix + "-" + generateThreadId(this.namePrefix);
        Thread thread = new Thread(r, threadName);
        thread.setDaemon(this.isDamon);
        System.out.println("创建线程" + threadName);
        return thread;
    }

    private static int generateThreadId(String namePrefix) {

        // 判断后执行 concurrentHashMap不能保证完全线程安全 用了putIfAbsent
        if (!THREAD_ID_TABLE.containsKey(namePrefix)) {
            THREAD_ID_TABLE.putIfAbsent(namePrefix, new AtomicInteger(0));
        }
        return THREAD_ID_TABLE.get(namePrefix).getAndIncrement();
    }
}
```

#### 6.handler

当满足以下两个条件其中一个的时候，如果继续向线程池中提交新的任务，那么线程池会调用内部的RejectedExecutionHandler对象的rejectedExecution()方法，表示拒绝执行这些新提交的任务：

**（1）当线程池处于SHUTDOWN状态时（不论线程池和阻塞队列是否已满）**

**（2）当线程池中所有的线程都处于运行状态并且线程池中的阻塞队列已满。**

一个demo去演示这两个情况：[执行handler的两种情况](https://github.com/zhanglijun1217/juc/blob/master/src/threadPool/ThreadPoolExecutorRejectNewTaskDemo.java)

当采用默认的拒绝策略，线程池会使用抛出异常的方式来拒绝新任务的提交，这种拒绝方式在线程池中被称为AbortPolicy，我们可以来看下有哪些拒绝策略：
**（1）AbortPolicy**

这中处理方式是直接抛出RejectedExecutionException异常，如果在ThreadPoolExecutor的构造函数中未指定RejectedExecutionHandler参数，那么线程池将使用defaultHandler参数，而这个就是采用的AbortPolicy。

**（2）CallerRunsPolicy**

将提交的任务放在ThreadPoolExecutor.execute()方法所在的那个线程执行。

**（3）DiscardPolicy**

直接不执行新提交的任务

**（4）DiscardOldestPolicy**

这个可以看源码中的解释：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181203001823589.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psajEyMTc=,size_16,color_FFFFFF,t_70)

由源码就可以知道，这种处理方式有两种情况：一，当线程池处于SHUTDOWN状态时，就默认不执行这个任务，即DiscardPolicy；二，当线程池处于运行状态时，会将队列中处于队首（head）的那个任务从队列中移除，然后将这个新提交的任务加入到阻塞队列中的队尾（tail）等待执行。

当然，RejectedExecutionHandler其实是个接口，我们可以自定义类去实现这个接口，重写rejectedExecution方法使用自己想要的拒绝策略即可。

## 下一篇

这篇把线程池中的核心参数进行了一些解释，在下一篇中我们将介绍线程池进行任务调度的原理。