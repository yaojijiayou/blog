# [jQuery源码拾遗-1]jQuery.extend——半吊子的深拷贝

------

用过jQuery库的小伙伴应该都用过/听过jQuery提供的extend接口，其主要功能是:将两个或更多对象的内容合并到第一个对象。

详细接口文档可见[extend api](http://api.jquery.com/?s=extend)

使用extend时，如果第一个参数传入的是true，则表示需要进行“深拷贝”。但为何我“毁谤”其为“半吊子的深拷贝”呢？

首先，请看下面的一段代码:


```
function Obj(){
	this.a = 1;
}

var obj = new Obj();

var tobeCloned = {o:obj};

var result  = $.extend(true,{},tobeCloned);

tobeCloned.o.a = 2;

console.log(result.o.a);//输出是什么呢？

```
请问，输出应该是1还是2呢？如果从字面意义上理解“深拷贝”，是不是应该输出1？

但结果却不如我们所愿，输出的是2。

这是为什么呢？

请看jquery.extend函数的注释版源码(版本为1.7)


```
jQuery.extend = jQuery.fn.extend = function() {

	var options, name, src, copy, copyIsArray, clone,
		target = arguments[0] || {},
		i = 1,
		length = arguments.length,
		deep = false;
	/*
	变量 options：指向某个源对象。
	变量 name：表示某个源对象的某个属性名。
	变量 src：表示目标对象的某个属性的原始值。
	变量 copy：表示某个源对象的某个属性的值。
	变量 copyIsArray：指示变量 copy 是否是数组。
	变量 clone：表示深度复制时原始值的修正值。
	变量 target：指向目标对象,申明时先临时用第一个参数值。
	变量 i：表示源对象的起始下标，申明时先临时用第二个参数值。
	变量 length：表示参数的个数，用于修正变量 target。
	变量 deep：指示是否执行深度复制，默认为 false。

	ps:源对象指的是把自己的值付给别人的对象；目标对象指的是被源对象赋值的对象
	*/

	// 如果第一个参数传入的是布尔值
	if ( typeof target === "boolean" ) {
		deep = target;//设置deep变量，确定是深拷贝还是浅拷贝
		target = arguments[1] || {};//将目标对象设为第二个参数值。
		i = 2;//源对象的起始下标设为2（即从第三个参数开始算源对象）
	}

	// Handle case when target is a string or something (possible in deep copy)
	//嗯，原英文解释的很清楚
	if ( typeof target !== "object" && !jQuery.isFunction(target) ) {
		target = {};
	}

	// 如果没有目标对象，那么目标对象就是jquery对象
	if ( length === i ) {
		target = this;
		--i;
	}

	拷贝的核心部分代码
	for ( ; i < length; i++ ) {//遍历源对象
		// Only deal with non-null/undefined values
		if ( (options = arguments[ i ]) != null ) {//options就是源对象
			// Extend the base object
			for ( name in options ) {//遍历源对象的属性名
				src = target[ name ];//获取目标对象上，属性名对应的属性
				copy = options[ name ];//获取源对象上，属性名对应的属性

				// 如果复制值copy 与目标对象target相等，
				//为了避免深度遍历时死循环，因此不会覆盖目标对象的同名属性。
				if ( target === copy ) {
					continue;
				}

				//递归地将源对象上的属性值合并到目标对象上
				//如果是深拷贝，且待拷贝的对象存在，且是普通对象或是数组
				//这一个判断条件非常关键，这正是之前疑问的症结
				//首先，普通对象的定义是:通过 "{}" 或者 "new Object" 创建的
				//回到之前的疑问，目标对象tobeCloned的属性o对象的obj不是普通对象，也不是数组，所以程序不会走到下面的分支
				if ( deep && copy && ( jQuery.isPlainObject(copy) || (copyIsArray = jQuery.isArray(copy)) ) ) {
					if ( copyIsArray ) {
						//如果是数组
						copyIsArray = false;
						clone = src && jQuery.isArray(src) ? src : [];

					} else {
						clone = src && jQuery.isPlainObject(src) ? src : {};
					}

					// 递归地拷贝
					target[ name ] = jQuery.extend( deep, clone, copy );

				} else if ( copy !== undefined ) {
				//会走到这个分支，这个分支的处理很暴力，就是把源对象的属性，直接赋给源对象。
				//对于上文中tobeCloned对象的属性o，没有进一步递归地拷贝，而是直接把引用赋给源对象
				//所以改变tobeCloned的o属性时，目标对象的o属性也被改变了。
					target[ name ] = copy;
				}
			}
		}
	}

	// Return the modified object
	return target;
};
```

以上。