---
layout:     post
title:      "2-1-2.Netty源码:注册服务端channel"
subtitle:   "注册服务端channel"
date:        2019-01-29
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

将新创建和初始化完毕的channel注册到selector中

```java
ChannelFuture regFuture = config().group().register(channel);
```
` config().group()`对应的类型是bossGroup类型，在MultithreadEventLoopGroup中有个`next().register(channel)`
next的方法具体实现是`chooser.next();`见【1-1.Netty源码:NioEventLoopGroup构造方法#1-1-3】
就是使用线程选择器`chooser`选定了一个NioEventLoop线程进行后面的注册操作

具体是在`AbstractChannel#register`中实现的
```java
//SingleThreadEventLoop.java->AbstractChannel.java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
  ...
  AbstractChannel.this.eventLoop = eventLoop;
  
  if (eventLoop.inEventLoop()) {
    //真正注册地方
    register0(promise);
  } else {
    try {
      eventLoop.execute(new Runnable() {
        @Override
        public void run() {
          register0(promise);
        }
      });
    } catch (Throwable t) {
        logger.warn(
        ...
    }
  }
}

```
* 【3】将当前线绑定为evenloop线程
* 【5-8】如果是在当前线程，就直接执行方法否则就计入到任务队列中，等时间循环执行。实际注册事件如下

  ```java
  private void register0(ChannelPromise promise) {
    try {
      ...
      boolean firstRegistration = neverRegistered;
      doRegister();
      neverRegistered = false;
      registered = true;
      //传播新增handle事件
      pipeline.invokeHandlerAddedIfNeeded();

      safeSetSuccess(promise);
      //传播已经注册成功事件。
      if (isActive()) {
        if (firstRegistration) {
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();
        }
      }
    } catch (Throwable t) {
        ...
    }
  }
  ```
  * 【5】在`AbstractNioChannel#doRegister`具体实现，使用底层jdk注册，并返回对应的selectionKey
    
     ```java
    //底层jdk调用  服务端channel(NioServerSocketChannel)绑定到selector上并且返回 selectionKey
    selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this); 
    ```
  * 【15】第一次进来时候返回的是false 因为最初只是用来将NioServerSocketChannel线程绑定到selector上，并没有进行实际的端口绑定,正真调用fireChannelActive方法是在`DefaultChannelPipeline#channelActive`中。具体方法见【2-1-3.Netty源码:端口绑定】
*  【10-15】如果在当前调用方法的的线程与SingleThreadEventExecutor所维护的线程不是同一个，就把这个注册以任务的形式提交给eventloop中的线程执行。

初始化channel并注册已经全部说完，后面有一段从《netty实战中》引入的话，也是非常重要的：
>1. 一个 EventLoopGroup 包含一个或者多个 EventLoop;
>2. 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定;
>3. 所有由 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理;
>4. 一个 Channel 在它的生命周期内只注册于一个 EventLoop;
>5. 一个 EventLoop 可能会被分配给一个或多个 Channel。
  