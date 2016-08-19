---
layout: post
title: jQuery 源码精粹5 -- event 与 effect
date: 2016-08-17
categories: Programming
tags: [JavaScript, jQuery]
---

## 引言

翻过`Defer`和`Promise`的大山，我们来到了本系列博客的最后一站，这一篇我们来研究`event.js`与`animate.js`。

<!--more-->

## event.js

关于`jQuery`的`event`模块，最有趣的代码位于`src/event/alias.js`

```javascript
jQuery.each( ( "blur focus focusin focusout resize scroll click dblclick " +
	"mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave " +
	"change select submit keydown keypress keyup contextmenu" ).split( " " ),
	function( i, name ) {

	// Handle event binding
	jQuery.fn[ name ] = function( data, fn ) {
		return arguments.length > 0 ?
			this.on( name, null, data, fn ) :
			this.trigger( name );
	};
} );
```

我们可以看到，几乎所有常用的event，底层都在调用`on`或者`trigger`方法（根据是否传入一个额外的fn参数）， 以`on`方法为例，它是如何定义的呢？

```javascript
jQuery.fn.extend( {

	on: function( types, selector, data, fn ) {
		return on( this, types, selector, data, fn );
	},
	...
}
```
在`on`方法内部，调用`jQuery.event.add`方法，给每个选中的DOM 节点绑定相应的事件。

```javascript
function on( elem, types, selector, data, fn, one ) {
	...
	return elem.each( function() {
		jQuery.event.add( this, types, fn, data, selector );
	} );
}
```

在`jQuery.event.add`方法定义内部：
	
```javascript
	if ( elem.addEventListener ) {
		elem.addEventListener( type, eventHandle );
	}
```

可以看到底层调用`addEventListener`方法将事件绑定到相应的节点。

## effect.js

`effect.js`定义了`jQuery`各种酷炫的动画效果，通读其代码，首先看到的有趣部分：

```javascript
// Generate shortcuts for custom animations
jQuery.each( {
	slideDown: genFx( "show" ),
	slideUp: genFx( "hide" ),
	slideToggle: genFx( "toggle" ),
	fadeIn: { opacity: "show" },
	fadeOut: { opacity: "hide" },
	fadeToggle: { opacity: "toggle" }
}, function( name, props ) {
	jQuery.fn[ name ] = function( speed, easing, callback ) {
		return this.animate( props, speed, easing, callback );
	};
} );
```

可以看到绝大多数的动画定义，底层都是调用的`animate`方法，区别在于传入了略微不同的`prop`参数，`animate`方法是如何定义的呢？

```javascript
jQuery.fn.extend( {
	animate: function( prop, speed, easing, callback ) {
		var empty = jQuery.isEmptyObject( prop ),
			optall = jQuery.speed( speed, easing, callback ),
			doAnimation = function() {
				...
			};
			doAnimation.finish = doAnimation;

		return empty || optall.queue === false ?
			this.each( doAnimation ) :
			this.queue( optall.queue, doAnimation );
	},
	...
}
```

`animate`方法在每个DOM节点上执行`doAnimation`函数。`doAnimation`的定义十分简单：

```javascript
			doAnimation = function() {

				// Operate on a copy of prop so per-property easing won't be lost
				var anim = Animation( this, jQuery.extend( {}, prop ), optall );

				// Empty animations, or finishing resolves immediately
				if ( empty || dataPriv.get( this, "finish" ) ) {
					anim.stop( true );
				}
			};
```

可以看到`doAnimaiton`方法内部主要是初始化了一个`Animation`对象：

```javascript
function Animation( elem, properties, options ) {
	...
	var animation = deferred.promise( {
			...
		} ),
		
	....
	
	jQuery.fx.timer(
		jQuery.extend( tick, {
			elem: elem,
			anim: animation,
			queue: animation.opts.queue
		} )
	);

	// attach callbacks from options
	return animation.progress( animation.opts.progress )
		.done( animation.opts.done, animation.opts.complete )
		.fail( animation.opts.fail )
		.always( animation.opts.always );
}
```

`Animation`的主体代码十分清晰，内部定义了一个私有的`Promise`对象`animation`, 按指定的时间间隔触发相应的动画操作，并根据传入的参数绑定完成或失败后的回调函数。

## 跋

写到这里，jquery 源码分析系列就告一段落了。分享一些感悟：

* 知其然也要知其所以然，只要洞悉了源代码，方可对使用的库有透彻的理解。
* 带着问题去阅读（无论是书和代码），可以让阅读变得有趣，让记忆变得深刻。

一千个读者眼中有一千个哈姆雷特，同一份源代码，不同的人读来应有不同的体会。希望本系列博客，可以抛砖引玉，为你阅读jQuery源码提供一些指引。