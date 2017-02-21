# iScroll5.2源码与知识点解析(三)-事件系统之自定义事件

接上一篇[iScroll5.2源码与知识点解析(二)-事件系统之浏览器标准事件](https://github.com/yaojijiayou/blog/blob/master/iframe%E5%BC%95%E5%85%A5%E9%A1%B5%E9%9D%A2%E4%B8%8E%E4%B8%BB%E9%A1%B5%E9%9D%A2%E7%9A%84viewport%E8%AE%BE%E7%BD%AE%E5%86%B2%E7%AA%81%E9%97%AE%E9%A2%98%E6%8E%A2%E8%AE%A8.md)，本篇继续介绍iScroll的事件系统中的自定义事件部分。

iScroll自定义事件系统的实现很简单，但对于我们自行实现一套事件机制还是有一点借鉴意义的。主要涉及到的代码如下:

```
on: function (type, fn) {
	if ( !this._events[type] ) {
		this._events[type] = [];
	}

	this._events[type].push(fn);
},

off: function (type, fn) {
	if ( !this._events[type] ) {
		return;
	}

	var index = this._events[type].indexOf(fn);

	if ( index > -1 ) {
		this._events[type].splice(index, 1);
	}
},

_execEvent: function (type) {
	if ( !this._events[type] ) {
		return;
	}

	var i = 0,
		l = this._events[type].length;

	if ( !l ) {
		return;
	}

	for ( ; i < l; i++ ) {
		this._events[type][i].apply(this, [].slice.call(arguments, 1));
	}
}
```

on、off、_execEvent三个函数分别对应:绑定事件处理函数、解绑、触发事件。

数据结构也比较简单，用了变量_events来记录不同事件对应的处理函数，形如:

{
	"scrollEnd":[handler1,handler2],
	"eventXXX":[handlerA,handlerB]
}



## 本篇总结

这篇有点水啊，下一篇开始讲最核心的部分:**如何让scroller能顺滑地滚动**；





