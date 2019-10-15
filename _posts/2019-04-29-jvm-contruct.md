---
layout:     post
title:      "JVM内存结构"
subtitle:   "JVM内存结构"
date:        2019-04-29
author:     "ZhaoLe"
header-img: "img/jvm/gc/post-bg.png"
catalog: true
tags:
    - JVM
---
参考资料：

**《深入理解Java虚拟机——JVM高级特性与最佳实践(第2版)》**

>对jvm的学习笔记

# JVM内存结构

### 方法区
方法区是各个线程共享的内存区域，它用于存储已经被虚拟机加载的类信息，常量,构造函数，静态变量，及时编译器编译后的代码等数据，一般来说方法区上执行GC的情况非常少，但是不代表完全没有GC，GC主要是针对常量池的回收和已经加载类的卸载，虽然java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它有个别名叫：非堆，目的就是与java堆区分，jdk8之后叫Metasapce ，jdk6，7则是叫PermSpace

### 运行常量池 
是方法区的一部分，用于存储编译器生成的常量和引用，一般来说，常量的分配在编译时就能确定，但也不完全是，它可以存储运行期产生的常量，比如String类的intern(),作用是String类维护了一个常量池，如果调用的字符”hello”已经在常量池中，则直接返回常量池中的地址，否则新建一个常量加入池中，并返回地址


### 虚拟机栈 
虚拟机栈是线程私有的，它的生命周期跟线程是相同的，它描述的是java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧，用于存储局部变量表，操作数，动态链接，方法出口等信息，一个方法从调用到执行完成的过程，就对应一个栈帧在虚拟机栈中入栈到出栈的过程

### 本地方法栈
它与虚拟机栈的作用类似，它们之间的区别不过是虚拟机栈是虚拟机执行java方法(字节码)服务，而本地方法栈则为虚拟机使用到的Native方法服务
  
### 局部变量表
存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。
其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot，字节码中有体现），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

### 堆
堆是java虚拟机管理内存中最大的一块，堆是被所有线程功效的一块内存区域，在虚拟机启动时创建，此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在里面分配。java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可。栈是运行时的单位，而堆是存储的单位

### 程序计数器
JVM支持多线程同时执行，每个线程都有自己的程序计数器，线程正在执行的方法叫做当前方法，如果是java代码，计数器里存放当前正在执行的指令的地址。



