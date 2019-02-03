---
layout:     post
title:      "4-1.Netty源码:新连接检测"
subtitle:   "新连接检测"
date:        2019-02-03
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---


使用telnet 连接服务端的demo，充当客户端的连接。
`telnet 127.0.0.1 8888`

先读取客户端channelHander  -> AbstractNioMessageChannel#read ->再读取服务端channelHandler

在`NioEventLoop# processSelectedKey(SelectionKey k, AbstractNioChannel ch)`继续` unsafe.read();`

新连接其实有两大部分 
* 第一部分获取新连接返回jdk底层的一个SocketChannel，并且将这个channel包装成netty自己的NioSocketChannel
* 第二部分就是 注册 并且传播handler事件

> 服务端负责获取新连接并且处理。

```java
//AbstractNioMessageChannel.java
  @Override
  public void  read() {
    ...
    boolean closed = false;
    Throwable exception = null;
    try {
          do {
              int localRead = doReadMessages(readBuf);
              if (localRead == 0) {
                  break;
              }
              if (localRead < 0) {
                  closed = true;
                  break;
              }

              allocHandle.incMessagesRead(localRead);
          } while (allocHandle.continueReading());
          ...
    //...新连接注册
    }
    ...
  }
}
```
* 【9】获取新连接
 
  ```java
  //AbstractNioMessageChannel.java
  @Override
  protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());
  
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } 
    return 0;
  }
  ```
  * 【3】 创建jdk的channel，通过ServerSocketChannel.accept()方法监听新进来的连接。当accept()方法返回的时候，它返回一个包含新进来的连接的SocketChannel。
  * 【8】 `new NioSocketChannel(this, ch)`将新连接构造成netty自己的实体`NioSocketChannel`，第一个参数是来自服务端`NioServerSocketChannel`,具体初始化方法见[【4-1-1.Netty源码:新连接NioSocketChannel 初始化】](http://jinlipool.com/2019/02/03/netty-4-1-1-NioSocketChannel-construct/)
* 【19】实现方法`DefaultMaxMessagesRecvByteBufAllocator#continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier)`每次读取连接数不能超过默认的16个连接
* 【21】新连接注册[【4-2.Netty源码:NioSocketChannel的线程分配和selector注册】](http://jinlipool.com/2019/02/03/netty-4-2-NioSocketChannel-register/)