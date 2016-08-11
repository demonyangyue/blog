---
layout: post
title: jQuery 源码精粹0 -- 引言
date: 2016-08-04
categories: Programming
tags: [JavaScript, jQuery]
---

## 缘起

作为世界上被[误解最深的编程语言](http://javascript.crockford.com/javascript.html)，JavaScript承载了很多并不公平的指责。

相对于它一无是处的远房表兄Java，JavaScript至少在这三处设计得非常优雅：

* 对象字面量表示法。（以及JSON）
* 基于原型链的继承模型
* 匿名函数块

以我对JavaScript粗浅的理解，编写优秀的JavaScript程序，基本是把这三点用到极致，并且避免所有其余JS糟粕的过程。jQuery, 作为应用最广的JavaScript库，是一个很好的学习范例。

本系列博客，旨在和你一起暂时远离html和css的泥沼，徜徉在jQuery源码的沙滩，寻找那些闪光的贝壳。

<!--more-->
## 环境准备

下载代码：

```
git clone https://github.com/jquery/jquery
```

切换到最新的release 版本

```
git checkout tags/3.1.0
```

jQuery代码随着版本的更新，自身也在不断地进化，推荐阅读最新版本的代码。本文写作的时候，jQuery 最新的版本是3.1.0


## 代码架构

jQuery的实现代码位于src目录，统计一下代码行数：

```
find src -name "*.js" | xargs wc -l
```
总计9008行，代码量并不大，正好适于学习。

首先分析一下jQuery主要功能与源码之间的映射关系：

* 代码入口
	- jquery.js
* 核心模块
	- core.js
* 选择器模块
	- selector-sizzle.js
	- selector-native.js
* 节点处理模块
	- attributes.js
	- css.js
	- data.js
	- manipulation.js
	- offset.js
	- traversing.js
* ajax 模块
	- ajax.js
	- deferred.js
	- callbacks.js
* 事件模块
	- event.js	
* 效果及动画模块	
	- effects.js
	- queue.js
	
## jquery.js

```javascript
define( [
	"./core",
	"./selector",
	"./traversing",
	"./callbacks",
	"./deferred",
	......
	"./exports/amd"
], function( jQuery ) {

"use strict";

return ( window.jQuery = window.$ = jQuery );

} );

```

代码并不长，主要是导入相关的模块，并设置著名的`$`的全局变量。一些有趣的问题：

* define并非JavaScript的关键字，它是在哪里被定义，又是如何实现模块导入的呢？

jQuery利用grunt定义编译任务，从代码`build/tasks/build.js`中，我们可以看到代码块：

```javascript
function convert( name, path, contents ) {
//...
// Remove define wrappers, closure ends, and empty declarations
  contents = contents
	.replace( /define\([^{]*?{\s*(?:("|')use strict\1(?:;|))?/, "" )
	.replace( rdefineEnd, "" );
//...
}
```

在执行`build`任务时，`require.js`会拼接所有的源码文件，作为`contents`参数传入`convert`函数，在`convert`执行的过程中，`define`包含的内容被相应的JS代码替换。


* 在最终发布时，jquery.js会被编译成什么样子呢？

可以在`dist/jquery.js`找到答案，简化后的版本：

```javascript
( function( global, factory ) {
    factory(global);
})(window, function (window, noGlobal){
  
  "use strict";

  var arr = [];
  var document = window.document;
  var getProto = Object.getPrototypeOf;
  var slice = arr.slice;
  
  //....
  var jQuery =  function( selector, context ) {
		return new jQuery.fn.init( selector, context );
  };
  
  //...
  window.jQuery = window.$ = jQuery;
  return jQuery;

});
```

* 为什么要设计define呢？看起来好像发明了新的语法。

现代的构建工具，比如ruby世界的rake， scala世界的sbt以及JS世界的grunt，都拥有自定义的构建语法，这些工具被称作领域专属语言（DSL），基于原有的宿主语言实现，看起来像是发明了一套新的语法。用户在定义构建任务时，不仅可以使用工具提供的功能，同时也可以使用原宿主语言的所有特性，强大而灵活。



