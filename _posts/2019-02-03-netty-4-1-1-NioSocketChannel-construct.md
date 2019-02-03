---
layout:     post
title:      "4-1-1.Netty源码:新连接NioSocketChannel初始化"
subtitle:   "新连接NioSocketChannel初始化"
date:        2019-02-03
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---

NioSocketChannel是基于NIO选择器的实现。

NioSocketChannel继承关系(有删减)
![IMAGE](/img/netty/4-1-1/1.jpg)

```java
public NioSocketChannel(Channel parent, SocketChannel socket) {
  super(parent, socket);
  config = new NioSocketChannelConfig(this, socket.socket());
}
```
    
* config中禁止了Nagle算法(Nagle算法:通过减少必须发送包的个数来增加网络软件系统的效率。)

初始化初始化的时候根据上图的循序依次调用父类。
* AbstractNioByteChannel
```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
  super(parent, ch, SelectionKey.OP_READ);
}
```
* 【2】新连接接入，`SelectionKey.OP_READ` 读事件

* AbstractNioChannel
```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
  super(parent);
  this.ch = ch;
  this.readInterestOp = readInterestOp;
  ...
      ch.configureBlocking(false);
  ...
}
```
  * 【3】客户端的channel，jdk底层产生的channel`ServerSocketChannel.accpet()`
  * 【4】新连接接入的话,事件就是OP_READ
  * 【6】设置非堵塞

* AbstractChannel
```java
protected AbstractChannel(Channel parent) {
  this.parent = parent;
  id = newId();
  unsafe = newUnsafe();
  pipeline = newChannelPipeline();
}
```
  * 【2】创建这个客户端channel的服务端channel`NioServerSocketChannel`

> 大部分初始化的方式跟NioServerSocketChannel类似

![IMAGE](/img/netty/4-1-1/2.jpg)