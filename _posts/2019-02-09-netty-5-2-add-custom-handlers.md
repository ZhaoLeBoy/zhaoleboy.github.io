---
layout:     post
title:      "5-2.Netty源码 新增用户自定义ChannelHandler"
subtitle:   "新增用户自定义ChannelHandler"
date:        2019-02-09
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---
通过一个例子来展开说明
```java
 public class MyChannelInitializer extends ChannelInitializer {
    public void initChannel(Channel channel) {
        channel.pipeline().addLast("myHandler", new MyHandler());
    }
 }
```
### addLast()
用户一般使用addLast()方法增加自定义handler
```java
//DefaultChannelPipeLine.java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
  final AbstractChannelHandlerContext newCtx;
  synchronized (this) {
  
    checkMultiplicity(handler);
    
    newCtx = newContext(group, filterName(name, handler), handler);
    
    addLast0(newCtx);

    if (!registered) {
        newCtx.setAddPending();
        callHandlerCallbackLater(newCtx, true);
        return this;
    }

    EventExecutor executor = newCtx.executor();
    if (!executor.inEventLoop()) {
      newCtx.setAddPending();
      executor.execute(new Runnable() {
          @Override
          public void run() {
              callHandlerAdded0(newCtx);
          }
      });
      return this;
    }
  }
  callHandlerAdded0(newCtx);
  return this;
}
```
* 【4】检查是否重复添加handle
    ```java
      private static void checkMultiplicity(ChannelHandler handler) {
        if (handler instanceof ChannelHandlerAdapter) {
            ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
            if (!h.isSharable() && h.added) {
                throw new ChannelPipelineException(。。。);
            }
            h.added = true;
        }
    }
    ```
    判断用户添加的ChannelHandler是否重复添加的最直接方法就是判断是否非共享(被Sharable注解标注)并且被添加过就抛出重复的异常。否则继续后面步骤并且标记添加状态为true
*  【8】 为这个ChannelHandler构建一个ChannelHandlerContext，一个ChannelHandler对应一个ChannelHandleContext
    
    ```java
    //DefaultChannelHandlerContext.java
    DefaultChannelHandlerContext(
        DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
      super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
      ...
      this.handler = handler;
    }
    
    ```
    * 【4】在`isInbound(handler)`, `isOutbound(handler)`来判断是入栈处理器还是出栈处理器，然后作为参数传入。super方法比较简单，就是在`DefaultChannelHandlerContext`类中调用父类的构造方法。构造方法里面给一些成员变量赋值，
  
* 【10】添加到链表
  ```java
  //DefaultChannelPipeline.java
  private void addLast0(AbstractChannelHandlerContext newCtx) {
      AbstractChannelHandlerContext prev = tail.prev;
      newCtx.prev = prev;
      newCtx.next = tail;
      prev.next = newCtx;
      tail.prev = newCtx;
  }
  ```
  * 标准的双向链表操作。把新创建的AbstractChannelHandlerContext对象加入到链表中。方法虽然叫addLast，但是tail永远是在最后一个节点，新加入的节点都会放在tail节点的前一个位置。
  
* 【30】用户自定编写的handler对象有个`handlerAdded`方法，所以这个顺序是，Pipeline创建后，生成AbstractChannelHandlerContext`并且添加到Pipeline中，最后这个回调方法调用(第一个被调用到的回调方法)，将handler对象添加到`AbstractChannelHandlerContext`这个对象中

  ```java
    private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
      ctx.setAddComplete();
      ctx.handler().handlerAdded(ctx);
      ...
    }
  ```
  
Pipeline是一个容器，它里面存放多个handler，更准确的说法是Pipeline是一个容器，它里面存放的是多个channelHandlerContext对象，而channelHandlerContext里面是对应的channelHandler对象，chanelHandlerContext是netty帮你创建的。
    
---
### ChannelInitializer

channel被注册的时候`initChannel`被调用，当这个方法返回的时候，这个实例会从Channel的ChannelPipeline中移除.
删除的原因是：无论是用户自己定义的类去继承`ChannelInitializer`，还是直接实例化出来的`ChannelInitializer`对象。主要的作用是将多个正真有用的多个处理器一次性添加到Pipeline中。所以`ChannelInitializer`对象本身并不算是个handler，在添加完有用的处理器后结束使命，就可以被移除了。

```java
//ChannelInitializer.java
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        initChannel(ctx);
    }
}
```
`initChannel`方法触发是在`handlerAdded`中，也就是说`ChannelInitializer`所属的`ChannelHandlerContext`对象被添加到Pipeline中后被触发。

`initChannel具体实现`
```java
//ChannelInitializer.java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
  if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) {
      try {
          initChannel((C) ctx.channel());
      } catch (Throwable cause) {
          exceptionCaught(ctx, cause);
      } finally {
          remove(ctx);
      }
      return true;
  }
  return false;
}
```
【5】执行回调，就是执行写在`initChannel`方法里面的那些操作。
【9】删除`ChannelPipeline`这个实例
  ```java
   private void remove(ChannelHandlerContext ctx) {
    try {
        ChannelPipeline pipeline = ctx.pipeline();
        if (pipeline.context(this) != null) {
            pipeline.remove(this);
        }
    }
  }
  ```
* 【4-6】如果pipeline中指定channelHandler所属的的上下文对象不为空，就从pipeline中删除这个当前对象删除。
    