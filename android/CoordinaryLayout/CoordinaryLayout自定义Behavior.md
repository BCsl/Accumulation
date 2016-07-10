# CoordinatorLayout自定义Behavior

## 两种形式

- 某个view监听另一个view的状态变化，例如大小、位置、显示状态等

- 某个view监听CoordinatorLayout里的滑动状态

第一种情况需要重写`layoutDependsOn`和`onDependentViewChanged`方法

第二种情况需要重写`onStartNestedScroll`和`onNestedPreScroll`方法

### 两种角色

被依赖和依赖被依赖的View，提供依赖的View必须实现`NestedScrollingChild`接口（CoordinaryLayout已经实现了`NestedScrollingParent`接口）

这种依赖关系的建立由`CoordinatorLayout#LayoutParam`来指定，假设此时有两个View: A 和B，那么有两种情况会导致依赖关系

- A的anchor是B
- A的behavior对B有依赖

#### 依赖的判断

```java
boolean dependsOn(CoordinatorLayout parent, View child, View dependency) {
    return dependency == mAnchorDirectChild|| (mBehavior != null && mBehavior.layoutDependsOn(parent, child, dependency));
}
```

其中`Behavior`的初始化比较简单，通过setter或者xml指定通过反射实例化；Anchor也是通过通过setter或者xml指定，但是为了不需要每次都根据ID通过`findViewById`去解析出AnchorView，所以会使用`mAnchorView`变量缓存好，需要注意的是这个AnchorView不可以是起所在`CoordinatorLayout`，另外也不可以是当前View的一个子View，变量`mAnchorDirectChild`记录的就是AnchorView的所属的`ViewGroup`或自身（当它是`CoordinatorLayout`的子View的时候）

CoordinatorLayout中维护了一个`mDependencySortedChildren`列表，里面含有所有的子View，按依赖关系排序，被依赖者排在前面，会在每次测量前重新排序，确保处理的顺序是**先处理被依赖的View**

```java
final Comparator<View> mLayoutDependencyComparator = new Comparator<View>() {
    @Override
    public int compare(View lhs, View rhs) {
        if (lhs == rhs) {
            return 0;
        } else if (((LayoutParams) lhs.getLayoutParams()).dependsOn(CoordinatorLayout.this, lhs, rhs)) {
            return 1;
        } else if (((LayoutParams) rhs.getLayoutParams()).dependsOn(CoordinatorLayout.this, rhs, lhs)) {
            return -1;
        } else {
            return 0;
        }
    }
};
```

### 监听视图的重绘

CoordinatorLayout需要在每次视图发生重绘的时候检查被依赖视图的变化进行相应的回调，为了避免内存泄漏，每次Detach的时候回移除这个监听器并在重新Attach的时候添加回来

```java
void addPreDrawListener() {
    if (mIsAttachedToWindow) {
        // Add the listener
        if (mOnPreDrawListener == null) {
            mOnPreDrawListener = new OnPreDrawListener();
        }
        final ViewTreeObserver vto = getViewTreeObserver();
        vto.addOnPreDrawListener(mOnPreDrawListener);
    }
    // Record that we need the listener regardless of whether or not we're attached. We'll add the real listener when we become attached.
    mNeedsPreDrawListener = true;
}
```

所以每次重绘的时候都可能会调用到`dispatchOnDependentViewChanged`方法，这方法就是判断被依赖的View是否改变并作相应回调

```java
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
    @Override
    public boolean onPreDraw() {
        dispatchOnDependentViewChanged(false);
        return true;
    }
}
```

### Behavior的一些方法的调用时间

#### layoutDependsOn(CoordinatorLayout parent, V child, View dependency)

该方法用来判断dependency是否是child的依赖，

#### onDependentViewChanged(CoordinatorLayout parent, V child, View dependency)

dependency作为child的依赖，在位移、大小上发生改变的时候调用

## 参考

[精通 CoordinatorLayout Behavior](http://blog.chengyunfeng.com/?p=906)

[CoordinatorLayout高级用法-自定义Behavior](http://blog.csdn.net/qibin0506/article/details/50290421)

[源码看CoordinatorLayout.Behavior原理](http://blog.csdn.net/qibin0506/article/details/50377592)

[SwipeDismissBehavior用法及实现原理](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1103/3650.html)
