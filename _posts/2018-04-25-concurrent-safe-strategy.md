---
layout:     post
title:      "三.线程安全策略"
subtitle:   "线程安全的一些信息"
date:       2018-04-25 21:00:00
author:     "ZhaoLe"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java
    - 高并发
---


参考慕课网[并发课程](https://coding.imooc.com/class/195.html)做的笔记

### 不可变对象

* 不可变的对象需要满足的条件

	1. 对象创建以后期状态就不能改变。
	2. 对象所有域都是final类型
	3. 对象是正确创建的(对象创建期间，this引用没有逸出)

* 创建不可变对象的方式（参考String类型）

	1. 将类声明成final类型，使其不可以被继承
	2. 将所有的成员设置成私有的，使其他的类和对象不能直接访问这些成员
	对变量不提供set方法
	3. 将所有可变的成员声明为final，这样只能对他们赋值一次
	4. 通过构造器初始化所有成员，进行深度拷贝
	5. 在get方法中，不直接返回对象本身，而是克隆对象，返回对象的拷贝

* final关键字：类、方法、变量

	1. 修饰类：不能被继承（final类中的所有方法都会被隐式的声明为final方法）。
	2. 修饰方法：1、锁定方法不被继承类修改；2、提升效率（private方法被隐式修饰为final方法）。
	3. 修饰变量：基本数据类型变量（初始化之后不能修改）、引用类型变量（初始化之后不能再修改其引用）。

* 除了final 还有其他可以定义不可变的方法
	
	1. Collections.unmodifiabnleXXX:Collection,List,Set,Map...
	
	2. Guava:ImmutableXXX: Collection,List,Set,Map...


### 线程封闭
把对象封装到一个线程里，只有这个线程能看到这个对象

* 实现线程封闭
	1. Ad-hoc 线程封闭，程序控制实现 最糟糕
	2. 堆栈封闭:局部变量，无并发问题
	3. ThreadLocal线程封闭:特别好的封闭方法(可以看下源码)

### 线程不安全类的写法
如果一个类的对象可以被多个线程访问，不做特殊的同步和并发处理，很容表现出异常

1. StringBuilder 线程不安全，StringBuffer线程安全
原因:StringBuffer几乎所有的方法都加了synchronized关键字
2. SimpleDateFormat 在多线程共享使用的时候回抛出转换异常，应该在用堆栈封闭在每次调用方法的时候在方法里创建一个SimpleDateFormat,joda-time的DateTimeFormatter是线程安全的
3. ArrayList,HashMap,HashSet等Collections线程不安全
4. 先检查再执行这种方式线程不安全 


```java
// 因为可能不是非原子性(如果是在多线程下 要不然加锁 要不然操作是原子性)
if(condition(a)){
  handle(a);
}
```

### 线程安全-同步容器

**ArrayList -> Vector,Stack**

**HasMap -> HashTable(key,value不能为null)**

**Collections.synchronizedXXX(List,Set,Map)**

但是使用同步容器的情况下 也不一定能做到线程安全（见代码）同步容器性能可能会稍差 而且不一定会保证性能安全,所以一般会使用并发容器来代替同步容器。

```java
/**
 * 并发测试
 * 同步容器不一定线程安全
 * @author gaowenfeng
 */
@Slf4j
@NotThreadSafe
public class VectorExample2 {

    /** 请求总数 */
    public static int clientTotal = 5000;
    /** 同时并发执行的线程数 */
    public static int threadTotal = 50;

    public static List<Integer> list = new Vector<>();

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            list.add(i);
        }
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                list.remove(i);
            }
        });

        Thread thread2 = new Thread(() -> {
            // thread2想获取i=9的元素的时候，thread1将i=9的元素移除了，导致数组越界
            for (int i = 0; i < 10; i++) {
                list.get(i);
            }
        });

        thread1.start();
        thread2.start();
    }

}
```

```java
@Slf4j
@NotThreadSafe
public class VectorExample3 {
    //java.util.ConcurrentModificationException
    //对集合遍历的时候 有对集合进行增删的操作 会报异常
    //如果要做remove操作 可以遍历的过程中先标记下 再遍历完后再操作
    private static void test1(Vector<Integer> v1) { //foreach
        for (Integer i : v1) {
            if (i.equals(3)) {
                v1.remove(i);
            }
        }
    }

    //java.util.ConcurrentModificationException
    private static void test2(Vector<Integer> v1) { //iterator
        Iterator<Integer> iterator = v1.iterator();
        while (iterator.hasNext()) {
            Integer i = iterator.next();
            if (i.equals(3)) {
                v1.remove(i);
            }
        }
    }

    //success
    private static void test3(Vector<Integer> v1) {
        for (int i = 0; i < v1.size(); i++) {
            if (v1.get(i).equals(3)) {
                v1.remove(i);
            }
        }
        log.info("size--{}",v1.size());
    }

    public static void main(String[] args) {
        Vector<Integer> vector = new Vector<>();
        vector.add(1);
        vector.add(2);
        vector.add(3);
        test3(vector);

    }
}

```