---
layout: post
title: Akka in Scala Part 5 - 监护与错误恢复
date: 2016-05-09
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

在分布式系统中，由于集群规模庞大，系统环境复杂，出错是常态，容错是一个非常重要的内容。

传统的异常处理，通常是捕获某种类型的异常并调用相应的异常处理函数，但是在分布式系统中，由于环境的复杂性，通常无法预料错误的类型并进行有效的处理，传统的异常处理方式并不适用。

`Akka`沿袭了`Erlang`的`let it crash`哲学，当`actor`内部发生异常时，并不试图捕捉异常并处理，而是重建一个新的`actor`, 使得整个系统在错误发生的时候可以自动恢复。

<!--more-->

## 监护策略

在`Akka`中，在子`actor`被创建后，父`actor`就成为了子`actor`的监护这，在子`actor`出错时负责处理， 有一对一和一对多两种策略：

* OneForOneStrategy
	- 只诛首恶，余者不问。只有出错的`actor`会被处理。`Akka`默认采用这种机制。
* OneForAllStrategy
	- 城门失火，殃及池鱼。当出错时，不仅出错的`actor`,其兄弟`actor`也采用同样的策略一并处理。

## 恢复策略

`actor`出错时具体采用何种策略呢？共有四种：

* Stop - 停止出错的`actor`,不再让它处理任何消息。
* Restart - 这是默认策略，杀死旧的`actor`,重新创建一个新的`actor`。
* Resume - 忽略本次错误，恢复`actor`对消息的处理。
* Escalate - 交给父`actor`来决定处理策略。

## 代码示例

回到我们的红包程序，这里设计了一个采用`Resume`策略的监护者，对于比较重要、不能重启的`actor`, 通常可以采用`resume`策略进行错误恢复。

```scala
/**
 * Define a customized supervision strategy
 */

import akka.actor.{SupervisorStrategy, OneForOneStrategy}
import akka.actor.{ActorKilledException, ActorInitializationException}
import akka.actor.SupervisorStrategy._

object ResumeSupervisor {
  def apply() = OneForOneStrategy() {
    case _: ActorInitializationException => Stop
    case _: ActorKilledException => Stop
    // resume the actor for general execption
    case _: Exception => Resume
    case _ => Escalate
  }
}
```

## Error Kernel Pattern

整个Akka系统，被设计成一个树形结构，底层的子Actor由其上一层的父Actor创建并管理。

![](/images/error-kernel-pattern.png)

`Error kernel pattern`的同样来源于`Erlang`，其核心思想在于，处于上层的`Actor`承担风险较小的任务，出错之后尽量采用`resume`的方式来恢复，处于底层的`Actor`承担更加危险的任务，出错之后通常采用`restart`的方法来恢复。

由于不同层级的`actor`在整个系统中的地位不同，在设计`actor`系统的时候，倾向于将重要的数据放在上层的`actor`,而将具体要执行的操作下放到下层的`actor`。

## Death Watch
对于那些不存在父子关系的`actor`, 一个`actor`如果想要获得另一个`actor`生命周期相关的信息，可以使用`death watch`, 在被关注的`actor`挂掉的时候收到`Terminated`消息并采取相应动作。

```scala
import akka.actor.{ Actor, Props, Terminated }
 
class WatchActor extends Actor {
  val child = context.actorOf(Props.empty, "child")
  context.watch(child) // <-- this is the only call needed for registration
  var lastSender = context.system.deadLetters
 
  def receive = {
    case "kill" =>
      context.stop(child); lastSender = sender()
    case Terminated(`child`) => lastSender ! "finished"
  }
}
```

`death watch`一个典型的应用是关闭`ActorSystem`。

`ActorSystem`应该在什么时候被关闭呢？一个自然的答案 —— 在`actor`处理完所有的消息之后。如何知道`actor`已经处理完所有的消息呢？由于`actor`消息处理是异步的，这是个很难回答的问题。

一个可能的方案是，估算一下系统处理消息所需的时间，设置一个超时，到达时限后关闭`ActorSystem`，但这只是一个粗略的实现。

如果需要更加精确的方案，可以考虑使用`death watch`, 参考[`akka shutdown patterns`](http://letitcrash.com/post/30165507578/shutdown-patterns-in-akka-2), 核心思想是设计一个单独的`actor`来关注所有其他的`actor`,当这些`actor`都发出`Teminated`消息之后，关闭整个系统。


## 总结

`Akka`的错误恢复机制，遵循了`Let it crash`哲学，体现了优雅的简洁性，使得构建稳定健壮的分布式系统成为可能。
