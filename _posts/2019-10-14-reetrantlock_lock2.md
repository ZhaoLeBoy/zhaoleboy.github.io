---
layout:     post
title:      "2.ReentrantLock-独占锁"
subtitle:   "2.ReentrantLock-独占锁"
date:        2019-10-14
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - ReetrantLock 
    - Lock
---


```java
  lock.lock();
  ...  
  lock.unlock();
```
ReentrantLock默认锁的实现是是独占，非公平，可重入锁。

在ReentrantLock中内部有两个静态内部类`NonfairSync`和`FairSync`，集成`Sync`的抽象类，`Sync`是公平锁和非公平锁的一个子类，里面有尝试获取非公平锁的实现方式等等，它的子类就是`AbstractQueuedSynchronizer`。

继承关系如下(略有删除)。
![dfa58d6b1e9aea7a3ce214318fc049b0](/img/lock/reetrantlock/reentrantlock-exclusive.png)

代码就围绕非公平独占锁这块代码来说

##### 加锁过程

```java
//NonfairSync#lock
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

`AQS`中是根据`state`来判断锁的状态,独占锁中如果`state=0` 说明没有被占用，如果`state=1`，则说明锁已经被占用了，`compareAndSetState`用CAS方式将state标记占用判断然后对应走进不同判断分支。

* 1. `setExclusiveOwnerThread(Thread.currentThread());`锁没有被占用，则直接将当前线程设置为这个锁的独占线程。

* 2. `acquire()`锁已经被其他线程给占用了，则会在独占模式下尝试获取锁，通过至少一次调用`tryAcquire`获取锁成功，否则，线程将会进入排队(CLH双向列表)，并且反复堵塞直到调用`tryAcquire`成功。

```java
//AQS
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`tryAcquire`尝试获取锁，在AQS是个抽象方法，具体实现是在对应的类中,非公平锁的实现实在它子类`Sync`中。

* ```java
    //NonfairSync#tryAcquire 
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

    //Sync#nonfairTryAcquire 
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
    * 1. 这个方法比较简单，获取当前的线程。判断锁的状态。如果这时候锁没有被占用，则用CAS操作为当前线程加锁，
    * 2. 如果当前线程和占有锁的线程是同一个，则state计数器加一 返回获取锁成功，此处也说明ReentrantLock是可重入的。

`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`这个方法分两步。

* 由于在`acquire`这一步当前线程获取锁失败，所以先调用`addWaiter`方法将当线程封装为一个Node节点。

*   ```java
    //AQS
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    ```

一开始队列都是空的索引调用`enq()`进行初始化。 如果不是第一次进来并且CLH链表已经初始化完成。则将当前线程节点加作为tail加入到CLH链表中，方法如下:

*   ```java
    //AQS
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    ```

由于一开始head和tail都是null 所以会进入第一个逻辑块，将初始化好的Node节点使用CAS方式挂到链表的head节点。并将head赋值给tail.这个步骤就是初始化下CLH链表。
由于这个方法是死循环，紧接着进入else这个逻辑块，将当前线程节点作为tail加入到CLH中。

* `acquireQueued`自旋方式的尝试从链表中获取锁。

* ```java
     //AQS
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ```
    在循环中，先用`node.predecessor()`获取当前线程节点的前置节点。如果这个前置节点是head节点并且，当前线程在`tryAcquire()`(详细见之前的说明)方法中获取锁成功。这一行的判断说明之前持有锁的线程已经释放掉了，并且当前线程已经获取锁成功了，算是一个预防式判断。由于当前节点线程获取锁成功，就把当前节点设置为head节点(由此可见，CLH链表中的Head节点就是已经获取锁的节点)，把前置节点的的next置为null是为了方便GC回收。

   * 如果当前线程不是head节点或者当前线程获取锁失败了则会走到后面的逻辑`shouldParkAfterFailedAcquire(p, node)`

        ```java
        private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            int ws = pred.waitStatus;
            if (ws == Node.SIGNAL)
                return true;
            if (ws > 0) {
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                pred.next = node;
            } else {

                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }
        ```
        这个方法主要是在获取锁失败后检查和更新节点中的状态。如果线程应该被堵塞则返回true，在所有获取锁循环中它是主要的信号控制方法.
        `此处判断都是对前置节点的状态判断`
        `SIGNAL`前置节点的线程正在持有锁，当前线程应当被堵塞。
        `CANCEL（ws>0）`前置节点线程有异常(中断或者超时)要被剔除。
        `0或者PROPAGATE`这种状态前置节点需要跟新`waitStatus`，使用CAS方式跟新为`SIGNAL`但此时并不堵塞当前线程，等在下次外层循环的时候让前置节点进行尝试获取锁的判断，如果前置节点线程还是获取不到锁，则堵塞当前节点线程。这么做个人猜测一是为了规范代码编程，因为这个方法主要就是对信号状态的处理，二是为了线程更及时获取到锁，三是因为堵塞耗费资源，为了减少资源消耗

    * `parkAndCheckInterrupt()`如果前置节点的`waitStatus==SIGNAL`则将当前节点中的线程堵塞。返回当前线程的中断状态`Thread.interrupted()`。

* 最后回到之前方法
    ```java
    //AQS
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    ```
    如果当前线程获取锁失败并且线程处于了中断这个状态，则调用`selfInterrupt()`中断当前线程。
    上述对中断处理不敏感，也就是说由于线程获取锁的时候出现中断,这个节点仍然CLH同步队列中，后续对线程进行中断操作时也不从同步队列中移除。


##### 释放锁过程

```java
//AQS
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

`tryRelease`是在Sync这个抽象类中实现，主要就是做计数器减法，默认是对一个线程进行释放锁
```java
//ReentrantLock
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
就是对`state`进行减法操作。先做一个预防判断，如果当前的线程不是获取锁的线程则抛出异常。
拿`state`进行减法操作，如果是0说明锁可以释放了,进行标记，如果不等于0说明还有线程再持有。最后保存`state`的结果。

上述操作如果返回的结果是当前锁可以释放了则进行后续操作。也是先预判断当前head节点(就是持有锁的线程节点)不能为空,并且这个节点的`waitStatus`是无状态也就是0,然后执行后续节点唤醒的操作。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
再次线程释放锁的节点一定是head节点。如果节点中`waitStatus`仍然是小于0，则先将它置为无状态`0`。
获取head节点的下一个节点，如果节点存在则直接` LockSupport.unpark(s.thread);`唤醒这个节点线程。
但如果这个节点被取消或明显无效，则从尾部向后遍历以找到实际的未被取消的节点。最后调用`LockSupport.unpark(s.thread)`方法唤醒该线程。

---

>为什么不是从head开始遍历

[《The java.util.concurrent Synchronizer Framework》 JUC同步器框架（AQS框架）原文翻译](!https://www.cnblogs.com/dennyzhangdd/p/7218510.html)
[Java AQS unparkSuccessor 方法中for循环从tail开始而不是head的疑问？](!https://www.zhihu.com/question/50724462)

论文中有段话很重要，我就直接抄中文翻译了。
`AQS队列的节点包含一个next链接到它的后继节点。但是，由于没有针对双向链表节点的类似compareAndSet的原子性无锁插入指令，因此这个next链接的设置并非作为原子性插入操作的一部分，而仅是在节点被插入后简单地赋值：pred.next = node; next链接仅是一种优化。如果通过某个节点的next字段发现其后继结点不存在（或看似被取消了），总是可以使用pred字段从尾部开始向前遍历来检查是否真的有后续节点。`

源码中很多地方都有`pred.next = node`,这个操作不是原子性。所以在多线程下会出问题。
举个例子，就是初始化列表这块`addWaiter()`
```java
//AQS
private Node enq(final Node node) {
    ...
    node.prev = t;
    if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
    }
}
```
如果在CAS操作后，节点赋值之前，`unparkSuccessor`中从head开始循环就会出现next节点为空或者被取消的节点，所以采用tail反向循环找到一个可用的线程。最后调用LockSupport的unpark(Thread thread)方法唤醒该线程。







