# 关于 RxJava 的理解

`Rx`->`Reactvie` + `Extension`

- `Reactvie`

  响应式编程。和观察者模式？

  - 观察者模式模型，被观察者变化 --通知--> 观察者
  - 响应式编程模型，被依赖的数据发生变化 --通知--> 依赖者

  认为不必过度区分两者，观察者模式模型也适用于响应式编程模型

- `Extension`

  1、支持事件序列（不可预知的，按钮点击等），还支持数据流（一般是固定的，数组等）

  2、大量的操作符（数据流的单独处理）

  3、线程的切换

  心得：只用一次 `subscribeOn`，放在最前或者最后，用来指定数据源的执行线程 ，多用 `observeOn`，指定数据处理、数据会滴的线程

## 上游事件的产生

参考 `Retrofit`、`RxBinding`

# 参考

- [zhihu-rxjava-meetup](https://github.com/zhihu/zhihu-rxjava-meetup)
