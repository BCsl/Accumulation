# 基本语法(主要根据和JAVA的区别来记录)

- 弱类型，变量定义 `var x`，常量 `const` 和 `let` 都可以用于定义块级作用域变量，全局变量（包括函数）自动绑定到 `window` 对象，`ES5` 只有两种声明变量的方法：`var` 命令和 `function` 命令。`ES6` 除了添加 `let` 和 `const` 命令、`import` 命令和 `class` 命令。[变量和作用域](http://es6.ruanyifeng.com/#docs/let)

- 不区分整数和浮点数，其中 `NaN` 表示 Not a Number，`Infinity` 代表无限大

- 比较运算，`===` 比较不会自动转换数据类型，使用 `==` 会转换类型，导致问题

  ```javascript
  var a = 1;    
  var b = true;    
  console.log(a == b);//true
  ```

  浮点数比较，`1 / 3 === (1 - 2 / 3); // false`，只能计算它们之差的绝对值，看是否小于某个阈值，null、undefined、0、NaN 和空字符串 '' 视为 false

- null 和 undefined ，null 和 JAVA 的 null 差不多，`undefined` 表示值未定义，`undefined` 仅仅在判断函数参数是否传递的情况下有用，两者的区分也没意义，大多数的时候使用 null 即可

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

  js中一切都是对象，需要注意的地方

  - 使用`new Number()`、`new Boolean()`、`new String()`创建包装对象后类型为`object`，不再是`number`、`boolean`、`string`，所以尽量不要使用即可

  - 用`Number.parseInt()`或`Number.parseFloat()`来转换任意类型到`number`

  - 用`String()`来转换任意类型到`string`，或者直接调用某个对象的`toString()`方法

  - 判断`Array`要使用`Array.isArray(arr)`，`Array`是`object`类型

  - 判断`null`请使用`myVar === null`，`null`也是`object`类型

  - 判断某个全局变量是否存在/函数内部判断某个变量是否存在用`typeof window.myVar === 'undefined'`；

- 函数，定义`function name(...params)`，不需要制定返回值，允许传入任意个参数而不影响调用（但没意义），隐藏参数`arguments`，表示所有输入参数

- 高阶函数，函数其实都指向某个变量。既然 **变量可以指向函数**，函数的参数能接收变量，那么一个函数就可以接收另一个函数作为参数，这种函数就称之为高阶函数

  - Array.prototype.[map|reduce]，用于大数据集合的处理，`map` 单独处理 `Array` 的每个元素，`reduce` 的每个元素的处理都可能影响最后的结果

- **闭包**，高阶函数除了可以接受函数作为参数外，还可以把**函数作为结果值返回**，返回闭包时牢记的一点就是：**返回函数不要引用任何循环变量，或者后续会发生变化的变量**,[深入理解javascript原型和闭包](http://www.cnblogs.com/wangfupeng1988/p/3977924.html)

- `this` 关键字，和 JAVA 一样 **指向调用的对象**。但 JS 设计上的错误，JS 的**函数内部**如果调用了 this，this 的指向就要事情况而定，`strict` 模式下，ECMA 规定指向 `undefined`，关于如何控制 `this` 的指向，可以使用 `apply|call|bind` 这几个方法，[加深对 JavaScript This 的理解](https://juejin.im/post/596a28f6f265da6c360a2716?utm_source=gold_browser_extension)

- 箭头函数，ES6标准，箭头函数相当于匿名函数，并且简化了函数定义，实际上有点像JAVA的lambda表达式，且修复了this的指向问题

- generator，`function *` 定义，可以返回多次，关键字 yield

- 面向对象，类和实例是大多数面向对象编程语言的基本概念，JS没有类的概念，通过原型来实现面向对象编程，对象的 `__proto__` 属性就是某个对象的原型，`Object.create(proto[, propertiesObject])` 方法来创建指定原型类型的对象

  ## 参考

- [JS秘密花园](https://bonsaiden.github.io/JavaScript-Garden/zh/)

- [廖雪峰的JS教程](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)

- [JS代码风格](https://github.com/sivan/javascript-style-guide/blob/master/es5/README.md#types)

- [What the f*ck JavaScript?](https://github.com/denysdovhan/wtfjs#readme)
