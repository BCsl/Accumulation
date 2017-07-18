# Generator 函数

**异步编程解决方案，也是实现状态机的最佳结构**

## 语法

- 语法

1.`function*` 来 **定义方法** ； 2.函数体内使用 `yield` 表达式，定义不同的状态 ；3.`yield` 关键字只能用在 `Generator` 函数里面

```javascript
function* helloWorldGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}
var hw = helloWorldGenerator();
```

定义了三个状态 `hello``，world` 和 `return` 语句（结束执行）

- 执行，`Generator.prototype.next([Object])`

和普通函数一样调用，但不会马上执行，返回的是一个 `Iterator` 对象，指向内部状态，通过 `next` 返回一个有着 `value` 和 `done` 两个属性的对象并使得指针移向下一个状态，`next` 方法可以接受一个参数，代表上一个 `yield` 表达式的返回值

```javascript
hw.next()
// { value: 'hello', done: false }
hw.next()
// { value: 'world', done: false }
hw.next()
// { value: 'ending', done: true }
hw.next()
// { value: undefined, done: true }
```

### `Generator.prototype.throw()`

可以在函数体外抛出错误，然后在 `Generator` 函数体内捕获

### `Generator.prototype.return()`

返回特定的值并结束 `Generator` 函数，以后再调用 `next` 方法就返回 `{ value: undefined, done: true }`

### `yield*` 表达式

`yield* [[expression]];`，用来在一个 `Generator` 函数里面执行另一个 `Generator` 函数，实际上任何数据结构只要有 `Iterator` 接口，就可以被 `yield*` 遍历

## 应用

特定：`Generator` 可以暂停函数执行，返回任意表达式的值

- 异步操作的同步化表达

```javascript
  function* loadUI() {
  showLoadingScreen();
  yield loadUIDataAsynchronously(); //异步操作写在yield表达式
  hideLoadingScreen();  //后续操作可以放在yield表达式下面
}
var loader = loadUI();
// 加载UI
loader.next()
// 卸载UI
loader.next()
```

- 控制流管理
- 部署 `Iterator` 接口
- 作为数据结构

## 异步应用

### Generator 函数

`Javascript` 语言的执行环境是"单线程"的，`Generator` 函数是协程在 ES6 的实现，最大特点就是可以交出函数的执行权（即暂停执行）。

ES6 以前的异步解决方案：

- 回调函数

  嵌套地狱

- 事件监听

- 发布/订阅

- `Promise` 对象

  允许将回调函数的嵌套，改成链式调用

#### 协程和 Generator 函数实现

多个线程互相协作，完成异步任务，有点像函数，又有点像线程

```javascript
function *asyncJob() {
  // ...其他代码
  var f = yield readFile(fileA);  //yield 命令是异步两个阶段的分界线
  // ...其他代码
}
```

协程遇到 `yield` 命令就暂停，等到执行权返回，再从暂停的地方继续往后执行。

整个 `Generator` 函数就是一个封装的异步任务，或者说是异步任务的容器。异步操作需要暂停的地方，都用 `yield` 语句注明

### 异步任务的封装

封装

```javascript
var fetch = require('node-fetch');

function* gen(){
  var url = 'https://api.github.com/users/github';
  var result = yield fetch(url);  //返回一个 Promise 对象
  console.log(result.bio);
}
```

执行

```javascript

var g = gen();
var result = g.next();  //返回一个 Promise 对象，异步任务的第一阶段

//数据处理
result.value.then(function(data){
  return data.json();
}).then(function(data){
  g.next(data); //异步任务的第二阶段
});
```

### Thunk 函数

Thunk 函数是自动执行 Generator 函数的一种方法。编译器的方法"传名调用"实现

伪代码如下：

```javascript

function f(m) {
  return m * 2;
}

f(x + 5);

// 等同于
//将参数放到一个临时函数之中，再将这个临时函数传入函数体，这个临时函数就叫做 Thunk 函数
var thunk = function () {
  return x + 5;
};

function f(thunk) {
  return thunk() * 2;
}
```

JS 中，Thunk 函数替换的不是表达式，而是 **多参数函数**，将其替换成一个 **只接受回调函数作为参数的单参数函数**，其实就是执行参数和回调分成两个函数。只要参数有回调函数，就能写成 Thunk 函数的形式。

```
fn(args, callback)--->变成--->thunk(fn)(args)(callback)
```

### Thunkify 模块

`$ npm install thunkify`

源码

```javascript

function thunkify(fn) {
  return function() {
    var args = new Array(arguments.length); //函数参数，一般接受非回调函数外的参数
    var ctx = this;

    for (var i = 0; i < args.length; ++i) {
      args[i] = arguments[i]; //构造参数数组，可以用 Array.prototype.slice.call(arguments) 方法，但在 V8 引擎上性能没那么好
    }

    return function (done) {  //done 是回调函数
      var called; //保证只能执行一次回调

      args.push(function () { //封装一个内部的代理回调函数
        if (called) {
          return;
        }
        called = true;
        done.apply(null, arguments);  //这里的 arguments 是回调函数回调时候的 arguments，不是上面的 arguments
      });

      try {
        fn.apply(ctx, args);
      } catch (err) {
        done(err);
      }
    }
  }
};
```

源码中，把参数 `callback` 和其他参数分离出来，并使用代理的回调函数来代理真实回调函数

使用 thunkify

```javascript

function f(a, b, callback){
  var sum = a + b;
  callback(sum);
  callback(sum);  //只回调一次，所以这里会忽略
}

var ft = thunkify(f);
var print = console.log.bind(console);
ft(1, 2)(print);
// 3
```

### Generator 函数的流程管理

传统情况下按顺序执行两个异步操作，如果任务越来越多，就会陷入嵌套回调，而且多任务耦合

```javascript
var fs = require('fs');
var thunkify = require('thunkify');
var readFileThunk = thunkify(fs.readFile);

readFileThunk('/etc/fstab')(function(err, data){
  if(err) throw err;
    console.log(r1.toString());
  readFileThunk('/etc/shells')(function(err, data)){
      if(err) throw err;
      console.log(r2.toString());
  })
});
```

下面使用 Generator 函数封装了两个异步操作

```javascript
var fs = require('fs');
var thunkify = require('thunkify');
var readFileThunk = thunkify(fs.readFile);  //thunk 函数

var gen = function* (){
  var r1 = yield readFileThunk('/etc/fstab'); //yield命令用于将程序的执行权移出 Generator 函数
  console.log(r1.toString());
  var r2 = yield readFileThunk('/etc/shells');
  console.log(r2.toString());
};
```

未优化的执行方式如下

```javascript

var g = gen();

var r1 = g.next();  //返回一个 value 为 thunk 函数，接受一个 callback 函数
r1.value(function (err, data) {
  if (err) throw err;
  var r2 = g.next(data);  //再次
  r2.value(function (err, data) {
    if (err) throw err;
    g.next(data);
  });
});
```

如上，在第一个 yield 命名将程序执行权移出 Generator 函数后，待第一个异步任务成功回调再执行第二个 yield 命名

进一步优化后得到通用的 Thunk 函数对 Generator 函数流程管理的方式

```javascript
function run(fn) {
  var gen = fn();

  function next(err, data) {
    var result = gen.next(data);  //返回 value 为 thunk 函数
    if (result.done) return;
    result.value(next); //next 函数传递到 thunk 函数，响应回调
  }
  next();
}

function* g() {
  // ...
}

run(g);
```

可以看到，Thunk 是基于回调函数来控制 Generator 函数的执行流程，同样地，使用 Promise 对象也可以轻松完成。

使用 Promise 的时候可以写成这样

```javascript

function run(gen){
  var g = gen();

  function next(data){
    var result = g.next(data);  // 返回 Promise 对象
    if (result.done) return result.value;
    result.value.then(function(data){
      next(data);
    });
  }

  next();
}
```

### co 模块

作用：用于 Generator 函数的自动执行。

co 模块其实就是将两种自动执行器（Thunk 函数和 Promise 对象），包装成一个模块。

语法：Generator 函数的 yield 命令后面，只能是 Thunk 函数或 Promise 对象（4.0 后不支持 Thunk 函数）。

```javascript
function* g() {
  // ...
}
var gen =g();
var co = require('co');
//co 函数返回一个 Promise 对象，因此可以用 then 方法添加回调函数
co(gen).then(function (){
  console.log('Generator 函数执行完成');
});
```

源码，用 Promise 对象包装 Generator 函数，并自动执行

```javascript

function co(gen) {
  var ctx = this;

  return new Promise(function(resolve, reject) {
    //1.检查是否是 Generator 函数
    if (typeof gen === 'function') {
      gen = gen.call(ctx);  // 得到 generator 函数
    }
    if (!gen || typeof gen.next !== 'function') {
      return resolve(gen);  // 不是 generator 函数，返回 promise 对象
    }
    //包装成onFulfilled函数，要是为了能够捕捉抛出的错误
    onFulfilled();
    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);  //执行到 yield 并把执行权移除 Generator 函数，返回 Promise 对象
      } catch (e) {
        return reject(e); //移除捕捉
      }
      next(ret);
    }
  });
}

// 自动执行的关键
/**
* 参数 ret 上次执行的结果
*/
function next(ret) {
  if (ret.done) {
    return resolve(ret.value);
  }
  var value = toPromise.call(ctx, ret.value);
  if (value && isPromise(value)){ //确保每一步的返回值，是 Promise 对象。
     return value.then(onFulfilled, onRejected);  
   }
  return onRejected(
    new TypeError(
      'You may only yield a function, promise, generator, array, or object, '
      + 'but the following object was passed: "'
      + String(ret.value)
      + '"'
    )
  );
}
```

上面说的都是把多个异步任务控制成同步任务顺序执行，如果需要处理并发异步操作，把并发的操作都放在数组或对象里面，跟在 `yield` 语句后面即可

# 更多

- [Generator 函数的语法](http://es6.ruanyifeng.com/#docs/generator) )
