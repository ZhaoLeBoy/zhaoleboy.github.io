---
layout:     post
title:      "4-3.Netty源码:NioSocketChannel读事件注册"
subtitle:   "NioSocketChannel读事件注册"
date:        2019-02-03
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---

客户端的读事件注册是紧接着【4.2.Netty源码:NioSocketChannel的线程分配和selector注册】后面

```java
//AbstractChanne.java
 private void register0(ChannelPromise promise) {
      try {
          ...
          //调用jdk底层 接口是否绑定了。当服务端启动的时候，isActive()返回false
          if (isActive()) {
              if (firstRegistration) {
                  pipeline.fireChannelActive();
              } else if (config().isAutoRead()) {
                  beginRead();
              }
          }
      } catch (Throwable t) {
          // Close the channel directly to avoid FD leak.
          closeForcibly();
          closeFuture.setClosed();
          safeSetFailure(promise, t);
      }
  }
```

* 【8】扩展说明，如下

    ```java
      public void channelActive(ChannelHandlerContext ctx) throws Exception {
        //向下传播事件了
        ctx.fireChannelActive();
        readIfIsAutoRead();
      }
    ```
  * 【4】继续深入会发现是从tail节点往前传播，再到客户端连接（`NioSocketChannel`）的`unsafe.beginRead()`，最终实现方法是在`AbstractNioChannel#doBeginRead`
  
    ```java
       //AbstractNioChannel.java
       protected void doBeginRead() throws Exception {
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }
  
        readPending = true;
  
        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
      }
    ```
    将读事件绑定到selector上。下次如果有数据进来就可以进行数据读写了。此时的`interestOps`是读事件。再构造NioSocketChannel的时候默认定义的.
    ```java
      protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
        super(parent, ch, SelectionKey.OP_READ);
    }
    ```