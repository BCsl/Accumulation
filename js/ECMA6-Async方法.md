# async 函数

ES2017 标准引入了 async 函数，是 Generator 函数的语法糖

async 函数就是将 Generator 函数的星号 `*` 替换成 async，将 yield 替换成 await，仅此而已，执行后返回 Promise 对象

使用 Generator 函数执行两个异步操作

```javascript

var fs = require('fs');
// 返回 promise 对象
var readFile = function (fileName) {
  return new Promise(function (resolve, reject) {
    fs.readFile(fileName, function(error, data) {
      if (error) reject(error);
      resolve(data);
    });
  });
};

var gen = function* () {
  var f1 = yield readFile('/etc/fstab');  //返回 promise 对象
  var f2 = yield readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};
```

写成 async 函数

```javascript
var asyncReadFile = async function () {
  var f1 = await readFile('/etc/fstab');  //await 方法返回 promise 对象的执行结果，或者表达式本身，如果非 promise 对象
  var f2 = await readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};
```

上面是顺序执行的，还改成并发执行

方式1

```javascript
var asyncReadFile = async function () {
  var f1 = readFile('/etc/fstab');
  var f2 = readFile('/etc/shells');
  var result = await Promise.all([f1, f2]); //Array 结果
  console.log(result.toString());
};
```

方式2

```javascript

async function asyncRead(name){
  var f = await readFile(name);
  console.log(f.toString());
}

async function asyncTask() {
  let f1Promise = asyncRead('/etc/fstab');  //返回 promise
  let f2Promise = asyncRead('/etc/shells');
  let f1 = await f1Promise;
  let f2 = await f2Promise;
}
```

## 基本用法

async 函数执行返回一个 Promise 对象，可以使用 then 方法添加回调函数，await 后接 Promise 对象，一旦遇到 await 就会先返回，等到异步操作完成，再接着执行函数体内后面的语句

```javascript
function asyncTask(ms) {
    return new Promise(resolve => {
        "use strict";
        console.log('cost time task run');
        setTimeout(()=> {
            console.log('cost time task complete');
            resolve(ms)
        }, ms);
    });
}
async function printTask(str, ms) {
    await asyncTask(ms);
    console.log(str);
    return 'PrintTask finish';  //return 返回值将作为 async 函数的 Promise 对象的 then方 法回调函数的参数
}

printTask('hello world', 50).then(msg=> {
    "use strict";
    console.log(msg);
});

//cost time task run
//cost time task complete
//hello world
//PrintTask finish
`
`
```

## 注意点

- 1、await 命令后面的 Promise 对象，运行结果可能是 rejected，将会结束整个 async 函数，可能出现异常的情况下最好把 await 命令放在 try...catch 代码块中，或者使用 Promise 对象的 catch 方法，使 Promise 对象状态为 resolved

- 2、多个 await 命令后面的异步操作，如果不存在继发关系，最好让它们同时触发，使用 `Promise#all` 方法

- 3、await 命令只能用在 async 函数之中，如果用在普通函数，就会报错

## 实现原理

将 Generator 函数和自动执行器，包装在一个函数里

```javascript

async function fn(args) {
  // ...
}

// 等同于

function fn(args) {
  //转换成 Generator 函数
  return spawn(function* () {
    // ...
  });
}
```

`spawn` 函数就是自动执行器

```javascript

function spawn(genF) {
  return new Promise(function(resolve, reject) {
    var gen = genF();
    function step(nextF) {
      try {
        var next = nextF();
      } catch(e) {
        return reject(e);
      }
      if(next.done) {
        return resolve(next.value);
      }
      Promise.resolve(next.value).then(function(v) {
        step(function() { return gen.next(v); });
      }, function(e) {
        step(function() { return gen.throw(e); });
      });
    }
    step(function() { return gen.next(undefined); });
  });
}
```

## 异步遍历器

一般情况下遍历器的 `next` 都是同步的，只要调用就可以得到返回值。所以对于 Generator 函数里面的异步操作，yield 返回的是 Thunk 函数或 Promise 对象

新提案是为异步操作提供原生的遍历器接口

### for await...of

`for...of`循环用于遍历同步的 `Iterator` 接口。新引入的 `for await...of` 循环，则是用于遍历异步的 `Iterator` 接口

# 更多

- [ECMAScript6 - async 函数](http://es6.ruanyifeng.com/#docs/async)
