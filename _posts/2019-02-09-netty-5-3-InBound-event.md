---
layout:     post
title:      "5-3.Netty源码:InBound事件传播"
subtitle:   "InBound事件传播"
date:        2019-02-09
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---


首先自定义三个inbound处理器
```java
//InBoundA
public class InBoundA extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("In Bound A" + msg);
        ctx.fireChannelRead(msg);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().pipeline().fireChannelRead("Test");
    }
}
//InBoundB
public class InBoundB extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("In Bound B" + msg);
        ctx.fireChannelRead(msg);
    }

}
//InBoundC
public class InBoundC extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("In Bound C" + msg);
        ctx.fireChannelRead(msg);
    }
}
```
InBound事件是按照ChannelHandler的顺序进行执行的。
![IMAGE](/img/netty/5-3/1.jpg)
从head节点的`channelRead`开始，使用debug方式启动服务器端，然后在使用客户端连接（下面代码有删减，只保留传播流程中部分代码）。

当有新连接来到的时候，InBoundA中的`channelActive`方法触发，事件从调用`fireChannelRead`开始传播。

```java
//DefaultChannelPipeline.java
public final ChannelPipeline fireChannelRead(Object msg) {
  AbstractChannelHandlerContext.invokeChannelRead(head, msg);
  return this;
}
```
* 【3】注意这个head说明事件是从head开始传播的。head则是在创建pipeline时候已经创建好了。

```java
//AbstractChannelHandlerContext.java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
  next.invokeChannelRead(m);
}

private void invokeChannelRead(Object msg) {
  ((ChannelInboundHandler) handler()).channelRead(this, msg);
}

//DefaultChannelPipeline.java
 @Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.fireChannelRead(msg);
}
```
```java
//AbstractChannelHandlerContext.java
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}
```
  *  `findContextInbound()`
      ```java
         private AbstractChannelHandlerContext findContextInbound() {
          AbstractChannelHandlerContext ctx = this;
          do {
              ctx = ctx.next;
          } while (!ctx.inbound);
          return ctx;
      }
      ```
      while循环，去找下一个Inbound节点

```java
//AbstractChannelHandlerContext.java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
  next.invokeChannelRead(m)    
}

private void invokeChannelRead(Object msg) {
  ((ChannelInboundHandler) handler()).channelRead(this, msg);
}
```
调用到服务器端中`InBoundA#channelRead()`然后在方法内部继续调用`ctx.fireChannelRead(msg);`向下传播，继续调用上述的几个方法直到进去tail节点。
需要说明的是`ctx.channel().pipeline().fireChannelRead();`是从head节点向下进行事件传播
`ctx.fireChannelRead();`是从当前节点向下进行事件传播。

一直在最后到Tail的节点。会调用`TailContext`中的ChannelRead方法
```java
//DefaultChannelPipeline.java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    onUnhandledInboundMessage(msg);
}

//DefaultChannelPipeline.java
protected void onUnhandledInboundMessage(Object msg) {
  ReferenceCountUtil.release(msg);
}
```
【9】如果msg是实现`ReferenceCounted`接口的消息类型类型(也就是ByteBuf)，那么传到tail节点后会被释放掉(引用计数减一)。