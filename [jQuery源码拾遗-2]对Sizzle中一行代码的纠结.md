# [jQuery源码拾遗-2]对Sizzle中一行代码的纠结

------

我一直在犹豫纠结要不要写这篇文章，犹豫的原因是因为本文所述的问题有一些微不足道。

但是它确确实实的我使我苦思冥想了一晚上。

如果不产出一篇文章，我觉得特别对不起这段时间死去的脑细胞。

但要申明一下:即使是在写这篇文章的时候，我也不百分百确定我的结论是正确的。

让我百思不得其解的就是Sizzle(版本1.7)中的一行代码:

```
set = Expr.relative[ parts[0] ] ?
				[ context ] :
				Sizzle( parts.shift(), context );

```

光看这一行代码，完全没有特殊之处，其上下文是这样的:

```
/*
几点说明:
1,parts存放了正则 chunker 从选择器表达式中提取的块表达式和块间关系符
比如选择器路径是:"div#a .b:nth(1) > .c",那么parts = ["div#a",".b:nth(1)",">",".c"]

2,Sizzle查询DOM时，有两种查询方向，分别是从左到右和从右到左。如果选择器路径存在位置伪类，则从左向右查找。如"div#a .b:nth(1) > .c"中的":nth(1)"就是位置伪类。下面的条件分支就是存在位置伪类时的逻辑代码。（本文只讲这种查询方式）

3,可能还是要读过Sizzle源码的人来看，我感觉背景太多了啊，讲也讲不完...
*/

if ( parts.length > 1 && origPOS.exec( selector ) ) {
	//块表达式数量大于1，且存在位置伪类，此时需要从左到右查找。
	//查询的思路是：不断缩小上下文，即不断缩小查找范围
	//以"div#a .b:nth(1) > .c"为例，
	//第一次查询。以外部传入的context（大部分是document）为上下文，先查找"div#a"，得到的结果集set，
	//第二次查询，以上一次的结果集set为上下文，查询".b:nth(1)",得到结果集为下一次查询的上下文
	//依次类推，直到结束

	if ( parts.length === 2 && Expr.relative[ parts[0] ] ) {
		//如果数组parts中只有两个元素，并且第一个是块间关系符，
		//则可以直接调用函数posProcess( selector, context, seed ) 查找匹配的元素集合
		set = posProcess( parts[0] + parts[1], context, seed );

	} else {

		set = Expr.relative[ parts[0] ] ?
			[ context ] :
			Sizzle( parts.shift(), context );
		//这一行就是我的疑惑所在，现在这行的功能是:
		//如果parts中大头的元素是块关系符，则把set赋为[context]，
		//否则，则递归调用Sizzle函数，把parts数组第一个元素作为Sizzle函数的selector参数传入
		//此时，这个参数是一个块表达式，形如:"div"或"div.className"或".className"等，只要不是关系符就行了

		<font color=#0099ff size=12 face="黑体">//我的疑问是:为什么要根据第一个元素是否是块关系符而分情况呢？</font>
		//为什么当第一个元素是块表达式时，需要立即直接执行Sizzle( parts.shift(), context )，
		//为什么：对parts首对象（为块表达式时）的处理，不放在while循环中的posProcess中呢？
		//这样的话这一行代码只需要set = context，这样代码看起来不更简单吗？

		
		//从左至右，根据块表达式一个接一个的缩小范围查询
		while ( parts.length ) {   
			selector = parts.shift();

			if ( Expr.relative[ selector ] ) {
				selector += parts.shift();
			}
			//如果第一个元素是块关系符，则再弹出一个元素，组成类似于“ > .className”的形式
			//这样查找才有意义
			
			set = posProcess( selector, set, seed );
			//posProcess：函数posProcess( selector, context, seed ) 在指定的上下文数组context下，
			//查找与选择器表达式selector 匹配的元素集合，并且支持位置伪类。
			//这里对于高云所著的《jQuery技术内幕》中的描述“选择器表达式selector由一个块间关系
符和一个块表达式组成。”
			//我有异议，此处的selector可能只由一个块表达式组成
		}
	}

}

```

posProcess函数:

```
var posProcess = function( selector, context, seed ) {
	var match,
		tmpSet = [],
		later = "",
		root = context.nodeType ? [context] : context;
		//如果context不是数组，那么就把它转成数组（当然不是它自己转）

	//把选择器路径中的伪类从路径中剔除，存入later变量
	while ( (match = Expr.match.PSEUDO.exec( selector )) ) {
		later += match[0];
		selector = selector.replace( Expr.match.PSEUDO, "" );
	}
	
	//如果选择器表达式就是一个块间关系符，则追加一个通配符"*"
	selector = Expr.relative[selector] ? selector + "*" : selector;
	
	//遍历上下文数组，调用函数Sizzle( selector, context, results, seed ) 
	//查找删除伪类后的选择器表达式匹配的元素集合，将查找结果合并到数组tmpSet 中
	for ( var i = 0, l = root.length; i < l; i++ ) {
		Sizzle( selector, root[i], tmpSet, seed );
	}
	
	//调用方法Sizzle.filter( expr, set, inplace, not )
	//用记录的伪类later 过滤元素集合tmpSet，并返回一个新数组，其中只包含了过滤后的元素
	return Sizzle.filter( later, tmpSet );
};

```


回到本文的终极疑问，我中途自己给自己找了很多答案，比如posProcess的selector参数必须是一个块间关系
符和一个块表达式组成，等。

但后来都被我自己否决了。

我的终极答案:这样写是为了

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

...

更快更节约空间。省去执行posProcess中一堆在当时情况下多余的代码，省去更多函数调用栈的开销。

（如果有不同想法，欢迎来喷...）

（确实是一篇比较无聊的文章。）

以上。