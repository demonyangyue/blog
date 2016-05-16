---
layout: post
title: Akka in Scala Part 3 - Actor
date: 2016-04-29
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

在Akka的世界中，一切皆Actor。Akka程序中，发生的唯一事情就是Actor之间传输并处理消息。

<!--more-->

## 创建Actor

### 定义Actor类
在Akka源码中，Actor 是一个base trait：

```scala
trait Actor {
	  def receive: Actor.Receive
}
```

用户代码通过自定义的`receive`方法，创建自己的Actor类。
`receive`方法的类型为`PartialFunction[Any, Unit]`, 是一个接受任意类型的消息，并且并返回结果的*PartialFunction*.

本系列中，我们会实现一个模拟微信摇一摇红包的程序。首先定义我们的红包生成器`RedPacketGenerator`, 每次收到`Shake`消息时，就生成一个新的随机红包：

```scala
class RedPacketGenerator extends Actor with ActorLogging {
	def receive = {
  		case Shake => 
        val amount = Random.nextInt(100)
        log.info("Generate a $%d red packet".format(amount))
  }	

}
```

### 创建Actor实例

完成类定义之后，我们来创建实例。不同于传统的scala，Akka采用配置类`Props`来定义actor实例的创建。

```scala
val redPacketGenerator = system.actorOf(Props[RedPacketGenerator], "redPacketGenerator")
```

如果Actor类的构造函数包含参数，可以用`Props(classOf[ActorWithArgs], "arg")`的形式。


在上面的代码片段中，`redPacketGenerator` 的类型是`ActorRef`而非`Actor`。在Akka中，actorRef提供了对内部actor对象的封装，用户只能与actorRef进行交互，这样设计的优点在于：

* 类似代理模式，向用户屏蔽了内部actor的实现细节
* 当发生错误时，内部actor会被重新创建，而actorRef并不发生任何改变，使得错误恢复对用户透明。

## Actor结构

每个actor有其自己的生命周期，由创建它的父actor进行管理。Actor内部代码定义了一系列的hook，用户代码可以利用这些hook，定义actor在生命周期各个阶段的行为：

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

用户代码可以通过actor提供的context属性来获取Actor上下文信息，例如：

* acotor 所属的akka system: `context.system`
* 父 actor： `context.parent`
* 子 actors: `context.children`
* 创建子actor的工厂方法：`context.actorOf()`

## 消息

在Akka中，actor之间传递的消息是强类型的，并且是不可变的(immutable)。

最佳实践是通过case object（如果消息类的构造函数不含参数） 或者 case class（如果消息类的构造函数包含参数） 定义消息，并放在companion object 中。

```scala
object RedPacketShaker {
    case object Initialize
    case class RedPacket(amount: Int)
}
```


## 发送消息

### Tell

最常用的消息发送方法是`tell`，按"send and forget"形式。Tell保证最多只发送一次消息(`at most once`)，但并不保证消息一定会被送达目的地。

相对于其它的消息传输保证机制, 比如 `at least once`（如GFS中的写副本操作）或`exactly once`（如TCP传输）,`at most once`机制最为简单灵活，并且易于容错。 Akka框架本身并不提供额外的消息送达保证，而是让客户代码自己基于业务逻辑去实现。

tell 方法的定义为：

```scala
final def tell(msg: Any, sender: ActorRef): Unit = this.!(msg)(sender)
```
实际代码中，我们通常不会直接调用tell，而是用柯里化的函数`!`,其定义为：

```scala
def ! (message: Any)(implicit sender: ActorRef = null): Unit
```

每次我们调用`!`方法的时候，会有一个隐式的sender被传入,这个ActorRef类型的sender是什么呢？在Actor源码中，sender并定义为消息的发送者：

```scala
implicit val self: ActorRef
```

回到红包程序，让我们让创建一个`RedPacketShaker` Actor, 每次被实例化的时候就向红包生成器发送一条`Shake`消息：

```scala
class RedPacketShaker(val redPackerGenerator: ActorRef) extends Actor with ActorLogging {

  def receive = {
      case Initialize =>
          redPackerGenerator ! Shake
  }
}
```
### Ask

另外一种消息发送的方式是ask，不同于tell，ask遵循的是`send and expect`的形式，发送出一条消息之后，异步等待回复的消息。

在用户代码中，通常我们以`?`的形式调用ask方法， ask返回一个Future供我们后续处理。

```scala
def ?(message: Any)(implicit timeout: Timeout, sender: ActorRef = Actor.noSender): Future[Any]
```

现在假设我们现在要发送一种"待拆开红包" -- 只有当用户点击"打开红包"之后，才显示数额：

```scala
class RedPacketShaker(val redPackerGenerator: ActorRef) extends Actor with ActorLogging {

    implicit val askTimeout = Timeout(1.second)

    def receive = {
    	case Initialize =>
            redPackerGenerator ! Shake

        	(redPackerGenerator ? FuzzyShake) onSuccess {
            	case UnopenedRedPacket(amount) => self ! OpenPacket(amount)
            }
    }
}
```

## 回复消息

actor 收到消息后，经常需要向发送者作出相应的回复，我们可以通过上文提到的`sender`回复消息：

```scala
class RedPacketGenerator extends Actor with ActorLogging {

    def receive = {
        case Shake => 
            val amount = Random.nextInt(100)
            sender ! RedPacket(amount)
    }	

}
```

## 编译运行

可以从[github](https://github.com/demonyangyue/RedPacket)下载本节完整代码并运行：

```bash
git clone https://github.com/demonyangyue/RedPacket
cd RedPacket
sbt run
```




