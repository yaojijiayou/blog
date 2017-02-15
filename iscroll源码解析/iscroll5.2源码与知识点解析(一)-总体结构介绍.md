# iScroll5.2源码与知识点解析(一)-1,初探

## 简介:

iScroll是一个比较知名的js库，主要用来解决浏览器滚动不够顺滑的问题。本系列文章将对iScroll5.2.0版本的源码做一些梳理，同时挑一些有学习意义的知识点做讲解。

## 代码skeleton:

IScroll的代码结构很简单:

```

//代码写在一个自调用函数里
//这样写的好处是为了创造了一个独立的作用域，
//使得该作用域内定义的函数、变量等不会和第三方库冲突
(function (window, document, Math) {
//以外部入参的形式获取window，document，Math对象，
//这样可以缩短访问这些变量的时间，因为访问的作用域链路径变短了
	
	//rAF变量先按下不表
	var rAF = window.requestAnimationFrame	||
		window.webkitRequestAnimationFrame	||
		window.mozRequestAnimationFrame		||
		window.oRequestAnimationFrame		||
		window.msRequestAnimationFrame		||
		function (callback) { window.setTimeout(callback, 1000 / 60); };

	//定义一个工具变量，里面实现了一些工具方法
	var utils = (function () {
				...
	})();

	//IScroll对象的构造函数
	function IScroll (el, options) {
		...
	}
	
	//给IScroll原型赋值，这样所有IScroll的实例，就可以共用原型上的方法了
	IScroll.prototype = {
		...
	}
	
	//创建一个默认的滚动条
	function createDefaultScrollbar (direction, interactive, type) {
		...
	}

	//Indicator的构造函数，Indicator的定义是什么，请看下文
	function Indicator (scroller, options) {
		...
	}

	//构造Indicator的原型
	Indicator.prototype = {
		...
	}
	
	//将工具函数作为IScroll的一个静态属性
	IScroll.utils = utils;
	
	//导出IScroll
	if ( typeof module != 'undefined' && module.exports ) {
		//检测是否运行于遵循 CommonJS 规范的环境中，比如 NodeJS
		module.exports = IScroll;
	} else if ( typeof define == 'function' && define.amd ) {
		//检测是否是运行在遵循 AMD 规范的环境之中
	    define( function () { return IScroll; } );
	} else {
		window.IScroll = IScroll;
	}

})(window, document, Math);
```

## 术语解释

具体请看下面的示意图:

![术语示意图](https://github.com/yaojijiayou/blog/blob/master/img/glossary.png)

####  1.wrapper是外部的容器，其高度、位置正常情况下是固定的。可以把它理解为一个窗口，窗口保持不动，但是映入窗户的内容在动；

####  2.scroller（下图中红色部分）是实际“滚动”的对象，当滚动的时候，实际上是scroller的位置在调整；

如下图所示:

![scroller示意图](https://github.com/yaojijiayou/blog/blob/master/img/scroller.png)

所以，**IScroll库的核心内容就是:如何让scroller部分能不断随着用户的动作来变换位置，从而实现滚动的效果。**；

####  3.indicator代表滑块，可以体现目前显示的内容，在整个对象中大概所处的位置；

####  4.scrollbar代表滚动槽。

####  5.参看官网上提供的**最简IScroll初始化代码**

```
<head>
...
<script type="text/javascript" src="iscroll.js"></script>
<script type="text/javascript">
var myScroll;
function loaded() {
    myScroll = new IScroll('#wrapper');
}
</script>
</head>
...
<body onload="loaded()">
<div id="wrapper">
    <ul>
        <li>...</li>
        <li>...</li>
        ...
    </ul>
</div>
</body>

```

其中id为wrapper的div就是wrapper

wrapper的第一个子元素是scroller，上方代码中的ul标签就是wrapper

## 本篇总结

敬请关注下一篇！





