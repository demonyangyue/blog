---
layout: post
title: jQuery 源码精粹2 -- 选择器
date: 2016-08-11
categories: Programming
tags: [JavaScript, jQuery]
---

## 引言

jQuery 主要解决两个问题：

* 对DOM节点进行操作。
* 发送Ajax请求并解析响应。

操作DOM节点之前，首先需要选中节点，这一篇我们来研究和节点选择器相关的代码。

<!--more-->

## init.js

### 片段一

在上一篇我们提到，在生成`jQuery.fn.init`实例时，通过解析传入的`selector`参数，根据不同的`selector`形式，生成相应的`jQuery object`，那么背后的代码具体是如何实现的呢？

```javascript
	init = jQuery.fn.init = function( selector, context, root ) {
		var match, elem;

		// HANDLE: $(""), $(null), $(undefined), $(false)
		if ( !selector ) {
			return this;
		}
		.....
	}
```
构造函数接收`selector`, `context` 和 `root`三个参数，`context`通常是一个`DOM element`，如果为空的话则是`document`, `root`通常是`document`。

如果传入的`selector`为空，则不作任何处理直接返回`jQuery object`。

### 片段二

```javascript
		// Handle HTML strings
		if ( typeof selector === "string" ) {
			if ( selector[ 0 ] === "<" &&
				selector[ selector.length - 1 ] === ">" &&
				selector.length >= 3 ) {

				// Assume that strings that start and end with <> are HTML and skip the regex check
				match = [ null, selector, null ];

			} else {
				match = rquickExpr.exec( selector );
			}
			...
		}
```

首先处理的是`selector`为`html string`的情况，相应的`match`变量不为空:

```javascript
			if ( match && ( match[ 1 ] || !context ) ) {

				// HANDLE: $(html) -> $(array)
				if ( match[ 1 ] ) {
					jQuery.merge( this, jQuery.parseHTML(
						match[ 1 ],
						context && context.nodeType ? context.ownerDocument || context : document,
						true
					) );
					
					return this;
				} 
```
在这种情况下，调用`jQuery.parseHTML()`方法来解析传入的`html string`,生成一组DOM节点，并调用`jQuery.merge()`方法，将这组DOM节点插入`jQuery object`。

### 片段三

```javascript
				// HANDLE: $(#id)
				else {
					elem = document.getElementById( match[ 2 ] );

					if ( elem ) {

						// Inject the element directly into the jQuery object
						this[ 0 ] = elem;
						this.length = 1;
					}
					return this;
				}


```
当传入的`string`是`#id`格式时，直接调用JavaScript原生的`getElementById`方法获取DOM节点，并将其插入`jQuery object`。

### 片段四

```javascript
			// HANDLE: $(expr, $(...))
			else if ( !context || context.jquery ) {
				return ( context || root ).find( selector );

			// HANDLE: $(expr, context)
			// (which is just equivalent to: $(context).find(expr)
			} else {
				return this.constructor( context ).find( selector );
			}
```

如果`selector`字符串是表达是形式，则调用`find`函数处理。根据`selector.js`及`selector-sizzle`中的定义： `jQuery.find = Sizzle;`, find函数调用`Sizzle`引擎，来解析selector， 关于`Sizzle`引擎的原理可以参考其[官方文档](https://sizzlejs.com)，本文限于篇幅就不作展开了。

可以从jQuery[官方文档](https://api.jquery.com/category/selectors/)获得完整的`selector`形式列表，其中的绝大部分是通过`Sizzle`引擎支持的。

### 片段五

```javascript
		// HANDLE: $(DOMElement)
		  else if ( selector.nodeType ) {
			this[ 0 ] = selector;
			this.length = 1;
			return this;

```
这里处理的是`selector`为`DOMElement`的情形，比较简单，直接将`DOMElement`插入`jQuery object`。

### 片段六

```javascript
		// HANDLE: $(function)
		// Shortcut for document ready
		  else if ( jQuery.isFunction( selector ) ) {
			return root.ready !== undefined ?
				root.ready( selector ) :

				// Execute immediately if ready is not present
				selector( jQuery );
		}
```
这里处理的是一个特殊情况，对于简写形式`$(function)`,调用其完整版函数`document.ready(selector)`。