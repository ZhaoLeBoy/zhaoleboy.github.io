---
layout:     post
title:      "二.Netty源码:NioServerSocketChannel创建"
subtitle:   "netty源码学习"
date:       2018-11-28 10:00:00
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
tags:
    - Netty
---

服务端的channel是通过反射创建出来的，具体这个`NioServerSocketChannel`的初始化是在什么时候发生的后面会介绍。先看这个类的无参构造方法。

`NioServerSocketChannel`的类的继承关系图

![NioServerSocketChannel.png][image-1]

```java
//无参构造方法 
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```
NioServerSocketChannel初始化做的三件事情:
* 2-1. 调用jdk底层创建ServerSocketChannel
* 2-2. 继承父类设置channel相关配置
* 2-3. 创建`ServerSocketChannelConfig`类

### 2-1. 调用jdk底层创建ServerSocketChannel

```java
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             * 默认实现是在{@Link ServerSocketChannel#open }
             * SelectorProvider.provider().openServerSocketChannel();
             *
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
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
    }
```

### 2-2. 继承父类设置channel相关配置

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
  //注意这个SelectionKey.OP_ACCEPT 
  //这个是服务端的channel所以设置OP_ACCEPT
  //在轮训中，表示请求在接受新连接并创建 Channel 时获得通知
  super(null, channel, SelectionKey.OP_ACCEPT);
}
```
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
          //调用jdk底层设置channel的非阻塞模式
          ch.configureBlocking(false);
      } catch (IOException e) {
         ...
      }
    }
    ```
  
* AbstractChannel

  ```java
  protected AbstractChannel(Channel parent) {
      this.parent = parent;
      //channel的唯一标识
      id = newId();
      //底层读写跟tcp有关。
      unsafe = newUnsafe();
      //TODO:.....
      pipeline = newChannelPipeline();
  }
  ```

### 2-3. 创建`ServerSocketChannelConfig`类

```java
  //将ServerSocketChannel作为构造函数参数之一，用于给channel配置TCP的一些配置
  //TODO:.....
 config = new NioServerSocketChannelConfig(this, javaChannel().socket());
```
[image-1]: /img/netty/NioServerSocketChannel.png