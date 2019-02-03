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
* **[【2-1.Netty源码:服务端启动】](http://jinlipool.com/2019/01/29/netty-2-1-ServerStart/)**
  * **[【2-1-1.Netty源码:初始化服务端的channel】](http://jinlipool.com/2019/01/29/netty-2-1-1-server-channel-initialization/)**
  * **[【2-1-2.Netty源码:注册服务端channel】](http://jinlipool.com/2019/01/29/netty-2-1-2-server-channel-register/)**
  * **[【2-1-3.Netty源码:端口绑定】](http://jinlipool.com/2019/01/29/netty-2-1-3-server-socket-bind/)**

#### 三.EventLoop
* **[【3-1.Netty源码:NioEventLoop启动】](http://jinlipool.com/2019/01/30/netty-3-1-evetloop-start/)**
* **[【3-2.Netty源码:NioEventLoop循环检测事件】](http://jinlipool.com/2019/01/30/netty-3-2-evetloop-event-detection/)**
  * **[【3-2-1.Netty源码:避免空轮循的bug】](http://jinlipool.com/2019/01/31/netty-3-2-1-avoid-epoll-bug)**
* **[【3-3.Netty源码:NioEventLoop处理事件】](http://jinlipool.com/2019/01/30/netty-3-3-handle-event/)**
* **[【3-4.Netty源码:任务和定时任务队列】](http://jinlipool.com/2019/01/30/netty-3-4-task.md/)**

#### 四.新连接接入
* **[【4-1.Netty源码:新连接检测】](http://jinlipool.com/2019/02/03/netty-4-1-connection-detection/)**
  * **[【4-1-1.Netty源码:新连接NioSocketChannel 初始化】](http://jinlipool.com/2019/02/03/netty-4-1-1-NioSocketChannel-construct/)**
* **[【4-2.Netty源码:NioSocketChannel的线程分配和selector注册】](http://jinlipool.com/2019/02/03/netty-4-2-NioSocketChannel-register/)**
* **[【4-3.Netty源码:NioSocketChannel读事件注册】](http://jinlipool.com/2019/02/03/netty-4-3-NioSocketChannel-read-event-register/)**


#### 扩展
  * **[【Netty源码:ChannelFuture】](http://jinlipool.com/2019/01/28/netty-extend-1-ChannelFuture/)**