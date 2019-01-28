---
layout:     post
title:      "1-1.Netty源码:构建NioEventLoopGroup"
subtitle:   "构建NioEventLoopGroup"
date:        2019-01-27
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


### 关系图
先上NioEventLoopGroup的继承关系图，有删减
![IMAGE](/img/netty/1-1/1.jpg)

一般来说Netty在启动的时候，会实例化两个NioEventLoopGroup，一个是boss，一个是worker,
boss是用来接受新的链接，worker是用来处理新链接的网络读写，具体初始化的类方法是在`MultithreadEventExecutorGroup`中进行的。

```java
//实例化参数是线程数，不传的化默认是cpu核数*2
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workGroup = new NioEventLoopGroup();
```
>EventLoopGroup是一个接口，用途是在事件循环中（例如等待连接，等待输入输出流等的一个死循环），在进行selection操作中允许注册Channel。
 NioEventLoopGroup是EventLoopGroup的一个实现类，MultithreadEventLoopGroup的实现，是基于Channel的Nio选择器(Selector)。

### 概述
NioEventLoopGroup初始化 做了三件事
 1. 构造线程执行器`ThreadPerTaskExecutor`
 2. 根据线程数和线程创建器`ThreadPerTaskExecutor`构造多个事件执行器`EventExecutor`数组。
 3. 创建线程事件执行器的选择器chooser，类型是`EventExecutorChooserFactory.EventExecutorChooser`


### 1.构造线程执行器`ThreadPerTaskExecutor`
```java
//MultithreadEventLoopGroup.java
 executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
```
`newDefaultThreadFactory()`自定义一个线程工厂。具体见[【1-1-1.Netty源码:线程执行器ThreadPerTaskExecutor】](http://jinlipool.com/2019/01/27/netty-1-1-1-ThreadPerTaskExecutor/)


### 2.创建事件执行器`EventExecutor`

```java
//  具体实现是在`NioEventLoop`中实现
children[i] = newChild(executor, args);
```
创建了新的事件执行器，调用next()方法进行访问，是可以在MultithreadEventExecutorGroup中每个线程调用此方法

NioEventLoop继承关系如下，有删减
![IMAGE](/img/netty/1-1/2.jpg)

初始化中继承关系如上图右侧
>NioEventLoop->SingleThreadEventLoop->SingleThreadExecutor->AbstractScheduledEventExecutor->AbstractEventExecutor
最重要的几个方法就是在`NioEventLoop`和`SingleThreadExecutor`这三个类中

* NioEventLoop

  ```java
    NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
      //继承链。。。
      super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
      if (selectorProvider == null) {
          throw new NullPointerException("selectorProvider");
      }
      if (strategy == null) {
          throw new NullPointerException("selectStrategy");
      }
      provider = selectorProvider;
      //TODO:优化
      final SelectorTuple selectorTuple = openSelector();
      //TODO:创建selector
      selector = selectorTuple.selector; //创建selector
      unwrappedSelector = selectorTuple.unwrappedSelector;
      selectStrategy = strategy;
    }
  ```

  * 【11】 这个`selectorProvider`是在`NioEventLoopGroup`初始化中由SelectorProvider.provider() （一个静态方法 用于提高效率）。
    
  * 【12】优化selector的创建


* SingleThreadExecutor

  ```java
  protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, int maxPendingTasks,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    this.executor = ObjectUtil.checkNotNull(executor, "executor");
    taskQueue = newTaskQueue(this.maxPendingTasks);
    //拒绝策略
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
  }
  ```
    * 【8】内部是一个LinkedBlockingQueue队列，LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。。

### 3.创建线程事件执行器的选择器chooser

```java
chooser = chooserFactory.newChooser(children);
```
线程选择器创建好的NioEventLoop进行选择，给新来的连接绑定NioEventLoop，循环绑定,此处的选择方式netty进行了优化

```java
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {//长度是否是2的幂
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```
```java
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    ...
    @Override
    public EventExecutor next() {
        //  二进制与运算。效率更高
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
}
```
```java
private static final class GenericEventExecutorChooser implements EventExecutorChooser {
    ...
    @Override
    public EventExecutor next() {
       //基数取模运算，
        return executors[Math.abs(idx.getAndIncrement() % executors.length)];
    }
}
```