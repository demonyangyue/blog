---
layout: post
title: Rails源码管窥1 - 工具篇
date: 2015-05-27
categories: Programming
tags: [Rails, Web]
---

## 引言

优秀的工具可以使阅读代码变得轻松高效。

说到工具，很多人首先会想到——工欲善其事，必先利其器。

我更愿意引用古龙在《圆月弯刀》中的点睛之笔——有些人纵有神刀在手，仍是无法成为刀中之神。

工具永远只是工具，它们只是人类大脑的延伸，而非替代。

是为引言。

## Vim + Ctags

Vim 作为编辑器的神，配合上Ctags插件，阅读代码也是行云流水。

## Byebug

Ctags的不足在于，它只对代码做静态分析，但实际的程序中， 同样名字的方法可能在多个类中有不同的定义，这时候就需要在debug模式下运行代码，由调试器告诉你到底执行了哪个方法。

[Byebug](http://guides.rubyonrails.org/debugging_rails_applications.html)是ruby2.0之后默认的调试器，除了基本的调试功能外，还支持在调试时动态执行ruby语句，并且通过ruby的强大的元编程机制，反射出运行时的上下文信息。

如果你以前用过其它语言如java, perl的调试器，请试试Byebug吧，它会让你感动到流泪~

## Learn by doing

如引言中所说，最好的工具永远是大脑和双手，修改和运行代码是理解代码最高效的方式。

让我们开启rails源码的探索之旅吧~
