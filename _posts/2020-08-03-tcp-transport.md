---
layout:     post
title:      "2.TCP超时与重传"
subtitle:   "2.TCP超时与重传"
date:        2020-08-02
author:     "ZhaoLe"
# header-img: "img/redis/post-bg.png"
catalog: true
tags:
    - TCP/IP
---


### TCP超时与重传

TCP接收端返回发送端的一系列确认信息来判断是否出现丢包，当数据包和确认信息丢失，TCP则启动重传操作。TCP拥有两套独立的机制来完成重传一个是基于时间一个是基于确认信息构成的。
针对超时做出的处理 ，快速重传，重传超时RTO
针对丢失做出的处理，选择确认

### 基于计时器的重传(RTO)
由于TCP是适应不同的网络环境，所以正对于超时值的设定不可能是静态的，RTT(往返时间)和RTO(重传超时)会随着时间变化而变化的， 

**经典方法**
TCP在收到数据返回确认信息的时候会在信息中携带一个字节的数据来测量传输该确认信息所需要时间，称这个测量结果为SampleRTT
EstimatedRTT（TCP维持一个SampleRTT均值）
当一旦获取一个新的SampleRTT 就会根据下列公式来更新EstimatedRTT
>EstimatedRTT = (1 - α ) * EstimatedRTT + α  *  SampleRTT  \\
α = 0.125 （1/8）

**标准方法**
用于估算SampleRTT一般偏离EstimatedRTT 的程度
>DevRTT = （1-β）* DevRTT + β * | SampleRTT - EstimatedRTT | \\
β =  0.25 

经典方法适合相对稳定的网络环境，而标准方法适合运行在RTT变化较大的网络中。
最后需要注意的是，如果TCP先与RTT就触发重传 会出现不必要的重复数据。如果延迟大于RTT间隔重传数据，整体网络吞吐量会下降。

**TODO:后期对这些概念补充完**

### 快速重传
快速重传跟超时重传相比，更能有及时的修复丢包情况，
当时有失序报文被接受到的时候，TCP立刻生成确认信息(重复的ACK)，   当TCP等到一定数目的重复ACK，来决定数据是否丢失并触发快速重传。默认阈值是3次。
![1](/img/tcp/transport/1.png)

当不使用SACK的时候，在接收到有效的ACK前至多只能重传一个报文段。由于数据包2丢失，导致3，4，5返回的ACK都是重复的，接收端并不知道返回的ACK是是接收端谁发的，所以会重新再发送2，3，4，5..虽然TCP协议是累计确认，但是仍造成不必要的包发送了。

### SACK (Selective Acknowledgment)
为了解决一个RTT可能有多个空缺和尽可能的保证不重传正确接受的数据，引入了SACK.
这种方式是在TCP头的可选项里增加SACK，SACK包含已接收成功但由于非连续而未被确认的数据段的序列号范围列表。这样发送端就可以根据回传的SACK知道哪些数据到达了。哪些没有，这个协议需要接收方和发送方都要支持。
![1](/img/tcp/transport/2.png)

//TODO:个人不明白的问题
就是当成功接受ACK0和ACK1时候 发送方会继续发新的分组4，5.其中这部分跟[<The Guide TCP/IP> ](tcpipguide.com/free/t_TCPNonContiguousAcknowledgmentHandlingandSelective-4.htm)理解上有点冲突

### D-SACK(重复DSACK)
很多情况下没有出现数据丢失也会出现重传，是因为过早的判定超时导致的。引入D-SACK ，主要用于告诉发送方有哪些数据被重复接受了。

D-SACK使用了SACK的第一个段来做标志，
		* 如果SACK的第一个段的范围被ACK所覆盖，那么就是D-SACK
		* 如果SACK的第一个段的范围被SACK的第二个段覆盖，那么就是D-SACK

**发送重复数据**
```
 Transmitted    Received    ACK Sent
 Segment        Segment     (Including SACK Blocks)
 3000-3499      3000-3499   3500 (ACK dropped)
 3500-3999      3500-3999   4000 (ACK dropped)
 3000-3499      3000-3499   4000, SACK=3000-3500
```

接收端丢失了几个ACK包，发送端重新发送数据包(3000-3499) ，接收端发现是重复接受，由于累计确认 则返回ACK 4000 并且SACK 标记成3000-3500 ，说明这个包其实之前是已经接受到了，只不过是ACK 丢失。

**无序和重复的数据**
```
Transmitted    Received    ACK Sent
Segment        Segment     (Including SACK Blocks)
3000-3499      3000-3499   3500 (ACK dropped)
3500-3999      3500-3999   4000 (ACK dropped)
4000-4499      (data packet dropped)
4500-4999      4500-4999   4000, SACK=4500-5000 (ACK dropped)
3000-3499      3000-3499   4000, SACK=3000-3500, 4500-5000
```

接收端丢失了几个ACK包，并且有个数据包(4000-4499) 没有发送成功。再新的的数据包(4500-4999 )发送后，接收端返回最后一次收到的ACK，但是返回的ACK依旧丢失了。最后数据包(3000-3499)重传,因为累计确认，接收端返回ACK4000，SACK=3000-3500, 4500-5000意思是  3000-3500之前接受过，只不过是ACK丢失，4500-5000也已经被接收到。后续如果 继续重新发送数据包应该是 
```
4000-4499   4000-4499   5000, SACK = 4500-5000
```

**网络延时**
```
Transmitted    Received    ACK Sent
Segment        Segment     (Including SACK Blocks)
500-999        500-999     1000
1000-1499      (delayed)
1500-1999      1500-1999   1000, SACK=1500-2000
2000-2499      2000-2499   1000, SACK=1500-2500
2500-2999      2500-2999   1000, SACK=1500-3000
1000-1499      1000-1499   3000
               1000-1499   3000, SACK=1000-1500
                                 ---------
```

数据包(1000-1499 )由于网络延时没有收到ACK，等到重新发送的时候，接收端ACK已经返回3000，所以SACK=1000-1500 说明之前已经接受到了，只不过是ACK丢失而已
