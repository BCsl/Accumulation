# 使用 Redux 进行 ReactNative 开发二

## Redux 和 React 的结合

```
//和 React 绑定
npm install --save react-redux
```

需要安装 `react-redux`，并且最好了解容器组件和展示组件的区别，一般来说展示组件只专注于视图的展示，数据源来自 `props` 且不应该依赖 Redux，而容器组件则是专注如何运行（数据获取和状态更新），可以依赖 Redux 并监听 Redux 的状态

技术上讲，容器组件就是使用 `store.subscribe()` 从 Redux `state` 树中读取部分数据，并通过 `props` 来把这些数据提供给要渲染的组件。你可以手工来开发容器组件，但建议使用 `react-redux` 库的 `connect()` 方法来生成，这个方法做了性能优化来避免很多不必要的重复渲染

具体的结合方式还是看 [搭配React](http://cn.redux.js.org/docs/basics/UsageWithReact.html)

### react-redux 的原理

#### Provider 组件

`Provider` 是一个 React 组件，是一个容器组件，使用组合的形式把应用的组件作为子组件来渲染，并为子组件提供统一的 `Store` 对象，且 `Provider` 仅支持单个子组件，且重写了 `getChildContext` 方法，将 `Store` 对象放入 `context` 对象中，使子孙组件可以直接访问到 `context` 对象中的 `Store`

```javascript

export function createProvider(storeKey = 'store', subKey) {
    const subscriptionKey = subKey || `${storeKey}Subscription`   //storeSubscription

    class Provider extends Component {
        //把 store 记录在 context
        getChildContext() {
          return { [storeKey]: this[storeKey], [subscriptionKey]: null }
        }

        constructor(props, context) {
          super(props, context)
          this[storeKey] = props.store;
        }

        render() {
          return Children.only(this.props.children) //Children.only 用于获取仅有的一个子组件，没有或超过一个均会报错
        }
    }
    //...
    Provider.propTypes = {
        store: storeShape.isRequired,
        children: PropTypes.element.isRequired,
    }
    Provider.childContextTypes = {
        [storeKey]: storeShape.isRequired,
        [subscriptionKey]: subscriptionShape,
    }
    Provider.displayName = 'Provider'
    return Provider
}

export default createProvider()
```

#### connect

`connect` 是一个返回高阶组件的高阶函数，返回的函数就是一个高阶组件，该高阶组件返回一个与 Redux Store 关联起来的新组件

```javascript
function connect(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  {
    pure = true,
    areStatesEqual = strictEqual,
    areOwnPropsEqual = shallowEqual,
    areStatePropsEqual = shallowEqual,
    areMergedPropsEqual = shallowEqual,
    ...extraOptions
  } = {}
) {...}
```

把视图组件和

# 更多

- [Redux 中文文档 - Context](https://discountry.github.io/react/docs/context.html)
- [解析 Redux 源码](https://zhuanlan.zhihu.com/p/22809799)
- [容器组件和展示组件相分离](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)
- [如何合理地设计 Redux 的 State](https://juejin.im/post/59a16e2651882511264e8402?utm_source=gold_browser_extension)
