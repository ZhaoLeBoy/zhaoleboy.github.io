---
layout:     post
title:      "5-1.Netty源码:Pipeline初始化"
subtitle:   "Pipeline初始化"
date:        2019-02-09
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: false
tags:
    - Netty
---

在Channel创建的时候Pipeline被创建出来,一个Channel对应一个Pipeline。

```java
//AbstractChannel.java->DefaultChannelPipeline.java
protected DefaultChannelPipeline(Channel channel) {
  this.channel = ObjectUtil.checkNotNull(channel, "channel");
  succeededFuture = new SucceededChannelFuture(channel, null);
  voidPromise = new VoidChannelPromise(channel, true);

  tail = new TailContext(this);
  head = new HeadContext(this);

  head.next = tail;
  tail.prev = head;
}
```
Pipeline是双向链表的结构。有两个节点

共同继承的抽象类。有删减
![IMAGE](/img/netty/5-1/1.jpg)

### tail
```java
//AbstractChannelHandlerContext.java
TailContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, TAIL_NAME, true, false);
    setAddComplete();
}

```

### head
```java
//AbstractChannelHandlerContext.java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
    setAddComplete();
}
```


```java
//AbstractChannelHandlerContext.java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                              boolean inbound, boolean outbound) {
  this.name = ObjectUtil.checkNotNull(name, "name");
  this.pipeline = pipeline;
  this.executor = executor;
  this.inbound = inbound;
  this.outbound = outbound;
  ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```
tail和head都是pipeline中的数据节点，它们用布尔值在父类构造函数中来标识是inbound还是outbound。