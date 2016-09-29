---
layout: post
title: Akka in Scala Part 6 - Router
date: 2016-05-10
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

`Akka`是一个自由的世界，`actor`之间可以互相发送任意类型的消息，但是随着`actor`数量的增多，如果不加限制地任由`actor`之间互相通信，很快整个系统将变得复杂且难以管理。

在互联网的世界，将一群节点组成一个内网，不同网络间的通信，需要通过路由器作为中介。路由使消息传递变得有序。

`Akka`世界中，也有扮演路由器角色的`router`，通过`router`可以管理一组`routee`的创建、通信和销毁。

<!--more-->

## 创建`router`

可以用两种不同的方式创建`router` 和 `routee`：

- Pool - `routee`由`router`创建，成为`router`的子`actor`, 由`router`管理其生命周期。优势在于:
	* 由`Router`来定义监护策略。
	* `routee`的路径和名字由系统自动生成。
	* 代码示例： 
	
		```scala
		val router: ActorRef = context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router")
		```
	
- Group - `routee`是独立的`actor`, 在`router`创建时，将其路径传递给`router`的配置，`router`并不负责管理`routee`的生命周期。优势在于:
	* `routee`的创建更加灵活，不受`router`的限制。
	* 对`routee`的监护更加灵活，因为每个`routee`由它自己的父`actor`来决定监护策略。
	* 代码示例： 
	
		```scala
		val router: ActorRef = context.actorOf(RoundRobinGroup(paths).props(), "router")
		```

## 路由策略

`Akka`提供了多种形式的路由策略，常用的有：

    - akka.routing.RoundRobinRoutingLogic
    	* 轮盘路由策略，依次选取各个`routee`来处理消息。
    - akka.routing.RandomRoutingLogic
    	* 随机路由策略，随机选取`routee`来处理消息。
    - akka.routing.SmallestMailboxRoutingLogic
    	* 选取当前收件箱size最小的`routee`来处理消息，这种方式具有更好的负载均衡性。
    - akka.routing.BroadcastRoutingLogic
    	* 将消息广播给所有的`routee`。

    
## 监护策略

对于用`Pool`方式创建的`router`,默认的监护策略是`escalate`，当`routee`抛出异常时，将错误向上传递给`router`, `router`继续向上传递给它的父`actor`，由父`actor`来决定如何恢复。

如果要替换默认的`escalate`策略，可以通过`router`的`supervisorStrategy`来指定。

## 远端actor

`routee`不仅可以是本地`actor`,也可以是远程`actor`。由于`Akka`中`actor`的地址透明性，本地`routee`和远程`routee`的创建方式并没有太大区别。

以`Pool`创建方式举例：

```scala
val addresses = Seq(
  Address("akka.tcp", "remotesys", "otherhost", 1234),
  AddressFromURIString("akka.tcp://othersys@anotherhost:1234"))
val routerRemote = system.actorOf(
  RemoteRouterConfig(RoundRobinPool(5), addresses).props(Props[Echo]))
```
    
## 示例代码

红包程序中为了模拟大规模并发请求，创建`generatorRouter`和`clientRouter`来分发消息给各个红包生成器及客户端。

```scala
import akka.actor.{ActorSystem, Props}
import akka.routing.RoundRobinPool

    val generatorRouter = system.actorOf(RoundRobinPool(100, supervisorStrategy = ResumeSupervisor()).props(Props[RedPacketGenerator]), "generatorRouter")
    val clientRouter = system.actorOf(RoundRobinPool(100).props(Props(classOf[RedPacketClient], generatorRouter)), "clientRouter")
```