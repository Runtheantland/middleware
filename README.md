# js中间件
js中间件

当我们在编写业务代码时候，我们无法避免有些业务逻辑复杂而导致业务代码写得又长又乱，如果再加上时间紧凑情况下写出来的代码估计会更让人抓狂。以至于我们一直在寻求更好的架构设计和更好的代码设计，这是一个没有终点的求知之路，但是在这条路上会越走越好。

在前端，我们可以借用这种思想通过before和after函数来实现：

Function.prototype.before = function(fn){//函数处理前执行fn
  var self = this;
   return function(){
     fn.call(this);
     self.apply(this, arguments);
   }
}
Function.prototype.after = function(fn){//函数处理后执行fn
  var self = this;
   return function(){
     self.apply(this, arguments);
     fn.call(this);
   }
}
实现思路是对被处理的函数通过闭包封装在新的函数里，在新的函数内部按照顺序执行传入的参数fn和被处理的函数。

1 举个例：

用户提交表单数据之前需要用户行为统计，代码应该是这样写：

function report(){
   console.log('上报数据');
}
function submit(){
   console.log('提交数据');
}

submit.before(report)(); //提交之前执行report
//结果： 上报数据
//      提交数据
从代码可以看出已经把统计和数据提交业务隔离起来，互不影响。

但是如果提交数据之前，需要数据验证并且依据验证结果判断是否能提交，怎么做？这里要改动before函数，看下代码：

Function.prototype.before = function(fn){//函数处理后执行fn
  var self = this;
   return function(){
     var res = fn.call(this);
     if(res)//返回成功则执行函数
       self.apply(this, arguments);
   }
}

function report(){
	console.log('上报数据');
   return true;
}
function validate(){
   console.log('验证不通过');
   return false;
}
function submit(){
   console.log('提交数据');
}

submit.before(report).before(validate)();
//结果： 
// 验证不通过 

function report(){
   console.log('上报数据');
   return true;
}
function validate(){
   console.log('验证通过');
   return true;
}
function submit(){
   console.log('提交数据');
}

submit.before(report).before(validate)();
//结果： 
// 验证通过
// 上报数据
// 提交数据
上面的例子如果很复杂会出现很长的链式，后期维护也很容易看晕，并且before和after也没有考虑到异步操作，显然还是有些不足的，那么还有没有其他解决办法呢，既能隔离业务，又能方便清爽地使用～我们可以先看看其他框架的中间件解决方案。

2 express

express是非常轻量的框架，express是集合路由和其他几个中间件合成的web开发框架，koa是express原班人马重新打造一个更轻量的框架，所以koa已经被剥离所有中间件，甚至连router中间件也被抽离出来，任由用户自行添加第三方中间件。解析express的写法

express的中间件写法如下：

var express = require('express');
var app = express();
 
app.use(function(req, res, next) {
  console.log('数据统计');
  next();//执行权利传递给
});

app.use(function(req, res, next) {
  console.log('日志统计');
  next();
});

app.get('/', function(req, res, next) {
  res.send('Hello World!');
});

app.listen(3000);
//整个请求处理过程就是先数据统计、日志统计，最后返回一个Hello World！
过程图 从上图来看，每一个“管道”都是一个中间件，每个中间件通过next方法传递执行权给下一个中间件，express就是一个收集并调用各种中间件的容器。

中间件就是一个函数，通过express的use方法接收中间件，每个中间件有express传入的req，res和next参数。如果要把请求传递给下一个中间件必须使用 next() 方法。当调用res.send方法则此次请求结束，node直接返回请求给客户，但是若在res.send方法之后调用next方法，整个中间件链式调用还会往下执行，因为当前hello world所处的函数也是一块中间件，而res.send只是一个方法用于返回请求。

3 参照express我们可以仿写

我们可以借用中间件思想来分解我们的前端业务逻辑，通过next方法层层传递给下一个业务。做到这几点首先必须有个管理中间件的对象，我们先创建一个名为Middleware 的对象：

function Middleware(){
   this.cache = [];
}
Middleware通过数组缓存中间件。下面是next和use 方法：

Middleware.prototype.use = function(fn){
  if(typeof fn !== 'function'){
    throw 'middleware must be a function';
  }
  this.cache.push(fn);
  return this;
}

Middleware.prototype.next = function(fn){
  if(this.middlewares && this.middlewares.length > 0 ){
    var ware = this.middlewares.shift();
    ware.call(this, this.next.bind(this));
  }
}
Middleware.prototype.handleRequest = function(){//执行请求
  this.middlewares = this.cache.map(function(fn){//复制
    return fn;
  });
  this.next();
}
我们用Middleware简单使用一下：

  var middleware = new Middleware();
middleware.use(function(next){console.log(1);next();})
middleware.use(function(next){console.log(2);next();})
middleware.use(function(next){console.log(3);})
middleware.use(function(next){console.log(4);next();})
middleware.handleRequest();
//输出结果： 
//1
//2
//3
//

4没有出来是因为上一层中间件没有调用next方法，我们升级一下Middleware 高级使用


var middleware = new Middleware();
middleware.use(function(next){
  console.log(1);next();console.log('1结束');
});
middleware.use(function(next){
   console.log(2);next();console.log('2结束');
});
middleware.use(function(next){
   console.log(3);console.log('3结束');
});
middleware.use(function(next){
   console.log(4);next();console.log('4结束');
});
middleware.handleRequest();
//输出结果： 
//1
//2
//3
//3结束
//2结束
//1 结束
每一个中间件执行权利传递给下一个中间件并等待其结束以后又回到当前并做别的事情，方法非常巧妙。
