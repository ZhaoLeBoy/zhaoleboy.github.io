---
layout:     post
title:      "3-2.Netty源码:NioEventLoop循环检测事件"
subtitle:   "NioEventLoop循环检测事件"
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
        switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
          ...
            case SelectStrategy.SELECT:
                //检测IO事件
                select(wakenUp.getAndSet(false));
          ...
        }
         ...
      } 
     ...
    }
  }
```

>因为这段代码过长，所以解释方法就直接写在代码里面了方便查看

```java
private void select(boolean oldWakenUp) throws IOException {
  Selector selector = this.selector;
  try {
      int selectCnt = 0;     //记录轮训次数
      long currentTimeNanos = System.nanoTime(); //当前时间
    
      long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

      for (; ; ) {
          if (selector.selectNow() > 0) {
              System.out.println("selectNow");
          }
          if (selector.select() > 0) {
              System.out.println("select");
          }
          // 本次select任务超时时间  500000L做个微调值   除1000000L 是为了纳秒转为毫秒
          long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;

          if (timeoutMillis <= 0) {//超时了
              if (selectCnt == 0) {//select次数是0
                  // 返回 Channel 新增的感兴趣的就绪 IO 事件数量。
                  selector.selectNow();//非堵塞的select方法 立即返回事件
                  selectCnt = 1;
              }
              break;
          }

          //wakenUp唤醒&& 任务队列是有任务
          if (hasTasks() && wakenUp.compareAndSet(false, true)) {
              // selectNow() 方法会检查当前是否有就绪的 IO 事件, 如果有, 则返回就绪 IO 事件的个数; 如果没有, 则返回0
              selector.selectNow();//非堵塞select方法
              selectCnt = 1;
              break;
          }

          //堵塞， 调用了 selector.select(timeoutMillis), 而这个调用是会阻塞住当前线程的, timeoutMillis 是阻塞的超时时间.
          int selectedKeys = selector.select(timeoutMillis);
          selectCnt++;

          //说明有任务或者有正在处理的任务，
          if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
              // - Selected something,
              // - waken up by user, or
              // - the task queue has a pending task.
              // - a scheduled task is ready for processing
              break;
          }

          //线程断了。。。
          if (Thread.interrupted()) {
              if (logger.isDebugEnabled()) {
                 ...
              }
              selectCnt = 1;
              break;
          }

          //到这个地方已经发生过堵塞操作 记录时间
          long time = System.nanoTime();

          //time - currentTimeNanos 小于TimeUnit.MILLISECONDS.toNanos(timeoutMillis)
          //完成一次堵塞操作的时间 - 轮训前的时间 < 任务超时时间   说明实际上没有发生堵塞 ，空轮训了。。
          /**
           * 大致可以这么看 ，执行select前记录下时间  30s ， 执行一个select时间要60s   , 执行select后 记录时间 50s 
           * [如果执行select后的时间]  - [执行selelct需要的时间] < [执行select前记录时间] ，说明了 这个select 操作是没有发生，空轮循了一次
           */
          if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
              selectCnt = 1;
          } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                  selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {//512次
              //大于一个阈值 默认512
              //老的select重新注册新的selector 防止空轮训
              rebuildSelector();
              selector = this.selector;

              selector.selectNow();
              selectCnt = 1;
              break;
          }

          currentTimeNanos = time;
      } 
    ...
}
```

* 【7】`delayNanos(currentTimeNanos)`获取延迟任务队列中第一个任务截止时间，如果返回时空则默认1s，selectDeadLineNanos是记录这次select操作的截止时间。
