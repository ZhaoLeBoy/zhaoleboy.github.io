---
layout:     post
title:      "Netty源码阅读-目录"
subtitle:   "目录"
date:        2019-01-27
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


## 一.Nio相关
* [BIO与NIO、AIO的区别(这个容易理解)](https://blog.csdn.net/skiof007/article/details/52873421) 
这篇文章很清楚的描述几种IO的区别，也很清楚说明了异步，同步，堵塞，非堵塞的基本概念
* [Scalable IO in Java》学习笔记](http://jinlipool.com/2018/12/29/scalable-io-in-java)

## 二.Netty源码相关

> 一个netty的服务端的例子

```java
public static void main(String[] args) throws InterruptedException {
  EventLoopGroup bossGroup = new NioEventLoopGroup();
  EventLoopGroup wordGroup = new NioEventLoopGroup();
  try {
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup, wordGroup).channel(NioServerSocketChannel.class)
      .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
          ChannelPipeline pipeline = ch.pipeline();
          ...
        }
      });

    ChannelFuture channelFuture = serverBootstrap.bind(8899).sync();
    channelFuture.channel().closeFuture().sync();
  } finally {
    bossGroup.shutdownGracefully();
    wordGroup.shutdownGracefully();
  }
}
```
无论写简单的还是复杂的netty程序，服务端程序都有以下三个步骤：
1. 定义两个事件循环组，一般来说是NioEventLoopGroup。
2. 然后使用Netty定义好的服务启动器ServerBootStrap。
3. 最后在childHandler中定义ChannelInitializer这个类，这个类里面定义的跟我们请求流程有关的handler。

Netty开发基本就是这个模板套路,后面的对源码的理解和分析也是基于这个demo的，会有一些扩展的解读文章会在后续补上。


#### 一.基本构造方法
* **[【1-1.Netty源码:NioEventLoopGroup构造方法】](http://jinlipool.com/2019/01/27/netty-1-1-NioEventLoopGroup-construct/)**
  * **[【1-1-1.Netty源码:线程执行器ThreadPerTaskExecutor】](http://jinlipool.com/2019/01/27/netty-1-1-1-ThreadPerTaskExecutor/)**
* **[【1-2.Netty源码:构建ServerBootstrap以及它的调用链】](http://jinlipool.com/2019/01/27/netty-1-2-ServerBootstrap-construct/)**
  * **[【1-2-1.Netty源码:服务端NioServerSocketChanne构造方法】](http://jinlipool.com/2019/01/27/netty-1-2-1-NioServerSocketChannel-construct/)**

#### 二.服务端启动