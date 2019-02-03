---
layout:     post
title:      "4-2.Netty源码:NioSocketChannel的线程分配和selector注册"
subtitle:   "NioSocketChannel的线程分配和selector注册"
date:        2019-02-03
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---

具体实现是在`AbstractNioMessageChannel#read()`

> 服务端负责获取新连接并且处理。所以实现方法是在AbstractNioMessageChannel中

```java
  @Override
  public void read() {
    try {
     // ...新连接检测+封装NioSocketChannel

      int size = readBuf.size();
      for (int i = 0; i < size; i ++) {
          readPending = false;
          pipeline.fireChannelRead(readBuf.get(i));
      }
    ...//TODO.
  }
}
```
* 【4】见[【4-1-1.Netty源码:新连接NioSocketChannel 初始化】](http://jinlipool.com/2019/02/03/netty-4-1-1-NioSocketChannel-construct/)
* 【7-10]具体实现是在ServerBootStrap中的`channelRead(ChannelHandlerContext ctx, Object msg)`
   
    ```java
    //ServerBootStrap.java
     public void channelRead(ChannelHandlerContext ctx, Object msg) {
      final Channel child = (Channel) msg;

      child.pipeline().addLast(childHandler);

      setChannelOptions(child, childOptions, logger);

      for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
      }

      try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
          @Override
          public void operationComplete(ChannelFuture future) throws Exception {
            if (!future.isSuccess()) {
                forceClose(child, future.cause());
            }
          }
        });
      }
      ...
    }
   ```

  > 方法中的`childHandler`,`childAttrs`,`childOptions`,`childGroup`的这些值是来自于(NioServerSocketChannel初始化的时候,服务端channel的pipline上增加了`ServerBootstrapAcceptor`适配器，这些变量被包装在其中。见【2-1-2.Netty源码:注册服务端channel】，然后逐层传播到现在的channelRead方法中
  
  * 【4】给新连接添加childHandler，这里添加的childHandler是服务端demo启动的时候childHandler中的`ChannelInitializer`,这个里面是用户自己定义的handler集合
  * 【6-10】给新连接设置childOptions和childAttrs
  * 【13】`childGroup`中选择一个NioEventLoop并且注册
  
    ```java
    //MultithreadEventLoopGroup.class
    public EventLoop next() {
      return (EventLoop) super.next();
    }

    public ChannelFuture register(Channel channel) {
        //选用事件执行器中的一个线程
        return next().register(channel);
    }
    ```
    
    ```java
    //MultithreadEventExecutorGroup.class
     public EventExecutor next() {
      return chooser.next();
    }
    ```
    这个`chooser`是NioEventLoopGroup初始化时候构建的，见【1-1.Netty源码:NioEventLoopGroup构造方法】，通过chooser拿到NioEventLoop并且进行注册，注册方法见[【2-1-2.Netty源码:注册服务端channel】](http://jinlipool.com/2019/01/29/netty-2-1-2-server-channel-register/),跟服务器端注册类似。
    
    ```java
     private void register0(ChannelPromise promise) {
      try {
        ....
        //真正注册地方 channel注册到selector
        doRegister();
        neverRegistered = false;
        registered = true;
        ...
      } 
    }
    ```
  