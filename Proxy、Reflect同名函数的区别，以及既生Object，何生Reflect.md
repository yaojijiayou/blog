# Proxy、Reflect同名函数的区别，以及既生Object，何生Reflect

------

如题，本文主要从个人角度解释了在学习JavaScript ES6中的Proxy和Reflect时，遇到的两个疑问。

------

## Proxy、Reflect同名函数的区别
Proxy一共支持13种拦截操作(不是Proxy自身的函数，而是Proxy的handler支持的操作)：

> * apply(target, object, args)
> * construct(target, args)
> * get(target, propKey, receiver)
> * set(target, propKey, value, receiver)
> * defineProperty(target, propKey, propDesc)
> * deleteProperty(target, propKey)
> * has(target, propKey)
> * ownKeys(target)
> * isExtensible(target)
> * preventExtensions(target)
> * getOwnPropertyDescriptor(target, propKey)
> * getPrototypeOf(target)
> * setPrototypeOf(target, proto)

Reflect对象一共有13个静态方法：

> * Reflect.apply(target, thisArg, args)
> * Reflect.construct(target, args)
> * Reflect.get(target, name, receiver)
> * Reflect.set(target, name, value, receiver)
> * Reflect.defineProperty(target, name, desc)
> * Reflect.deleteProperty(target, name)
> * Reflect.has(target, name)
> * Reflect.ownKeys(target)
> * Reflect.isExtensible(target)
> * Reflect.preventExtensions(target)
> * Reflect.getOwnPropertyDescriptor(target, name)
> * Reflect.getPrototypeOf(target)
> * Reflect.setPrototypeOf(target, prototype)

各自的13个函数是严格一一对应的，很多文章也经常会把Proxy和Reflect放在一起讲。那么它们两者的区别是什么的？
其实两者的函数虽然同名，且经常联合起来使用，但完全不是一回事，具体的区别是：**Proxy的函数负责的是：拦截并定义拦截时具体的操作；Reflect的静态函数负责的是：最终执行对象的操作**

举一个例子：
```
let obj = {
  a: '1'
};

let handler = {
  //下一行中的set是Proxy支持的操作，意思是需要拦截target的设置值的操作，
  //同时定义了设置值操作被拦截后会执行的操作，本例中就是输出‘set’,然后执行Reflect.set函数
  set(target, key, value, receiver) {
    console.log("set");
    //下一行执行的是Reflect的set函数，是它最终执行了对target对象属性的操作。
    Reflect.set(target, key, 100, receiver)
  }
};

let objProxy = new Proxy(obj, handler);
objProxy.a = '2';

```
顺便补充两个问题：
> * console.log(objProxy.a);输出什么？
> * obj.a = 10; console.log(objProxy.a);输出什么？


## 既生Object，何生Reflect
Reflect函数中大量函数，和Object的部分函数一一对应，且有着类似的功能。为什么在已经有了Object的情况下，再增加这样一个Reflect对象呢？

在大家平日自己写程序的时候，是否经历过这样的场景：程序一开始只有一个类A，它需要用到一个工具函数funcA，一开始为了图方便，就把这个函数直接写在A里了。后来随着程序规模的扩大，发现很多地方都会用到这个funcA，且随着情况的复杂化，这个funcA需要考虑的情况变多，自身也亟需优化。到了优化的阶段，索性创建一个新的Util类，把类似的函数提取出来，统一放到Util类中。

我个人觉得Reflect的产生和上述的例子有很大的相似性。首先是Object上的函数，在实际使用中存在不便利之处，具体主要有：
```
(具体可参见[ES6新特性：JavaScript中的Reflect对象](http://taobaofed.org/blog/2016/11/03/es6-advanced/))
1：更加有用的返回值： Reflect有一些方法和ES5中Object方法一样样的， 比如： Reflect.getOwnPropertyDescriptor和Reflect.defineProperty,  不过, Object.defineProperty(obj, name, desc)执行成功会返回obj， 以及其它原因导致的错误， Reflect.defineProperty只会返回false或者true来表示对象的属性是否设置上了， 如下代码可以重构：

try {
  Object.defineProperty(obj, name, desc);
  // property defined successfully
} catch (e) {
  // possible failure (and might accidentally catch the wrong exception)
}
重构成这样：

if (Reflect.defineProperty(obj, name, desc)) {
  // success
} else {
  // failure
}
其余的方法， 比如Relect.set ， Reflect.deleteProperty, Reflect.preventExtensions, Reflect.setPrototypeOf， 都可以进行重构；

2：函数操作，  如果要判断一个obj有定义或者继承了属性name， 在ES5中这样判断：name in obj ； 或者删除一个属性 ：delete obj[name],  虽然这些很好用， 很简短， 很明确， 但是要使用的时候也要封装成一个类；

有了Reflect， 它帮你封装好了， Reflect.has(obj, name),  Reflect.deleteProperty(obj, name);

3：更加可靠的函数式执行方式： 在ES中， 要执行一个函数f，并给它传一组参数args， 还要绑定this的话， 要这么写：

f.apply(obj, args)
但是f的apply可能被重新定义成用户自己的apply了，所以还是这样写比较靠谱：

Function.prototype.apply.call(f, obj, args)
上面这段代码太长了， 而且不好懂， 有了Reflect， 我们可以更短更简洁明了：

Reflect.apply(f, obj, args)
4：可变参数形式的构造函数： 想象一下， 你想通过不确定长度的参数实例化一个构造函数， 在ES5中， 我们可以使用扩展符号， 可以这么写：

var obj = new F(...args)
不过在ES5中， 不支持扩展符啊， 所以， 我们只能用F.apply，或者F.call的方式传不同的参数， 可惜F是一个构造函数， 这个就坑爹了， 不过有了Reflect， 我们在ES5中能够这么写：

var obj = Reflect.construct(F, args)
5：控制访问器或者读取器的this： 在ES5中， 想要读取一个元素的属性或者设置属性要这样：

var name = ... // get property name as a string
obj[name] // generic property lookup
obj[name] = value // generic property update
Reflect.get和Reflect.set方法允许我们做同样的事情， 而且他增加了一个额外的参数reciver， 允许我们设置对象的setter和getter的上下this：

var name = ... // get property name as a string
Reflect.get(obj, name, wrapper) // if obj[name] is an accessor, it gets run with `this === wrapper`
Reflect.set(obj, name, value, wrapper)
访问器中不想使用自己的方法，而是想要重定向this到wrapper：

var obj = {
    set foo(value) { return this.bar(); },
    bar: function() {
        alert(1);
    }
};
var wrapper = {
    bar : function() {
        console.log("wrapper");
    }
}
Reflect.set(obj, "foo", "value", wrapper);
6：避免直接访问 __proto__ ： ES5提供了 Object.getPrototypeOf(obj)，去访问对象的原型， ES6提供也提供了Reflect.getPrototypeOf(obj) 和  Reflect.setPrototypeOf(obj, newProto)， 这个是新的方法去访问和设置对象的原型：
```
既然要优化，索性优化的好一些。
使用一个新的Reflect对象有如下好处：
#### 1. 可以把一些不仅仅是Object含有的工具函数，统一起来，比如Reflect.apply,这是针对Function，而不是针对Object的，这些函数放到Object上显然不合适，放在Reflect上，第一没有什么歧义，第二增加了使用的便利性；

#### 2. 把散落的工具函数集中到一个统一的对象上，可以保持其他对象的简洁性；

#### 3. 把类似delete等命令式操作，改为Reflect.delete的函数式操作，可以减少保留字的个数；

