---
layout:     post
title:      "Netty源码:ChannelHandlerContext "
subtitle:   "ChannelHandlerContext"
date:        2019-02-07
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

对Netty中的ChannelHandlerContext的文档大概翻译一下.

### 概述
`ChannelHandlerContext` 可以让一个`ChannelHandler`与它的`ChannelPipeline`和其他handler交互。此外一个处理器可以通知`ChannelPipeline`中下一个`ChannelHandler`并且可以动态修改它所属的`ChannelPipeline`对象

### 通知
可以通过调用里面的一些方法通知同一个`ChannelPipeline`中最近的handler

### 修改pipeline
可以调用`pipeline()`方法来获取这个handler所属的`ChannelPipeline`对象，应用可以在pipeline运行中动态的插入删除替换handler

### 获取供后续使用
可以获取一个ChannelHandlerContext供后续使用，例如在handler方法之外，甚至不同的线程中触发一个事件

```java
public class MyHandler extends {@link ChannelDuplexHandler} {

private  ChannelHandlerContext ctx;

public void beforeAdd({@link ChannelHandlerContext} ctx) {
  this.ctx = ctx;
}

public void login(String username, password) {
    ctx.write(new LoginMessage(username, password));
}
...
}
```

### 存储状态信息
`attr(AttributeKey`允许你存储和访问跟handler和它的context有关的状态的信息

#### 一个handler可以拥有多个context
一个ChannelHandler可以多次添加到ChannelPipeline中，这就意味这个一个ChanndlHandler实例可以拥有多个ChannelHandlerContext ，因此一个单例可以在不同的ChannelHaandlerContext被调用，如果它被添加到
ChannelPipeline中超过一次。
举个例子。如下的handler会拥有多个独立的AttributeKey，取决于添加到pipeline中次数，不论它是被添加到相同pipeline多次，还是不同pipeline中多次。
```java
 public class FactorialHandler extends  ChannelInboundHandlerAdapter {
 
    private final AttributeKey<Integer>; counter = AttributeKey.valueOf("counter");
 
    // This handler will receive a sequence of increasing integers starting
    // from 1.
    @Override 
    public void channelRead (ChannelHandlerContext  ctx, Object msg) {
      Integer a = ctx.attr(counter).get();
 
      if (a == null) {
        a = 1;
      }
 
      attr.set(a * (Integer) msg);
    }
  }
```
```java
/*
*不同的context对象赋值"f1", "f2", "f3", and "f4"，尽管它们是同一个handler实例
* 因为FactorialHandler在context对象(使用AttributeKey)中存储了自己的状态。
* 当两个pipeline处于活动状态下，它会被计算四次
*/
ChannelPipeline  p1 =   Channels.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);

ChannelPipeline  p2 =  Channels.pipeline();
p2.addLast("f3", fh);
p2.addLast("f4", fh);
```

>下面是摘自《Netty实战》

__关系图__
![IMAGE](/img/netty/extend/channelHandlerContext/1.jpg)

__事件的流动__
![IMAGE](/img/netty/extend/channelHandlerContext/2.jpg)

ChannelHandlerContext 有很多的方法，其中一些方法也存在于 Channel 和 ChannelPipeline 本身上，但是有一点重要的不同。如果调用Channel或者ChannelPipeline 上的这 些方法，它们将沿着整个 ChannelPipeline 进行传播。而调用位于 ChannelHandlerContext 上的相同方法，则将从当前所关联的 ChannelHandler 开始，并且只会传播给位于该 ChannelPipeline 中的下一个能够处理该事件的 ChannelHandler。