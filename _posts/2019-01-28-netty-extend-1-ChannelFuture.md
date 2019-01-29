---
layout:     post
title:      "Netty源码:ChannelFuture"
subtitle:   "ChannelFuture"
date:        2019-01-28
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---

主要就是对Netty中的Future和ChannelFuture的文档大概翻译一下，Netty的文档写的比较详细，认真看下来也是收获颇多。
Netty中也有个Future接口，在`io.netty.util.concurrent`包中。
jdk自带的Future接口是在`java.utail.concurrent`。

先上ChannelFuture,Future和jdk的Future的关系图
![IMAGE](/img/netty/extend/channelfuture/1.jpg)


### java.utail.concurrent.Future

> 文档说明

Future表示异步计算的结果，Future中提供一些方法用来检测计算是否完成，等待计算完成，获取计算方法的方法等。
计算完成只能通过`get` 获取。get是阻塞的，直到计算完成。取消操作是`cancel`完成，Future提供额外的方法来
判断任务是正常完成还是取消掉了，如果计算一旦完成就不能被取消。如果想使用Future去取消任务的目的但是没有
提供可用的结果，可以声明Future<?> 返回一个null 来表示底层的结构

```java
interface ArchiveSearcher { String search(String target); }
class App {
  ExecutorService executor = ...
  ArchiveSearcher searcher = ...
  void showSearch(final String target) throws InterruptedException {
    Future<String> future = executor.submit(new Callable<String>() {
      public String call() {
        return searcher.search(target);
      }});
    displayOtherThings(); // do other things while searching
    try {
      displayText(future.get()); // use future
    } catch (ExecutionException ex) { cleanup(); return; }
  }
}}
```

FutureTask类是Future的一个实现类，他继承了Runable，所以Executor可以执行FutureTask
上面代码优化如下
```java
FutureTask<String> future =
  new FutureTask<String>(new Callable<String>() {
      public String call() {
        return searcher.search(target);
    }});
  executor.execute(future);
}
```
内存一致性的影响：由异步计算异步计算的行为一定在另个线程future.get结果返回之前发生的。

>内部方法

```java
boolean cancel(boolean mayInterruptIfRunning);
boolean isCancelled();
boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
```

###  io.netty.util.concurrent.Future

> 文档说明

异步操作的结果，是在jdk自带的future接口之上进行方法扩展。
```java
//如果并只能如果io操作完成才会返回true
boolean isSuccess();

// 只有当操作可以通过cancel(boolean)返回true才返回true
boolean isCancellable();

// 如果io失败返回原因
Throwable cause();

// future中添加了指定的监听，当future的isDone()完成就会收到通知，(观察者模式)
Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
Future<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

//移除监听器
Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
Future<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

// 等待future 直到完成。如果失败则抛出异常
Future<V> sync() throws InterruptedException;
Future<V> syncUninterruptibly();

// 等待future完成
Future<V> await() throws InterruptedException;
Future<V> awaitUninterruptibly();

//等待fure完成
boolean await(long timeout, TimeUnit unit) throws InterruptedException;
boolean await(long timeoutMillis) throws InterruptedException;
boolean awaitUninterruptibly(long timeout, TimeUnit unit);
boolean awaitUninterruptibly(long timeoutMillis);

//如果没有被堵塞直接返回结果，如果future没有完成 就返回null
//如果null被表示结果为成功，你需要坚持结果是否真的完成 使用isDone，最好不要依赖null返回结果
V getNow();

@Override
boolean cancel(boolean mayInterruptIfRunning);
```
---
### io.netty.channel.ChannelFuture
异步Channel的io操作的结果

> 文档说明

 在Netty中所有的IO操作都是异步的，意味着任何io调用都会立刻返回，并不能保证所请求的io操作在调用结束后
 会返回，相反，你会获取一个ChannelFuture 它会返回关于io操作的结果或者信息。
一个ChannelFuturen要不是完成或者未完成，当io操作开始，Netty会创建一个新的future对象，新的future对象
是未完成，不是成功的也不是失败，也不是取消，如果io完成，要不然成功完成，失败完成要不是取消future被标
识完成，携带一些信息，如失败的原因，注意，失败和取消都是完成的状态。
![IMAGE](/img/netty/extend/channelfuture/2.jpg)
提供的各种方法让你去检测io操作是否完成了，还是在等待完成，可以获取io操作的结果，允许你添加
`GenericFutureListener`，当io操作完成后可以收到通知.推荐使用`addListener(GenericFutureListener)`，不推荐使用`await()`。原因是`addListener(GenericFutureListener)`方法是非阻塞的，仅仅是将ChannelFutureListener添加到ChannelFuture中当与这个ChannelFuture关联的io操作完成时候，进行对通知。ChannelFutureListener性能好和资源利用率高，因为它什么都不会堵塞。不过如果不熟悉事件驱动模型。实现逻辑循序比较困难。

await是堵塞，虽然实现顺序逻辑容易单线程会堵塞，成本太高。容易出现死锁。
不要在ChannelHandler中调用await方法， ChannelHandler 中事件方法通常被io线程调用 如果await事件在事件处理方法(比如io连接的处理等)被调用，这个事件处理方法又是被io线程调用，这个io操作将一直等待 不会完成，因为await可以堵塞io操作，造成死锁。
```java
// BAD - NEVER DO THIS
public void channelRead({@link ChannelHandlerContext} ctx, Object msg) {
 {@link ChannelFuture} future = ctx.channel().close();
  future.awaitUninterruptibly();
  // Perform post-closure operation
  // ...
}

// GOOD
{@code @Override}
public void channelRead({@link ChannelHandlerContext} ctx, Object msg) {
  {@link ChannelFuture} future = ctx.channel().close();
  future.addListener(new {@link ChannelFutureListener}() {
    public void operationComplete({@link ChannelFuture} future) {
        // Perform post-closure operation
        // ...
    }
  });
}
```
不要在io线程中掉await 否则会抛出BlockingOperationException 防止死锁
不要将io超时 和等待超时 混淆。 ,await(long, TimeUnit),awaitUninterruptibly(long)，waitUninterruptibly(long, TimeUnit)  这些方法跟io超时没有任何关系。如果io操作超时，future会被标记失败，e.g.如果连接超时会通过一个特定传输选项进行配置

```java
// BAD - NEVER DO THIS
Bootstrap  b = ...;
ChannelFuture  f = b.connect(...);
f.awaitUninterruptibly(10, TimeUnit.SECONDS);//等待超时
if (f.isCancelled()) {
    // 用户取消连接
} else if (!f.isSuccess()) {
    //你可能获取到一个NullPointerException异常，因为future并没有完成
    f.cause().printStackTrace();
} else {
    //连接成功
}
/*
* 因为ChannelFuture是异步的，等到下面判断逻辑的的时候b.connect还不能建立的上
*，f是空，所以会得到NullPointerException
*/

// GOOD
 Bootstrap b = ...;
// 配置连接超时时间
b.option( ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000);
ChannelFuture} f = b.connect(...);
f.awaitUninterruptibly();

//此时已经确定任务已经完成
assert f.isDone();

if (f.isCancelled()) {
    // Connection attempt cancelled by user
} else if (!f.isSuccess()) {
    f.cause().printStackTrace();
} else {
    // Connection established successfully
}
```
