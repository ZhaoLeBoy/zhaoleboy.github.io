---
layout:     post
title:      "1.ReetrantLock-加锁过程"
subtitle:   "1.ReetrantLock-加锁过程"
date:        2019-10-14
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - ReetrantLock
---

之前写过两篇文章,是关于ReentrantLock。
* [1.ReetrantLock:单线程下的加锁过程](http://jinlipool.com/2019/06/24/reetrantlock_1/)
* [2.ReetrantLock:多线程下的加锁过程](http://jinlipool.com/2019/06/24/reetrantlock_2/)

总感觉还差点什么，放了许久后，再次进入源码中学习，这次又有更深的理解,准备换个角度从纯流程开始描述。

##### 概念

AQS 全称是 AbstractQueuedSynchronizer，顾名思义，是一个用来构建锁和同步器的框架，它底层用了 CAS 技术来保证操作的原子性，同时利用 FIFO 队列实现线程间的锁竞争，将基础的同步相关抽象细节放在 AQS，这也是 ReentrantLock、CountDownLatch 等同步工具实现同步的底层实现机制。它能够成为实现大部分同步需求的基础，也是 J.U.C 并发包同步的核心基础组件。

##### 加锁过程描述
先大概用文字描述下加锁的过程，建立个大概的印象，然后会跟着代码详细解释。

Reentrant实现的锁默认是非公平的，具体是在`NonfairSync`中实现，lock加的是独占锁(互斥锁)。

```java
//NonfairSync#lock
final void lock() {
if 
    setExclusiveOwnerThread(Thread.currentThread());
else
    acquire(1);
}
```
首先会去判断同步锁状态(state)，如果这个同步锁没有被占用，则当前的线程(这里称为threadB)可以直接持有。如果被threadA持有了，threadB会进入下一步尝试去获取锁。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
获取锁的方式有两种，一种是根据threadB的情况来判断，还有一种是从CLH链表中获取。如果threadB获取锁失败但是从CLH链表中获取到锁了，则需要将threadB标记中断。//TODO.........

* tryAcquire(arg)中干的事是，判断同步锁状态，如果这个时候同步锁没有被持有，可以直接让threadB直接持有。如果threadB 跟已经持有锁的threadA是同一个线程，直接计数器+1(可重入)，如果上述的条件都不满足返回false。

* acquireQueued(addWaiter(Node.EXCLUSIVE), arg)两个方法组成。
    * `addWaiter`用来将threadB封装成Node(以下称为NodeB)，并将它装载到CLH双向链的尾部
    * `acquireQueued`不间断的尝试从chl队列中获取独占锁。

        ```java
        final boolean acquireQueued(final Node node, int arg) {
           ...
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            ...
        }
       ```

       1.  如果threadB的`前一个节点`（以下称为NodeA）是head节点，并且再次尝试获取锁（tryAcquire(arg)）成功了(说明NodeA节点的threadA已经执行完，释放锁了)，则把threadB设置为head节点（由此可见，CHL链表中的头结点是持有锁的那个线程）并且返回true  
     
        ```java
        final boolean acquireQueued(final Node node, int arg){
            ...
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
               interrupted = true;

            finally {
                if (failed)
                    cancelAcquire(node);
            }
        }
        ```
      
        2. 在上述尝试获取锁失败后会检测是否挂起threadB。     
              * `shouldParkAfterFailedAcquire`根据NodeA节点(注意是NodeB的前一个节点)的`waitStatus` 对节点进行处理，如果标记为`SIGNAL`说明NodeA处于持有锁的状态，返回true，说明NodeA中的线程正在持有锁，threadB可以被挂起。如果是`CANCELLED`则删除这个节点。如果是`0`或者`PROPAGATE`则将其设置为为`SIGNAL`。但此时返回的是false，threadB可以不被挂起，原因是只是刚改了NodeA的状态，但不确定NodeA中线程能不能获取到锁(多线程竞争锁的时候容易出现这种事情)，需要再次循环重试。
              * `parkAndCheckInterrupt` 先挂起当前线程,返回该线程是否中断的状态。如果false说明threadB没有中断。否则标记下interrupted=true,最后回到`acquire()`方法中，由于threadB是中断状态，所以调用`selfInterrupt()`中断threadB。其实中断问题也不大，因为后面会再唤醒的。
              *  对出了异常的节点。会将节点标记为`CANCEL`，在下一次循环的`shouldParkAfterFailedAcquire`方法中剔除。


  
