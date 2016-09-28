---
layout: post
title: Akka in Scala Part 8 - Future
date: 2016-09-09
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

<!--more-->

## 定义
Future is monad。
Promise - to hold the value which would be later consumed by Future

- object Future { def apply[T](body: => T)(implicit execctx: ExecutionContext): Future[T] }

    - 可以把ExecutionContext看成线程池，通常通过默认参数传递
    
## 应用场景
- 非阻塞IO：通常不需要使用Future（除非你在实现no-blocking的库）
- 阻塞IO：把相关的代码放到Future块中
- 耗时的计算：把相关的代码放到Future块中
- akka有其自己的应用场景
    - 一定要根据某个消息的回复进行后续处理
    - 和本次消息在同一上下文执行
    
## 使用方法

- Future callbacks

    - onSuccess
    - onFailure
    - onComplete
- Compose Futures

    - map
    - flatmap
    - for comprehension(注意彼此独立的future需要在For外部初始化),否则将串行执行
    
- Future.sequence is taking the List[Future[Int]] and turning it into a Future[List[Int]]

- 在actor中使用Future

## 代码示例

## 小结