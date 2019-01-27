---
layout:     post
title:      "1-2.Netty源码:构建ServerBootstrap以及它的调用链"
subtitle:   "构建ServerBootstrap以及它的调用链"
date:        2019-01-27
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


先上部分代码，这块代码就是服务端启动demo的一部分，这篇文章就是对这块代码进行描述
```java
...
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(bossGroup, wordGroup)
  .channel(NioServerSocketChannel.class)
  .childHandler(new ChannelInitializer<SocketChannel>() { 
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
      ChannelPipeline pipeline = ch.pipeline();
      ...
    }
  });
...
```
### 概述
1. 实例化`ServerBootstrap`。
2. `ServerBootstrap`链式调用

### 1.实例化`ServerBootstrap`。
实例化`ServerBootstrap`，其实内部是个空的构造方法，文档中对其解释是`ServerBootstrap`这个类是`BootStrap`的子类，允许便捷的启动`ServerChannel` 。
其实`ServerBootstrap`和`BootStrap`的关系如下图，并没有直接继承关系，总感觉描述应该是`ServerBootstrap`和`BootStrap`都是同级的子类。

![IMAGE](/img/netty/1-2/1.jpg)
    
 `ServerChannel`是一个标记接口，用于对类进行标识。`ServerChannel`是一个Channel用于接受另一端发来的连 接，并通过接受来创建它的子Channel(就是连接的Channel)
  ```java
  public interface ServerChannel extends Channel {
      // This is a tag interface.
  }
  ```
    
### 2. ServerBootstrap链式调用
* ` group(EventLoopGroup parentGroup, EventLoopGroup childGroup)` 
  ```java
      //ServerBootStrap.java
  public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    ...
    this.childGroup = childGroup;
    return this;
  }
  ```
  设置服务端(acceptor)和客户端的EventLoopGroup，这些EventLoopGroup用于处理跟ServerChannel和Channel的所有事件和IO，方法里面做了非空判断和给成员变量赋值。 
  * 【3】parentGroup作为参数传入到父类
      ```java
      //AbstractBootstrap.java
      public B group(EventLoopGroup group) {
      ...
      this.group = group;
      return self();
      }
      ```
      父类goup方法意义是这个`EventLoopGroup`用于处理将要创建的Channel的所有事件，由子类传入的是`parentGroup`，所以它是用于处理所有的跟客户端连接有关的事件。方法里也是做非空判断和给成员变量赋值。
    
* `channel(Class<? extends C> channelClass)`
  ```java
  //AbstractBootstrap.java
   public B channel(Class<? extends C> channelClass) {
    ...
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
  }
  ```
  使用channelClass通过反射创建一个Channel实例，这个前提是这个Channel是要有无参的构造方法，如果有的话只能使用`channelFactory(io.netty.channel.ChannelFactory)`来获取实例。
    ```java
    //ReflectiveChannelFactory.java
     public ReflectiveChannelFactory(Class<? extends T> clazz) {
      ...
      this.clazz = clazz;
    }
    ```
    `ReflectiveChannelFactory`是一个Channel工厂类，是通过反射形式创建有一个Channel实例。这个方法里只是做个简单的成员变量赋值
    ```java
    //AbstractBootstrap.java
    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
      ...
      this.channelFactory = channelFactory;
      return self();
    }
    ```
    只有当调用`bind`方法（见demo），ChannelFactory才会去创建一个channel实例，所以现在只是设置好成员变量。
    最后对通过反射实例化的`NioServerSocketChannel`这个类的无参构造方法究竟做了什么请看[【1-2-1.Netty源码:服务端NioServerSocketChannel构造方法】]()
  
* `childHandler(ChannelHandler childHandler)`
  ```java
  public ServerBootstrap childHandler(ChannelHandler childHandler) {
    ...
    this.childHandler = childHandler;
    return this;
  }
  ```
  设置ChannelHandler，专门服务于Channel的请求。这个方法里只是做个简单的成员变量赋值。
  
  
到此为止ServerBootStrap所必须的成员变量也设置好，纵观这个方法链，大部分是都做了非空判断+成员变量赋值再然后就是一些构建一些工厂。这些工厂和成员变量都是为后面`bind`启动来服务的。
    
    