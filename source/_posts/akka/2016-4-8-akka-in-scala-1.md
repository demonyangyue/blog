---
layout: post
title: Akka in Scala Part 1: Java is Shit
date: 2016-04-08
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## Java 之殇

我无意挑起编程宗教间的争论，仅仅想阐述一个简单的事实 -- Java is shit.

你从最优秀的Java书籍中学到了解耦接口与实现，于是开始热衷于设计出各种各样的接口，然而这些接口要么支持的方法太少，沦为几乎无用的鸡肋，要么需要实现方法太多，给子类造成沉重的负担。Java还慷慨地给了你额外的’奖励’ -- 同样的方法声明在各个子类中需要一遍又一遍地重复，如同遭受了计算机世界中恶灵的诅咒。

你从最优秀的Java博客中掌握了设计模式，开始实现各种各样的单件类，如同发现新大陆一般对访问权限控制，枚举变量之类的奇技淫巧兴奋不已。然而所有的设计模式，背后都对应着现实中的高频需求，在Java的内在世界，这些需求却被无情地置于荒漠之中。

Java最大的魅力在于，这门语言总在试图表达正确的概念，却总能展现丑陋的实现，这一点在并发编程领域，体现得淋漓尽致。

## 无尽的绝望

Java 并发基于线程和锁模型，相对于多进程，更加轻量并且易于数据共享。

然而 **共享可变状态**， 这个并发世界的魔鬼一旦从瓶中放出，厄运的齿轮开始不停转动。

我们创建一个最简单计数器类，为了支持多个线程同时进行读写，于是引入synchoronized进行同步：

```java
class Counter {
    private String name = "";
    private int count = 0;

    public String getName(){ return name; }
    public synchronized void setName(String n){ name = n; }

    public synchronized void increase() { ++count; }
    public int getCount() { return count; }
}
```

看起来一切都很美好，多个线程可以同时读取数据，提高了效率，写操作进行了同步，于是不会有竞争。

### 乱序执行

为了提高代码运行的效率，编译器会做静态优化，JVM会做动态优化，这些优化可能会打乱代码的执行顺序：

```java
class Hell {
    private static Counter counter = new Counter();

    static Thread t1 = new Thread() {
        public void run() {
            counter.increase();
            counter.setName("counter");
        }
    };

    static Thread t2 = new Thread() {
        public void run() {
            if (!counter.getName().isEmpty()) {
                System.out.println(counter.getName() + " now is " + counter.getCount());
            } else {
                System.out.println("The counter doesn't have a name yet");
            }     
        }
    };

    public static void main (String [] args) throws InterruptedException {
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
```

输出可能是"counter now is 0", 看起来像是`set_name()`方法在`increase()`之前被调用。

### 内存可见性

比乱序执行更糟的是，一个线程的修改会对令一个线程不可见： 

```java
static Thread t2 = new Thread() {
    public void run() {
        while (counter.getName().isEmpty()) {
            System.out.println("The counter doesn't have a name yet");
        } 
    }
}

```

在一个线程中的修改可能在另一个线程中不可见，导致这个循环可能永远无法停止。

### Java内存模型
为了解决乱序执行和内存可见性的问题，依据Java内存模型(Java Memory Mode, JMM) [JSR 133](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html) 的定义, 我们需要把读操作和写操作都加锁进行同步。

```java

    public synchronized String getName(){ return name; }
    public synchronized void setName(String n){ name = n; }

    public synchronized void increase() { ++count; }
    public synchronized int getCount() { return count; }
```

但是随着越来越多的锁被加入，性能会变得越来越差，讽刺的是，我们引入并发的初衷是为了提高性能。

竞态条件和内存可见性并不是最难缠的问题，真正的灾难在于 -- 死锁。

### 死锁


按照CSAPP的解释，当多个线程以错误的顺序获取和释放资源时，就会造成死锁。

![](/images/deadlock.png)

一个典型的例子是[哲学家进餐问题](https://www.cs.mtu.edu/~shene/NSF-3/e-Book/MUTEX/TM-example-philos-1.html)。

解决的方案是按全局一致的顺序进行加锁和解锁，

但是现实情况是，系统通常比较复杂，我们根本无从知晓全局顺序。

绝望么？

还有更难缠的，第三库提供的API，由于我们并不清楚其内部实现细节，当这些API内部使用锁时，我们对此一无所知。如果我们自己的synchronized代码块内部调用了这些API，就会在毫不知情的情况下使用了两把锁，这就有可能发生死锁。

现在，绝望么？

## 反思

你可以使用java.concurrent.util 之类的帮助解决某些特定的问题，但是核心的缺陷——**共享可变状态**， 并没有被真正克服。

线程，锁，信号量，条件变量，关于并发编程我们已经学习了很多，最宝贵的经验是——并发编程很难，难以开发，[难以测试](http://stackoverflow.com/questions/12159/how-should-i-unit-test-threaded-code)，难以重构。

编写并发程序的时候，我们经常处于这样一种怪圈：

1. 想让程序执行得更快，所以增加并发。
1. 发现各种bug，于是增加同步机制来保证确定性。
1. 于是陷入了串行和等待，发现程序性能下降。
1. 循环步骤1

性能的损失并不是最致命的，最大的困难是存在太多的陷阱，大部分时候我们已经深陷其中却不自知，最终我们发现几乎无法写出完全正确的程序。

并发程序之所以难，是因为**错误**的工具为我们提供了**错误**的抽象。

传统的并发模型与人类的认知模型格格不入。人的大脑本质上是单线程的，我们无法在同一时刻同时做两件事 - 比如左手画方，右手画圆，并且当人们专注于一件事，却突然被另外一件事打断时，上下文切换的成本是很高的。编程的本质是把世界运行的逻辑翻译给计算机听，当我们试图通过线程和锁编程时，实际上是在以一种违反认知模型的方式去对事情进行建模，注定要饱受摧残。

## The Akka Way

不同于传统的基于共享状态的并发模型，akka基于消息传递模型实现并发，并提供了一组强大的工具，将我们从Java地狱中拯救出来。

Akka是Scala的标准并发库，scala这类函数式编程语言的基石是不可变量(immutable value)，从根本上杜绝了可变状态。Akka 鼓励使用不可变量，但如果不可避免地要使用变量，那么这个变量会被隐藏在actor内部，只能通过消息传递来获取或改变其状态，而不和外界共享，保证了不会落入传统并发编程**共享可变状态**的陷阱。

让我们开启Akka的探索之旅吧！
