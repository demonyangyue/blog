---
layout: post
title: jQuery 源码精粹1 -- 核心模块
date: 2016-08-10
categories: Programming
tags: [JavaScript, jQuery]
---

## 引言

核心模块`core.js`是`jquery.js`第一个包含的模块，浏览`jquery`源码文件，可以发现大部分文件都把它作为依赖。作为`jquery`最重要的模块，它到底做了哪些事情呢？

<!--more-->

## core.js

### 片段一

`core.js`首先声明了一系列的依赖模块：

```
define( [
	"./var/arr",
	"./var/document",
	"./var/getProto",
	"./var/slice",
	"./var/concat",
	"./var/push",
	"./var/indexOf",
	"./var/class2type",
	"./var/toString",
	"./var/hasOwn",
	"./var/fnToString",
	"./var/ObjectFunctionString",
	"./var/support",
	"./core/DOMEval"
], function( arr, document, getProto, slice, concat, push, indexOf,
	class2type, toString, hasOwn, fnToString, ObjectFunctionString,
	support, DOMEval ) {...} );
```

这些模块提供了对通用功能的支持，通常比较简短，比如`var/document`的内容为：

```
define( function() {
	"use strict";

	return window.document;
} );
```

`define(["./var/document"])`在编译时，会被替换成`var document = window.document;`。

在jQuery每个源文件的开头，都会看到类似的`define`语句，以优雅的形式定义了所需的依赖模块。

### 片段二：

```
var
	version = "3.1.0",

	// Define a local copy of jQuery
	jQuery = function( selector, context ) {

		// The jQuery object is actually just the init constructor 'enhanced'
		// Need init if jQuery is called (just allow error to be thrown if not included)
		return new jQuery.fn.init( selector, context );
	},
```

jQuery的API调用形式是`$(selector).action(props)`, 从这段代码可以看出，`$(selector)`返回的`jQuery object`,实际上是一个`jQuery.fn.init`实例，通过阅读`core/init.js`文件，我们可以知道在初始化实例时，解析传入的`selector`参数，根据不同的`selector`形式，生成相应的`jQuery object`(具体逻辑在下一篇分析), `jQuery object`是一个类似数组的对象，代表了一组被选中的DOM节点的集合。

## 片段三

`core.js`下一个代码片段:

```
jQuery.fn = jQuery.prototype = {
	each: function() {...},
	map: function() {...},
	first: function(){...},
	...
}
```
通过jQuery原型，定义了一些列诸如`each`,`map`,`first`等通用方法，支持类似数组的操作，所有的`jQuery object`都可以调用这些方法。

向`jQuery.fn`添加方法定义，是`jQuery`各个模块以及用户自定义plugin, 扩展`jQuery`对象的标准做法。

同时在`src/init.js`的末尾，有这段代码：

```
init.prototype = jQuery.fn;
```

于是`jQuery object`实际和`jQuery.fn.init object`实际是等价的，正如我们在片段二所提到的。

## 片段四

`core.js`末尾的代码片段：

```
jQuery.extend = jQuery.fn.extend = function() {...};
jQuery.extend({
	error: function( ) {},
	noop: function() {},
	isFunction: function() {},
	isArray: function() {},
	...
})；

```

在`jQuery.extend`代码内部，会迭代变量传入参数的每个属性，并将其赋值到`jQuery`命名空间，如果将`deep`参数设为`true`,则会递归遍历传入参数的所有属性。

`jQuery extend`很像类方法，可以让用户直接在`jQuery`命名空间定义全局方法，并以`$.function_name()`的形式进行调用。





