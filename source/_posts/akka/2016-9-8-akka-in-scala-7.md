---
layout: post
title: Akka in Scala Part 7 - State
date: 2016-09-08
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

<!--more-->

## become

## orElse
组合行为

## FSM
- State(S) x Event(E) -> Actions (A), State(S’) 这里State包含state name 和 state data
- stay
- using
- OnTransition
- Internal Monitoring

    - onTransition(handler)
- External Monitoring

    - SubscribeTransitionCallBack(actorRef)
    - SubscribeTransitionCallBack(actorRef)
- Timers

    - 不用actor 默认的schedule
- Stop

    - 有自己的stop方式  - stop([reason[, data]])
## 设计
使单个actor的逻辑保持简单，尽量不要使用状态切换。

