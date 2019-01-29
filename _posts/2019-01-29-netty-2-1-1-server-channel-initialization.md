---
layout:     post
title:      "2-1-1.Netty源码:初始化服务端的channel"
subtitle:   "初始化服务端的channel"
date:        2019-01-29
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

### 初始化并且注册一个服务端的Channel
>服务端的channel是通过反射创建出来的

```java
//AbstractBootstrap.java
final ChannelFuture initAndRegister() {
  Channel channel = null;
  try {
      channel = channelFactory.newChannel();
      init(channel);
  } catch (Throwable t) {
  ...
  }

  ChannelFuture regFuture = config().group().register(channel);
   return regFuture;
}
```

* 【4】通过反射来创建channel的这个反射的类就是在启动代码里`.channel(NioServerSocketChannel.class)`里设置的`NioServerSocketChannel`的类,反射调用这个类的构造函数进行初始化,具体方法见[【1-2-1.Netty源码:服务端NioServerSocketChanne构造方法】](http://jinlipool.com/2019/01/27/netty-1-2-1-NioServerSocketChannel-construct/)
* 【6】初始化channel，具体初始化的方法在`ServerBootstrap#init()`中

  ```java
  //ServerBootstrap.java
   void  init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    //设置新channel的channeloptions
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    //设置新channelchanneloptions
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }

    p.addLast(new ChannelInitializer<Channel>() {
      @Override
      public void initChannel(final Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) { 
            pipeline.addLast(handler);
        }
         
        ch.eventLoop().execute(new Runnable() {
          @Override
          public void run() {
            //跟新连接接入有关，把新连接绑定到新的NioEventLoop线程上去
              pipeline.addLast(new ServerBootstrapAcceptor(
                      ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
          }
        });
      }
    });
  }
  ```
  * 【3-15】设置服务端channel的option和attrs,这些配置都是在启动代码中用户自行定义的好的。将option绑定到服务端channel`NioServerSocketChannel`中的`ServerSocketChannelConfig`去。
  * 【17】获取这个channel的管道对象。
  * 【19-26】保存用户自定义的childOptions和childAttrs到局部变量。
  * 【29-46】在管道(pipeline)对象最后面新增ChannelInitializer的实例。
    *【32】添加用户自定义个handle ,这个handler是由用户在demo的方法链中实现了。专门处理跟bossgoup有关（跟childHandler专门处理workgroup类似）
    *【37-43】服务端启动时候在最后默认会添加`ServerBootstrapAcceptor`，类似Reactor模型的Acceptor
      1. currentChildOptions,currentChildAttrs用户自定设置的属性。
      2. currentChildHandler也是用户在代码中设置e.g. ` .childHandler(new ChannelInitializer<SocketChannel>() {...})`
      3. currentChildGroup就是用户代码中的`workerGroup`
* 【10】详细见[【2-1-2.Netty源码:注册服务端channel】](http://jinlipool.com/2019/01/29/netty-2-1-2-server-channel-register/)

---
TODO:ChannelInitializer没有做说明

  