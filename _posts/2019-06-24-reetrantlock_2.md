---
layout:     post
title:      "2.ReetrantLock:多线程下的加锁过程"
subtitle:   "2.ReetrantLock:多线程下的加锁过程"
date:        2019-06-24
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - ReetrantLock
---

### 一个简单的demo

通过一个Demo来展示在多线程模式下ReetrantLock加锁和释放锁的过程

```java
//Demo
private static ReentrantLock lock = new ReentrantLock();

public static void main(String[] args) {
    Thread thread1 = new Thread(lockTask(),"线程一号");
    thread1.start();
    Thread thread2 = new Thread(lockTask(),"线程二号");
    thread2.start();
}


public static Runnable lockTask() {
    return () -> {
        while (true) {
            System.out.println(Thread.currentThread().getName() + "准备获取锁。。。");
            lock.lock();
            System.out.println(Thread.currentThread().getName() + "获取锁成功。。。");
            threadSleep();

            lock.unlock();
            System.out.println(Thread.currentThread().getName() + "的锁已经释放。。。");
        }
    };
}

private static void threadSleep() {
    try {
        Thread.sleep(5000L);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}

```
然后点击运行，可以看下控制台打印的结果，如果线程1先获取锁，那线程2线程就要等它先释放后才能持有。


### Lock

第一个线程获取锁的逻辑已经在[【1.ReetrantLock:单线程下的加锁过程】](http://jinlipool.com/2019/06/24/reetrantlock_1/)解释过了，现在主要说线程2在线程1没有释放锁的情况下发生了什么。

类调用：`NonfairSync#lock->AbstractQueuedSynchronizer#acquire`
```java
//AbstractQueuedSynchronizer.java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

* `tryAcquire(arg)`具体实现是在`ReentrantLock#nonfairTryAcquire`

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

   * 【5-10】第一个线程已经处于占有锁还未释放的状态 所以`state`已经是1 ，这个方法并不会进去
   * 【11-17】当前线程二在访问并且线程一已经持有锁，所以这个判断方法也不会进去。
   * 【18】`nonfairTryAcquire`返回false

* `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`把这个方法拆分成两个说。

    * `addWaiter(Node.EXCLUSIVE), arg)`
        
        ```java
          //AbstractQueuedSynchronizer.java
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
        新增个为`EXCLUSIVE`类型的Node节点,`waitStatus`为0，第一次进来这个链表结构还没有初始化,都是null，所以会直接进入到`enq(node)`。
        * 【5-11】如果不是第一次进来。此时已经有链表。这个方法会把新来的Node放到链表之后排队。
        
        ```java
        //AbstractQueuedSynchronizer.java
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
         这个方法主要做两件事,一个是初始化链表，二是将新的node节点插入到链表，使用的CAS的方式保证入队列方式的一致性。
         * 【3】算是个自旋，如果第一次进来会做初始化链表+挂node节点。
         * 【5】第一次进方法，链表没有初始化，这时候tail是null，要走一步初始化。
         * 【6-7】这个方法使用CAS方式将新建的Node节点赋值到`head`节点上，再将head节点赋值给tail，此时`tail = head`。 
         * 【9-13】初始化成功后会进入这个方法里面，将传入的Node节点设置为tail节点的后置，构成一个双向链表。
         
   * `acquireQueued`
   
       ```java
         //AbstractQueuedSynchronizer.java
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
      * 【7-13】参数Node是上一步刚组建好的的链表，首先从Node中获取上一个节点，判断是否是head节点（如果知道当前节点的上个节点是头节点，就说明当前的node链表中只有一个等待的线程）。然后在` tryAcquire(arg)`，由于在我们例子中，到这一步的时候线程一还是没有释放掉锁，所以这个判断里的代码是走不到的。这个判断里面的方法意思就是如果当前线程已经获取到锁，我们需要按照FIFO原则清除链表上的节点。
           
        ```java
        //AbstractQueuedSynchronizer.java
        private void setHead(Node node) {
            head = node;
            node.thread = null;
            node.prev = null;
        }
        ```
       
      * 【14-16】如果获取不到锁就会进这个方法，这里是实现排队阻塞等待的功能。这里还是分成两个方法来说明。
            
          * `shouldParkAfterFailedAcquire(p, node)`
               
               这里面要先说几个`waitStatus`状态。
               `SIGNAL`: waitStatus的状态值，-1，表明后面的线程需要当前线程唤醒。
               `CANCELLED`: waitStatus的状态值，1，标记这个线程已经取消了。
               
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
               * 【3-4】如果上个节点的`waitStatus`是`SIGNAL`说明这个线程可以直接被挂起
               * 【6-9】暂时没有调用。
               * 【11】改变`waitStatus`为`Node.SIGNAL`
               
          * `parkAndCheckInterrupt()`调用`UNSAFE.park`挂起线程。
            ```java
             public static void park(Object blocker) {
                Thread t = Thread.currentThread();
                setBlocker(t, blocker);
                UNSAFE.park(false, 0L);
                setBlocker(t, null);
            }
            ```

### unLock

```java
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
跟单线程的比多了对链表中下一个节点线的程进行唤醒
```java
 //AbstractQueuedSynchronizer.java
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
* 【3】获取节点的`waitStatus`状态。
* 【4-5】将node节点状态初始化为0
* 【8-12】没怎么看明白，而且debug模式也没进入。
* 【14-15】唤醒线程去获取锁。