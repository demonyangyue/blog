---
layout: post
title: jQuery 源码精粹3 -- 节点处理
date: 2016-08-15
categories: Programming
tags: [JavaScript, jQuery]
---

## 引言

jQuery支持丰富的DOM节点操作：

* 属性设置
	- attributes.js
	- css.js
	- data.js
* 插入或删除节点
	- manipulation.js
* 节点遍历
	- traversing.js

由于JavaScript已经对操作DOM节点提供了基础性支持，jQuery在实现这些功能的时候，主要是提供一层封装。仔细研读源码实现，仍可收获许多启发：

<!--more-->

## attributes.js

关注其核心方法`attr`，根据jQuery API [文档](http://api.jquery.com/attr/)， `attr`方法主要有两种调用形式：

* `.attr( attributeName )` ：获取首个匹配节点的属性值
* `.attr( attributeName, value )` ： 设置所有匹配节点的属性值， 有三种调用形式：
	- `.attr( attributeName, value )` : `value`是普通的`String`类型
	- `.attr( attributes )` ： `attributes` 是一个由`attribute-value`键值对构成的`PlainObject`
	- `.attr( attributeName, function )`: 需设置的属性值由`function`的返回值提供
	
### 片段一

实际的代码实现：

```jvascript
jQuery.fn.extend( {
	attr: function( name, value ) {
		return access( this, jQuery.attr, name, value, arguments.length > 1 );
	}
} );

```

`access`方法：

```javascript
// Multifunctional method to get and set values of a collection
// The value/s can optionally be executed if it's a function
var access = function( elems, fn, key, value, chainable, emptyGet, raw ) {
... };	
```

这里需要指出的是，之前提到的`attr`方法的各种调用形式，无论是`get`还是`set`, 无论是传递`plain object`还是`function`,所有的实现都是在`access`中，导致`access`的内部代码比较凌乱复杂。

这是一个值得商榷的设计，因为它违反了KISS原则，为了获得灵活性牺牲了可读性。当然，由于JavaScript本身传递函数参数的缺陷（例如不支持函数重载）， 这么做也是无奈之举。

## css.js和data.js

和`attr.js`类似，`css.js`的目的是为了存取节点的css属性，`data.js`的目的是为了存取节点私有数据。这三个文件的实现代码存在很大程度的重复，是jQuery未来实现可以改进的地方。

## manipulation.js

`manipulation.js`中实现了操作DOM节点（比如插入和删除）的逻辑。

### 片段二

拿插入方法`append`举例：

```javascript
	append: function() {
		return domManip( this, arguments, function( elem ) {
			if ( this.nodeType === 1 || this.nodeType === 11 || this.nodeType === 9 ) {
				var target = manipulationTarget( this, elem );
				target.appendChild( elem );
			}
		} );
	},
```

可以看到，底层调用了javascript的`appendChild`方法，来实现插入节点，比较简单。

### 片段三

这一段是`manipulation.js`中最有趣的代码：

```javascript
jQuery.each( {
	appendTo: "append",
	prependTo: "prepend",
	insertBefore: "before",
	insertAfter: "after",
	replaceAll: "replaceWith"
}, function( name, original ) {
	jQuery.fn[ name ] = function( selector ) {
		var elems,
			ret = [],
			insert = jQuery( selector ),
			last = insert.length - 1,
			i = 0;

		for ( ; i <= last; i++ ) {
			elems = i === last ? this : this.clone( true );
			jQuery( insert[ i ] )[ original ]( elems );

			// Support: Android <=4.0 only, PhantomJS 1 only
			// .get() because push.apply(_, arraylike) throws on ancient WebKit
			push.apply( ret, elems.get() );
		}

		return this.pushStack( ret );
	};
} );
```

对于`append`方法，有一个对应的类似方法`appendTo`, 二者的唯一区别在于调换了source和target，相应的还有`prependTo`, `insertBefore`, `insertAfter`和`replaceAll`, 他们都是基于原方法，调用`jQuery( insert[ i ] )[ original ]( elems );`得以交换source和target。

这类优雅的实现，得益于JavaScript语言本身较强的动态性，极大地减少了重复代码。

## traversing.js

`traversing.js`中实现了遍历DOM节点树的逻辑。由于JavaScript本身已经对遍历DOM节点树提供了较好的支持，jQuery只是提供了一层很浅的封装。

### 片段四

抽取一段有趣的代码：

```javascript

jQuery.each( {
	parents: function( elem ) {
		return dir( elem, "parentNode" );
	},
	parentsUntil: function( elem, i, until ) {
		return dir( elem, "parentNode", until );
	},
	
	...
}， function (name, fn) {
...
} );
```
许多遍历方法如`parents`，`nextAll`, `prevUntil`,底层都是调用了`dir`方法实现,`dir`方法是如何定义的的呢？

### 片段五

```javascript
return function( elem, dir, until ) {
	var matched = [],
		truncate = until !== undefined;

	while ( ( elem = elem[ dir ] ) && elem.nodeType !== 9 ) {
		if ( elem.nodeType === 1 ) {
			if ( truncate && jQuery( elem ).is( until ) ) {
				break;
			}
			matched.push( elem );
		}
	}
	return matched;
};
```

根据传入的`dir`及`until`参数，按方向遍历获取所有满足条件的节点，再次展现了JavaScript较强的动态性。