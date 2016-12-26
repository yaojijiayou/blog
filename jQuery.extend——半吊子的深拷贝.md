# jQuery.extend——半吊子的深拷贝

------

用过jQuery库的小伙伴应该都用过/听过jQuery提供的extend接口，其主要功能是:将两个或更多对象的内容合并到第一个对象。

详细接口文档可见：
+ [extend api](http://api.jquery.com/?s=extend)


```
//父类
function Parent(){
	this.name = "P";
}
Parent.prototype.func=function(){
	console.log("Parent");
}
//子类
function Child(){
}
Child.prototype=new Parent();

```

有同学问我，为什么最后一行的代码是：
```
Child.prototype=new Parent();
```

而不是
```
Child.prototype=Parent.prototype;
```
呢？

## 在解答之前我，我先把两种实现方式的结果给出：
```
//父类
function Parent(){
	this.name = "P_Name";
}
Parent.prototype.func=function(){
	console.log("Parent Func");
}
//子类
function Child(){
}
Child.prototype=new Parent();

Child.prototype.func = function(){
	console.log("Child Func");
}
var child = new Child;
var parent = new Parent;

child.func();//Child Func
console.log(child.name);//P_Name
parent.func();//Parent Func

function Child2(){
}
Child2.prototype=Parent.prototype;

Child2.prototype.func = function(){
	console.log("Child2 Func");
}

var child2 = new Child2;
child2.func();//Child2 Func
console.log(child2.name);//undefined
parent.func();//Child2 Func
```

## 来一张说明一切的无码大图：

![](https://raw.githubusercontent.com/yaojijiayou/blog/master/img/2.png)

## 一切都是这么清晰，yeah~