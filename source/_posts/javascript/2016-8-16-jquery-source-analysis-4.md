---
layout: post
title: jQuery 源码精粹4 -- defer与promise
date: 2016-08-16
categories: Programming
tags: [JavaScript, jQuery]
---

## 引言

JavaScript采用回调函数来处理异步编程，但是用户经常会陷入一种被称作回调金字塔([Pyramid of Doom](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)))的困境，即在回调函数内部嵌套其它回调函数，层层嵌套使得代码清晰性急剧下降。

```javascript
asyncOperation(function(data){
  anotherAsync(function(data2){
      yetAnotherAsync(function(){
      });
  });
});
```

defer和promise是jQuery实现中，最难也最有趣的部分，它们以优雅的方式实现了JS世界中的异步回调。

```javascript
promiseSomething()
.then(function(data){
    return anotherAsync();
})
.then(function(data2){
    return yetAnotherAsync();
})
.then(function(){
});
```

<!--more-->

## 基础知识

### Defer和Promise

一个 Promise 对象代表一个目前还不可用，但是在未来的某个时间点可以被解析的值。

类似于代理模式，Promise对象扮演真实数据对象的代理。Defer对象又对Promise做了一层封装，向用户屏蔽了Promise对象的内部私有方法。

通过给调用直接返回一个Defer对象，用户代码不必阻塞等待结果，可以继续向下执行。Defer和Promise使得用户可以用同步的方式编写异步代码，同步为表，异步为里，兼具同步代码的清晰性和异步代码的高效性。

许多其它语言也提供了对Defer和Promise的支持。

* 函数式编程是Defer和Promise的起源，函数式世界里通常称作Future和Promise，可以在Scala, Erlang 及 Cloure看到相应实现。
* 著名的Python twisted库，也有自己的[Defer实现](https://twistedmatrix.com/documents/14.0.1/core/howto/defer.html)。

### 回调函数与回调链

Promise对象可以有三种状态：

* 未完成 (unfulfilled)
* 完成 (fulfilled)
* 失败 (failed）

Promise 的状态只能由**未完成**转换成完成，或者**未完成**转换成**失败**。

可以在Promise 对象上绑定一串回调函数，构成一个回调函数链。一旦真实数据变得可用，基于当前Promise对象的状态，调用相应的回调函数。

![](/images/deferred-process.png)

## 代码分析

### callbacks.js

`jQuery.Callbacks`对象本质上维护了一个回调列表：

```javascript
jQuery.Callbacks = function( options ) {
	...
	// Actual callback list
	list = [],
	...
}
```
其核心方法为`fire`, 定义了依次触发列表中每个回调函数的逻辑：

```javascript

		// Fire callbacks
		fire = function() {

			// Execute callbacks for all pending executions,
			// respecting firingIndex overrides and runtime changes
			fired = firing = true;
			for ( ; queue.length; firingIndex = -1 ) {
				memory = queue.shift();
				while ( ++firingIndex < list.length ) {

					// Run callback and check for early termination
					if ( list[ firingIndex ].apply( memory[ 0 ], memory[ 1 ] ) === false &&
						options.stopOnFalse ) {

						// Jump to end and forget the data so .add doesn't re-fire
						firingIndex = list.length;
						memory = false;
					}
				}
			}


```

核心的代码是`list[ firingIndex ].apply( memory[ 0 ], memory[ 1 ] `, 定义了如何触发回调函数。

作为一个列表，`Callbacks object`也定义了相应的`add` 和 `remove`方法，用来添加或删除某个回调函数。

初始化`Callbacks object`的时候，可以传入一个`option`字符串，具体可参考。jQuery API [文档](http://api.jquery.com/jQuery.Callbacks)

### deferred.js

阅读`jQuery.Deferred`对象的定义，首先发现的有趣地方在于，其定义了一个`promise`方法，返回一个扩展的`promise`对象：

```javascript
	Deferred: function( func ) {
		// Get a promise for this deferred
		// If obj is provided, the promise aspect is added to the object
		promise: function( obj ) {
			return obj != null ? jQuery.extend( obj, promise ) : promise;
		}
	}
```

对于内部的`promise`对象来说，最核心的是`then`方法：

```javascript
	promise = {
		then: function( onFulfilled, onRejected, onProgress ) {
		...
		}
	}
```

按照`Promises/A`规范的定义，then 方法可以接受 3 个函数作为参数。前两个函数对应 promise 的两种状态 fulfilled 和 rejected 的回调函数。第三个函数用于处理进度信息。

`then`方法内部，最核心的定义是`resolve`方法，用来计算Promise对象代理的真实数据，实现状态的变迁, 并触发相应的回调函数`handler`。

```javascript
	function resolve( depth, deferred, handler, special ) {
		...
	}
```

实际的`Promise`对象，其代理的真实数据也可能是一个`Promise`对象。`resolove`方法的有趣之处在于通过递归的方式解析层层嵌套的`Promise`对象:

```javascript
returned = handler.apply( that, args );

then = returned && ( typeof returned === "object" || typeof returned === "function" ) && returned.then;

if ( jQuery.isFunction( then ) ) {
	maxDepth++;

	then.call(
		returned,
		resolve( maxDepth, deferred, Identity, special ),
		resolve( maxDepth, deferred, Thrower, special ),
		resolve( maxDepth, deferred, Identity, deferred.notifyWith )
	);
}
```

读完`deferred.js`的实现，试着解答困惑自己的一些问题:

* [Q] Promise对象如何计算其代理的真实数据对象?
* [A] 当客户端代码调用`then`或者`done`方法时，底层会调用`Promise`对象的`resolve`方法，计算其代理的真实数据对象并实现状态变迁，根据对象的不同状态调用相应的`fulfil`, `reject`或`progress`函数。	

* [Q] 何时触发该计算？
* [A] 解析`Promise`对象代理的数据是一个异步的操作，可以看到`window.setTimeout( process );` 这样的代码，表明由JS解析器选择合适的时机来触发`process`。

* [Q] 回调链是如何构建的？
* [A] 在`then`方法实现的最后，我们可以看到代码`return jQuery.Deferred(...).promise(); `, 证明其返回的仍然是一个`Promise`对象，于是可以继续添加`.then`调用，构成回调链。


### ajax.js

理解了`defer`和`promise`, 再来看`ajax.js`的实现，就会变得十分简单。核心的定义是`$.ajax`方法:

```javascript
jQuery.extend( {

	// Main method
	ajax: function( url, options ) {
	...
	}
})

```

`ajax`底层调用的是JS中的XHR(XmlHttpRequest),所以我们在代码中可以看到一个对应的`jqXHR`对象：

```javascript
// Fake xhr
jqXHR = {
	readyState: 0,
	...
}
```

为了保证异步，`jqXHR`被实现成一个`Promise`对象：

```javascript

// Attach deferreds
deferred.promise( jqXHR );
```

我们可以在后续代码中看到`jqXHR`对象上绑定的回调函数：

```javascript
// Install callbacks on deferreds
completeDeferred.add( s.complete );
jqXHR.done( s.success );
jqXHR.fail( s.error );
```

其余代码涉及`ajax`通信的琐碎细节，在此不再赘述。

## 参考文档
1. [deferred对象详解](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)

1. [Promise 的工作原理](https://blog.coding.net/blog/how-do-promises-work)
