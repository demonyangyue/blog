---
layout: post
title: Akka in Scala Part 3 - Actor
date: 2016-04-29
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

在Akka的舞台上，Actor是其主角。Akka程序中，发生的主要事情就是Actor之间传输并处理消息。

<!--more-->

## 红包程序

在整个系列博客，我们会通过一个红包生成和发送程序(类似支付宝的咻一咻或者微信摇一摇)，来解释Akka中的主要组件。

可以从[github](https://github.com/demonyangyue/RedPacket)下载完整代码并运行：

```bash
git clone https://github.com/demonyangyue/RedPacket
cd RedPacket
sbt run
```

整个程序的框架为:

![](/images/actor-dispatcher.png)

后续章节我们会逐步解释各个模块。

## 创建Actor

### 定义Actor类
在Akka源码中，Actor 是一个base trait：

```scala
trait Actor {
  def receive: Actor.Receive
}
```

用户代码通过定义自己的`receive`方法，创建自己的Actor类。
`Actor.Receive`的实际类型为`PartialFunction[Any, Unit]`, 是一个接受任意类型的消息，并且不返回结果的偏函数（我们执行`receive`方法，是为了获得其定义的副作用，而非一个返回值，所以`receive`并非一个纯函数）。


现在定义我们的红包生成器`RedPacketGenerator`, 每次收到客户端发来的`RedPacket`请求时，就生成一个新的随机红包：

```scala
class RedPacketGenerator extends Actor with ActorLogging {

  def receive = {

    case RedPacket => 
      // generate a random red packet, at most $100 !
      val amount = Random.nextInt(100)
      log.info("Generate a new red packet!")
  }	

}
```

`receive`方法可以接受任何类型的消息，这是一个非常优雅的设计, 使得akka各个组件之间，可以用一致的方式进行通信。

### 创建Actor实例

完成类定义之后，我们来创建实例。不同于传统的scala，Akka采用配置类`Props`来创建actor实例。

```scala
val redPacketGenerator = system.actorOf(Props[RedPacketGenerator], "redPacketGenerator")
```

如果Actor类的构造函数包含参数，可以用`Props(classOf[ActorWithArgs], "arg")`的形式。


在上面的代码片段中，`redPacketGenerator` 的类型是`ActorRef`而非`Actor`。在Akka中，`actorRef`提供了对内部`actor`对象的封装，用户只能与`actorRef`进行交互，这样设计的优点在于：

* 类似代理模式，向用户屏蔽了内部actor，防止内部actor被直接修改。
* 当发生错误时，内部actor会被重新创建，而actorRef并不发生任何改变，使得错误恢复对用户变得透明。

## Actor内部结构

每个`actor`有其自己的生命周期，由创建它的父`actor`进行管理。Actor内部代码定义了一系列的hook，用户代码可以利用这些hook，定义actor在生命周期各个阶段的行为：

```scala
  def preStart(): Unit = ()
  
  def postStop(): Unit = ()
  
  def preRestart(reason: Throwable, message: Option[Any]): Unit = {
  context.children foreach { child ⇒
    context.unwatch(child)
    context.stop(child)
  }
  postStop()
}
 
  def postRestart(reason: Throwable): Unit = {
  preStart()
}
```

用户代码可以通过`actor`提供的`context`属性来获取上下文信息，例如：

* acotor 所属的akka system: `context.system`
* 父 actor： `context.parent`
* 子 actors: `context.children`
* 创建子actor的工厂方法：`context.actorOf()`

## 消息

在Akka中，actor之间传递的消息是强类型的，并且是不可变的(immutable)。

通常我们不会把消息直接定义为`String`或`Int`类型，而是通过`case object`（如果消息类的构造函数不含参数） 或者 `case class`（如果消息类的构造函数包含参数）定义消息，并放在`companion object` 中。

对于红包生成器的代码，我们在相应的`companion object`中定义消息`RedPacket`:

```scala
object RedPacketGenerator {
  case object RedPacket
}
```

## 发送消息

### Tell

最常用的消息发送方法是`tell`，遵循`fire and forget`的方式，`tell`保证最多只发送一次消息(`at most once`)，并不保证消息一定会被送达目的地。

相对于其它的消息传输保证机制, 比如 `at least once`（如GFS中的写操作）或`exactly once`（如AMQP协议),`at most once`机制最为简单灵活，并且易于容错。Akka框架本身并不提供额外的消息送达保证，而是让客户代码自己基于业务逻辑去实现。

tell 方法的定义为：

```scala
final def tell(msg: Any, sender: ActorRef): Unit = this.!(msg)(sender)
```
实际代码中，我们通常不会直接调用tell，而是用柯里化的方法`!`,其定义为：

```scala
  def !(msg: Any)(implicit sender: ActorRef = Actor.noSender) = tell(msg, sender)
```

每次我们调用`!`方法的时候，会有一个隐式的`sender`被传入,这个`ActorRef`类型的`sender`是什么呢？从Actor源码中我们看到，`sender`被定义为消息的发送者自身：

```scala
implicit val self: ActorRef
```

回到红包程序，让我们的红包生成器发出一个红包：

```scala
class RedPacketGenerator extends Actor with ActorLogging {

  def receive = {

    case RedPacket => 
      // generate a random red packet, at most $100 !
      val amount = Random.nextInt(100)
      log.info("Generate a new red packet!")
      sender ! UnopenedRedPacket(amount)

  }	
}
```
### Ask

另外一种消息发送的方式是`ask`，不同于`tell`，`ask`遵循的是`send and expect`的形式，发送出一条消息之后，异步等待回复的消息。

在用户代码中，通常我们以`?`的形式调用`ask`方法， `ask`返回一个`Future`。我们可以向`Future`对象注册自己的回调函调函数，待Future完成任务后调用这些函数。后续文章会对`Future`作详细解释。

akka源码将`ask`方法定义为：

```scala
def ?(message: Any)(implicit timeout: Timeout, sender: ActorRef = Actor.noSender): Future[Any] =
  internalAsk(message, timeout, sender)

```

这里有个隐式的`timeout`参数，如果在指定的时间内仍未完成消息的处理，则Actor会抛出超时错误。

举个例子，在我们的红包程序中，客户端接收一种"待拆开红包" -- 只有当用户点击"打开红包"之后，才显示具体数额：

```scala
class RedPacketClient(val generator: ActorRef) extends Actor with ActorLogging {

  implicit val askTimeout = Timeout(1.second)
  def receive = {

    // simulate the shake action in Mobile
    case Shake =>
      (generator ? RedPacket) onSuccess {
        case UnopenedRedPacket(amount) => self ! OpenPacket(amount)
      }
  }
}
```

## 回复消息

`actor`收到消息后，经常需要向发送者作出相应的回复，Akka会将消息及其发送者打包在一起发送到接收端，我们可以通过`sender`方法来获取消息的发送者，例如

```scala
sender ! UnopenedRedPacket(amount)
```

## 小结

在这一篇，我们学习了`actor`的创建，消息的发送和回复，下一篇我们会学习`akka`中的测试框架。




