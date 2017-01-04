# 基本语法(主要根据和JAVA的区别来记录)

- 弱类型，变量定义`var x`，常量`const`和`let`都可以用于定义块级作用域变量，全局变量（包括函数）自动绑定到`window`对象

- 不区分整数和浮点数，其中`NaN`表示Not a Number，`Infinity`代表无限大

- 比较运算，`===`比较不会自动转换数据类型，浮点数比较，`1 / 3 === (1 - 2 / 3); // false`，只能计算它们之差的绝对值，看是否小于某个阈值，null、undefined、0、NaN和空字符串''视为false

- null和undefined，null和JAVA的null差不多，`undefined`表示值未定义，`undefined`仅仅在判断函数参数是否传递的情况下有用，两者的区分也没意义，大多数的时候使用null即可，JS中所有变量都可以当做对象使用，除了这两个例外

- 数组，因为是弱类型语言，所以数组的元素可以是任何类型数据，`[x,y,z]`或者`new Array(x,y,z)`来定义

- 对象，键-值组成的无序集(所以可以当成hash表来使用)，可以自由地给一个对象添加或删除属性（delete）,**键值** 必须是字符串，方法也是变量，所以可以作为值

  ```javascript
  var person = {
    name: 'Bob',
    age: 20,
    tags: ['js', 'web', 'mobile'],
    city: 'Beijing',
    hasCar: true,
    'middle-school': 'No.1 Middle School' //key不是一个有效的变量，就需要用''括起来，且需用['xxx']来访问
  };
  ```

- 函数，定义`function name(...params)`，不需要制定返回值，允许传入任意个参数而不影响调用（但没意义），隐藏参数`arguments`，表示所有输入参数

- 高阶函数，函数其实都指向某个变量。既然**变量可以指向函数**，函数的参数能接收变量，那么一个函数就可以接收另一个函数作为参数，这种函数就称之为高阶函数

  - Array.prototype.[map|reduce]，用于大数据集合的处理，map单独处理`Array`的每个元素，reduce的每个元素的处理都可能影响最后的结果

- **闭包**，高阶函数除了可以接受函数作为参数外，还可以把**函数作为结果值返回**，返回闭包时牢记的一点就是：**返回函数不要引用任何循环变量，或者后续会发生变化的变量**,[深入理解javascript原型和闭包](http://www.cnblogs.com/wangfupeng1988/p/3977924.html)

- `this`关键字，和JAVA一样指向调用的对象。但JS设计上的错误，JS的函数内部如果调用了this，this的指向就要事情况而定，strict模式下，ECMA规定指向`undefined`，关于如何控制`this`的指向，可以使用`apply`

- 箭头函数，ES6标准，箭头函数相当于匿名函数，并且简化了函数定义，实际上有点像lambda表达式，且修复了this的指向问题

## 参考

- [JS秘密花园](https://bonsaiden.github.io/JavaScript-Garden/zh/)

- [廖雪峰的JS教程](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)
