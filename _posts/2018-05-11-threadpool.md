---
layout:     post
title:      "四.线程池"
subtitle:   "一些高并发的基础"
date:       2018-05-11 12:00:00
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java
    - 高并发
---


### new Thread 弊端
1. 每次new Thread 新建对象 性能差
2. 线程缺乏统一管理，可能无限制的新建线程，互相竞争，有可能占用过多系统资源导致死机或者OOM
3. 缺少更多功能，如更多执行，定期执行，线程中断

### 线程池的好处
1. 重用存在的线程，减少对象创建，消亡的开销，性能好
3. 可有效控制最大并发线程数，提高系统资源利用率，同事可以避免过多资源竞争，避免阻塞
3. 提供定时指定，定期执行，单线程，并发数控制等功能

### ThreadPoolExecutor 线程池

```java
 /**
    /**
     * 线程最大线程数
     */
    private volatile int maximumPoolSize;

    /**
     * 核心线程数量
     */
    private volatile int corePoolSize;

    /**
     * 阻塞队列，存储等待执行的任务，很重要 会对线程池运行过程产生重大影响
     */
    private final BlockingQueue<Runnable> workQueue;
    
    /**
     * 线程没有任务执行时最多保持多久时间终止
     * 当线程池中的线程数量大于corePoolSize，如果没有新的任务提交，
     * 核心线程外的线程不会立刻销毁，先进入等待直道超过keepAliveTime
     */
    private volatile long keepAliveTime;
    
    /**
     * 线程工程 用来创建线程，
     */
    private volatile ThreadFactory threadFactory;

    /**
     * 当拒绝处理任务时候的策略 
     * 1.直接抛出异常(默认)。
     * 2.用调用者所在的线程执行任务 
     * 3.丢弃队列中最靠前的任务并执行当前任务 
     * 4.直接丢弃这个任务
     */
    private volatile RejectedExecutionHandler handler;
```

ThreadPoolExecutor 对线程池和队列的使用方式如下：

1. 从线程池中获取可用线程执行任务，如果没有可用线程则使用ThreadFactory创建新的线程，直到线程数达到corePoolSize限制
2. 线程池线程数达到corePoolSize以后，新的任务将被放入队列，直到队列不能再容纳更多的任务
3. 当队列不能再容纳更多的任务以后，会创建新的线程，直到线程数达到maxinumPoolSize限制
4. 线程数达到maxinumPoolSize限制以后新任务会被拒绝执行，调用 RejectedExecutionHandler 进行处理



### 线程池状态 
![线程池状态][image-1]

1. running：能接受新提交的任务，也能处理阻塞队列中的任务
2. shutdown：不能处理新的任务，但是能继续处理阻塞队列中任务
3. stop：不能接收新的任务，也不处理队列中的任务
4. tidying：如果所有的任务都已经终止了，这时有效线程数为0
5. terminated：最终状态

### Executor框架接口
1. Executors.newCachedThreadPool ：创建一个可缓存的线程池，如果线程池长度超过处理的需要，可以灵活回收空闲线程，如果没有可以回收的则新建线程。
2. Executors.newFixedThreadPool：创建一个定长的线程池，可以控制线程的最大并发数，超出的线程会在队列中等待
3. Executors.newScheduledThreadPool：创建一个定长的线程池，支持定时和周期性的任务执行。
4. Executors.newSingleThreadPool：单线程化的线程池，只会用唯一的工作线程去执行任务，保证任务按指定顺序执行(先入先出，优先级顺序等)

>这几个方法使用起来比较方便，但是这些方法返回的是`ExecutorService`,在ExecutorService中只会返回简单启动关闭等方法，缺少`ThreadPoolExecutor`中很多实用的功能。

### 线程池-合理配置
1. cpu密集型任务，就需要尽量压榨cpu，参考值可以设置为NCPU+1
2. IO密集任务 参考值可以设置为2*NCPU


### 一些线程的问题
[ThreadPoolExecutor策略配置以及应用场景](https://segmentfault.com/a/1190000008394155)这篇文章对几种常用的ThreadPoolExecutor写得非常详细


#### 为什么newFixedThreadPool中要将corePoolSize和maximumPoolSize设置成一样？
```
  public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
因为newFixedThreadPool中用的是LinkedBlockingQueue（是无界队列），只要当前线程大于等于corePoolSize来的任务就直接加入到无界队列中，所以线程数不会超过corePoolSize，这样maximumPoolSize没有用。例如，在 Web 页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。


#### 为什么newFixedThreadPool中队列使用LinkedBlockingQueue？
设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池是大小固定的，要保证线程池大小固定则需要LinkedBlockingQueue（无界队列）来保证来的任务能够放到任务队列中，不至于触发拒绝策略。


#### 为什么newFixedThreadPool中keepAliveTime会设置成0？
因为corePoolSize和maximumPoolSize一样大，KeepAliveTime设置的时间会失效，所以设置为0

#### 为什么newCachedThreadPool中要将corePoolSize设置成0？
因为队列使用SynchronousQueue，队列中只能存放一个任务，保证所有任务会先入队列，用于那些互相依赖的线程，比如线程A必须在线程B之前先执行。

#### 为什么newCachedThreadPool中队列使用SynchronousQueue？
线程数会随着任务数量变化自动扩张和缩减，可以灵活回收空闲线程，用SynchronousQueue队列整好保证了CachedTheadPool的特点。

#### 为什么newSingleThreadExecutor中使用DelegatedExecutorService去包装ThreadPoolExecutor？
SingleThreadExecutor是单线程化线程池，用DelegatedExecutorService包装为了屏蔽ThreadPoolExecutor动态修改线程数量的功能，仅保留Executor中的方法。

[image-1]: /img/concurrent-threadpool/BBD60F5BF4C01F8C32457FCF9A7BA371.jpg



