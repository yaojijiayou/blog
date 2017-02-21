# iScroll5.2源码与知识点解析(二)-事件系统之浏览器标准事件系统

在上一篇[iScroll5.2源码与知识点解析(一)-初探](https://github.com/yaojijiayou/blog/blob/master/iframe%E5%BC%95%E5%85%A5%E9%A1%B5%E9%9D%A2%E4%B8%8E%E4%B8%BB%E9%A1%B5%E9%9D%A2%E7%9A%84viewport%E8%AE%BE%E7%BD%AE%E5%86%B2%E7%AA%81%E9%97%AE%E9%A2%98%E6%8E%A2%E8%AE%A8.md)中，我大致介绍了一下iScroll的代码结构和术语，本篇将着重介绍一下iScroll的事件系统。iScroll的事件系统可以分为两部分:
> * 浏览器标准事件系统
> * 自定义事件系统


## 浏览器标准事件系统:

所谓浏览器标准事件系统，其涵盖的事件主要就是类似于:touchstart、touchmove、touchcancel、touchend、click、mousedown、mousemove、mousecancel等浏览器支持的事件。

这部分事件的绑定，主要涉及到的代码如下:

```
var utils = (function () {

	var me = {};

	...

	me.addEvent = function (el, type, fn, capture) {
		el.addEventListener(type, fn, !!capture);
	};

	me.removeEvent = function (el, type, fn, capture) {
		el.removeEventListener(type, fn, !!capture);
	};

	...

	return me;

})();


IScroll.prototype = {

	...

	_initEvents: function (remove) {

		var eventType = remove ? utils.removeEvent : utils.addEvent,
			target = this.options.bindToWrapper ? this.wrapper : window;

		eventType(window, 'orientationchange', this);
		eventType(window, 'resize', this);

		if ( this.options.click ) {
			eventType(this.wrapper, 'click', this, true);
		}

		if ( !this.options.disableMouse ) {
			eventType(this.wrapper, 'mousedown', this);
			eventType(target, 'mousemove', this);
			eventType(target, 'mousecancel', this);
			eventType(target, 'mouseup', this);
		}

		if ( utils.hasPointer && !this.options.disablePointer ) {
			eventType(this.wrapper, utils.prefixPointerEvent('pointerdown'), this);
			eventType(target, utils.prefixPointerEvent('pointermove'), this);
			eventType(target, utils.prefixPointerEvent('pointercancel'), this);
			eventType(target, utils.prefixPointerEvent('pointerup'), this);
		}

		if ( utils.hasTouch && !this.options.disableTouch ) {
			eventType(this.wrapper, 'touchstart', this);
			eventType(target, 'touchmove', this);
			eventType(target, 'touchcancel', this);
			eventType(target, 'touchend', this);
		}

		eventType(this.scroller, 'transitionend', this);
		eventType(this.scroller, 'webkitTransitionEnd', this);
		eventType(this.scroller, 'oTransitionEnd', this);
		eventType(this.scroller, 'MSTransitionEnd', this);
	},

	handleEvent: function (e) {
		switch ( e.type ) {
			case 'touchstart':
			case 'pointerdown':
			case 'MSPointerDown':
			case 'mousedown':
				this._start(e);
				break;
			case 'touchmove':
			case 'pointermove':
			case 'MSPointerMove':
			case 'mousemove':
				this._move(e);
				break;
			case 'touchend':
			case 'pointerup':
			case 'MSPointerUp':
			case 'mouseup':
			case 'touchcancel':
			case 'pointercancel':
			case 'MSPointerCancel':
			case 'mousecancel':
				this._end(e);
				break;
			case 'orientationchange':
			case 'resize':
				this._resize();
				break;
			case 'transitionend':
			case 'webkitTransitionEnd':
			case 'oTransitionEnd':
			case 'MSTransitionEnd':
				this._transitionEnd(e);
				break;
			case 'wheel':
			case 'DOMMouseScroll':
			case 'mousewheel':
				this._wheel(e);
				break;
			case 'keydown':
				this._key(e);
				break;
			case 'click':
				if ( this.enabled && !e._constructed ) {
					e.preventDefault();
					e.stopPropagation();
				}
				break;
		}
	}
}
```

其中值得讲的知识点如下:

#### 1. 不兼容ie11以前的IE浏览器

给某个target绑定监听器的逻辑，封装在util的addEvent函数中，代码如下

```
me.addEvent = function (el, type, fn, capture) {
	el.addEventListener(type, fn, !!capture);
};

```
我们注意到，归根结底是调用了支持W3C标准的addEventListener函数。看过一点点jQuery源码的同学知道，单纯的addEventListener有兼容性问题，因为在ie11之前，需要用ie专属的attachEvent。从源码来看，iscroll已经抛弃掉兼容ie11之前的版本了。

如果要兼容，推荐一种写法：
```
function addEvent(elm, evType, fn, useCapture) {
	if (elm.addEventListener) {
		elm.addEventListener(evType, fn, useCapture);//DOM2.0
		return true;
	}
	else if (elm.attachEvent) {
		var r = elm.attachEvent(‘on‘ + evType, fn);//IE5+
		return r;
	}
	else {
		elm['on' + evType] = fn;//DOM 0
	}
}
```

#### 2. addEventListener的一点知识补充

我对于addEventListener函数的用法，停留在addEventListener(type,handler,useCapture)上。type是事件类型，handler是事件触发后的处理函数，useCapture决定了是在冒泡阶段还是捕获阶段触发handler。

但是读了iscroll的源码，让我学习到了一种新的addEventListener用法:
> * target.addEventListener(type, listener[, useCapture]);
> * listener 必须是一个实现了EventListener接口的对象，或者是一个函数

源码中，传入的listener参数都是this，即iscroll对象，而iscroll的原型中有一个handleEvent函数，这就使得iscroll对象成了一个“实现了EventListener 接口的对象”，当EventListener 所注册的事件发生的时候，handleEvent方法会被调用。


#### 3. 事件绑定在哪个对象上？

iscroll的配置属性中有一个bindToWrapper，该属性决定了move事件具体监听哪个对象，如果bindToWrapper设为true，则监听iscroll容器，否则则监听整个document。默认是监听document，这样的好处是:当你的手指\鼠标移出iscroll的wrapper范围，滚动仍会继续。如下图中，中间的大箭头是鼠标的拖动轨迹，如果只监听wrapper，那么途中蓝色部分的拖动，将无效。
![拖动示意图](https://github.com/yaojijiayou/blog/blob/master/img/bindtowrapper.png)

那么问题来了，假设有两个iscroll对象，事件监听都绑定在document上，那么document的move事件触发的时候，如何来判断:应该使哪个iscroll的scroller继续滚动呢？

请看下面的代码:

```
eventType(this.wrapper, 'touchstart', this);
eventType(target, 'touchmove', this);
eventType(target, 'touchcancel', this);
eventType(target, 'touchend', this);
```
touchstart事件的监听对象必须为wrapper，而move、cancel、end事件的监听对象才会根据bindToWrapper的值决定是document还是wrapper。

touchstart事件的处理对象是iscroll原型中的_start函数，在_start函数中，还把变量initiated设为非零，表示已被初始化，在结束滚动的时候，会把它设为initiated设为0，所以在move事件的回调函数中，只要判断initiated变量是否为零，就能决定是否需要滚动。


## 本篇总结

没想到我以为三言两语就能讲完的[浏览器标准事件系统]也写了这么多内容。先讲这么多吧，下一篇再讲自定义事件系统！





