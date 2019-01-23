---
layout:     post
title:      "《Scalable IO in Java》学习笔记"
subtitle:   "《Scalable IO in Java》学习"
date:        2018-12-29
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Netty
---




[Doug Lea](!https://baike.baidu.com/item/Doug%20Lea/6319404?fr=aladdin) 大师写的《Scalable IO in Java》

文章地址:[《Scalable IO in Java》](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)


我就当边学变翻译文章了，水平渣，如有错误定当改之。😂

---

### 网络服务
在大部分的网络服务系统中(例如.web服务，分布式对象)基础结构，如下:
* 读请求
* 对请求进行解码
* 处理业务逻辑
* 对处理完对结果进行编码
* 发送响应

但不同网络服务对于上面的每一步的处理的方式和成本大都不一样。

### 传统的服务设计
![IMAGE](/img/netty/scalable-io/scalable-io-1.jpg)
每个处理器(handler)都会在自己线程中进行。

> 示例代码

```java
class Server implements Runnable {
  public void run() {
try {
      ServerSocket ss = new ServerSocket(PORT);//绑定服务端口号
      while (!Thread.interrupted())
        /**
        *客户端发起一个请求，服务端就会new个线程，由这个线程跟客户端进行数据双向处理
        *ServerSocket返回个socket(堵塞)作为参数传入处理器，
        */
        new Thread(new Handler(ss.accept())).start();
      // or, single-threaded, or a thread pool
    } catch (IOException ex) { /* ... */ }
  }
  static class Handler implements Runnable {
    final Socket socket;
    Handler(Socket s) { socket = s; }
    public void run() {
    try {
      //获取输入流，读取客户端的数据，数据处理最后写入输出流
        byte[] input = new byte[MAX_INPUT]; 
        socket.getInputStream().read(input);
        byte[] output = process(input);
        socket.getOutputStream().write(output);
      } catch (IOException ex) { /* ... */ }
    }
    private byte[] process(byte[] cmd) { /* ... */ }
  }
}
Note: most exception handling elided from code examples
```

### 传统服务设计存在的不足
如果只是少量的连接数，传统模型是可以应付的，但是如今的互联网都是几十万到百万的流量，连接数肯定也是成百上千，物理机有线程数的限制，线程之间的切换也很耗性能，所以传统的设计肯定是驾驭不了的。所以要对原有的模型进行扩展。希望达到一个目标。

### 可以扩展的目标
* 更多的客户端连接，负载增加时的优雅降级。
* 随着资源(CPU、内存、磁盘、带宽)的增加系统性能也随之提高。
* 还要满足可用性和性能目标:
  * 延迟低
  * 高负载
  * 服务的质量可控 
  
>分治策略通常是实现可伸缩性目标的最佳方法。

### 分治策略(Divide and Conquer)
  * 将处理过程分解更小的任务，每个任务在不阻塞情况下执行一个action。
  * 执行可用的任务，例如，某个任务io操作没有完成系统不必等待，去执行下个可用的任务，IO事件作为触发器使用，某个IO事件产生，对应的读的功能就会被触发执行。
  * java.nio中支持的基本机制：
    * 非堵塞的读写。
    * 分派感知到IO事件相关的任务。

### 事件驱动设计

**这种设计方式比其他方案更有效**
* 占用更少的资源，不用为每个客户端开启新线程。
* 更小的开销，少了线程就少了线程直接的切换，也就少了锁的使用。
* 派发将会变得慢，因为必须手动将操作(action)绑定到事件。

**但是这种设计编程更加复杂**
* 必须拆分为简单的非阻塞操作(action),不过无法消除所有阻塞（例如，gc，页面错误等）。
* 必须持续的跟踪服务的逻辑状态。

### Reactor模式
* 感应器(Reactor)，相应IO事件，并派发适当的处理器(handler)进行事件处理。
* 处理器(Handler)用来处理非堵塞的操作(action)。
* 将handler绑定到对应事件上进行管理，当有事件产生，会有相应的handler进行对应处理。

>单线程版本

![IMAGE](/img/netty/scalable-io/scalable-io-2.jpg)

reactor是个线程对象用来监听客户端向服务器发起的连接。
当有客户端有事件传入，Reactor检测到后分发(dispatch)给特定的处理器来处理客户端数据。

### java.nio支持
Chnnels:连接到支持非阻塞读取的文件、套接字等。
Buffers:类似数组对象可以直接由channel中读写。
Selectors:通知哪些Channel有io事件。
SelectionKeys:里面维护这io事件状态和绑定事件。

>文章中的示例代码

```java
//1.Setup
class Reactor implements Runnable {
  final Selector selector;
  final ServerSocketChannel serverSocket;
  
  Reactor(int port) throws IOException {
    selector = Selector.open();
    serverSocket = ServerSocketChannel.open();
    serverSocket.socket().bind(new InetSocketAddress(port));
    serverSocket.configureBlocking(false);
    SelectionKey sk = serverSocket.register(selector,SelectionKey.OP_ACCEPT);
    //serverSocket中附加一个Acceptor方法
    sk.attach(new Acceptor());
  }

  //2.Dispatch Loop
  public void run() {  // normally in a new Thread
    try {
        while (!Thread.interrupted()) {
          selector.select();
          Set selected = selector.selectedKeys(); 
          Iterator it = selected.iterator(); 
          while (it.hasNext())
            //不做任何客户端处理。全部派发下去
             dispatch((SelectionKey)(it.next()); 
          selected.clear();
        }
    } catch (IOException ex) { /* ... */ }
  }
  
  void dispatch(SelectionKey k) {
    //获取attachment对象也就是在前面的的Acceptor()
    Runnable r = (Runnable)(k.attachment()); 
    if (r != null)
      r.run();
  }

  //3.Acceptor 
  class Acceptor implements Runnable { // inner
      public void run() {
        try {
          SocketChannel c = serverSocket.accept();
          if (c != null)
            new Handler(selector, c);
        }catch(IOException ex) { /* ... */ }
     }
  } 
}

//4.Handler setup
//建立注册客户端socket
final class Handler implements Runnable {
  final SocketChannel socket;
  final SelectionKey sk;
  ByteBuffer input = ByteBuffer.allocate(MAXIN); 
  ByteBuffer output = ByteBuffer.allocate(MAXOUT); 
  static final int READING = 0, SENDING = 1;
  int state = READING;
  
  Handler(Selector sel, SocketChannel c) throws IOException {
      socket = c; c.configureBlocking(false);
      // Optionally try first read now
   sk = socket.register(sel, 0); 
   sk.attach(this); 
   sk.interestOps(SelectionKey.OP_READ); 
   sel.wakeup();
  }
  
  boolean inputIsComplete() { /* ... */ } 
  boolean outputIsComplete() { /* ... */ } 
  void process() { /* ... */ }
  
  public void run() {
  //区分状态，处理用户自己的业务逻辑
    try {
      if      (state == READING) read();
      else if (state == SENDING) send();
    } catch (IOException ex) { /* ... */ }
  }
  void read() throws IOException {
    socket.read(input);
    if (inputIsComplete()) {
      process();
      state = SENDING;
      // Normally also do first write now 
      sk.interestOps(SelectionKey.OP_WRITE);
    } 
  }
 void send() throws IOException { 
  socket.write(output);
  if (outputIsComplete()) 
    sk.cancel();
  } 
}
```

### 多线程的设计

>现代计算机都是多核，为了更好的应用现有资源，策略性地添加线程以获得可伸缩性。
  * 1.引入Worker线程，Reactors会快速的触发处理器，但是处理器的处理速度会拖慢Reactor,建议将非IO处理交给其他线程完成。
  * 2.多Reactor线程可以做到负载均衡，均衡CPU和IO之间速率。

#### 1. worker线程设计
* 移除非io的操作，加快reactor线程
* 比在事件驱动模型中重新计算绑定更简单，还是非阻塞计算,有有足够的处理量以解决过载问题。
* 但是处理IO计算重叠比较困难，最好的方式是一开始就将所有的输入读进buffer，再进行后续操作。
* 使用线程池技术，所以方便调节，通常需要的线程数要远小于客户端的数量。

![IMAGE](/img/netty/scalable-io/scalable-io-3.jpg)

```java
class Handler implements Runnable {
  // uses util.concurrent thread pool
  //可以使用java自带的线程池的一些实例对象
  static PooledExecutor pool = new PooledExecutor(...);
  static final int PROCESSING = 3;
  // ...
  synchronized void read() { // ...
    socket.read(input);
    if (inputIsComplete()) {
      //读取完毕输入流后放入线程池中执行，并且标记状态
      state = PROCESSING;
      pool.execute(new Processer());
    }
  }
  synchronized void processAndHandOff() {
    process();
    state = SENDING; // or rebind attachment
    //处理写事件
    sk.interest(SelectionKey.OP_WRITE);
  }
  class Processer implements Runnable {
    public void run() { processAndHandOff(); }
  }
}
```
* 每个任务都可以启用，触发或者调用下一个任务。
* 回调每个处理器(per-handler)的分派器(dispatcher)，设置状态、附件等。
* 队列可以跨阶段传递缓冲区。
* 每个任务的结果可以用wait/notify或者join进行协同。


#### 2.多Reactor线程设计
* 使用reactor池。用于匹配CPU和IO的速率。
* 可以静态或者动态构造，每个reactor都有自己的slector thread dispatch loop。
* 主acceptor向其他reactor分配。

![IMAGE](/img/netty/scalable-io/scalable-io-4.jpg)

主接收器进行其他reactor的分发
```java
Selector[] selectors; // also create threads int next = 0;
class Acceptor { // ...
  public synchronized void run() { ...
    Socket connection = serverSocket.accept(); 
    if (connection != null)
      new Handler(selectors[next], connection); 
      if (++next == selectors.length) 
        next = 0;
    } 
}
```

### 使用其他nio的特性
* 每个Reactor都有多个selector，去绑定不同处理器到不同io事件中，需要考虑到同步去协调。
* 文件到网络，网络到文件的自动化拷贝。
* 使用buffer访问文件。
* 使用直接缓冲区(Direct buffers)可以实现零拷贝,但是有初始化和回收开销,最适合长连接的应用程序。