# 对象的扩展

## 属性的简洁表示

`ES6` 允许直接写入变量和函数，作为对象的属性和方法。这样的书写更加简洁。

## 属性表达式

对象属性名可以这样写 `[表达式]`，如 [`ab`+`c`]，属性名为 `abc`，但感觉意义不大

## 方法的 name 属性

打印对象方法、函数名

## Object.is

比较两个值是否相等，与严格比较 `===` 基本一致，但是修改了 +0 等于 -0 和 NaN 不等于 NaN 的问题。

## Object.assign

- 用于对象的合并，将源对象（source，不包括继承属性）的所有**可枚举属性**，复制到目标对象（target）

  ```javascript
  var v1 = 'abc';
  console.log(Object(v1));    //String {0: "a", 1: "b", 2: "c", length: 3, [[PrimitiveValue]]: "abc"}
  var v2 = true;
  console.log(Object(v2));    //Boolean {[[PrimitiveValue]]: true}
  var v3 = 10;
  console.log(Object(v3));    //Number {[[PrimitiveValue]]: 10}
  var v4 = {[Symbol('c')]: 'b'};
  console.log(Object(v4));    //Object {Symbol(c): "b"}
  //v2,v3没有枚举属性
  var obj = Object.assign({}, v1, v2, v3, v4);   //Object {0: "a", 1: "b", 2: "c", Symbol(c): "b"}
  console.log(obj);
  ```

- `Object.assign` 方法实行的是**浅拷贝**，而不是深拷贝

- 遇到属性名相同的属性合并，后面的属性值会覆盖前面的

- 操作数组要注意，会把数组视为对象

  ```javascript
  Object.assign([1, 2, 3], [4, 5])
  // [4, 5, 3]
  ```

常见用途

1.为对象添加属性和方法

2.克隆对象

3.合并多个对象

4.为属性指定默认值

## 属性的可枚举性

`Object.getOwnPropertyDescriptor` 方法可以获取该属性的描述对象

下列操作会忽略 `enumerable` 为 `false` 的属性

- `for...in` 循环：只遍历对象自身的和继承的可枚举的属性 ES5
- `Object.keys()` 返回对象自身的所有可枚举的属性的键名 ES5
- `JSON.stringify()` 只串行化对象自身的可枚举的属性 ES5
- `Object.assign()` ES6

## `__proto__`属性，`Object.setPrototypeOf()`，`Object.getPrototypeOf()`

**每一个对象（函数也是对象、原型也是对象）都有 `__proto__` 属性，指向对应的构造函数的 `prototype` 属性**，但无论从语义的角度，还是从兼容性的角度，都不要使用这个属性，而是使用下面的 `Object.setPrototypeOf()`（写操作）、`Object.getPrototypeOf()`（读操作）、`Object.create()`（生成操作）代替

## Null 传导运算符

如果读取对象内部的某个属性，往往需要判断一下该对象是否存在，Null 传导运算符 `?.` 可以简化这种操作

```javascript
// 如果 a 是 null 或 undefined, 返回 undefined
// 否则返回 a.b.c().d
a?.b.c().d

// 如果 a 是 null 或 undefined，下面的语句不产生任何效果
// 否则执行 a.b = 42
a?.b = 42

// 如果 a 是 null 或 undefined，下面的语句不产生任何效果
delete a?.b
```

# 参数

[ECMAScript6 - 对象扩展](http://es6.ruanyifeng.com/#docs/object#Object-assign)
