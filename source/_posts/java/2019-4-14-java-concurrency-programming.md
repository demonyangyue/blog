---
layout: post
title: Java并发编程
date: 2019-04-14
categories: Programming
tags: [Java, Concurrent Programming]
---

## 序言

过去的数十年，摩尔定律是推动IT行业发展的底层引擎，`集成电路芯片上所集成的电路的数目，每隔18个月就翻一倍`， 计算能力的提升使得许多想象不到的场景成为了现实。

然而随着硅片线路密度的增加，其复杂性和差错率也呈指数增长，到达纳米级别时，材料的物理、化学性能将发生质的变化，摩尔定律也就要走到尽头。

当一匹马拉不动车的时候，我们不是去换一匹更强壮的马，而是用四匹马来拉。近年来，我们不再拼命提升单个CPU的计算能力，而是靠多个CPU或者多台计算机(分布式集群)同时计算，来获得更强大的计算能力。为了驱动多CPU同时计算， 我们需要具有并发能力的程序。

<!--more-->

但是世界上没有免费的午餐，要获得更强的计算能力，也要付出相应的代价，并发程序要比单线程程序难上一个数量级，主要体现在:

- 难以理解

  多核计算机可以很好地并发执行任务，和计算机模型不同，人类的大脑，其思维方式，天然是串行的。`左手画圆，右手画方`，在人类大脑的控制下是做不到的，只能依靠肌肉记忆实现。

- 难以开发

  操作系统底层的并发模型，与语言框架提供的并发语义之间，并不是无缝连接的，开发者一不小心，可能会掉入那些潜在的深渊。

- 难以测试

  由于并发任务本质的复杂性，并发程序也是难以测试的。你无法知道线程之间会以怎样的顺序执行，会在哪个资源上面产生竞争，之前正常运行了很久的并发程序，有可能在某个深夜莫名崩溃。

还记得第一次遇到并发Bug的无助感吗？

本文主要讨论如何在Java项目中编写**正确**的并发程序。

## 并发的困境

并发程序需要面对两类问题: 线程安全性和线程活跃性，它们互为硬币的两面。

线程安全性代表了并发程序的正确性，指的是在多线程环境下，应用程序始终能够表现出正确的行为。

线程活跃性代表了并发程序的执行效率，指的是引入多线程后，比原来的单线程程序降低了多少执行时间。

在并发程序中，遇到资源竞争时，为了保证线程安全性，通常会引入某种同步手段，保证任意时刻只有一个线程访问资源。但是这种情况下，其它线程会因为等待资源而被挂起，延长总体执行时间，可能会引起线程活跃性问题。我们引入并发程序的目的是为了提高程序的执行性能，活跃性问题让我们与目标背道而驰。

## 灾难从何而来

### 线程安全性问题的根源

所有的线程安全性问题，都可以归结于同一个原因: 共享的可变状态。

首先来看状态的共享，在Java中，如果类的某个域被声明为public，或者通过public方法返回了某个private域的引用，那么这个域就可以被其它对象访问到，可以认为基于该类创建的对象，共享了其状态。

一旦状态被共享，宿主对象就失去了状态的完全控制权，你无法预知其它对象会对共享状态做怎样的误操作。

接下来看状态的可变。`变量`是Java世界中最常见的公民，绝大多数的对象都是可变的。在并发代码中，状态的可变性会面临3个问题:

- 原子性

  修改变量状态的单次操作，未必是原子的。在线程A修改变量的过程中，线程B可能看到的是状态修改到一半的变量。比如以下的请求统计程序，就是非线程安全的，因为自增(++)操作是非原子的。

  自增操作包含了三步子操作: 从内存中读出当前值，在线程本地缓存将当前值加1，将新值写回本地缓存。在线程A执行中执行任何一步子操作时，都可能被挂起转而执行线程B，导致线程B看到的是修改之前的旧值。

  

  ```java
  @NotThreadSafe
  public class UnsafeSequence {
      private int value;
  
      /**
       * Returns a unique value.
       */
      public int getNext() {
          return value++;
      }
  }
  ```

- 可见性

  在线程A中修改的变量，线程B中未必可以立即看到，这是由计算机的内存模型决定的:

  ![](/images/visibility.png)

  线程A中对变量的修改，首先存储到线程的本地缓存(Cache)中，只有等刷新到主存(Main Memory)之后，线程B才可以看到变量的新值。

- 有序性

  现代微处理器会通过指令乱序执行(out-of-order execution)来提升执行效率，除了处理器，Java自身的JIT编辑器也会对指令做重排序，最终生成的机器指令可能与字节码顺序并不一致。

  在并发程序中，指令重排序可能会导致预期之外的执行结果，比如以下的程序，在多线程执行时，线程1中的语句可能会被乱序执行，`flg=true`可能会先于`a=1`被执行，则线程2可能会出乎意料地打印出 `a = 2`。

  ![](/images/order.jpg)

### 线程活跃性问题的根源

遇到活跃性问题的并发程序，被称为(Poor Concurrency)应用程序，活跃性问题可能有很多原因引起:

- 线程开销。线程虽然比进程轻量，但是线程的管理仍然需要消耗一定的系统资源。比如线程上下文切换，需要5000~10000个时钟周期，大约是几微秒，如果线程上下文切换过于频繁，就会对活跃性造成影响。
- 阻塞。当线程被不恰当地置为阻塞状态时，后续的指令得不到执行，于是就会出现活跃性问题。
- 死锁。死锁是最常见的活跃性风险。当两个线程互相等待对方持有的资源时，就会发生死锁。不恰当的加锁解锁顺序，以及错误的资源管理策略，都有可能导致死锁。死锁往往出现在最糟糕的时候 —— 高负载的情形。
- 活锁。当线程不断地重试某个失败的操作时，就会发生活锁。此时线程虽然不会被阻塞，但也不能继续执行。

## 走出困境

既然并发程序有这么多问题，那么我们该如何避免或者解决这些问题呢?

### 解决线程安全性问题

线程安全性是并发代码最重要也是最基本的要求，我们不应容忍大部分时候可以正确运行，但是在偶然情况下会出错的并发程序。

上文已经提到，共享可变状态是造成线程不安全的唯一原因，那么为了解决线程安全性问题，可以先从避免共享状态或者避免可变状态入手。

#### 避免共享状态

如何避免共享状态呢?理想的情况是构造无状态的程序，没有状态自然也就不会共享。一个典型的例子就是Servlet程序，各Servlet自身并不持有状态，彼此隔离，互不相扰。

如果持有状态不可避免，则可以使用线程封闭技术，将状态'隐藏起来'，不让别的线程访问到。常见的有栈封闭和ThreadLocal类两种形式。

栈封闭是线程封闭的一种特例，在栈封闭中，只能通过局部变量访问对象，这些局部变量被封闭在执行线程的栈内部，其它线程无法访问到它们。举个栗子:

```java
    public int loadTheArk(Collection<Animal> candidates) {
        SortedSet<Animal> animals;
        int numPairs = 0;
        Animal candidate = null;

        // animals confined to method, don't let them escape!
        animals = new TreeSet<Animal>(new SpeciesGenderComparator());
        animals.addAll(candidates);
        for (Animal a : animals) {
            if (candidate == null || !candidate.isPotentialMate(a))
                candidate = a;
            else {
                ark.load(new AnimalPair(candidate, a));
                ++numPairs;
                candidate = null;
            }
        }
        return numPairs;
    }
```

在上面的代码中，`animals`和`candidate`是函数的局部变量，被封闭在栈帧内部，不会逸出，被其它线程访问到，所以该方法是线程安全的。

ThreadLocal类能使线程中的某个值与保存值的对象关联起来，在单个线程内部共享这个变量，而其它线程无法访问。一个典型的示例是数据库连接会话，将连接会话存储为ThreadLocal对象，线程内部共享同一个连接会话，不同线程之间的连接会话互不影响。举个栗子:

```java
 private static ThreadLocal<Connection> connectionHolder
            = new ThreadLocal<Connection>() {
                public Connection initialValue() {
                   return DriverManager.getConnection(DB_URL);
                }
            };

   public static Connection getConnection() {
        return connectionHolder.get();
   }
```

#### 避免可变状态

线程安全性是不可变对象的固有属性，对于不可变对象，所有线程看到状态必然是一致的。

纯函数式编程语言中，没有变量，只有常量，状态不能被持有，只能通过函数参数来传递，所以是天然的线程安全。

Java没有这样得天独厚的基因，不可变类型需要自己实现，具体的实现方式可以参考《Effective Java》 "最小化可变性"这一节，概括来讲需要遵循以下5条原则:

1. 不要提供修改对象状态的方法
2. 确保这个类不能被继承
3. 把所有属性设置为final
4. 把所有的属性设置为private
5. 禁止访问类内部的可变域

[Guava库](https://github.com/google/guava)也提供了一组不可变类，比如`ImmutabelList`、`ImmutableSet` 这些，我们应该在代码中尽可能地使用它们。

#### 同步机制

如果共享和可变都无法避免，那么只有使用下策 —— 同步机制，来保证线程安全性。

在Java代码中，通常使用`synchronized`关键字，对类或者对象加锁，来实现同步。被`synchronized`修饰的代码块及方法，在同一时间，只能被单个线程访问。`synchronized`关键字以'退化到单线程'的方法，解决并发安全性的问题。

举个栗子:

```java
public class SynchronizedDemo {
     //同步方法
    public synchronized void syncMethod(){
        System.out.println("Hello World");
    }

    //同步代码块
    public void syncBlock(){
        synchronized (this){
            System.out.println("Hello World");
        }
    }
}
```

以上的示例代码，编译后的字节码为:

```java
public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return

  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #5                  // class com/hollis/SynchronizedTest
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #3                  // String Hello World
        10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          23
        18: astore_2
        19: aload_1
        20: monitorexit
        21: aload_2
        22: athrow
        23: return
```

对于同步方法，JVM使用`ACC_SYNCHRONIZED`标记符，对于同步代码块，JVM采用`monitorenter`、`monitorexit`指令。这两种方式殊途同归，都是通过获取和对象关联的监视器锁(monitor), 来实现同步的。

synchronized 关键字，同时解决了原子性、可见性、有序性问题:

- 原子性。通过监视器锁，可以保证`synchronized`修饰的代码在同一时间，只能被一个线程访问，在锁未释放之前其它线程无法进入该方法或代码块，保证了操作的原子性。
- 可见性。按照Java内存模型（Java Memory Model ,JMM）的规范，对一个变量解锁之前，必须先把此变量同步回主存中，这样解锁后，后续线程就可以访问到被修改后的值。所以被synchronized锁住的对象，其值具有可见性。
- 有序性。synchronized 关键字并不禁止指令重排，但是由于程序是以单线程的方式执行的，所以执行的结果是确定的，不会受指令重排的干扰，有序性不再是个问题。

需要注意的是，当我们使用synchronized 关键字，管理某个状态时，必须对访问这个对象的所有操作，都加上synchronized 关键字， 否则仍然会有并发安全性问题。

### 解决线程活跃性问题

要避免线程活跃性问题，需要我们对并发机制有深刻了解，并养成良好的并发编程习惯。常见的解决编码活跃性问题的手段有:

- 避免使用锁。这是釜底抽薪、从源头解决的问题的办法。没有买卖就没有伤害，没有锁就不会陷入单线程执行模式，就不会有线程活跃性问题。可以使用上文提到的避免可变状态、避免共享状态等手段，来规避对锁的使用。
- 降低锁的粒度。如果加锁不可避免，那么可以尝试降低锁的粒度，只在确实需要使用锁的地方才使用它。比如可以在一个方法内部，只对其中的某几行代码，引入`synchornized`对代码块进行同步。
- 加上超时限制。并发程序可能会以出乎意料的方式，陷入长时间的锁等待，甚至是死锁。作为止血方案，可以使用显示锁(Lock类)，并指定超时时限(Timeout), 在超过该时间之后就返回一个失败信息，避免永久等待。

当出现线程活跃性问题时，我们可以借助一些工具进行诊断:

- Jstack。通过`jstack`命令，获取线程执行信息，找出其中的线程阻塞和死锁问题。
- Heap dump。通过`jmap`命令dump出当前的jvm 堆栈信息，然后使用内存分析工具识别线程阻塞和死锁。
- [Arthas](<https://github.com/alibaba/arthas>)。作为阿里开源的Java诊断利器，arthas也提供了线程分析诊断功能, 可以通过`arthas`的`thread`命令，查找出当前阻塞的线程。

### Java并发类库

`java.util.concurrent` 包提供了一系列的并发工具类，让我们能够站在巨人的肩膀上，更加高效地实现并发程序。需要注意的是，我们一定要在正确地理解了当前所要处理的并发问题，以及工具类的机制原理之后，再去选择相应的并发工具类。如果只是一知半解就去盲目使用，很可能会给自己挖坑。

下面选择一些比较常用的并发工具类并简要介绍。

#### Executor

在并发程序中，线程的创建和管理是一个重要的命题。实际的生产代码中，不能为每一个任务就创建一个线程，也就是不能出现`new Thread(runnable).start()`这样的代码，因为线程是昂贵的系统资源，不能无节制地创建，需要使用线程池对线程进行管理。

Excutor类支持了线程资源的管理和多线程任务的调度。可以使用Executors中的静态方法之一来创建一种线程池(newFixedThreadPool、newCachedThreadPool、newSingleThreadPool、newScheduledThreadPool等)，可以使用Runalbe、Callable来提交并发任务，Excutor类会自己负责任务的调度，解耦了任务的提交和执行。

举个栗子:

```java
Callable<Integer> task = () -> {
    try {
        TimeUnit.SECONDS.sleep(1);
        return 123;
    }
    catch (InterruptedException e) {
        throw new IllegalStateException("task interrupted", e);
    }
};

ExecutorService executor = Executors.newFixedThreadPool(1);
Future<Integer> future = executor.submit(task);

System.out.println("future done? " + future.isDone());

Integer result = future.get();

System.out.println("future done? " + future.isDone());
System.out.print("result: " + result);
```

在使用Excutor时，如果线程池的任务之间存在依赖，线程池中的某些任务需要无限期地等待一些其它任务提供的资源；或者某些任务运行耗时较长，其它任务得不到运行资源，则会出现线程饥饿，引发活跃性问题，需要避免。

#### Atomic

从Java5.0开始，提供了一组原子类变量(例如`AtomicInteger`,`AtomicLong`,`AtomicBoolean`,`AtomicReference`等)， 来支持对单个变量的原子性操作。

内置的监视器锁，是一种悲观锁，任何时候只有一个线程可以持有该锁，其它想获取该锁的线程必须阻塞等待。而`Atomic` 类提供了一种乐观机制，任何线程都可以尝试获取资源并更新，如果在更新的过程中存在来自其它线程的干扰，那么这个操作将失败并可以重试。

`Atomic`的实现依赖于处理器提供的CAS(Compare and Swap)指令。CAS是一个原子性操作，包含三个操作数：需要读写的内存位置V、旧值A和拟写入的新值B。线程读取V的值，如果等于A，则将V的值更新为B，返回成功，否则返回失败。

举个栗子，以下的代码就是线程安全的，而不需要显示地同步。

```java
@ThreadSafe
public class SafeSequence {
    private AtomicInteger value;

    /**
     * Returns a unique value.
     */
    public int getNext() {
        return value.getAndIncrement();
    }
}
```

相对于内置的监视器锁，`Atomic`更加地轻量和高效，不存在死锁和活跃性问题。其主要劣势在于，需要调用者来处理竞争问题，决定在CAS操作失败时是重试、回退还是放弃。

#### Lock

Lock类提供了一种可轮询的、定时的以及可中断的锁获取操作，当内置锁无法满足使用场景的要求时，可以考虑使用显式的Lock。举一个带有时间限制的Lock示例:

```java
public class TimedLocking {
    private Lock lock = new ReentrantLock();

    public boolean trySendOnSharedLine(String message,
                                       long timeout, TimeUnit unit)
            throws InterruptedException {
        long nanosToLock = unit.toNanos(timeout)
                - estimatedNanosToSend(message);
        if (!lock.tryLock(nanosToLock, NANOSECONDS))
            return false;
        try {
            return sendOnSharedLine(message);
        } finally {
            lock.unlock();
        }
    }
}
```

#### ConcurrentHashMap

上文中提到，为了解决线程活跃性问题，提高并发执行效率，一种可行的方案是降低锁的粒度。`ConcurrentHashMap`可以看作这种思路的优秀实践，内部使用了分段锁(Lock Striping)的方式来降低锁的粒度，使用一个包含16个锁的数组，每个锁保护所有散列桶的1/16，其中第N个散列桶，由 N%16 个锁来维护。其原理如下图所示:

![](/images/lock-strip.jpg)

#### CountDownLatch

`CountDownLatch`的作用相当于一扇门，在到达结束状态之前，这扇门一直是关闭的，没有任何线程可以通过，当到达结束状态时，这扇门会打开，并允许所有的线程通过。`CountDownLatch` 可以用来保证某些活动直到其它活动都完成之后才继续执行。举个栗子:

```java
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task)
            throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException ignored) {
                    }
                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
```

以上测试并发程序执行时间的示例中，通过插入'启动门'(startGate对象)，避免了将线程启动时间统计在内，通过插入'结束门'(endGate对象)，保证了所有线程执行完成之后，再获取结束时间。

#### Semaphore

信号量`Semaphore`，用来控制同时访问某个特定资源的操作数量。`Semaphore`中管理着一组虚拟许可，如果没有许可，则`acquire`操作将阻塞直到有许可。可以把锁看做一种特殊的二元信号量。举一个代码示例:

```java
public class BoundedHashSet <T> {
    private final Set<T> set;
    private final Semaphore sem;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        sem.acquire();
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded)
                sem.release();
        }
    }

    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
```

这里通过`Semaphore`实现了一个有界集合。

### 测试

基于测试的目的，并发程序测试可以分为两类:安全性测试和活跃性测试。

编写正确的并发测试程序，比编写正确的并发程序本身更加困难，它要求测试用例的编写者对被测代码运行机制有深切的洞察，否则很难写出有效的测试程序。

由于并发问题常常隐藏得比较深，并发测试程序可能需要多次、长时间地执行，才能暴露目标程序中的问题，压测是发现并发问题的有效方案。

### 文档

要写出完全线程安全的Java代码，现实中困难重重，你调用的第三方类库常常是一个黑盒，既没有文档说明其线程安全性保证，又无法看到其内部具体实现，于是用户很难知道在并发环境中使用该类时，是否要添加额外的同步机制。

我们可以通过文档缓解这个问题。在自己发布的类库中，如果使用时需要额外的同步机制，那么就显式地在文档中标注出来。给用户一个解决并发问题的机会，否则就是在制造问题坑人。

## 总结

并发程序，需要在设计之初，就要尽可能保证其正确性。草率设计完之后再去优化的代价太高了，潘多拉的魔盒一旦打开，厄运的齿轮就会开始转动。

如果用户在使用一门语言解决并发问题时，必须要理解很多底层原理和细节，才能开始编码，战战兢兢写完代码之后，发现制造的问题比解决的问题还要多，那么这门语言针对并发场景的解决方案就算不得成功。

愿岁月对Java工程师温柔以待。
