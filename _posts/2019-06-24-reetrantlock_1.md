---
layout:     post
title:      "1.ReetrantLock:单线程下的加锁过程"
subtitle:   "1.ReetrantLock:单线程下的加锁过程"
date:        2019-06-24
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - ReetrantLock
    - Lock
---

### 一个简单的demo

首先通过一个Demo来展示在单线程模式下ReetrantLock加锁和释放锁的过程
  
```java
//Demo
private static ReentrantLock lock = new ReentrantLock();

public static void main(String[] args) {
    lock.lock();
    lock.lock();
    System.out.println("Hello World");
    lock.unlock();
    lock.unlock();
}
```

通过例子+运行时候的断点，我们开始学习源码。

### Lock
```java
//ReentrantLock.java
public ReentrantLock() {
    sync = new NonfairSync();
}
```
首先是实例化一个ReentrantLock实例，然后可以跟着断点看看实例化具体流程。

`NonfairSync->abstract Sync- abstract >AbstractQueuedSynchronizer`

`NonfairSync->abstract Sync`

默认情况下ReentrantLock是使用非公平锁方式，也是就是默认使用`NonfairSync`这个类来做具体实例化工作。

实例化完后准备获取锁了，具体代码流程如下。

```java
//NonfairSync.java
final void lock() {
if (compareAndSetState(0, 1))
    setExclusiveOwnerThread(Thread.currentThread());
else
    acquire(1);
}
```
* 【3-4】由于是刚启动，第一次调用`lock`方法，所以在调用`compareAndSetState`直接加锁成功。`state`变为1，并且通过`setExclusiveOwnerThread`将当前线程设置为已经拥有锁的线程。业务代码第一次调用`lock.lock()`加锁就成功了，结束第一次调用。

* 【6】然后紧接着在业务代码第二次调用`lock.lock()`，由于前一次线程加锁成功，所以此时的CAS方法会失败，断点会走到`acquire(1);`,具体实现方法是在`NonfairSync#tryAcquire->Sync#nonfairTryAcquire`

```java
//ReentrantLock.java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
* 【3-4】获取当前线程已经线程状态，当前线程已经加锁了，所以state肯定不是0
* 【11-16】判断当前线程是不是已经获取到锁了，如果是已经获取到锁，则对state进行累加，所以`ReetrantLock`对于同一个线程获多次在内部是通过累加来进行记录的。

### unLock

加锁结束后继续跟进代码到释放这一快。

```java
//AbstractQueuedSynchronizer.java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
* 【3】主要是是对`state`做减发操作，默认传过来的`release`是1，如果减到`state`等于0则会将当前线程的锁释放掉。如果没有减完则继续保存`state`状态。

    ```java
    //ReentrantLock.java
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    ```
  
* 【4-7】在单线程的模式下。head一直是为空的，所以`unparkSuccessor`方法是进不去的。后面再分析。


