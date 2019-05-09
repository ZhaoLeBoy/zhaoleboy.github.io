---
layout:     post
title:      "JVM垃圾收集器"
subtitle:   "JVM垃圾收集器"
date:        2019-04-29
author:     "ZhaoLe"
header-img: "img/jvm/gc/post-bg.png"
catalog: true
tags:
    - JVM
---

参考资料：

**《深入理解Java虚拟机——JVM高级特性与最佳实践(第2版)》**

**[Java Hotspot G1 GC的一些关键技术](!https://zhuanlan.zhihu.com/p/22591838)**

---
>对jvm的学习笔记


JVM使用可达性分析算法来判断对象是否存活。
可达性分析算法中不可达的对象也并非是”非死不可“，这时候他们暂时处于缓刑阶段，要真正宣告一个对象死亡，至少要经历两次标记过程，如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选。筛选条件是此对象是否有必要执行finalize方法，当对象没有覆盖finalize方法，或者finalize方法已经被虚拟机调用过，虚拟机将这两种情况都被视为没有必要执行。

## 垃圾回收算法

#### 标记清除
算法分两个阶段，`标记`和`清除`，首先标记出所有需要回收的的对象，在标记后统一回收所有。

**缺点：**
效率不高，标记和清除的两个过程效率都不是很高。产生碎片，碎片太多会提前导致GC，所以针对这两点，产生了下面两种算法。

#### 标记整理
标记过程跟`标记清除`算法一样的，但后续步骤不是直接对可回收对象进行清理，而是让所欲存活的对象向一端移动，然后直接清理掉边界以外的内存。

**优缺点：**
没有内存碎片，但是整理比较耗时。

#### 复制
将可用内存按容量划分大小相等的两块，每次只使用其中一块，当这一块的内存用完了将还存活的对象复制到另外一块。再把已经使用过的内存空间一次性清理掉。

**优缺点：**
实现简单，运行高效，空间利用率低

---
Young区使用的是复制算法。
Old区用标记清除或者标记整理。

## 垃圾收集器
---

##### 垃圾回收器分类
* 串行收集器：Serial,Parallel New(Serial的多线程版本) ,Serial Old
* 并行收集器：Parallel Scavenge, Parallel Old
* 并发收集器：CMS,G1 

##### 并行和并发的区别
* 并行，多条来及收集线程并行工作，但用户线程依然处于等待状态
* 并发，用户现场与垃圾线程同时执行，但不一定是并行，可能会交替执行，用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。

##### 垃圾回收期按照下图的方式搭配使用：
![7bd0f81a22faa532125a7bb075d78a90](/img/jvm/gc/gc1.png)


##### Serial收集器
单线程收集器,采用复制算法。进行垃圾回收的时候必须停止其他所有工作线程（stop the world）

##### ParNew收集器
Serial收集器的多线程版本（复制算法）使用多条线程进行垃圾收集

##### Parallel Scavemge收集器
采用标记整理算法，并行多线程，目的是达到一个可控制的吞吐量，吞吐量 = 运行用户代码时间 / （运行用户代码时间+垃圾收集时间） 适合后台运算而不需要太多交互的任务

##### Serial Old 
采用标记整理算法 单线程收集器。

##### Parallel Old
采用标记整理算法，JDK1.6之后才出现

##### CMS 收集器
采用标记清除算法，以获取最短停顿时间为目标的收集器。  

整个过程分为四个步骤：
1. 初始标记：仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，需要“Stop The World”。
2. 并发标记：进行GC Roots Tracing的过程，在整个过程中耗时最长。
3. 重新标记：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。此阶段也需要“Stop The World”。
4. 并发清除

步骤1和3 仍然需要stw 步骤1仅仅标记下GC ROOTS能直接关联到的对象，并发标记是进行GC ROOTS Tracing过程，步骤3是为了修正并发标记期用户程序继续运作而导致的标记产生变动的那一部分对象的标记记录，这个阶段停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。步骤2，4可以与用户线程一起工作。

**CMS的缺点**
1. CPU敏感，默认回收线程是(cpu数量+3）/4,也就是当CPU 4个以上，并发回收垃圾收集线程不少于25%的CPU资源，随着CPU数量增加而下降。
2. 无法处理浮动垃圾,cms预留一部分空间提供并发收集时的程序运作使用，如果应用中老年代增长不是太快，可以适当提高参数，`-XX:CMSInitiatingOccupancyFraction` 的值来提高触发百分比，以便降低内存回收次数来获取更好性能。要是CMS预留的内存无法满足程序需要，就会出现一次`Concurrent Mode Failure`失败，虚拟机启动后备预案，临时启动`Serial Old`收集器从新进行老年代收集,这样停顿时间就更长了，所以`-XX:CMSInitiatingOccupancyFraction`设置太高，会很容易导致大量`Concurrent Mode Failure`失败。性能反而下降。

3. 有大量空间碎片，碎片过多会给大对象分配带来很大麻烦，往往老年代还有很大空间剩余，但是无法找到足够大连续空间来分配当前对象 不得不触发一次full GC

`-XX:UseCMSCompactAtFullCollection`开关参数（默认开启）用于cms收集器顶不住要进行FullGC时候开启内存碎片的合并整理过程，合并整理过程是无法并发的，碎片没有了但是停顿时间长了，所以还有另外个参数`-XX：CMSFullGCsBeforeCompaction`，这个参数用于设置执行多少次不压缩的FullGC 跟着来一次带压缩的（默认0，表示每次进入FullGC都进行碎片整理）

##### G1 收集器

G1将内存划分为多个大小相等的内存块（Region），每个Region是逻辑连续的一段内存。

**特点**
1. 并行与并发：G1能充分利用CPU来缩短stw的事件。
2. 分代收集：新生代和老年代都是使用G1
3. 空间整合：基于标记整理算法实现，从局部(Region)来看，基于复制算法实现。这两种算法意味着G1运行期间不会产生太多空间碎片，
4. 可预测停顿：G1和CMS关注点都是降低停顿时间，除此之外，G1还可以建立可预测的停顿时间模型。

G1将hava堆划分多个大小相等的独立区域(Region),每个Region被标记了E、S、O和H，说明每个Region在运行时都充当了一种角色，其中H代表Humongous，这表示这些Region存储的是大对象（humongous object，H-obj），当新建对象大小超过Region大小一半时，直接在新的一个或多个连续Region中分配，并标记为H。
   
**H-obj特点：**

1.  H-obj直接分配到了old gen，防止了反复拷贝移动。
2.  H-obj在global concurrent marking阶段的cleanup 和 full GC阶段回收。
3.  在分配H-obj之前先检查是否超过 initiating heap occupancy percent和the marking threshold, 如果超过的话，就启动global concurrent marking，为的是提早回收，防止 evacuation failures 和 full GC。     

    
G1之所以能建立停顿时间模型，是因为它有计划的避免在整个java堆中做全区域垃圾回收。G1跟踪各个Region里面垃圾堆积的价值大小（回收获得的空间大小和回收所需要的时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间（用户可自行设置`-XX:MaxGCPauseMillis`），优先回收价值最大的Region。

**Remembered Set概念：**
G1收集器中，Region之间的对象引用以及其他收集器中的新生代和老年代之间的对象引用是使用Remembered Set来避免扫描全堆。G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序对Reference类型数据进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于不同的Region之间(在分代中例子中就是检查是否老年代中的对象引用了新生代的对象)，如果是便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set中。当内存回收时，在GC根节点的枚举范围加入Remembered Set即可保证不对全局堆扫描也不会有遗漏。

G1中提供了两种模式垃圾回收模式，young gc、mixed gc ，在不同的条件下被触发。young，mixed GC 两种都是完全STW的。

* Young GC：选定所有年轻代里的Region。通过控制年轻代的region个数，即年轻代内存大小，来控制young GC的时间开销。
* Mixed GC：选定所有年轻代里的Region，外加根据global concurrent marking统计得出收集收益高的若干老年代Region。在用户指定的开销目标范围内尽可能选择收益高的老年代Region。

Mixed GC 不是full GC 它只能回收部分老年代的Region，如果mixed GC实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行Mixed GC，就会使用serial old GC（full GC）来收集整个GC heap。所以我们可以知道，G1是不提供full GC的。

上文中，多次提到了global concurrent marking，它的执行过程类似CMS，但是不同的是，在G1 GC中，它主要是为Mixed GC提供标记服务的，并不是一次GC过程的一个必须环节。global concurrent marking的执行过程分为四个步骤：

1. 初始标记：它标记了从GC Root开始直接可达的对象。
2. 并发标记：这个阶段从GC Root开始对heap中的对象标记，标记线程与应用程序线程并行执行，并且收集各个Region的存活对象信息。
3. 最终标记：标记那些在并发标记阶段发生变化的对象，将被回收。
    > 虚拟机将这段时间对象变化记录在线程Remembered Set Logs 中，最终标记阶段需要吧Rememberered Set Logs的数据合并到Remembered Set中。
4. 筛选回收。并发清除空Region（没有存活对象的），加入到free list。
    > 筛选回收阶段对各个Region的回收价值和成本进行拍讯，根据用于所期望的GC停顿时间来制定回收计划。