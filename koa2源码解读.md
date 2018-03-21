## **0，前言**

阅读本文前，需要对koa2（以下简称Koa）和es6有一定了解。
本文会以如下的顺序：

- 1，koa是什么；
- 2，初读koa源码；
- 3，精读koa源码；
    - 3.1，中间件机制解读
	- 3.2，如何将generator函数转成类async函数
	- 3.3，统一的错误处理机制
	- 3.4，context如何实现对request和response的代理
	- 3.5，等其他我还没想到的知识点

循序渐进，结合源码解读Koa框架。
阅读完本文之后，除了可以对Koa有一个全方面的了解，还可以对js的Promise，Generator，Async语法有深入理解。


## **1，koa是什么**

Koa是一个精简的web容器，它主要做了以下几件事：

- 为request和response对象赋能，并基于它们封装成一个context对象**（占了最多的代码篇幅，但很容易理解）**
- 基于async/await的中间件容器机制**（最重要，最有价值的，代码很精简但很难理解）**



## **2，初读koa源码**

koa的源码结构非常简单，共4个文件。
```
── lib
   ├── application.js
   ├── context.js
   ├── request.js
   └── response.js
```

以下是四个文件带注释的精简版，读懂它们可以对koa有一个大致的了解：

### **application.js**
```
//作者注：引入第三方库，实际不仅是下面几个，列出来的几个是比较关键的
const response = require('./response');
const compose = require('koa-compose');//作者注：实现基于async/await的洋葱式调用顺序的中间件容器的关键库，下文会重点介绍
const context = require('./context');
const request = require('./request');
const Emitter = require('events');//作者注：node的基础库，koa应用集成于它，主要用了其事件机制，来实现异常的处理
const http = require('http');//作者注：node实现web服务器功能的核心库
const convert = require('koa-convert');//作者注：为了支持koa1的generator中间件写法，对于使用generator函数实现的中间件函数，需要通过koa-convert转换


module.exports = class Application extends Emitter {

    constructor() {
        super();

        this.middleware = [];//作者注：该数组存放所有通过use函数引入的中间件函数

        //作者注：创建context、request、response对象
        this.context = Object.create(context);
        this.request = Object.create(request);
        this.response = Object.create(response);
    }


    //作者注：创建服务器实例
    listen(...args) {
        debug('listen');
        //作者注：通过执行callback函数返回的函数来作为处理每次请求的回调函数
        const server = http.createServer(this.callback());
        return server.listen(...args);
    }

    /*
      作者注：通过调用koa应用实例的use函数，形如：
      app.use(async (ctx, next) => {
          await next();
      });
      来加入中间件
    */
    use(fn) {
        //作者注：如果是generator函数，则需要通过koa-convert转换成类似类async/await函数
        //其核心原理是将 Generator 函数和自动执行器，包装在一个函数里。后文会重点解释
        if (isGeneratorFunction(fn)) {
            fn = convert(fn);
        }
        //作者注：将中间件加入middleware数组中
        this.middleware.push(fn);
        return this;
    }

    //作者注：返回一个形如(req, res) => {}的函数，该函数会作为参数传递给上文listen函数中的http.createServer函数，作为处理请求的回调函数
    //具体细节会在下文重点解释
    callback() {

        //作者注：将所有中间件函数通过koa-compose组合一下
        const fn = compose(this.middleware);

        //作者注：该函数会作为参数传递给上文listen函数中的http.createServer函数，
        const handleRequest = (req, res) => {

            //作者注:基于req和res，封装一个更强大的context对象
            const ctx = this.createContext(req, res);

            //作者注：当有请求过来时，需要基于办好了request和response信息的ctx和所有中间件函数，来处理请求。
            return this.handleRequest(ctx, fn);
        };

        return handleRequest;
    }

    handleRequest(ctx, fnMiddleware) {
        //作者注：略
    }

    //作者注:基于req和res对象，创建context对象
    createContext(req, res) {
        //作者注：略
    }


};
```

### **context.js**

```
const util = require('util');
const createError = require('http-errors');
const httpAssert = require('http-assert');
const delegate = require('delegates');
const statuses = require('statuses');



const proto = module.exports = {

    //作者注：
    //一些不甚重要的函数
};


/*
    作者注：
    在application.js的createContext函数中，
    被创建的context对象上会挂载基于response.js实现的response对象和基于request.js实现的request对象，
    下面两个delegate函数的作用是让context对象代理response和request的部分方法和属性
*/
delegate(proto, 'response')
    .method('attachment')
    ...
    .getter('writable');

/**
 * Request delegation.
 */

var a = delegate(proto, 'request')
    .method('acceptsLanguages')
    ...
    .getter('ip');
```


### **request.js**
```
module.exports = {

    //作者注，在application.js中的createContext函数中，会把node服务器的req对象作为request对象的属性，
    //request对象会基于req封装很多便利的函数和属性
    get header() {
        return this.req.headers;
    },

    set header(val) {
        this.req.headers = val;
    },

    //作者注：省略了大量类似的工具属性和方法
}

```

### **response.js**
与request.js类似，主要是基于node服务器的res对象,封装一系列便利的函数和属性


## **3，精读koa源码**
上面毕竟是走马观花，本着追根究底的学术精神，还需要对大量细节仔细揣度，下文就会对我认为很重要的细节进行解读，如果彻底读通的话，会对Promise，Generator，Async语法有更深入的理解。

### **3.1，精简但重要的中间件机制**

koa中的中间件本质上就是一个async函数，形如：
```
async (ctx, next) => {
  await next();
}
```
该函数接受两个参数，ctx和next，ctx即为application的context属性，其封装了req和res；next函数用于将程序控制权交个下一个中间件。
通过koa应用实例的use函数，可以将中间件加入到koa实例的middleware数组中。
当node服务启动的时候，会通过koa-compose的compose函数，将middleware数组组织成一个fn对象。
当有请求访问时，会调用callback函数内部的handleRequest函数，该函数主要做两件事:
- 根据req和res创建context对象；
- 执行koa实例的handleRequest函数（注意区分两个handleRequest函数）；
koa实例的handleRequest函数通过其最后一行代码，开启了中间件函数的洋葱式调用（具体什么是koa的洋葱式调用，请参见：[链接](https://eggjs.org/zh-cn/intro/egg-and-koa.html)）。
以上描述的对应下面三部分代码如下：
```
const server = http.createServer(this.callback());
```


```
  callback() {
    const fn = compose(this.middleware);

    if (!this.listeners('error').length) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
```


```
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

这里有两个细节很关键，第一是koa-compose对middleware做了什么；第二是如果实现洋葱式调用的？
答：
以下是koa-compose的精简版源码
```
module.exports = compose

function compose(middleware) {
    return function (context, next) {
        //略
    }
}
```
compose函数接收middleware数组作为参数，middleware中每个对象都是一个async函数；
返回一个以context和next作为入参的函数，我们姑且和源码一样，称其为fnMiddleware；

接下来运行中间件：
```
 return fnMiddleware(ctx).then(handleResponse).catch(onerror);
```
这里就需要关心fnMiddleware的实现了：
```
return function (context, next) {
        let index = -1
        return dispatch(0)
        function dispatch(i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))

            index = i
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, function next() {
                    return dispatch(i + 1)
                }))
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
```
解释前我先做一个假设：假设加入了两个中间件，代码如下：
```
app.use(async (ctx,next) => {
   console.log("1-start");
   await next();
   console.log("1-end");
});

app.use(async (ctx, next) => {
  console.log("2-start");
  await next();
  console.log("2-end");
});
```
然后我们逐步执行：
fnMiddleware(ctx)运行；
执行dispatch(0)；
进入dispatch函数，执行到

```
return Promise.resolve(fn(context, function next() {
            return dispatch(i + 1)
        }))
```

此时fn就是第一个中间件，**它是一个async函数，async函数会返回一个Promise对象，Promise.resolve（）中若传入一个Promise对象的话，那么Promise.resolve将不做任何修改、原封不动地返回这个Promise对象**。
    进入到第一个中间件代码内部：
    先执行‘console.log("1-start");’；
    然后执行'await next();'并开始等待next执行返回；
        进入到next函数后，主要是执行dispatch（1），于是老的dispatch（0）函数压栈，开始从头开始执行dispatch（1），即把第二个中间件函数付给fn，然后开始执行，**这步完成了程序控制权从第一个中间件到第二个中间件的转移**。
        进入到第二个中间件代码内部：
        先执行‘console.log("2-start");’；
        然后执行'await next();'并开始等待next执行返回；
            进入到next函数后，主要是执行dispatch（2），于是老的dispatch（1）函数压栈，开始从头开始执行dispatch（2），由于此时程序符合下面的条件：
            
```
if (i === middleware.length) fn = next
if (!fn) return Promise.resolve()
```
所以返回Promise.resolve()，此时第二个中间件的next函数返回了。
        所以接下来执行console.log("2-end");由此第二个中间件执行完成，把程序控制权传递给第一个中间件。
    第一个中间件执行console.log("1-end");。
    终于完成所有中间件的执行，若中间没有异常，则返回Promise.resolve()，执行handleResponse回调；若有异常，则返回Promise.reject(err),执行onerror回调。
    
    
### **3.2，如何将generator函数转成类async函数**
出于对于上个版本的兼容，如果中间件函数是generator函数的话，会使用koa-convert将其转为‘类async’函数。（不过到第三版本，该功能会取消）。
如何转换，这个话题还是很吸引人的。让我们来想想generator函数和async有啥区别？唯一的区别是async函数会自动执行，而generator每次需要调用next函数。仅此而已。
所以问题转变为：如果让generator函数自执行？
如果不考虑异步的情况的话，非常简单，用个while循环就行，但是如果有异步的话，就比较复杂一些。
这里就涉及到了著名的co插件。其功能是:传入generator函数，使其能自动执行。
每次执行generator的next函数，它会返回一个对象 { value: xxx, done: false }，返回对象后，如果能再次执行generator的next函数就可以达到自动执行generator的目的了。
请看下面的代码：
```
function * gen(){
    yield new Promise((resolve,reject){
        //异步函数1
        if（成功）{
            resolve（）
        }else{
            reject();
        }
    });
    
    yield new Promise((resolve,reject){
        //异步函数2
        if（成功）{
            resolve（）
        }else{
            reject();
        }
    })
}
let g = gen();
let ret = g.next();
```
此时ret = {value：promise实例，done：false}；
此时拿到了promise对象，那就可以自己定义成功/失败的回调函数了。即：
```
    ret.value.then(()=>{
        g.next();
    })
```
上面的代码是不是在第一次异步函数执行完成之后，自动执行了generator函数？
好好读读上面这几行代码，是不是觉得有点眉目了？
问题是如何把代码写得更通用，然后让自动执行一路执行下去呢？
上精简版co源码：
```
function co(gen) {

  return new Promise(function(resolve, reject) {
    //1，不管三七二十一先执行onFulfilled函数
    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        //2，第一次执行gen函数，返回一个{ value: xxx, done: false }
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      //3，然后执行next函数，并把ret作为入参
      next(ret);
    }

    function onRejected(err) {
    }

    function next(ret) {
      if (ret.done) return resolve(ret.value);
      //4，按照之前的说法，如果返回的是个Promise实例，就可以根据Promise实例的then回调来继续执行generator的next方法了。所以先要把ret.value转换为一个Promise实例
      var value = toPromise.call(ctx, ret.value);
      //5，让成功的回调指向onFulfilled函数，其实就是又从第1步开始执行了
      //这样就实现了自动执行generator
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
    }
  });
}
```
对！好好品味一下，就是这么简单。

除了使用Promise之外，还可以使用Trunk来实现。具体可以参见尊敬的阮一峰老师的[文章](http://es6.ruanyifeng.com/#docs/generator-async)
我这里就不赘述了，思路和Promise很像，中心思想都是：
    1，执行函数cb（我瞎取的名），调用next之后返回一个句柄ret，
    2，可以根据这个句柄ret为其定制回调函数，将回调函数设为cb。
就这样循环往复。


### **3.3，统一的错误处理机制**
[TODO]

### **3.4，context如何实现对request和response的代理**
[TODO]

### **3.5，等其他我还没想到的知识点**
[TODO]


