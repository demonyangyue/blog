---
layout: post
title: Scheme解释器开发札记
date: 2016-12-19
categories: Programming
tags: [Scheme, Scala, Interperter]
---

## 缘起

最早见到的`Scheme`解释器实现，是在[`The Litter Schemer`的最后一章](http://uternet.github.io/TLS/10.html)。虽然书中实现的是一个原型版本，只能解析最基本的形式，并且用外部的Scheme解释器去计算解析出的Scheme表达式(接近于作弊)，但是还是可以隐约地感受到，函数式编程在解决复杂问题时的简洁有力。

那么设计一个实际的`Scheme`解释器，会遇到哪些挑战呢？函数式编程思想在解决这些问题的时候，又会起到怎样的作用呢？带着对这些问题的好奇，用两个星期的业余时间，尝试写了一个自己的[scheme解释器](https://github.com/demonyangyue/SchemeInterpreter)。

<!--more-->

## 设计

具体细节可以参考源码，这里列出一些设计过程中的一些思考。

### 整体结构

对于Scheme解释器来说，代码主要分为两个部分 -- 解析和计算。

首先解析Scheme语句，生成语法树，然后递归计算每个节点的值。

### 执行环境

解释器程序中，我们需要全局执行环境，以查找某个符号对应的处理函数。

我们也需要局部执行环境，在函数调用时，绑定局部变量和实际的值，并在调用完成后释放。

在实际代码中，执行环境被定义为一个包含两个`Map`的元素的列表，第一个`Map`代表局部执行环境（按需生成），第二个`Map`代表全局执行环境（永久存在）。当进行符号查找时，先查找局部环境(于是局部变量具有更高优先级)，未找到再接着查找全局环境。

### 副作用的处理

在函数编程中，副作用是一个存在争议的话题，理想的函数式代码都是纯函数，不包含任何副作用(IO交互，抛出异常之类)。 

但是在现实的世界，程序会不可避免的带有副作用。函数式编程的解决方案是，尽量把副作用相关的调用放到项目的外层，保证内核由纯函数构成（Imperative shell， pure core)。

具体可以参考[`Functional Programming in Scala`](https://www.manning.com/books/functional-programming-in-scala) 第十三章 -- External Effects and I/O

### 测试

解释器需要处理各种极端情形和边界条件，没有测试的保障我们将寸步难行。

遵循TDD实践，首先写单元测试，然后再完成对应的实现代码。最后用全面的集成和验收测试保证程序的正确性。

Devil is in the details, test to rescue。

## FP 感悟

OO式编程的体验，类似于用一块块积木搭建一个房子。函数式编程的体验，更像是搭建一个多米诺骨牌阵列，奇迹发生在设计完成后推倒第一块骨牌的时刻，由此出发的连锁反应，会让你赞叹设计的精妙。如果你和我一样，被`V字仇杀队`片尾连环爆炸的场景点燃过，那么你也一定会爱上函数式编程带来的体验。

### Description over action

函数式编程的设计过程中，我们更多的是在描述问题，而不是陷于具体的实现细节。

实际编码时，就像在不断地和计算机签订契约 -- 给我一个这样的输入，我将返回给你一个那样的输出。

通过一个个契约，我们获得了对整个问题的完整描述。当我们真的给程序一个输入时，一切魔法就如当初预设的那样自然发生了。 Boom！

### Combinators

代数数据类型（Algebra Data Type）是函数式编程中强有力的工具。

你的类型只需要实现一些基本操作(Primitive funciton)，就能通过组合算子的方式自动或者大量的组合操作(Combinative function)。

[`Functional Programming in Scala`](https://www.manning.com/books/functional-programming-in-scala)，是一个非常好的示例。


### Recursive

如果要用一个词描述计算的本质的话，我会选`递归`。这一点，在读完[`The Little Schemer`](http://uternet.github.io/TLS)和[`SICP`](https://mitpress.mit.edu/sicp/full-text/book/book.html)之后体会尤深。

类似于三体里面的降维攻击，计算机世界里通过递归的方式，对复杂的问题进行降维攻击，最终将问题转化成人脑可以处理的复杂度。

## 结语

虽然在日常工作中，我们可能永远不会有机会，用`scala`或者`scheme`去做一个实际的项目，但是通过在业余时间的这个项目，增进了对函数式编程的领悟，真切地感受到了数学、代码和思考融合在一起时，所绽放出的艺术光辉。