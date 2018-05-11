---
layout:     post
title:      "四.线程池"
subtitle:   "线程安全的一些信息"
date:       2018-05-11 09:00:00
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java
    - 高并发
---


参考慕课网[并发课程](https://coding.imooc.com/class/195.html)做的笔记

* new Thread 弊端*
1. 每次new Thread 新建对象 性能差
2. 线程缺乏统一管理，可能无限制的新建线程，互相竞争，有可能占用过多系统资源导致司机或者OOM
3. 缺少更多功能，如更多执行，定期执行，线程中断

线程池的好处
1. 重用存在的线程，减少对象创建，消亡的开销，性能好
3. 可有效控制最大并发线程数，提高系统资源利用率，同事可以避免过多资源竞争，避免阻塞
3. 提供定时指定，定期执行，单线程，并发数控制等功能

ThreadPoolExecutor 线程池

```java
 /**
    /**
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     * 线程最大线程数
     */
    private volatile int maximumPoolSize;

    /**
     * 核心线程数量
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     */
    private volatile int corePoolSize;

    /**
     * The queue used for holding tasks and handing off to worker
     * threads.  We do not require that workQueue.poll() returning
     * null necessarily means that workQueue.isEmpty(), so rely
     * solely on isEmpty to see if the queue is empty (which we must
     * do for example when deciding whether to transition from
     * SHUTDOWN to TIDYING).  This accommodates special-purpose
     * queues such as DelayQueues for which poll() is allowed to
     * return null even if it may later return non-null when delays
     * expire.
     * 阻塞队列，存储等待执行的任务，很重要 会对线程池运行过程产生重大影响
     */
    private final BlockingQueue<Runnable> workQueue;

```

如果运行的线程数少于`corePoolSize` 直接创建新线程处理任务。即使线程池中其他线程是空闲的。
如果线程数大于等于`corePoolSize`🐧小于`maximumPoolSize` 



线程池状态 
RUNNING  shutdown stop tidying terminated

running状态时候调用shutdown()进入shutdown状态
shutdown: 关闭状态。当一个线程池实例处于该状态，不能再接受新的任务，但是能处理保存在阻塞队列里的任务。
running状态或者shutdown状态时候调用shutdownNow()进入stop状态
stop:不能接受新任务也不能处理阻塞队列任务，他会中断正在处理任务的线程池实例

tidying 如果所有任务已经终止了 所有有效线程数为0会进入该状态 
调用terminated() 
进入terminated


###方法
execute() 提交任务 交给线程池执行。
submit() 提交任务 能够执行返回结果 excute+future
shutdown() 关闭线程池，等待任务都执行完
shutdown() 关闭线程池 不等任务执行完