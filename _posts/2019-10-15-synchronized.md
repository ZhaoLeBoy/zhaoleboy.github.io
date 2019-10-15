---
layout:     post
title:      "悲观锁-synchronized"
subtitle:   "悲观锁-synchronized"
date:        2019-10-15
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - Synchronized
---

##### 原理
通过字节码，可以看出来，synchronized在修饰代码块同时 由monitorenter和monitorexit指令来实现同步的，进入monitorenter指令后，线程将持有`Monitor`对象，退出moniitorenter指令后，线程将释放`Monitor`对象

JVM 中的同步是基于进入和退出管程(Monitor)对象实现的。每个对象实例都会有一个Monitor，Monitor是和对象一起创建，销毁的。具体见[管程Monitors](http://jinlipool.com/2019/10/15/monitors/)


##### 优化
JDK1.6以后对`synchronized`进行了优化，引入偏向锁，轻量级锁，重量级锁来减少锁竞争带来的上下文切换。
当java对象被`synchronized`关键字修饰为同步锁后，围绕锁的一系列升级操作都跟java对象头有关。

###### java对象头
java对象实例在堆内存中分成了三个部分，对象头，实例数据，对齐填充。其中对象头就是由Mark Word，指向类的指针以及数组长度。
其中Mark Word中就记录这锁有关的信息。Mark Word在64位JVM中长度是64bit。
![430d405f57f20f1cabaf624c00dd2fb1](/img/lock/synchronized-optimize.png)
`synchronized`就是从偏向锁开始，随着竞争激烈而逐渐升级。偏向锁->轻量级锁->重量级锁

###### 1. 偏向锁
用于优化同一个线程多次申请同一个锁的竞争，大多数情况下锁总是被同一个线程多次获取。不存在多线程竞争的情况.
偏向锁的作用，就是当一个线程再次访问这个同步代码块或者方法的时候，该线程只需要去对象头的Mark Word中去判断一下是否有偏向锁指向它的线程id，无需进入Monitor去竞争对象了。
>检查Mark Word的优势:如果选择CAS，虽然不会造成上下文切换，但频繁的操作还是会损耗cpu性能，而偏向锁只需要在置换线程ID的时候做一次CAS操作即可。

`一旦有其他线程去竞争锁资源，偏向锁就会被撤销`，但是不它会立刻撤销，它会等到全局安全点（这个时间点没有字节码执行）暂停(sop-the-word)持有锁的线程，并且对该线程进行检查是否还在执行方法，如果是则升级锁，反之其他线程就抢占这个偏向锁。

###### 2. 轻量级锁
当其他线程竞争偏向锁成功时候，发现对象头Mark Word中的线程ID不是自己的，就会进行一次CAS操作获取锁，如果成功，这个线程就会保持偏向锁的状态去继续执行线程方法，如果获取失败，说明当前锁还是有其他的线程竞争，这个时候偏向锁就会升级为轻量级锁。
轻量级的锁适合线程交替执行同步块场景，通过自旋避免用户态和内核态的频繁切换。绝大部分的锁在整个同步周期内都不存在长时间的竞争。

###### 3. 重量级锁
轻量级锁CAS抢锁失败后，线程会被挂起，如果正在持有锁的线程在很短时间内释放资源，那么进入堵塞状态的线程又要申请锁资源。轻量级别锁通过自旋不断的尝试获取锁，如果再自旋一定次数后仍然没有抢到锁，为了不继续耗费cpu，轻量级锁就会升级为重量级锁

![7c48a2c666c7233badb48cd78199daf6](/img/lock/synchronized-flow.png)

##### 总结
JDK1.6以后引入了分级锁来优化synchronized，本质上是减少锁的竞争，当一个线程获取锁后，首先对象锁是个偏向锁，用来优化同一线程重复获取通一把锁而导致用户态和内核态的频繁切换。当多个线程竞争锁资源，锁会升级为轻量级锁，适合短时间持有，使用自旋来避免线程用户态和内核态频繁切换。如果锁的竞争还是很激烈，就会升级为重量级锁。

##### 资料引用
* [【不可不说的Java“锁”事】](!https://tech.meituan.com/2018/11/15/java-lock.html)
* [【多线程之锁优化（上）：深入了解Synchronized同步锁的优化方法】](!https://time.geekbang.org/column/article/101244)


