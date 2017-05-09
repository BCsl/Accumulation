# MVP 的一些见解

## 结构的迭代

### 传统的写法

传统的写法 `Activity/Fragment` 负责 `View` 加载，处理用户输入并响应直接操作 `View`，很明显 `Activity/Fragment` 并不是单纯的 `View`，还会有业务逻辑上的操作，这样大大增加了 `Activity/Fragment` 的**复杂性和耦合性**，不方便进行测试和复用

### 传统的 MVP

`MVP` 的目的，在于用 `P` 层来隔离了 `View` 和 `Model` 之间的直接联系，传统的以 `Google` 官方 `MVP` 为例，需要创建大量的接口，`Activity/Fragment` 作为单纯的 `View` 来使用，并组合一个 `Presenter` 来使用，带来的问题是，`Activity/Fragment` 本身就是 `Android`中的一个 `Context`，不论怎么去封装，都难以避免将业务逻辑代码写入到其中，且视图复用上就变得麻烦了，因为 `Activity/Fragment` 是 `View` ，就意味着需要多次的去在每个 `Activity/Fragment` 做一些初始化视图的操作，每个界面都要写一遍

### TheMVP

主要是把 `Activity/Fragment` 作为 `Presenter`，将 UI 操作抽到 `Delegate` 中，作为 `View` 层，并组合到 `P` 层来使用（传统的是 V 组合 P 来使用），`Presenter` 直接就可以和 `Activity/Fragment` 的生命周期做绑定并可以少写很多类，并且 `View` 可以直接独立出来并复用，但 `P` 的复用就变麻烦了。值得一提的是，支付宝也在使用。

## 参考

- [用MVP架构开发Android应用](https://www.kymjs.com/code/2015/11/09/01/)
- [传统MVP用在项目中是真的方便还是累赘?](http://www.jianshu.com/p/ac51c9b88af3)
