# 面向对象编程

JS不包含传统的类继承模型，而是使用`prototype`原型模型。

## 原型继承

```javascript

function Animal(name, age) {
    this.name = name;
    this.age = age;
    // 在每个通过new Animal方式创建的对象定义一个tellSelf函数,将会类似java中的overwrite一样
    //  this.tellSelf = function () {
    //     console.log("My name is :" + this.name + ",i am " + this.age);
    // }
}
//在Animal.prototype指向的某个对象(也就是xiaoming的原型)定义一个tellSelf函数
Animal.prototype.tellSelf = function () {
    console.log("Animal name is :" + this.name + ",i am " + this.age);
}

var xiaoming = new Animal("xiaoming", 12);
xiaoming.tellSelf();
```

## 类继承

ES6开始提供`class`关键字，有点类似于JAVA，用`class`关键字定义对象，`extends`代表继承某个对象
