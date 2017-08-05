# 快速入门

[入门教程](https://facebook.github.io/react/tutorial/tutorial.html)

## JSX

`JSX` 是 `JS` 的一种扩展，为了把 `HTML` 模板直接嵌入到 `JS` 代码里面，这样就做到了模板和组件关联，[深入 JSX](https://discountry.github.io/react/docs/jsx-in-depth.html)

### 内嵌表达式

使用 `{}` 符号来包含一个 `JS` 表达式，并且推荐使用 `()` 来包括一段 `JSX` 代码来避免 [`ASI`](http://www.cnblogs.com/fsjohnhuang/p/4154503.html) 带来的错误

```javascript
const element = (
  <h1>
    Hello, {formatName(user)}!  //formatName(user) 是一个函数，返回 User 信息的字符串，这里省略
  </h1>
);

ReactDOM.render(
  element,
  document.getElementById('root')
);
```

### JSX 同样也是表达式

编译之后，`JSX` 成为常规的 `JS` 对象。这意味着可以在 `if` 语句和 `for` 循环中使用 `JSX`，可以将其分配给变量，可以作为方法的参数，可以作为函数返回值

```javascript

function getGreeting(user) {
  if (user) {
    return <h1>Hello, {formatName(user)}!</h1>;
  }
  return <h1>Hello, Stranger.</h1>;
}
```

### 使用 JSX 指定属性

```javascript
const element = <img src={user.avatarUrl}></img>;
```

### JSX 所代表的对象

Babel 将 `JSX` 编译成 `React.createElement()` 方法调用，如下两个代码效果相同

```javascript
const element = (
  <h1 className="greeting">
    Hello, world!
  </h1>
);
```

```javascript
const element = React.createElement(
  'h1',
  {className: 'greeting'},
  'Hello, world!'
);
```

`React` 实际上创建了这样一个 `React` 元素，描述你要在屏幕上看到的内容

```javascript
// Note: this structure is simplified
const element = {
  type: 'h1',
  props: {  //后面学习自定义组件的时候就会知道 props 的用处
    className: 'greeting',
    children: 'Hello, world'
  }
};
```

## 元素渲染

使用 `ReactDOM.render` 方法来渲染

```javascript
const element = <h1>Hello, world</h1>;
ReactDOM.render(
  element,
  document.getElementById('root')
);
```

每个元素新建之后是不变的，想改变 UI 就需要重新新建元素并调用 `ReactDOM.render` 方法来渲染（实际上，大部分情况下只需要调用一次渲染方法，然后在自定义组件的状态改变的时候就会重新渲染）

## React组件和属性

组件可以将 `UI` 拆分成独立的可重复使用的部分

两种方式来新建一个 `React` 组件，另外组件名需要使用大写字母开头

使用 `JS` 函数

```javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```

使用类继承

```javascript

class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

组件可以直接使用 `HTML` 标签来使用，并指定属性，组件就可以使用 `this.props.xxx` 来获取属性，属性一旦初始设置完成，就不应该修改了（这是 React 的一个规则）

```javascript
const element = <Welcome name="Sara" />;
ReactDOM.render(
  element,
  document.getElementById('root')
);
```

### 使用 React.PropTypes 对组件属性进行类型检测

> 注意: React.PropTypes 自 React v15.5 起已弃用。请使用 [prop-types](https://www.npmjs.com/package/prop-types) 库代替

`React.PropTypes` 返回的是一系列验证函数，用于确保接收的数据类似是否是有效的

```javascript

class Greeting extends React.Component {
    render() {
        return (
            <h1>Hello {this.props.name}</h1>
        )
    };
}
Greeting.propTypes = {
    name: React.PropTypes.string.isRequired
};
```

上面例子，使用 `React.PropTypes.string.isRequire` 检测 `name` 是否为字符串，并且是必填的

更多的属性

### 设置默认的属性

```javascript

class Greeting extends React.Component {
    render() {
        return <h1>hello {this.props.name}</h1>;
    };
}

// 如果name没有传值，则会将name设置为默认的zhangyatao
Greeting.defaultProps = {
    name: 'HelloWorld'
}
```

## React组件的状态和生命周期

`state` 是组件的当前状态（需要使用 `class` 来定义，而不是 `function`），可以把组件简单看成一个"状态机"，根据状态 `state` 呈现不同的 `UI` 展示。一旦状态更改，组件就会自动调用 `render` 重新渲染 `UI`，这个更改的动作会通过组件的 `this.setState()` 方法来触发

正确的使用状态

- 1.不要直接修改状态，而是使用 `this.setState()` 方法
- [2.状态修改可能是异步的](https://facebook.github.io/react/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)

  为了更好的性能，React 可能将多次 `this.setState` 方法的调用合并到一次更新上。另外 `this.props`和 `this.state` 可能是异步更新的，所以你不应该依靠它们的值来计算下一个状态

- [3.状态更新会被合并](https://facebook.github.io/react/docs/state-and-lifecycle.html#state-updates-are-merged)

  `setState` 方法在执行时，会作多个要更改 `state` 的合并，以及有类似异步的处理，这是 `React` 中隐性的运行机制，目的主要是为了优化与性能。[为何说setState方法是异步的？](https://segmentfault.com/a/1190000007454080)

组件在销毁的时候释放资源，所以 React 提供了组件的生命周期回调

- `componentWillMount`

  只会在装载之前调用一次，在 `render` 之前调用，你可以在这个方法里面调用 `setState` 改变状态，并且不会导致额外调用一次 `render`

- `componentDidMount`

  组件渲染在 DOM 树之后回调

- `componentWillReceiveProps(nextProps)`

  在组件接收到一个新的 prop 时被调用。不会再第一次渲染的时候回调

- `shouldComponentUpdate(nextProps, nextState)`

  返回一个布尔值。在组件接收到新的 props 或者 state 时被调用，确认是否需要更新组件时使用，不会再第一次渲染的时候回调

- `componentWillUpdate(nextProps, nextState)`

  在组件接收到新的 props 或者 state 但还没有 render 时被调用，不会再第一次渲染的时候回调

- `componentDidUpdate(prevProps, prevState)`

  组件状态改变后马上回调，不会再第一次渲染的时候回调

- `componentWillUnmount`

  组件从 DOM 树中移除的时候回调

## 事件处理

和处理 DOM 元素事件类似，语法上的差别

- 1.React 的事件命名使用驼峰而不是小写
- 2.使用 JSX 处理一个事件处理的函数，而不是一个字符串

这是 `HTML` 的方式

```javascript
<button onclick = "activateLasers()"> //事件小写，传递方法名字符串
  Activate Lasers
</button>
```

使用 `React` 的方式

```javascript
<button onClick = {activateLasers} >  //事件驼峰命名，传递方法
  Activate Lasers
</button>
```

> 注意要显式调用 `bind(this)` 将事件函数上下文绑定要组件实例上（或者使用箭头函数），否则这个函数中的 `this` 是 `undefined`，这不是 `React` 的特殊行为，[它是函数如何在 JavaScript 中运行的一部分](https://www.smashingmagazine.com/2014/01/understanding-javascript-function-prototype-bind/) ，[中文版](http://blog.csdn.net/alex8046/article/details/51909744)，需要显示调用 `preventDefault` 方法来阻止默认的事件行为执行

```javascript

function ActionLink() {
  function handleClick(e) {
    e.preventDefault();
    console.log('The link was clicked.');
  }
}
//...
  return (
    <a href="#" onClick={handleClick}>
      Click me
    </a>
  );
  //...
```

## 操作 DOM

- 1.`findDOMNode`

`ReactDOM#render` 方法返回对组件的引用（对于无状态状态组件来说返回 `null`），当组件加载到页面上之后`（mounted）`，可以通过 `react-dom` 提供的 `findDOMNode()` 方法拿到组件对应的 `DOM` 元素

- 2.`Refs`

另外一种方式就是通过在要引用的 `DOM` 元素上面设置一个 `ref` 属性指定一个名称，然后通过 `this.refs.name` 来访问对应的 `DOM` 元素，如果 `ref` 是设置在原生 `HTML` 元素上，它拿到的就是 `DOM` 元素，如果设置在自定义组件上，它拿到的就是组件实例，这时候就需要通过 `findDOMNode` 来拿到组件的 `DOM` 元素

如果想要拿无状态组件的 `DOM` 元素的时候，就需要用一个状态组件封装一层，然后通过 `ref` 和 `findDOMNode` 去获取
