# ECMAScript6 - Promise

Js 提供 `Promise` 对象，可以用来处理 **异步操作**，支持链式回调，**统一的异常处理**

## 特点

- 状态不受外界影响且修改后不可改变

  `Pending`（进行中）、`Resolved`（已完成，又称 `Fulfilled`）和`Rejected`（已失败），不论最终是 `Resolved` 还是 `Rejected`，状态将不能再修改，且会保持结果，再次添加回调函数也能得到之前结果

- 需要手动 `catch` 异常，否则将被忽略

- 无法取消

- 支持回调，不像 JAVA 中的 `Future` 构成的 `Promise` 模式，需要用户主动检查结果

## 使用

创建一个 `Promise` 对象，且创建即刻运行

```javascript
var promise = new Promise(function(resolve, reject) {
  // ... some code

  if (/* 异步操作成功 */){
    resolve(value); //Promise.resolve(value)
  } else {
    reject(error);  //Promise.reject(reason)
  }
});
```

`Promise` 对象创建接受一个函数为参数，这个函数不需要用户自己传递，由 `JS` 引擎部署，该函数的参数是两个参数，分别在成功的时候回调结果的 `resolve` 和出错的时候提供错误信息的 `reject`

## 回调

`Promise.prototype.then(onFulfilled, onRejected)` 和 `Promise.prototype.catch(onRejected)`，一般来说 `then` 的第二个参数不需要传入，都用 `catch` 来处理异常，两个方法都返回一个 `Promise` 对象，所以可以链式调用

## 方法

### Promise.all(iterable)

合并多个 `Promise` 对象

- 如果所有的 `Promise` 都是 `fulfills`，才会收到 `fulfills` 状态，参数是一个数组，长度和 `Promise` 个数一样
- 只要有一个 `Promise` 状态是 `rejects`，那么就会 `rejects`，收到的第一个被 `reject` 的实例的返回值

> 注意，作为参数的 `Promise` 实例，自己定义了 `catch` 方法，那么它一旦被 `rejected`，并不会触发 `Promise.all()` 的 `catch` 方法，该实例执行完 `catch` 方法后，也会变成 `resolved`

### Promise.race(iterable)

如其名，第一个状态改变的 `Promise` 将作为结果状态

### done

`Promise` 对象的回调链，不管以 `then` 方法或 `catch` 方法结尾，要是最后一个方法抛出错误，都有可能无法捕捉到，`done`方法，总是处于回调链的尾端，保证抛出任何可能出现的错误

其实现代码如下：

```javascript
Promise.prototype.done = function (onFulfilled, onRejected) {
  this.then(onFulfilled, onRejected)
    .catch(function (reason) {
      // 抛出一个全局错误
      setTimeout(() => { throw reason }, 0); //向全局抛出
    });
};
```

### finally

`finally` 方法用于指定不管 `Promise` 对象最后状态如何，都会执行的操作。它与 `done` 方法的最大区别，它接受一个普通的回调函数作为参数，该函数不管怎样都必须执行

实现如下：

```javascript
Promise.prototype.finally = function (callback) {
  let P = this.constructor; //指向创建该 promise 的函数
  return this.then(
    value  => P.resolve(callback()).then(() => value),
    reason => P.resolve(callback()).then(() => { throw reason })
  );
};
```

### Promise.try

不管函数 `f` 是同步函数还是异步操作，但是想用 `Promise` 来处理它，想用 `then` 方法管理流程，一般情况下可以这样

```javascript
const f = () => console.log('now');
(
  () => new Promise(
    resolve => resolve(f())
  )
)();  //创建一个匿名函数并执行 new Promise()，这种情况下，同步函数也是同步执行的。
console.log('next');
// now
// next
```

`Promise.try` 就是替代上面的写法，而且有一个优点便是，还能捕获到同步块 `f` 的异常

# 更多

- [ECMAScript6 - Promise对象](http://es6.ruanyifeng.com/#docs/promise)
