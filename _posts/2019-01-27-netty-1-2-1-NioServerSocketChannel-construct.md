---
layout:     post
title:      "1-2-1.Netty源码:服务端NioServerSocketChanne构造方法"
subtitle:   "服务端NioServerSocketChanne构造方法"
date:        2019-01-27
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


### 概述
服务端的channel是通过反射创建出来的，具体这个`NioServerSocketChannel`的初始化是在什么时候发生的后面会介绍。先看这个类的无参构造方法。

`NioServerSocketChannel`的类的继承关系图,有删减

![IMAGE](/img/netty/1-2-1/1.jpg)

```java
//无参构造方法 
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

NioServerSocketChannel初始化做的三件事情:
1. 调用jdk底层创建ServerSocketChannel
2. 继承父类设置channel相关配置
3. 创建`ServerSocketChannelConfig`类

###  1. 调用jdk底层创建ServerSocketChannel

```java
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        /**
         * 默认实现是在{ ServerSocketChannel#open
         * SelectorProvider.provider().openServerSocketChannel();
         *See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
         *  当前的SelectorProviderde provider方法，这个方法包含了一个同步代码块，
         *  每创建一个新的channel都会去调用这个方法，执行里面的同步代码块。
         *  当应用创建许多的连接的时候，这个会导致不必要的阻塞，每秒创建5000个连接的时候，性能会下降1%
         *
         *  可能的解决方案是存储provider方法的结果，然后直接用selectorProvider的openSocketChannel创建channel,而不是使用SocketChannel的open方法。
         *  所以在Netty最近的版本中，确实对这个方案进行了实现，将provider直接设置成SocketChannel类的静态成员，并进行初始化赋值。
         */
  
        //创建一个新的channel
        return provider.openServerSocketChannel();
    } catch (IOException e) {
       ...
    }
}
```
  * 【16】使用jdk底层创建一个ServerSocketChannel
  
###  2. 继承父类设置channel相关配置

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
  super(null, channel, SelectionKey.OP_ACCEPT);
}
```
* 【2】注意这个SelectionKey.OP_ACCEPT，如果是服务端的channel，则是OP_ACCEPT，用来检测新连接在轮训中，表示请求在接受新连接并创建 Channel 时获得通知

> 继承链如下：
NioServerSocketChannel->AbstractNioMessageChannel->AbstractNioChannel->AbstractChannel
重要的几个方法是在`AbstractNioChannel`，`AbstractChannel`

  * AbstractNioChannel
  
    ```java
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);
        } catch (IOException e) {
           ...
        }
    }
    ```
     * 【6】 调用jdk底层设置channel的非阻塞模式
  
* AbstractChannel

  ```java
  protected AbstractChannel(Channel parent) {
      this.parent = parent;
      id = newId();
      unsafe = newUnsafe();
      pipeline = newChannelPipeline();
  }
  ```
  【3】channel的唯一标识
  【4】//TODO底层读写跟tcp有关。
  【5】TODO:.....

###  3. 创建`ServerSocketChannelConfig`类
```java
  //TODO:.....
 config = new NioServerSocketChannelConfig(this, javaChannel().socket());
```
* 【1】 将ServerSocketChannel作为构造函数参数之一，用于给channel配置TCP的一些配置