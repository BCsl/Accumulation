## Loader小结
* Loader框架中的角色
  - Activity和Fragment，Activity和Fragment依靠其自身的生命周期回调来主动管理LoaderManager，Activity和Fragment都有各自的LoaderManager，而Activity的`mAllLoaderManagers`则保存了其自身和其依附的Fragmnet的LoaderManager实例

  - LoaderManager，用于管理其所有的`Loader`（实际上是用`LoaderInfo`来描述）的生命周期，如`start`、`finishRetain`等

  - LoaderInfo，描述了Loader的信息，状态和数据结果等，并适当地回调其Loader的生命周期进行，同时监听实际Loader的加载结果

  - Loader，在各种生命周回调中做出相应的操作，基本空实现，需要子类重写

  - AsyncTaskLoader，继承自Loader，内部使用了LoaderTask，其机制如AsynTask，用于异步处理耗时操作，管理了异步加载过程中的周期回调

* 配置发生改变时候的保存和恢复
  - 由于LoaderManager是由Activity和Fragment主动来管理的，每个Activity用一个ArrayMap的`mAllLoaderManager`来保存当前Activity及其附属Frament的唯一LoaderManager；在Activity配置发生变化时，Activity在由于配置发生改变而`destory`前会保存`mAllLoaderManager`（具体看`ActivityThread#performDestroyActivity`方法），当Activity再重新创建时，会在`attach`、`onCreate`方法中恢复`mAllLoaderManager`，并在又需要的情况下再`onStart`的时候尝试恢复之前的数据，这部分尚未深入研究，挖坑。

* 如何自定义Loader
  - 简单地可以参考`CursorLoader`

## 一些bugs
- [Support Fragments do not retain loaders on rotation](https://code.google.com/p/android/issues/detail?id=183783)
