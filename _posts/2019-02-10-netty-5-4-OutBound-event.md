---
layout:     post
title:      "5-4.Netty源码:OutBound事件传播"
subtitle:   "OutBound事件传播"
date:        2019-02-10
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---

逻辑跟InBound类似。

首先自定义三个outBound处理器
```java
//OutBoundA
public class OutBoundA extends ChannelOutboundHandlerAdapter {
  @Override
  public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      System.out.println("OutBoundA " + msg);
      ctx.write(msg, promise);
  }
  
  @Override
  public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
      ctx.executor().schedule(() -> {
      //模拟服务端给客户端写的数据
          ctx.channel().pipeline().write("write ing");
      }, 5, TimeUnit.SECONDS);
  }
}

//OutBoundB
public class OutBoundB extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundB " + msg);
        ctx.write(msg, promise);
    }
}

//OutBoundC
public class OutBoundC extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundC " + msg);
        ctx.write(msg, promise);
    }
}
```
OutBound事件是按照ChannelHandler的反向进行执行的。
![IMAGE](/img/netty/5-4/1.jpg)
从tail节点的`write`开始，使用debug方式启动服务器端，然后在使用客户端连接(下面代码有删减，只保留传播流程中部分代码）。

当有新连接来到的时候，OutBoundA`handlerAdded`方法触发，事件从调用`write`开始传播。

```java
//DefaultChannelPipeline.java
public final ChannelFuture write(Object msg) {
  return tail.write(msg);
}
```
* 【3】注意这个tail说明事件是从tail开始传播的。tail则是在创建pipeline时候已经创建好了

```java
//AbstractChannelHandlerContext.java
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}

public ChannelFuture write(final Object msg, final ChannelPromise promise) {
  write(msg, false, promise);
  return promise;
}
```

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    next.invokeWrite(m, promise);
}
```
* 【2】while循环找之前一个是OutBound的节点并且返回
  ```java
    private AbstractChannelHandlerContext findContextOutbound() {
      AbstractChannelHandlerContext ctx = this;
      do {
          ctx = ctx.prev;
      } while (!ctx.outbound);
      return ctx;
    }
  ```

```java
//AbstractChannelHandlerContext.java
private void invokeWrite(Object msg, ChannelPromise promise) {
  invokeWrite0(msg, promise);
}

private void invokeWrite0(Object msg, ChannelPromise promise) {
  ((ChannelOutboundHandler) handler()).write(this, msg, promise);
}
```
  调用到服务器端中`OutBoundC#write()`然后在方法内部继续调用`ctx.write(msg, promise)`向上传播，继续调用上述的几个方法直到进去head节点。
需要说明的是`ctx.channel().pipeline().write()`是从tail节点向上进行事件传播
`ctx.write()`是从当前节点向上进行事件传播。

一直在最后到head的节点。会调用`HeadContext`中的write方法
```java
//DefaultChannelPipeline.java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
  unsafe.write(msg, promise);
}
```
如果一直没有找到有用的handler，就会在head节点调用这个方法`unsafe.write(msg, promise);`，最后msg是实现`ReferenceCounted`接口的消息类型类型(也就是ByteBuf)，那么传到tail节点后会被释放掉(引用计数减一)。