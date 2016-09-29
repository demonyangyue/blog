---
layout: post
title: Akka in Scala Part 7 - Future
date: 2016-09-09
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言
在`Akka`的世界中，消息传输是异步的，异步是一种高效的通信机制，使得消息的发送者无须阻塞等待先前的消息处理完成。

异步也意味着不确定性，但是很多时候业务逻辑要求消息按一定的顺序被处理，此时`Akka`的另一基石 -- `Future`就可以派上用场。

<!--more-->

## 定义

`Future`和`Promise`是一对孪生的概念。

一个 `Promise` 对象代表一个目前还不可用，但是在未来的某个时间点可以被解析的值, 这个值会在未来被`Future`对象消费。

通过给调用者返回一个`Future`对象，用户代码不必阻塞等待结果，可以继续向下执行。`Future`和`Promise`使得用户可以用同步的方式编写异步代码，同步为表，异步为里，兼具同步代码的清晰性和异步代码的高效性。

在`Akka`源码中，`Future`对象的定义为：

```scala
object Future { def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T] }
```
接收一个函数对象`body`以及执行上下文`execctx`作为参数。可以把`execctx`看作线程池，通常`actor`中已经包含`ExecutionContext`类型的默认参数，无须用户代码显示指定。
    
## 应用场景

`Future`有以下典型的应用场景：

- 阻塞IO：典型的如`Sleep`函数，可以把相关的代码放入`Future`块，使`actor`不必阻塞，可以继续处理后续消息。
- 耗时的计算：典型的如计算PI的值，可以把计算的代码放入`Future`, 而不必阻塞等待计算结果。

在`Akka`中，`Future`有更多的应用场景：
    
- 消息之间有顺序依赖。一条消息需要在某条消息之后被处理，由于`Akka`的异步处理机制并不保证消息处理的顺序，我们需要借助`Future`来达成目的。
- 多条消息需要在同一上下文执行。可以通过`Future`提供的闭包，来保证多条消息在同一上下文被处理。
    
## 使用方法

`Akka`的`Future`和其它语言中的一样，支持回调链，待`Promise`完成计算之后，触发注册的回调函数。

* `onSuccess`, `Promise`计算成功之后调用的函数。
* `onFailure`，`Promise`计算失败之后调用的函数。
* `onComplete`, 无论`Promise`计算结果如何，都会调用的函数。


`Future`本质上是单子（`monad`）, 支持一系列高阶函数操作，如`map`, `flatMap` 以及`filter`, 也可以通过`for comprehension`, 从`Future`中获得计算结果：

```scala
val f = for {
  a <- Future(10 / 2) // 10 / 2 = 5
  b <- Future(a + 1) //  5 + 1 = 6
  c <- Future(a - 1) //  5 - 1 = 4
  if c > 3 // Future.filter
} yield b * c //  6 * 4 = 24
```


`Future`另一个有趣的方法是`sequence`, 其作用是将`List[Future[T]]`类型的对象装换成`Future[List[T]]`类型, 维持了原来`List`对象各个元素的顺序。

在`actor`中, 当我们使用`ask`方法发送消息时，获得的就是一个`Future`对象，可以基于这个`Future`对象注册后续操作，例如，可以用`pipeTo`方法，将`Future`的结果发送到别的`actor`:

```scala
import akka.pattern.pipe        
val future = someActor ? SomeMessagefuture pipeTo anotherActor
```

## 代码示例
在红包客户端中，我们需要在发送红包请求之后，接收红包并打开，消息之间存在顺序依赖，于是引入`Future`, 代码示例：

```scala
  implicit val askTimeout = Timeout(1.second)
  def receive = {

    // simulate the shake action in Mobile
    case Shake =>
      (generator ? RedPacket) onSuccess {
        case UnopenedRedPacket(amount) => self ! OpenPacket(amount)
      }
  }
```
## 小结
`Akka`通过`Future`提供了对同步机制的支持，在特定的应用场景,`Future`是非常有力的工具。