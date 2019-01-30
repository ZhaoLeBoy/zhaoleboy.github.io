---
layout:     post
title:      "3-3.Netty源码:NioEventLoop处理事件"
subtitle:   "NioEventLoop处理事件"
date:        2019-01-30
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

```java
@Override
protected void run() {
  for (; ; ) {
    try {
      ...select部分... 
        final int ioRatio = this.ioRatio;
      if (ioRatio == 100) {
          try {
              processSelectedKeys();//处理io相关逻辑
          } finally {
              // Ensure we always run tasks.
              runAllTasks();//处理外部线程进入队列的任务
          }
      } else {
          final long ioStartTime = System.nanoTime();
          try {
              processSelectedKeys();
          } finally {
              // Ensure we always run tasks.
              final long ioTime = System.nanoTime() - ioStartTime;
              runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
          }
      } 
    } 
   ...
  }
}
```
* 【8】见[【3-2.Netty源码:NioEventLoop循环检测事件】](http://jinlipool.com/2019/01/30/netty-3-2-evetloop-event-detection/)
* 【9】 processSelectedKeys此处是处理相关io连接问题

  ```java
    private void processSelectedKeysOptimized() {
      //获取key
      for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();
    
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
    
      ...
      }
    }
  ```
  * 【4-5】获取selectedKeys，由于在Channel关闭后，SelectedSelectionKeySet可以保持对SelectionKey的强引用，所以每次取出一次要对应设置null，最后由GC 回收。
  * 【7】 这个attachment是来自服务员端`NioServerSocketChannel`的Channel信息,`AbstractNioChannel#doRegister`中`selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);`
  * 【10】  processSelectedKey(k, (AbstractNioChannel) a);
      ```java
       private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        ...
        /**
         * OP_ACCEPT: 请求在接受新连接并创建 Channel 时获得通知
         * OP_CONNECT: 请求在建立一个连接时获得通知
         * OP_READ: 请求当数据已经就绪，可以从 Channel 中读取时获得通知
         * OP_WRITE: 请求当可以向 Channel 中写更多的数据时获得通知。这处理了套接字缓冲区被完 全填满时的情况，
         *          这种情况通常发生在数据的发送速度比远程节点可处理的速度更 快的时候
         */
    
            ...
          if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
              unsafe.read();
          }
        }
      ```
      * 【14】处理读事件ing..........
    
  
*【14】 详情见[【3-4.Netty源码:任务和定时任务队列】](http://jinlipool.com/2019/01/30/netty-3-4-task.md/)