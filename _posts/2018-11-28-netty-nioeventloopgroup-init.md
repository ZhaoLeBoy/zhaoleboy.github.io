---
layout:     post
title:      "一.Netty源码:NioEventLoopGroup初始化"
subtitle:   "netty源码学习"
date:       2018-11-28 3:00:00
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
tags:
    - Netty
---

先上NioEventLoopGroup的继承关系图，有删减
![NioEventLoopGroup.png][image-1]

## 初始化开始
一般来说Netty启动的时候，启动的时候会实例化两个NioEventLoopGroup，一个是boss，一个是worker,
boss是用来接受新的链接，worker是用来处理新链接的网络读写

```java
//实例化参数是线程数，不传的化默认是cpu核数*2
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workGroup = new NioEventLoopGroup();
```
NioEventLoopGroup初始化 做了三件事
* 1-1.  构造一个线程创建器`ThreadPerTaskExecutor`
* 1-2.  根据线程数和线程创建器`ThreadPerTaskExecutor`构造多个事件执行器`EventExecutor`数组。
* 1-3.  创建线程事件执行器的选择器chooser，类型是`EventExecutorChooserFactory.EventExecutorChooser`


### 1-1  构造线程创建器`ThreadPerTaskExecutor`
```java
     executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
```
实例化实现主要是在`newDefaultThreadFactory()`

```java
 public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
        ...
        //给这个线程执行器命名`eventLoopGroup_` +自增器
        prefix = poolName + '-' + poolId.incrementAndGet() + '-'; 
        ...
    }
```

### 1-2  创建事件执行期`EventExecutor`

```java
  //  具体实现是在`NioEventLoop`中实现
  children[i] = newChild(executor, args);
```
NioEventLoop继承关系如下
![NioEventLoop.png][image-2]

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
          //是一个LinkedBlockingQueue队列
          //LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。
          taskQueue = newTaskQueue(this.maxPendingTasks);
          //拒绝策略
          rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
      }
  ```

### 1.3  创建线程事件执行器的选择器chooser
```java
chooser = chooserFactory.newChooser(children);
```
线层选择器创建好的NioEventLoop进行选择，给新来的连接绑定NioEventLoop，循环绑定,此处的选择方式netty进行了优化

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

[image-1]: /img/netty/NioEventLoopGroup.png
[image-2]: /img/netty/NioEventLoop.png