---
layout:     post
title:      "2-1.Netty源码:服务端启动"
subtitle:   "服务端启动"
date:        2019-01-29
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

>参考【Netty源码:目录】中写的服务端启动代码

>具体bind方法在`AbstractBootstrap`中实现的，表示新建一个Channel并且绑定它。删除了些后就这么多，后面围绕这块代码展开说

  ```java
  //AbstractBootstrap.java
  private ChannelFuture doBind(final SocketAddress localAddress) {
    //初始化和注册
    final ChannelFuture regFuture = initAndRegister();
  
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }
  
    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
       ...
    }
  }
  ```

*【4】初始化并且注册一个服务端的Channel
  * [【2-1-1.Netty源码:初始化服务端的channel】](http://jinlipool.com/2019/01/29/netty-2-1-1-server-channel-initialization/)
  * [【2-1-2.Netty源码:注册服务端channel】](http://jinlipool.com/2019/01/29/netty-2-1-2-server-channel-register/)

*【13】执行具体绑定端口方法
  * [【2-1-3.Netty源码:端口绑定】](http://jinlipool.com/2019/01/29/netty-2-1-3-server-socket-bind.md/)
    