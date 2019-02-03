---
layout:     post
title:      "Netty源码:ChannelPipeline "
subtitle:   "ChannelPipeline"
date:        2019-02-03
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

对Netty中的ChannelPipeline的文档大概翻译一下.

ChannelPipeline本质上是个ChannelHandler的集合，它会处理或者拦截Channel的入栈事件以及出栈的操作。ChannelPipeline实现了一种高级拦截过滤器模式
使用户可以完全控制事件的处理方式以及管道中的ChannelHandler如何相互交互。

### 管道创建
每个channel都有自己pipeline，它会在channel创建的时候创建。

### pupeline中事件流
下图展示了io事件如何被channelPipline中的若干个channelHandler处理的(pipeline相当于多个channelhandler的容器)，一个事件要不然被ChannelInboundHandler处理要不然ChannelOutboundHandler处理。处理完后会被事件传播给与之相近的下一个handler，这个是由ChannelHandlerContext 方法处理的。
 
![IMAGE](/img/netty/extend/channelPipeline/1.jpg)
 
 一个入栈的事件被入栈处理器自下向上进行处理的。入栈处理器通常处理由io线程生成的入栈数据， 入栈数据通常会从远端输入操作读取，例如SocketChannel#read。 如果入栈事件超出了上边界(见上图) ，就会被丢弃。
 一个出栈事件被出栈处理器 自上向下进行处理的。出栈处理器通常生成或者传输出出栈的数据，例如写请求，如果出栈事件超过最下面的边界(见上图)，就被与这个Channel相关的I/O线程给处理了。
I/O线程经常执行实际的输出操作，例如`SocketChannel#write(ByteBuffer)`

```java
 ChannelPipeline p = ...;
  p.addLast("1", new InboundHandlerA());
  p.addLast("2", new InboundHandlerB());
  p.addLast("3", new OutboundHandlerA());
  p.addLast("4", new OutboundHandlerB());
  p.addLast("5", new InboundOutboundHandlerX());
```
Inbound开头的是入栈处理器，Outbound开头的是出栈处理器，
ChannelPipeline会忽略某些处理器来减少栈的深度。
入栈时候事件处理是顺序1，2，5 
出栈时候事件处理顺序是5，4，3。

### 将事件转发给下个处理器
如上图，一个处理器需要调用`ChannelHandlerContext`类中的事件传播方法将事件转发到下个处理器。

```java
入栈事件传播方法
ChannelHandlerContext#fireChannelRegistered()
ChannelHandlerContext#fireChannelActive()
ChannelHandlerContext#fireChannelRead(Object)
ChannelHandlerContext#fireChannelReadComplete()
ChannelHandlerContext#fireExceptionCaught(Throwable)
ChannelHandlerContext#fireUserEventTriggered(Object)
ChannelHandlerContext#fireChannelWritabilityChanged()
ChannelHandlerContext#fireChannelInactive()
ChannelHandlerContext#fireChannelUnregistered()
出栈事件传播方法
ChannelHandlerContext#bind(SocketAddress, ChannelPromise)
ChannelHandlerContext#connect(SocketAddress, SocketAddress, ChannelPromise)
ChannelHandlerContext#write(Object, ChannelPromise)
ChannelHandlerContext#flush()
ChannelHandlerContext#read()
ChannelHandlerContext#disconnect(ChannelPromise)
ChannelHandlerContext#close(ChannelPromise)  
ChannelHandlerContext#deregister(ChannelPromise)
```

### 构建pipeline
  用户在在一个pipline中有多个ChannelHandler去接受io事件 (e.g. read) 去处理请求io操作。如 写和关闭一个典型的服务器在每个channel的pipline中会有如下处理器 但你可能根据实现的复杂度和协议的不同有些区别:
  
1. 协议解码器，二级制数据转换java对象
2. 协议编码器，将对象转换二进制__
3. 业务逻辑处理器，处理实际的业务逻辑，比如数据库访问
    
  ```java
   static final link EventExecutorGroup group = new  DefaultEventExecutorGroup(16);
  ...
 
  ChannelPipeline pipeline = ch.pipeline();
 
  pipeline.addLast("decoder", new MyProtocolDecoder());
  pipeline.addLast("encoder", new MyProtocolEncoder());
  //告诉pipeline运行MyBusinessLogicHandler的事件处理方法在不同的线程，这样io线程就不会被一个耗时任务给堵塞了。
  //如果你的业务逻辑是异步或者完成非常快，就可以不必要指定group
  //还有一种方法就是在MyBusinessLogicHandler业务里面用线程池进行业务处理，也可以避免io线程堵塞
  pipeline.addLast(group, "handler", new MyBusinessLogicHandler());
  ```
  
### 线程安全
一个ChannelHandler可以再任何时候被添加和删除，因为ChannelPupeline是线程安全的。举个栗子，当敏感数据被交换，你可以插入一个加密处理器，在交换完毕后可以删除。