---
layout:     post
title:      "1-1-1.Netty源码:线程执行器ThreadPerTaskExecutor"
subtitle:   "线程执行器ThreadPerTaskExecutor"
date:        2019-01-27
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


### 概述
ThreadPerTaskExecutor 字面意思是每个线程的任务执行器，是`java.util.concurrent`包下的`Executor`的一个实现类，`Executor`主要用于执行提交的Runnable任务，实现了任务提交和任务执行进行的解耦，所以可以大概猜到`ThreadPerTaskExecutor`跟线程执行任务有关。
```java
//MultithreadEventExecutorGroup.java
executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
```

### 构建一个线程工厂
```java
protected ThreadFactory newDefaultThreadFactory() {
  return new DefaultThreadFactory(getClass());
}
```
生成一个默认的线程工厂，传入的是当前类的对象,`DefaultThreadFactory`是ThreadFactory的一个实现。


```java
//DefaultThreadFactory.java
public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
  ...
  prefix = poolName + '-' + poolId.incrementAndGet() + '-';
  this.daemon = daemon;
  this.priority = priority;
  this.threadGroup = threadGroup;
}
```
* 【4】给这个工厂定义一个唯一名字前缀，`eventLoopGroup_` +自增器，在`DefaultThreadFactory`类中有具体方法来生成这个`poolName`可以自行查看。

### 线程执行器
```java
//ThreadPerTaskExecutor.java
private final ThreadFactory threadFactory;
public  ThreadPerTaskExecutor(ThreadFactory threadFactory) {
  ... 
  this.threadFactory = threadFactory;
}
```
【4】构造方法就干一件事，给成员变量赋值

由于`ThreadPerTaskExecutor`实现了`Executor`接口，它实现接口的`execute`方法
```java
//ThreadPerTaskExecutor.java 
@Override
public void execute(Runnable command) {
  threadFactory.newThread(command).start();
}
```
* 【3】使用了代理模式，由这个类里面的成员变量`threadFactory`进行线程创建和执行。