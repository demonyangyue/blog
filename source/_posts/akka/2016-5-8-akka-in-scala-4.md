---
layout: post
title: Akka in Scala Part 4 - 测试
date: 2016-05-08
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

按测试驱动开发(TDD, Test Driven Development)的理念，测试代码应先于产品代码被编写，这样不仅保证了产品代码的测试覆盖率，并且可以强制开发者在设计之初就考虑代码将如何被使用，如何才能易于测试。

Akka基于`ScalaTest`框架，提供了`akka-testkit`模块来支持单元测试和集成测试的编写。

<!--more-->

## 加入依赖

修改`build.sbt`并加入相应的测试模块：

```scala
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor" % "2.3.11",
  "com.typesafe.akka" %% "akka-testkit" % "2.3.11" % "test",
  "org.scalatest" %% "scalatest" % "2.2.4" % "test")
```

运行`sbt update`更新依赖。

## 加入测试代码

现在我们给`RedPakcetClient`加上相应的测试代码：

```scala
import akka.actor.{ActorSystem, Actor, Props}
import akka.testkit.{ TestActorRef, TestKit, ImplicitSender, TestProbe }
import org.scalatest.WordSpecLike
import org.scalatest.Matchers
import org.scalatest.BeforeAndAfterAll
import akka.testkit.EventFilter
import com.typesafe.config.ConfigFactory

class RedPacketClientSpec() extends TestKit(ActorSystem("RedPacketClientSpec", ConfigFactory.parseString("""akka.loggers = ["akka.testkit.TestEventListener"]"""))) with ImplicitSender
with WordSpecLike with Matchers with BeforeAndAfterAll {

  import RedPacketGenerator._
  import RedPacketClient._


  override def afterAll {
    TestKit.shutdownActorSystem(system)
  }

  "A red packet Client" must {
    val redPacketClient = TestActorRef(Props(classOf[RedPacketClient], testActor))

    "receive a RedPacket after Shake" in {
      redPacketClient.receive(Shake)
      expectMsgPF() {
        case RedPacket =>
      }
    }

    "output the amount after open" in {
      val amount = 100
      EventFilter.info(pattern = "Open a red packet, get \\$" + amount, occurrences = 1) intercept {
        redPacketClient.receive(OpenPacket(amount))
      }
    }

  }
}
```

## 辅助组件
`akka-testkit`提供了一系列组件来帮助我们实现测试代码：

* ImplicitSender: `akka-testkit` 提供了一个`testActor`来作为隐式的消息发送者，通常我们会使用`testActor`作为被测试actor的对端，这样做的好处在于：

	1. 让代码专注于被测Actor自身的消息收发，与外界Actor的交互由`testActor`代理。
	1. 通过隐式的`testActor`,可以直接调用Akka提供的一系列断言方法。
	
* TestKit: 提供了一个与Actor交互的基础框架，以及一系列的测试断言来帮助我们处理Actor的回复消息。

	1. `expectMsg[T](d: Duration, msg: T): T` 指定时间内期望收到回复的`msg`.
	1. `expectMsgPF[T](d: Duration)(pf: PartialFunction[Any, T]): T` 使用指定的偏函数对收到的回复消息进行验证。
	1. `expectNoMsg(d: Duration)` 在指定的时间内期望不收到任何消息。
	1. `fishForMessage(max: Duration, hint: String)(pf: PartialFunction[Any, Boolean]): Any` 指定时间内，持续接受消息直到偏函数返回`True`。
	1. 更多的方法请参考akka[文档](http://doc.akka.io/docs/akka/2.4.4/scala/testing.html)。

* TestActorRef: 不同于普通的ActorRef, TestActorRef可以让我们直接访问到内部Actor的状态。虽然打破封装通常不是一个好的实践，但是在某些情况下可以方便测试代码的实现。

```scala
  val testRedPacketClient = TestActorRef(Props(classOf[RedPacketClient], testActor))
  val a = redPacketShaker.underlyingActor
```

## 测试日志消息
`akka-testkit`也支持对日志消息的测试，客户端代码需要：

* 使用`TestEventListener`来替代通常的事件处理器。
* 使用`akka.testkit.EventFilter`来检查日志信息。

## 运行测试

从[github](https://github.com/demonyangyue/RedPacket)下载代码，并执行`sbt test`即可运行测试。

如果遵循测试驱动开发，希望每次保存代码修改后自动执行测试，可以运行命令：

```bash
sbt
~test
```

