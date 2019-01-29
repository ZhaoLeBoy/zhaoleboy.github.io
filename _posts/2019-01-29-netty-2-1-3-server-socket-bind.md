---
layout:     post
title:      "2-1-3.Netty源码:端口绑定"
subtitle:   "端口绑定"
date:        2019-01-29
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

端口绑定

```java
//AbstractBootstrap.java
 private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

  //把绑定接口的方法做成一个Task 扔到事件执行器 中执行
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
* 【11】具体实现绑定方法在AbstractChannel#bind() 
  ```java
   @Override
  public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
      assertEventLoop();
  
      if (!promise.setUncancellable() || !ensureOpen(promise)) {
          return;
      }
      ...
      //false
      boolean wasActive = isActive();
      try {
          doBind(localAddress);
      } catch (Throwable t) {
        ...
      }
  
      if (!wasActive && isActive()) {
          invokeLater(new Runnable() {
              @Override
              public void run() {
                  pipeline.fireChannelActive();
              }
          });
      }
  
      safeSetSuccess(promise);
  }
  ```
  * 【12】底层是使用了jdk底层进行绑定
  * 【17】判断是否绑定了`isactive`方法也是用jdk地层实现的 `javaChannel().socket().isBound();`
  * 【21】完成绑定后发送事件
  
      ```java
      public void channelActive(ChannelHandlerContext ctx) throws Exception {
          ctx.fireChannelActive();
          readIfIsAutoRead();
      }
      ```
      * 【2】传播channel激活事件
      * 【3】层层的调用最终实现方式是在`AbstractNioChannel#doBeginRead`
      
        ```java
        //AbstractNioChannel.java
         protected void doBeginRead() throws Exception {
          final SelectionKey selectionKey = this.selectionKey;
          if (!selectionKey.isValid()) {
              return;
          }
          ...
  
          final int interestOps = selectionKey.interestOps();
          if ((interestOps & readInterestOp) == 0) {
              selectionKey.interestOps(interestOps | readInterestOp);
          }
        }
        ```
        获取服务端channel注册到selector上的selectionKey，然后获取这个key的interestOps
        然后判断这个interestOps是否是0（SelectionKey.OP_ACCEPT，具体请见[【1-2-1.Netty源码:服务端NioServerSocketChanne构造方法】](http://jinlipool.com/2019/01/27/netty-1-2-1-NioServerSocketChannel-construct/#2-%E7%BB%A7%E6%89%BF%E7%88%B6%E7%B1%BB%E8%AE%BE%E7%BD%AEchannel%E7%9B%B8%E5%85%B3%E9%85%8D%E7%BD%AE)）如果等于0 做个或操绑定Accept事件，再重新绑定到selectionKey。
    
  
            