---
layout: post
title: Akka in Scala Part 6 - Router
date: 2016-05-10
categories: Programming
tags: [Akka, Scala, Cloud Computing]
---

## 引言

<!--more-->

## 概念
一种特殊类型的Actor

## 种类
- Pool - The router creates routees as child actors and removes them from the router if they terminate.
- Group - The routee actors are created externally to the router and the router sends messages to the specified path using actor selection, without watching for termination.

## 路由策略

- The routing logic shipped with Akka are:

    - akka.routing.RoundRobinRoutingLogic
    - akka.routing.RandomRoutingLogic
    - akka.routing.SmallestMailboxRoutingLogic
    - akka.routing.BroadcastRoutingLogic
    - akka.routing.ScatterGatherFirstCompletedRoutingLogic
    - akka.routing.TailChoppingRoutingLogic
    - akka.routing.ConsistentHashingRoutingLogic
    
## 监护策略

supervisor 策略默认是escalate，如果要改成restart的话需要自己设置

## 远端actor

- 可以连接远端已经存在的actor
- 可以在程序运行时创建远端actor

    - 在程序内部动态指定配置
    - 从本地配置文件读取
    
## 示例代码