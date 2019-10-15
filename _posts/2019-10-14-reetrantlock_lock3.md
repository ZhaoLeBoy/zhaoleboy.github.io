---
layout:     post
title:      "3.ReentrantLock-公平锁"
subtitle:   "3.ReentrantLock-公平锁"
date:        2019-10-14
author:     "ZhaoLe"
header-img: "img/lock/reetrantlock/post-bg.png"
catalog: true
tags:
    - ReetrantLock
---

ReentrantLock默认是用非公平锁，具体文章见[2.ReentrantLock-独占锁](http://jinlipool.com/2019/10/14/reetrantlock_lock2/)
那么公平锁是怎么实现的，它跟非公平锁到底有什么区别。
其实公平锁和非公平锁的实现几乎一样。只有在加锁有一出不同。公平锁会多出一个`hasQueuedPredecessors()`方法。如下图，左边是公平锁获取方式，右边是非公平锁获取方式。
![dd1c6b8ee22bff9043e9072e33f48892](/img/lock/reetrantlock/reentrantlock-code.png)

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
首先去判断有没有其他线程比当前线程等待的事件更长。
 1. head和tail不相等说明有等待的线程。
 2. head.next为空 说明可能有线程在入队(在[2.ReentrantLock-独占锁](http://jinlipool.com/2019/10/14/reetrantlock_lock2/)中有说明，是由于有节点要挂在链表上的操作不具备原子性导致的。)
 3. 当前线程不等于head.next节点的线程说明队列里面还是有多个节点


综上，公平锁就是通过同步队列来实现多个线程按照申请锁的顺序(FIFO原则)来获取锁，从而实现公平的特性。非公平锁加锁时不考虑排队等待问题，直接尝试获取锁，所以存在后申请却先获得锁的情况。

在非公平锁中，由于刚释放锁的线程再次获取同步状态的几率会非常大，所以使得其他线程只能在同步队列中等待。公平性锁保证了锁的获取按照FIFO原则，而代价是进行大量的线程切换。非公平性锁虽然可能造成线程“饥饿”，但极少的线程切换，保证了其更大的吞吐量。