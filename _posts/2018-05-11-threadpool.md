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
1. 如果运行的线程数少于`corePoolSize` 直接创建新线程处理任务。即使线程池中其他线程是空闲的。
2. 如果线程数大于等于`corePoolSize`并且小于`maximumPoolSize`，只有`等workQueue`满的时候才会去创建新的任务。
3. 如果corePoolSize和maximumPoolSize设置数量是一致的,则线程池是固定的,如果`workQueue`没有满则放入`workQueue`，等待有空闲的线程从workQueue取出任务去处理
4. 如果运行线程的数量大于`maximumPoolSiz`，如果`workQueue`也满了，则会根据一个拒绝策略的参数来指定策略 来处理任务。

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


[image-1]: /img/concurrent-threadpool/BBD60F5BF4C01F8C32457FCF9A7BA371.jpg



