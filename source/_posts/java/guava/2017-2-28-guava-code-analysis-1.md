---
layout: post
title: Guava 源码笔记系列第一章 -- ImmutableList
date: 2017-02-28
categories: Programming
tags: [Java, Guava]
---

本系列博客的目标读者是和我一样的初级Java 程序员。

## 有关源码阅读

入门一种编程语言，阅读经典类库或框架的代码，无疑是十分有效的方法，但是瀚若星辰的类库，选择哪一个阅读，常常让初学者无所适从。

我个人的体验是，通常一些Utils类库或者UT框架是一个不错的起点，这些库/框架解决的是通用的问题，不涉及复杂的应用场景，易于上手，并且由于被广泛使用，代码质量有可靠保证。

Guava, 类似Boost 之于 c++, 可以算作Java的“准标准库”，未来版本的JDK很可能吸收Guava中一些优秀的设计并作为标准库一部分，正是入门学习Java代码的好资料。

<!--more-->

## 初识guava

第一次邂逅guava是在[鸟瞰](http://iwant.cainiao-inc.com/iwatch/report/metrics?dep_name=%E8%8F%9C%E9%B8%9F-%E6%8A%80%E6%9C%AF%E9%83%A8&start_date=2016-08-01&end_date=2017-03-30)项目，@禅剑利用guava cache缓存数据库查询结果，使网页响应提高速度从4~5秒减少到1秒以内, 好奇心驱下使开始阅读guava源码。正好最近读完《Effective Java》,在guava源码中看到了大量书中所提的优秀实践，于是决定记录一些思考和感悟。

对于一门编程语言来说，集合类型处于核心地位，例如`Lisp`名字就源于`List Processor`, 本系列博客从`guava`实现的一个集合类型 -- `ImmutableList`类说起。

## Java的劣势

Java8中，`lambda`表达式、缓式集合类`Stream`等函数式特性的引入，标志着Java一只脚已经跨入了函数式编程的大门，函数虽然尚未成为一等公民，至少已能够在方法间传来传去。但是我觉得Java还有另外一只脚仍在函数式的大门之外，那就是**不可变量**。

在纯粹的函数式世界如`Haskell`, 是不存在可变量(`mutable data`)的，对象状态的变迁通过建立一个新的数据对象来传递，副作用通过`Monad`之类的复杂机制来表达。
如《Effective Java》Item70 所述，对象的线程安全性可分为5级: immutable, unconditionally thread-safe, confitionally thread-safe, not thread-safe, thread-hostile, 其中不可变量为第一等的线程安全对象。

不可变量与生俱来的线程安全性，使得函数式编程语言，天然适应大规模并发的场景。相对而言Java在这方面处于劣势，Java并发程序的编写，测试和调试都会遇到很多的陷阱和挑战。理论上来说，在Java中定义`Immutable Class`并不困难，如《Effective Java》Item15所述，需要遵循5条原则:

1. 不要提供任何改变对象状态的方法(比如set方法)。

1. 让该类无法被继承(public final class XXX)。

1. 所有的属性都声明为final。

1. 所有的属性都声明为private。

1. 如果内部有任何属性引用了可变对象，访问这些属性是需通过防御式拷贝(defensive copy)。

实际工程中，主要的难点在最后一条，一来难以确定对象间复杂的引用关系，二来Java中对象拷贝是个棘手的问题。让我们来一起看看`Guava`中`ImmutableList`的源码实现。

## ImmutableList初体验

首先来看类定义：

```
public abstract class ImmutableList<E> extends ImmutableCollection<E>
    implements List<E>, RandomAccess 
```

由于实现了`List`和`RandomAccess`接口，`ImmutableList`支持元素的随机访问，同时由于继承了`ImmutableCollection`，所有的修改操作如`add`,`addAll`,`remove`,都会直接抛出`UnsupportedOperationException`异常, 保证List本身是不可修改的。


## 创建ImmutableList对象

如《Effective Java》Item1 所述，我们在设计类的时候，倾向优先使用静态工厂方法(static factory method)而非构造函数(constructor)创建对象，优点在于:

1. 静态工厂方法多了一层名称信息，比构造函数更富表达性。
1. 可以更灵活地创建对象，比如缓式初始化，缓存已创建对象。
1. 静态方法内部返回的对象类型，可以是其声明类型的子类。

`ImmutableList`遵循了最佳实践。首先，`ImmutableList`不可以通过构造函数实例化，更准确地说，不可以在`package`外部通过构造函数实例化。具体是如何做到的呢？(可作为面试题^_^)其父类`Immutable Collection`的构造函数定义为:

```java
ImmutableCollection() {}
```
由于Java方法的默认访问权限是`package`, 所以只能在`package`内部调用构造函数。

其次，`ImmutableList`提供了三种方式来创建对象:

### ImmutableList.of

代码示例：

```java
ImmutableList<String> foobar = ImmutableList.of("foo", "bar", "baz");
```

该方法接受同类型的一组元素并生成一个`ImmutableList`, `of`方法的实现很有意思，不是简单的使用一个`varargs`参数，而是定义了一系列接受不同个数参数的方法:

```java
public static <E> ImmutableList<E> of() 

public static <E> ImmutableList<E> of(E element)

public static <E> ImmutableList<E> of(E e1, E e2)

public static <E> ImmutableList<E> of(E e1, E e2, E e3)
...
public static <E> ImmutableList<E> of(
      E e1, E e2, E e3, E e4, E e5, E e6, E e7, E e8, E e9, E e10, E e11, E e12, E... others)
  
```

直到参数个数超过12个时才用`varargs`，实际工作中看到这样的代码肯定会认为MDZZ，但是作为一个类库来说，这样设计一定有其道理，我猜想是为了性能优化。`varargs`在传递的时候，背后会有一次到`array`的隐式转换，性能不如普通的参数传递那么高。通常情况下，我们并不会传递超过12个参数，这样设计保证了绝大部分场景性能最优。

### ImmutableList.copyOf

代码示例：

```java
void sampleMethod(Collection<String> collection) {
   ImmutableList<String> defensiveCopy = ImmutableList.copyOf(collection);
   ...
}
```

该方法接受一个`Collection`作为参数并返回一个`ImmutableList`。

要实现`copyOf`方法，最简单的莫过于直接将底层的每个元素做深拷贝然后生成`ImmutableList`。但是对于所有情况都深拷贝的话，性能和存储开销必然比较大，源码里面是如何优化的呢？

```java
public static <E> ImmutableList<E> copyOf(Collection<? extends E> elements) {
    if (elements instanceof ImmutableCollection) {
      
      ImmutableList<E> list = ((ImmutableCollection<E>) elements).asList();
      return list.isPartialView() ? ImmutableList.<E>asImmutableList(list.toArray()) : list;
    }
    return construct(elements.toArray());
  }
```

可以看到，`copyOf`方法是比较智能的，在参数已经是`ImmutableCollection`类型的情况下，直接复用原来的`collection`即可，其它情况下，通过`construct`方法，底层调用`Arrays.copyOf`做深拷贝。

在源码注释中，提到了`ImmutableList`一个首要的特性:[`Shallow immutability`](https://github.com/google/guava/blob/0f025e5d5d39a5d1ab709917f725d48a8464df50/guava/src/com/google/common/collect/ImmutableCollection.java#L57-L70), 这里的`Shallow`用的非常精髓，让我们仔细品味一下。

在Java标准库中，`java.util.Collections`已经提供了一系列的`UnmodifiableCollection`实现，比如`UnmodifiableList`，但是这些实现有个严重的问题，当源collection被修改时，对应的`UnmodifiableCollection`也会受到影响，仿佛连体婴儿一般。不同于`UnmodifiableCollection`这样的妖艳贱货，`guava`中的`ImmutableCollection`由于使用了深拷贝，可以不受源Collection修改的影响。

相对于`UnmodifiableColletion`提供的表面的不可变性(surface immutability), `ImmutableCollection`更深一层，但为什么不用`Deep`而说是`Shallow`呢？因为它们都无法真正摆脱一个幽灵:`mutable data`。Collection 本身固然可以实现为`Immutable`, 但是Collection所包含的每个元素，却仍然可能是`mutable object`。

```java
UserBean user = new UserBean();
List<UserBean> userList = ImmutableList.of(user);

userList.get(0).setAge(27); //age is 27
userList.get(0).setAge(28); //age is changed, list also changed
```

### Builder

代码示例:

```java
public static final ImmutableList<Color> GOOGLE_COLORS =
       ImmutableList.<Color>builder()
           .addAll(WEBSAFE_COLORS)
           .add(new Color(0, 191, 255))
           .build();
```

类似`StringBuilder`, `ImmutableList`也提供了`Builder`类，来减少中间对象的创建，提高内存使用效率。

## 君欲何往
Java8为什么没有实现guava中类似的不可变集合？JDK会在以后版本的标准库中加入对不可变集合的支持么？[这里](http://softwareengineering.stackexchange.com/questions/221762/why-doesnt-java-8-include-immutable-collections)可以看到一些有趣的讨论。

我觉得, 就算是标准库支持了不可变集合, 那也没什么大不了, 由于无法禁绝可变量, Java也许永远不能完全迈入函数式编程的大门。不过这也挺好，Java就是Java，它不会，也没有必要成为另一个`Closure`或者`Scala`，我就是我，不一样的烟火~


