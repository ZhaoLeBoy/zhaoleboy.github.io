---
layout:     post
title:      "3-1.Netty源码:NioEventLoop启动"
subtitle:   "NioEventLoop启动"
date:        2019-01-30
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

```java
//AbstractBootStrap.java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
  channel.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
      if (regFuture.isSuccess()) {
          channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
      } else {
          promise.setFailure(regFuture.cause());
      }
    }
  });
}
```
把绑定的方法包装成一个任务，放到事件执行器 中执行。具体是在`SingleThreadEventExecutor#execute`中实现
具体可以看[【2-1-3.Netty源码:端口绑定】](http://jinlipool.com/2019/01/29/netty-2-1-3-server-socket-bind/)，已经介绍了bind()方法具体做的是什么。

```java
 @Override
  public void execute(Runnable task) {
    ...
    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
      startThread();
      if (isShutdown() && removeTask(task)) {
          reject();
      }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
  }
```
* 【4】判断当前线程是否是eventLoop线程，
* 【5】将任务放到任务队列，具体是`taskQueue.offer(task)`，详细可见[【3-4.Netty源码:任务和定时任务队列】](http://jinlipool.com/2019/01/30/netty-3-4-task.md/)
* 【7】开启线程
  
  ```java
  private void startThread() {
    if (state == ST_NOT_STARTED) {
      if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
        try {
            doStartThread();
        } 
            ...
      }
    }
  }
  ```
  
  ```java
  private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
      @Override
      public void run() {
        ...
        boolean success = false;
        ...
        try {
            SingleThreadEventExecutor.this.run();
            success = true;
        } 
         ...
      }
    });
  }
  ```
  * 【2】有个断言。线程不能为空
  * 【3】是调用`DefaultThreadFactory#execute(Runnable`方法新开启一个线程并且启动，netty将Runnable封装成`FastThreadLocalRunnable`，并且将这个线程封装`FastThreadLocalThread`后启动
  * 【10】线程执行。具体实现NioEventLoop#run，具体实现方法见下一篇文章。
  
主线程会调用bind方法 bind方法会作为一个task被放到任务队列里，然后调用服务端execute的方法，nett有会判断调用的方法不是在NioEventLoop线程中，然后调用`doStartThread`方法创建一个线程，并保存下来 最后调用`  SingleThreadEventExecutor.this.run();`进行启动