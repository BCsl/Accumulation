# 自定义Behavior的艺术探索-仿UC浏览器主页

## 前言&效果预览

最近几个周末基本在研究 CoordinatorLayout 控件和自定义 Behavior 当中，这期间看了不少这方面的知识，有关于 CoordinatorLayout 使用的文章，CoordinatorLayout 的源码分析文章等等，轻轻松松入门虽然简单，无耐于网上介绍的一些例子实在是太简单，很多东西都是草草带过，尤其是关于 NestedScroll 效果这方面的，最后发现自己到头来其实还是一头雾水，当然，自己在周末的时候效率的确不高，干扰因素也多。但无意中发现了一篇通过自定义 View 的方式实现的[仿 UC 浏览器主页的文章](http://ittiger.cn/2016/05/26/UC%E6%B5%8F%E8%A7%88%E5%99%A8%E9%A6%96%E9%A1%B5%E6%BB%91%E5%8A%A8%E5%8A%A8%E7%94%BB%E5%AE%9E%E7%8E%B0/)（大家也可以看看，对于自定义 View 也是有帮助的），顿时有了使用自定义 Behavior 实现这样的效果的想法，而且这种方式在我看来应该会更简单， **但重点是这货的解耦功能！！！你使用 `Behavior` 抽象了某个模块的 View 的行为，而不再是依赖于特定的 View ，以后可以随便地替换这部分的 View ，而你只需要为改变的 View 设置好对应的 `Behavior`** ，于是看了很多这方面的源码 CoordinatorLayout、NestedScrollView、SwipeDismissBehavior、FloatingActionButton.Behavior、AppBarLayout.Behavior 等，也是有所顿悟，于是有了今天的这篇文章。忆当年，自己也曾经在 UC 浏览器实习过大半年的时间，UC 也是自己一直除了 QQ 从塞班时代至今一直使用的 APP 了，只怪自己当时有点作死。。。。咳咳，扯多了，还是直接来看效果吧，因为文章比较长，不先放个效果图，估计没多少人能耐心看完（即使放了，估计也没多少能撑着看完，文章特长...要不直接看 [代码](https://github.com/BCsl/UcMainPagerDemo)？）

![效果图](https://raw.githubusercontent.com/BCsl/GoogleWidget/master/distribution/NestedScroll2.gif)

## 揭开 NestedScrolling 的原理

网上不少写文章写到自定义Behavior的实现方式有两种形式，其中一种是实现 NestedScrolling 效果的，需要关注重写 `onStartNestedScroll` 和 `onNestedPreScroll` 等一系列带 `Nested` 字段的方法，当你一看这样的一个方法 `onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed)` 是有多少个参数的时候，你通常会一脸懵逼，就算你搞懂了这里的每个参数的意思，你还是会有所疑问，这样的一大堆方法是在什么时候调用的，这个时候，你首先需要弄懂的是 Android5.0 开始提供的支持嵌套滑动效果的机制

NestedScrolling 提供了一套父 View 和子 View 滑动交互机制。要完成这样的交互，父 View 需要实现 `NestedScrollingParent` 接口，而子 View 需要实现 `NestedScrollingChild` 接口，系统提供的 `NestedScrollView` 控件就实现了这两个接口，千万不要被这两个接口这么多的方法唬住了，这两个接口都有一个 `NestedScrolling[Parent,Children]Helper` 辅助类来帮助处理的大部分逻辑，它们之间关系如下

![NestedScrollView](https://raw.githubusercontent.com/BCsl/Accumulation/master/android/CoordinaryLayout/img/NestedScrollView.png)

### 实现 NestedScrollingChild 接口

实际上 `NestedScrollingChildHelper` 辅助类已经实现好了 Child 和 Parent 交互的逻辑。原来的 View 的处理滑动 事件的逻辑大体上不需要改变。

需要做的就是，如果要准备开始滑动了，需要告诉 Parent，Child 要准备进入滑动状态了，调用 `startNestedScroll()`。Child 在滑动之前，先问一下你的 Parent 是否需要滑动，也就是调用 `dispatchNestedPreScroll()`。如果父类消耗了部分滑动事件，Child 需要重新计算一下父类消耗后剩下给 Child 的滑动距离余量。然后，Child 自己进行余下的滑动。最后，如果滑动距离还有剩余，Child 就再问一下，Parent 是否需要在继续滑动你剩下的距离，也就是调用 `dispatchNestedScroll()`，大概就是这么一回事，当然还还会有和 `scroll` 类似的 `fling` 系列方法，但我们这里可以先忽略一下

`NestedScrollView` 的 `NestedScrollingChild` 接口实现都是交给辅助类 `NestedScrollingChildHelper` 来处理的，是否需要进行额外的一些操作要根据实际情况来定

```java
// NestedScrollingChild
public NestedScrollView(Context context, AttributeSet attrs, int defStyleAttr) {
    //...
    mParentHelper = new NestedScrollingParentHelper(this);
    mChildHelper = new NestedScrollingChildHelper(this);
    //...
    setNestedScrollingEnabled(true);
}

@Override
public void setNestedScrollingEnabled(boolean enabled) {
    mChildHelper.setNestedScrollingEnabled(enabled);
}

@Override
public boolean isNestedScrollingEnabled() {
    return mChildHelper.isNestedScrollingEnabled();
}

//在初始化滚动操作的时候调用，一般在 MotionEvent#ACTION_DOWN 的时候调用
@Override
public boolean startNestedScroll(int axes) {
    return mChildHelper.startNestedScroll(axes);
}

@Override
public void stopNestedScroll() {
    mChildHelper.stopNestedScroll();
}

@Override
public boolean hasNestedScrollingParent() {
    return mChildHelper.hasNestedScrollingParent();
}

//参数和dispatchNestedPreScroll方法的返回有关联
@Override
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed,int dyUnconsumed, int[] offsetInWindow) {
    return mChildHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed,offsetInWindow);
}

//在消费滚动事件之前调用，提供一个让ViewParent实现联合滚动的机会，因此ViewParent可以消费一部分或者全部的滑动事件，参数consumed会记录ViewParent所消费掉的事件
@Override
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
    return mChildHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
}

@Override
public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
    return mChildHelper.dispatchNestedFling(velocityX, velocityY, consumed);
}

@Override
public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
    return mChildHelper.dispatchNestedPreFling(velocityX, velocityY);
}
```

实现 `NestedScrollingChild` 接口挺简单的不是吗？但还需要我们决定什么时候进行调用，和调用那些方法

#### startNestedScroll 和 stopNestedScroll 的调用

`startNestedScroll` 配合 `stopNestedScroll` 使用，`startNestedScroll` 会再接收到 `ACTION_DOWN` 的时候调用，`stopNestedScroll` 会在接收到 `ACTION_UP|ACTION_CANCEL` 的时候调用，`NestedScrollView`中的伪代码是这样

```java
onInterceptTouchEvent | onTouchEvent (MotionEvent ev){

   case MotionEvent.ACTION_DOWN:
      startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
   break;
   case  MotionEvent.ACTION_CANCEL | MotionEvent.ACTION_UP:
      stopNestedScroll();
   break;
}
```

`NestedScrollingChildHelper` 处理 `startNestedScroll` 方法，可以看出可能会调用 Parent 的 `onStartNestedScroll` 和 `onNestedScrollAccepted` 方法，只要 Parent 愿意优先处理这次的滑动事件，在结束的时候 Parent 还会收到 `onStopNestedScroll` 回调

```java
public boolean startNestedScroll(int axes) {
    if (hasNestedScrollingParent()) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
                mNestedScrollingParent = p;
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}

public void stopNestedScroll() {
    if (mNestedScrollingParent != null) {
        ViewParentCompat.onStopNestedScroll(mNestedScrollingParent, mView);
        mNestedScrollingParent = null;
    }
}
```

#### dispatchNestedPreScroll 的调用

在消费滚动事件之前调用，提供一个让 Parent 实现联合滚动的机会，因此 Parent 可以消费一部分或者全部的滑动事件，注意参数 `consumed` 会记录了 Parent 所消费掉的事件

```java
onTouchEvent (MotionEvent ev){
    //...
   case MotionEvent.ACTION_MOVE:
   //...
     final int y = (int) MotionEventCompat.getY(ev, activePointerIndex);
     int deltaY = mLastMotionY - y; //计算偏移量
     if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
       deltaY -= mScrollConsumed[1]; //减去被消费掉的事件
       //...
   }
   //...
   break;
}
```

`NestedScrollingChildHelper` 处理 `dispatchNestedPreScroll` 方法，会调用到上一步里记录的希望优先处理 Scroll 事件的 Parent 的 `onNestedPreScroll` 方法

```java
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dx != 0 || dy != 0) {
            int startX = 0;
            int startY = 0;
            //...
            consumed[0] = 0;
            consumed[1] = 0;
            ViewParentCompat.onNestedPreScroll(mNestedScrollingParent, mView, dx, dy, consumed);
            //...
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            //...
        }
    }
    return false;
}
```

#### dispatchNestedScroll 的调用

这个方法是在 Child 自己消费完 Scroll 事件后调用的

```java
onTouchEvent (MotionEvent ev){
    //...
   case MotionEvent.ACTION_MOVE:
   //...
   final int scrolledDeltaY = getScrollY() - oldY; //计算这个Child View消费掉的Scroll事件
   final int unconsumedY = deltaY - scrolledDeltaY; //计算的是这个Child View还没有消费掉的Scroll事件
   if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
       mLastMotionY -= mScrollOffset[1];
       vtev.offsetLocation(0, mScrollOffset[1]);//重新调整事件的位置
       mNestedYOffset += mScrollOffset[1];
   }
   //...
   break;
}
```

`NestedScrollingChildHelper` 处理 `dispatchNestedScroll` 方法，会调用到上一步里记录的希望优先处理 Scroll 事件的 Parent 的 `onNestedScroll` 方法

```java
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
        int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
            int startX = 0;
            int startY = 0;
            //...
            ViewParentCompat.onNestedScroll(mNestedScrollingParent, mView, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);

            //..
            return true;
        } else if (offsetInWindow != null) {
            // No motion, no dispatch. Keep offsetInWindow up to date.
            //..
        }
    }
    return false;
}
```

### 实现 NestedScrollingParent 接口

同样，也有一个 `NestedScrollingParentHelper` 辅助类来帮助 Parent 实现和 Child 交互的逻辑。**滑动动作是 Child 主动发起**，Parent 就受滑动回调并作出响应。从上面的 Child 分析可知，滑动开始的调用 `startNestedScroll()`，Parent 收到 `onStartNestedScroll()` 回调，决定是否需要配合 Child 一起进行处理滑动，如果需要配合，还会回调 `onNestedScrollAccepted()`

每次滑动前，Child 先询问 Parent 是否需要滑动，即 `dispatchNestedPreScroll()` ，这就回调到 Parent 的 `onNestedPreScroll()`，Parent 可以在这个回调中消费掉 Child 的 Scroll 事件，也就是优先于 Child 滑动

Child 滑动以后，会调用 `dispatchNestedScroll()` ，回调到 Parent 的 `onNestedScroll()` ，这里就是 Child 滑动后，剩下的给 Parent 处理，也就是后于 Child 滑动

最后，滑动结束 Child 调用 `stopNestedScroll`，回调 Parent 的 `onStopNestedScroll()` 表示本次处理结束

现在我们来看看 `NestedScrollingParent` 的实现细节，这里以 `CoordinatorLayout` 来分析而不再是 `NestedScrollView` ，因为它才是这篇文章的主角

在这之前，首先简单介绍下 `Behavior` 这个对象，你可以在 XML 中定义它就会在 `CoordinaryLayout` 中解析实例化到目标子 View 的 `LayoutParams` 或者获取到 `CoordinaryLayout` 子 View 的 `LayoutParams` 对象通过 setter 方法注入，如果你自定义的 `Behavior` 希望实现 NestedScroll 效果，那么你需要关注重写以下这些方法

- onStartNestedScroll ： boolean
- onStopNestedScroll ： void
- onNestedScroll ： void
- onNestedPreScroll ： void
- onNestedFling ： void
- onNestedPreFling ： void

你会发现以上这些方法对应了 `NestedScrollingParent` 接口的方法，只是在参数上有所增加，且都会在 `CoordiantorLayout` 实现 `NestedScrollingParent` 接口的每个方法中作出相应回调，下面来简单走读下这部分代码

```java
public class CoordinatorLayout extends ViewGroup implements NestedScrollingParent {
  //.....

//CoordiantorLayout的成员变量
private final NestedScrollingParentHelper mNestedScrollingParentHelper = new NestedScrollingParentHelper(this);

   // 参数child:当前实现`NestedScrollingParent`的ViewParent包含触发嵌套滚动的直接子view对象
   // 参数target:触发嵌套滚动的view  (在这里如果不涉及多层嵌套的话,child和target)是相同的
   // 参数nestedScrollAxes:就是嵌套滚动的滚动方向了.垂直或水平方法
   //返回参数代表当前ViewParent是否可以触发嵌套滚动操作
   //CoordiantorLayout的实现上是交由子View的Behavior来决定，并回调了各个acceptNestedScroll方法，告诉它们处理结果
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        boolean handled = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,nestedScrollAxes);
                handled |= accepted;
                lp.acceptNestedScroll(accepted);
            } else {
                lp.acceptNestedScroll(false);
            }
        }
        return handled;
    }
    //onStartNestedScroll返回true才会触发这个方法
    //参数和onStartNestedScroll方法一样
    //按照官方文档的指示，CoordiantorLayout有一个NestedScrollingParentHelper类型的成员变量，并把这个方法交由它处理
    //同样，这里也是需要CoordiantorLayout遍历子View，对可以嵌套滚动的子View回调Behavior#onNestedScrollAccepted方法
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
        mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
        mNestedScrollingDirectChild = child;
        mNestedScrollingTarget = target;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScrollAccepted(this, view, child, target, nestedScrollAxes);
            }
        }
    }

    //嵌套滚动的结束，做一些资源回收操作等...
    //为可以嵌套滚动的子View回调Behavior#onStopNestedScroll方法
    public void onStopNestedScroll(View target) {
        mNestedScrollingParentHelper.onStopNestedScroll(target);

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onStopNestedScroll(this, view, target);
            }
            lp.resetNestedScroll();
            lp.resetChangedAfterNestedScroll();
        }

        mNestedScrollingDirectChild = null;
        mNestedScrollingTarget = null;
    }
    //进行嵌套滚动
    // 参数dxConsumed:表示target已经消费的x方向的距离
    // 参数dyConsumed:表示target已经消费的x方向的距离
    // 参数dxUnconsumed:表示x方向剩下的滑动距离
    // 参数dyUnconsumed:表示y方向剩下的滑动距离
    // 可以嵌套滚动的子View回调Behavior#onNestedScroll方法
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        final int childCount = getChildCount();
        boolean accepted = false;

        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,dxUnconsumed, dyUnconsumed);
                accepted = true;
            }
        }

        if (accepted) {
            dispatchOnDependentViewChanged(true);
        }
    }
    //发生嵌套滚动之前回调
    // 参数dx:表示target本次滚动产生的x方向的滚动总距离
    // 参数dy:表示target本次滚动产生的y方向的滚动总距离
    // 参数consumed:表示父布局要消费的滚动距离,consumed[0]和consumed[1]分别表示父布局在x和y方向上消费的距离.
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        int xConsumed = 0;
        int yConsumed = 0;
        boolean accepted = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                mTempIntPair[0] = mTempIntPair[1] = 0;
                viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair);

                xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0]): Math.min(xConsumed, mTempIntPair[0]);
                yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1]): Math.min(yConsumed, mTempIntPair[1]);

                accepted = true;
            }
        }

        consumed[0] = xConsumed;
        consumed[1] = yConsumed;

        if (accepted) {
            dispatchOnDependentViewChanged(true);
        }
    }

    // @param velocityX 水平方向速度
    // @param velocityY 垂直方向速度
    // @param consumed 子View是否消费fling操作
    // @return true if this parent consumed or otherwise reacted to the fling
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {
        boolean handled = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                handled |= viewBehavior.onNestedFling(this, view, target, velocityX, velocityY,consumed);
            }
        }
        if (handled) {
            dispatchOnDependentViewChanged(true);
        }
        return handled;
    }

    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
        boolean handled = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted()) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                handled |= viewBehavior.onNestedPreFling(this, view, target, velocityX, velocityY);
            }
        }
        return handled;
    }
    //支持嵌套滚动的方向
    public int getNestedScrollAxes() {
        return mNestedScrollingParentHelper.getNestedScrollAxes();
    }
}
```

你会发现 `CoordiantorLayout` 收到来自 `NestedScrollingChild` 的各种回调后，都是交由需要响应的 `Behavior` 来处理的，所以这里可以得出一个结论，`CoordiantorLayout` 是 `Behavior` 的一个代理类，所以 `Behavior` 实际上也是一个 `NestedScrollingParent` ，另外结合 `NestedScrollingChild` 实现的部分来看，你很容就能搞懂这些方法参数的实际含义

`CoordiantorLayout` , `Behavior` 和 `NestedScrollingParent` 三者关系 ![NstesdScroll](https://raw.githubusercontent.com/BCsl/Accumulation/master/android/CoordinaryLayout/img/NestedScroll2.png)

### NestedScroll 小结

NestedScroll 的机制的简版是这样的，当子 View 在处理滑动事件之前，先告诉自己的父 View 是否需要先处理这次滑动事件，父 View 处理完之后，告诉子 View 它处理了多少滑动距离，剩下的还是交给子 View 自己来处理

你也可以实现这样的一套机制，父 View 拦截所有事件，然后分发给需要的子 View 来处理，然后剩余的自己来处理。但是这样就做会使得逻辑处理更复杂，因为事件的传递本来就由外先内传递到子 View ，处理机制是由内向外，由子 View 先来处理事件本来就是遵守默认规则的，这样更自然且坑更少，不知道自己说得对不对，欢迎打脸(￣ε(#￣)☆╰╮(￣▽￣///)

## CoordinatorLayout 的源码走读和如何自定义 Behavior

上面在分析 `NestedScrollingParent` 接口的时候已经简单提到了 `CoordinatorLayout` 这个控件，至于这个控件是用来做什么的？`CoordinatorLayout` 内部有个 `Behavior` 对象，这个 `Behavior` 对象可以通过外部 setter 或者在 xml 中指定的方式注入到 `CoordinatorLayout` 的某个子 View 的 `LayoutParams`，`Behavior` 对象定义了特定类型的视图交互逻辑，譬如 `FloatingActionButton` 的 `Behavior` 实现类，只要 `FloatingActionButton` 是 `CoordinatorLayout` 的子View，且设置的该 `Behavior`（默认已经设置了），那么，这个 FAB 就会在 `Snackbar` 出现的时候上浮，而不至于被遮挡，而这种通过定义 `Behavior` 的方式就可以控制 View 的某一类的行为，通常会比自定义 View 的方式更解耦更轻便，由此可知，`Behavior` 是 `CoordinatorLayout` 的精髓所在

### Behavior 的解析和实例化

简单来看看 `Behavior` 是如何从 xml 中解析的，通过检测 `xxx:behavior` 属性，通过全限定名或者相对路径的形式指定路径，最后通过反射来新建实例，默认的构造器是 `Behavior(Context context, AttributeSet attrs)` ,如果你需要配置额外的参数，可以在外部构造好 `Behavior` 并通过 setter 的方式注入到 `LayoutParams` 或者获取到解析好的 `Behavior` 进行额外的参数设定

```java
LayoutParams(Context context, AttributeSet attrs) {
    super(context, attrs);
    final TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CoordinatorLayout_LayoutParams);
    //....
    mBehaviorResolved = a.hasValue(R.styleable.CoordinatorLayout_LayoutParams_layout_behavior); //通过apps:behavior属性·
    if (mBehaviorResolved) {
        mBehavior = parseBehavior(context, attrs, a.getString(R.styleable.CoordinatorLayout_LayoutParams_layout_behavior));
    }
    a.recycle();
}

static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
    if (TextUtils.isEmpty(name)) {
        return null;
    }
    final String fullName;
    if (name.startsWith(".")) {
        // Relative to the app package. Prepend the app package name.
        fullName = context.getPackageName() + name;
    } else if (name.indexOf('.') >= 0) {
        // Fully qualified package name.
        fullName = name;
    } else {
        // Assume stock behavior in this package (if we have one)
        fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME) ? (WIDGET_PACKAGE_NAME + '.' + name) : name;
    }
    try {
        ///...
        if (c == null) {
            final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true, context.getClassLoader());
            c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
            c.setAccessible(true);
            //...
        }
        return c.newInstance(context, attrs);
    } catch (Exception e) {
        throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
    }
}
```

### 两种关系和两种形式

#### View 之间的依赖关系

`CoordinatorLayout` 的子 View 可以扮演着不同角色，一种是被依赖的，而另外一种则是主动寻找依赖的 View ，被依赖的 View 并不会感知到自己被依赖，被依赖的 View 也有可能是寻找依赖的 View

这种依赖关系的建立由 `CoordinatorLayout#LayoutParam` 来指定，假设此时有两个 View：A 和 B，那么有两种情况会导致依赖关系

- A 的 anchor 是 B
- A 的 behavior 对 B 有依赖

`LayoutParams` 中关于依赖的判断的依据的代码如下

```java
LayoutParams.class

boolean dependsOn(CoordinatorLayout parent, View child, View dependency) {
    return dependency == mAnchorDirectChild|| (mBehavior != null && mBehavior.layoutDependsOn(parent, child, dependency));
}
```

依赖判断通过两个条件判断，一个生效即可，最容易理解的是根据 `Behavior#layoutDependsOn` 方法指定，例如 FAB 依赖 `Snackbar`

```java
Behavior.java

@Override
public boolean layoutDependsOn(CoordinatorLayout parent, FloatingActionButton child, View dependency) {
    return Build.VERSION.SDK_INT >= 11 && dependency instanceof Snackbar.SnackbarLayout;
}
```

另外一个可以看到是通过 `mAnchorDirectChild` 来判断，首先要知道 AnchorView 的 ID 是通过 setter 或者 xml 的 anchor 属性形式指定，但是为了不需要每次都根据ID通过 `findViewById` 去解析出 AnchorView，所以会使用 `mAnchorView` 变量缓存好，需要注意的是这个 AnchorView 不可以是 `CoordinatorLayout` ，另外也不可以是当前 View 的一个子 View ，变量 `mAnchorDirectChild` 记录的就是 AnchorView 的所属的`ViewGroup`或自身（当它直接ViewParent是`CoordinatorLayout`的时候），关于 AnchorView的作用，也可以在 FAB 配合`AppBarLayout`使用的时候，`AppBarLayout` 会作为 FAB 的 AnchorView，就可以在 `AppBarLayout` 打开或者收缩状态的时候显示或者隐藏 FAB，自己这方面的实践比较少，在这也可以先忽略并不影响后续分析，大家感兴趣的可以通过看相关代码一探究竟

根据这种依赖关系，`CoordinatorLayout` 中维护了一个 `mDependencySortedChildren` 列表，里面含有所有的子 View，按依赖关系排序，被依赖者排在前面，会在每次测量前重新排序，确保处理的顺序是 **被依赖的 View 会先被 measure 和 layout**

```java
final Comparator<View> mLayoutDependencyComparator = new Comparator<View>() {
    @Override
    public int compare(View lhs, View rhs) {
        if (lhs == rhs) {
            return 0;
        } else if (((LayoutParams) lhs.getLayoutParams()).dependsOn(CoordinatorLayout.this, lhs, rhs)) { //lhs 依赖 rhs,lhs>rhs
            return 1;
        } else if (((LayoutParams) rhs.getLayoutParams()).dependsOn(CoordinatorLayout.this, rhs, lhs)) { // rhs 依赖 lhs ,lhs<rhs
            return -1;
        } else {
            return 0;
        }
    }
};
```

`selectionSort` 方法使用的就是 `mLayoutDependencyComparator` 来处理，list 参数是所有子 View 的集合，这里使用了_选择排序法_，递增的方式，所以最后被依赖的 View 会排在最前

```java
private static void selectionSort(final List<View> list, final Comparator<View> comparator) {
    if (list == null || list.size() < 2) { //只有一个的时候当然不需要排序了
        return;
    }
    final View[] array = new View[list.size()];
    list.toArray(array);
    final int count = array.length;
    for (int i = 0; i < count; i++) {
        int min = i;
        for (int j = i + 1; j < count; j++) {
            if (comparator.compare(array[j], array[min]) < 0) {
                min = j;
            }
        }
        if (i != min) {
            // 把小的交换到前面
            final View minItem = array[min];
            array[min] = array[i];
            array[i] = minItem;
        }
    }
    list.clear();
    for (int i = 0; i < count; i++) {
        list.add(array[i]);
    }
}
```

这里有个疑问？为什么不直接使用 `Collections#sort(List<T> list, Comparator<? super T> comparator)` 的方式来排序呢？我的想法是考虑到可能会出现这样的一种情况，A 依赖 B，B 依赖 C，C 依赖 A，这时候 Comparator 比较的时候，A > B，B > C，C > A，这就违背了 Comparator 所要求的传递性（根据传递性原则，A 应该大于 C ），所以没有使用 `sort` 方法来排序，不知道自己说得是否正确，有知道的一定要告诉我，经过选择排序的方式的结果是 [C，B，A] ，所以虽然 C 依赖 A，但也可能先处理了 C，这就如果你使用到这样的依赖关系的时候就需要谨慎且注意了，例如你在 C 处理 `onMeasureChild` 的时候，你并不能得到 C 依赖的 A 的测量结果，因为 C 先于 A 处理了

#### 依赖的监听

这种依赖关系确定后又有什么作用呢？当然是在主动寻找依赖的View，在其依赖的View发生变化的时候，自己能够知道啦，也就是如果`CoordinatorLayout`内的A依赖B，在B的大小位置等发生状态的时候，A可以监听到，并作出响应，`CoordinatorLayout`又是怎么实现的呢？

`CoordinatorLayout`本身注册了两种监听器，`ViewTreeObserver.OnPreDrawListener`和`OnHierarchyChangeListener`，一种是在绘制的之前进行回调，一种是在子View的层级结构发生变化的时候回调，有这两种监听就可以在接受到被依赖的View的变化了

_监听提供依赖的视图的位置变化_

`OnPreDrawListener`在`CoordinatorLayout`绘制之前回调，因为在`layout`之后，所以可以很容易判断到某个View的位置是否发生的改变

```java
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
    @Override
    public boolean onPreDraw() {
        dispatchOnDependentViewChanged(false);
        return true;
    }
}
```

`dispatchOnDependentViewChanged`方法，会遍历根据依赖关系排序好的子View集合，找到位置改变了的View，并回调依赖这个View的`Behavior`的`onDependentViewChanged`方法

```java
void dispatchOnDependentViewChanged(final boolean fromNestedScroll) {
    final int layoutDirection = ViewCompat.getLayoutDirection(this);
    final int childCount = mDependencySortedChildren.size();
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        // Check child views before for anchor
        //...
        // Did it change? if not continue
        final Rect oldRect = mTempRect1;
        final Rect newRect = mTempRect2;
        getLastChildRect(child, oldRect);
        getChildRect(child, true, newRect);
        if (oldRect.equals(newRect)) { //比较前后两次位置变化，位置没发生改变就进入下次循环得了
            continue;
        }
        recordLastChildRect(child, newRect);
        // 如果改变了，往后面位置中找到依赖当前View的Behavior来进行回调
        for (int j = i + 1; j < childCount; j++) {
            final View checkChild = mDependencySortedChildren.get(j);
            final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
            final Behavior b = checkLp.getBehavior();

            if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                if (!fromNestedScroll && checkLp.getChangedAfterNestedScroll()) {
                    // If this is not from a nested scroll and we have already been changed from a nested scroll, skip the dispatch and reset the flag
                    checkLp.resetChangedAfterNestedScroll();
                    continue;
                }
                final boolean handled = b.onDependentViewChanged(this, checkChild, child);
                if (fromNestedScroll) {
                    // If this is from a nested scroll, set the flag so that we may skip any resulting onPreDraw dispatch (if needed)
                    checkLp.setChangedAfterNestedScroll(handled);
                }
            }
        }
    }
}
```

_监听提供依赖的View的添加和移除_

`HierarchyChangeListener`在View的添加和移除都会回调

```java
private class HierarchyChangeListener implements OnHierarchyChangeListener {
    //...
    @Override
    public void onChildViewRemoved(View parent, View child) {
        dispatchDependentViewRemoved(child);
        //..
    }
}
```

根据情况回调`Behavior#onDependentViewRemoved`

```java
void dispatchDependentViewRemoved(View view) {
    final int childCount = mDependencySortedChildren.size();
    boolean viewSeen = false;
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        if (child == view) {
            // 只需要判断后续位置的View是否依赖当前View并回调
            viewSeen = true;
            continue;
        }
        if (viewSeen) {
            CoordinatorLayout.LayoutParams lp = (CoordinatorLayout.LayoutParams)child.getLayoutParams();
            CoordinatorLayout.Behavior b = lp.getBehavior();
            if (b != null && lp.dependsOn(this, child, view)) {
                b.onDependentViewRemoved(this, child, view);
            }
        }
    }
}
```

### 自定义 Behavior 的两种目的

我们可以按照两种目的来实现自己的 `Behavior`，当然也可以两种都实现啦

- 某个 view 监听另一个 view 的状态变化，例如大小、位置、显示状态等

- 某个 view 监听 `CoordinatorLayout` 内的 `NestedScrollingChild` 的接口实现类的滑动状态

第一种情况需要重写 `layoutDependsOn` 和 `onDependentViewChanged` 方法

第二种情况需要重写 `onStartNestedScroll` 和 `onNestedPreScroll` 系列方法（上面已经提到了哦）

对于第一种情况，我们之前分析依赖的监听的时候相关回调细节已经说完了，`Behavior` 只需要在 `onDependentViewChanged` 做相应的处理就好

对于第二种情况，我们在 NestedScoll 的那节也已经把相关回调细节说了

### CoordinatorLayout的事件传递

`CoordinatorLayout`并不会直接处理触摸事件，而是尽可能地先交由子View的`Behavior`来处理，它的`onInterceptTouchEvent`和`onTouchEvent`两个方法最终都是调用`performIntercept`方法，用来分发不同的事件类型分发给对应的子View的`Behavior`处理

```java
//处理拦截或者自己的触摸事件
private boolean performIntercept(MotionEvent ev, final int type) {
    boolean intercepted = false;
    boolean newBlock = false;

    MotionEvent cancelEvent = null;

    final int action = MotionEventCompat.getActionMasked(ev);

    final List<View> topmostChildList = mTempList1;
    getTopSortedChildren(topmostChildList); //在5.0以上，按照z属性来排序，以下，则是按照添加顺序或者自定义的绘制顺序来排列

    // Let topmost child views inspect first
    final int childCount = topmostChildList.size();
    for (int i = 0; i < childCount; i++) {
        final View child = topmostChildList.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior b = lp.getBehavior();
        // 如果有一个behavior对事件进行了拦截，就发送Cancel事件给后续的所有Behavior。假设之前还没有Intercept发生，那么所有的事件都平等地对所有含有behavior的view进行分发，现在intercept忽然出现，那么相应的我们就要对除了Intercept的view发出Cancel
        if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
            // Cancel all behaviors beneath the one that intercepted.
            // If the event is "down" then we don't have anything to cancel yet.
            if (b != null) {
                if (cancelEvent == null) {
                    final long now = SystemClock.uptimeMillis();
                    cancelEvent = MotionEvent.obtain(now, now, MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                }
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        b.onInterceptTouchEvent(this, child, cancelEvent);
                        break;
                    case TYPE_ON_TOUCH:
                        b.onTouchEvent(this, child, cancelEvent);
                        break;
                }
            }
            continue;
        }

        if (!intercepted && b != null) {
            switch (type) {
                case TYPE_ON_INTERCEPT:
                    intercepted = b.onInterceptTouchEvent(this, child, ev);
                    break;
                case TYPE_ON_TOUCH:
                    intercepted = b.onTouchEvent(this, child, ev);  
                    break;
            }
            if (intercepted) {
                mBehaviorTouchView = child; //记录当前需要处理事件的View
            }
        }

        // Don't keep going if we're not allowing interaction below this.
        // Setting newBlock will make sure we cancel the rest of the behaviors.
        final boolean wasBlocking = lp.didBlockInteraction();
        final boolean isBlocking = lp.isBlockingInteractionBelow(this, child); //behaviors是否拦截事件
        newBlock = isBlocking && !wasBlocking;
        if (isBlocking && !newBlock) {
            // Stop here since we don't have anything more to cancel - we already did
            // when the behavior first started blocking things below this point.
            break;
        }
    }
    topmostChildList.clear();
    return intercepted;
}
```

### 小结

以上，基本可以理清 `CoordinatorLayout` 的机制，一个 View 如何监听到依赖 View 的变化，和 `CoordinatorLayout` 中的 `NestedScrollingChild` 实现 NestedScroll 的机制，触摸事件又是如何被 `Behavior` 拦截和处理，另外还有测量和布局我在这里并没有提及，但基本就是按照依赖关系排序，遍历子 View，询问它们的 `Behavior` 是否需要处理，大家可以翻翻源码，这样可以有更深刻的体会，有了这些知识，我们基本就可以根据需求来自定义自己的 `Behavior` 了，下面也带大家来实践下我是如何用自定义 `Behavior` 实现 UC 主页的

## UC 主页实现分析

先来看看 UC 浏览器的主页的效果图

![UC主页效果](https://raw.githubusercontent.com/BCsl/GoogleWidget/master/distribution/UcMainPager.gif)

可以看到有一共有4种元素的交互，这里分别称为 Title 元素、Header 元素、Tab 元素和新闻列表元素

在往上拖动列表页而还没进入到新闻阅读状态的时候，我们需要一个容器来完全消费掉这个拖动事件，避免列表项向上滚动，同时 Tab 和 Title 则分别从列表顶部和 `CoordinatorLayout` 顶部出现，Header 也有往上偏移一段距离，而到底谁来扮演这个角色呢？我们需要先确定它们之间的依赖关系

### 确定依赖关系

在编码之前，首先还需要确定这些元素的依赖关系，看下图来比较下前后的状态

![状态变化](https://raw.githubusercontent.com/BCsl/Accumulation/master/android/CoordinaryLayout/img/translation.png)

根据前后效果的对比图，我们可以使 Header 作为唯一被依赖的 View 来处理，列表容器和 Tab 容器随着 Header 上移动而上移动，Title 随着 Header 的上移动而下移出现，在这个完整的过程中，我们定义 Header 一共向上移动了 offestHeader 的高度，Title 向下偏移了 Title 这个容器的高度，Tab 则向上偏移了 Tab 这个容器的高度，而列表偏移的高度是 [offestHeader - Title容器高度 - Tab容器高度]

### 实现头部和列表的 NestedScroll 效果

首先考虑列表页，因为列表页可以左右切换，所以这里使用 ViewPager 作为列表页的容器，列表页需要放置在 Header 之下，且随着 Header 的上移收缩，列表页也需要上移，在这里我们首先需要解决两个问题

- 1.列表页置于 Header 之下
- 2.列表页上移留白问题

首先来解决第一个问题-列表页置于 Header 之下，`CoordinatorLayout` 继承来自 `ViewGroup`，默认的布局行为更像是一个 `FrameLayout`，不是 `RelativeLayout` 所以并不能用 `layout_below` 等属性来控制它的相对位置，而某些情况下，我们可以给 Header 的高度设定一个准确值，例如 250dip ，那么我们的的列表页的 marginTop 设置为 250dip 就好了，但是通常，我们的 Header 高度是不定的，所以我们需要一种能够适配这种变化的方法，所以我能想到的就是重写列表页的 layout 过程，`Behavior` 提供了 `onLayoutChild` 方法可以让我们实现，很好；接着来思考列表页上移留白问题，这是因为在 `CoordinatorLayout` 测量布局完成后，记此时列表高度为 H，但随着 Header 上移 H2 个高度的时候，列表也随着移动一定高度，但是列表高度还是 H，效果不言而喻，所以，我们需要在子 View 测量的时候，添加上列表的最大偏移量 [H2 - Title容器高度 - Tab容器高度]，下面来看代码，其实这就和系统 `AppBarLayout` 下的滚动列表处理一样的，我们会在 `AppBarLayout` 下放置的 View 设定一个这样的 `app:layout_behavior="@string/appbar_scrolling_view_behavior" Behavior 属性，所以提供已经提供了这样的一个基类来处理了，只不过它是包级私有，需要我们另外 copy 一份出来，来看看代码吧，继承自同样 sdk 提供的包级私有的`ViewOffsetBehavior`类，`ViewOffsetBehavior`使用`ViewOffsetHelper` 方便对 View 进行偏移处理，代码不多且功能也没使用到，所以就不贴了，可以自己看

```java
public abstract class HeaderScrollingViewBehavior extends ViewOffsetBehavior<View> {
    private final Rect mTempRect1 = new Rect();
    private final Rect mTempRect2 = new Rect();

    private int mVerticalLayoutGap = 0;
    private int mOverlayTop;

    public HeaderScrollingViewBehavior() {
    }

    public HeaderScrollingViewBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onMeasureChild(CoordinatorLayout parent, View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
        final int childLpHeight = child.getLayoutParams().height;
        if (childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT || childLpHeight == ViewGroup.LayoutParams.WRAP_CONTENT) {
            // If the menu's height is set to match_parent/wrap_content then measure it with the maximum visible height
            final List<View> dependencies = parent.getDependencies(child);
            final View header = findFirstDependency(dependencies);
            if (header != null) {
                if (ViewCompat.getFitsSystemWindows(header) && !ViewCompat.getFitsSystemWindows(child)) {
                    // If the header is fitting system windows then we need to also, otherwise we'll get CoL's compatible measuring
                    ViewCompat.setFitsSystemWindows(child, true);
                    if (ViewCompat.getFitsSystemWindows(child)) {
                        // If the set succeeded, trigger a new layout and return true
                        child.requestLayout();
                        return true;
                    }
                }
                if (ViewCompat.isLaidOut(header)) {
                    int availableHeight = View.MeasureSpec.getSize(parentHeightMeasureSpec);
                    if (availableHeight == 0) {
                        // If the measure spec doesn't specify a size, use the current height
                        availableHeight = parent.getHeight();
                    }
                    final int height = availableHeight - header.getMeasuredHeight() + getScrollRange(header);
                    final int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(height,
                            childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT ? View.MeasureSpec.EXACTLY : View.MeasureSpec.AT_MOST);

                    // Now measure the scrolling view with the correct height
                    parent.onMeasureChild(child, parentWidthMeasureSpec, widthUsed, heightMeasureSpec, heightUsed);

                    return true;
                }
            }
        }
        return false;
    }

    @Override
    protected void layoutChild(final CoordinatorLayout parent, final View child, final int layoutDirection) {
        final List<View> dependencies = parent.getDependencies(child);
        final View header = findFirstDependency(dependencies);

        if (header != null) {
            final CoordinatorLayout.LayoutParams lp = (CoordinatorLayout.LayoutParams) child.getLayoutParams();
            final Rect available = mTempRect1;
            available.set(parent.getPaddingLeft() + lp.leftMargin, header.getBottom() + lp.topMargin,
                    parent.getWidth() - parent.getPaddingRight() - lp.rightMargin,
                    parent.getHeight() + header.getBottom() - parent.getPaddingBottom() - lp.bottomMargin);

            final Rect out = mTempRect2;
            GravityCompat.apply(resolveGravity(lp.gravity), child.getMeasuredWidth(), child.getMeasuredHeight(), available, out, layoutDirection);

            final int overlap = getOverlapPixelsForOffset(header);

            child.layout(out.left, out.top - overlap, out.right, out.bottom - overlap);
            mVerticalLayoutGap = out.top - header.getBottom();
        } else {
            // If we don't have a dependency, let super handle it
            super.layoutChild(parent, child, layoutDirection);
            mVerticalLayoutGap = 0;
        }
    }

    float getOverlapRatioForOffset(final View header) {
        return 1f;
    }

    final int getOverlapPixelsForOffset(final View header) {
        return mOverlayTop == 0 ? 0 : MathUtils.constrain(Math.round(getOverlapRatioForOffset(header) * mOverlayTop), 0, mOverlayTop);

    }

    private static int resolveGravity(int gravity) {
        return gravity == Gravity.NO_GRAVITY ? GravityCompat.START | Gravity.TOP : gravity;
    }

    //需要子类来实现，从CoordinatorLayout中找到第一个child view依赖的View
    protected abstract View findFirstDependency(List<View> views);

    //返回Header可以收缩的范围，默认为Header高度，完全隐藏
    protected int getScrollRange(View v) {
        return v.getMeasuredHeight();
    }

    /**
     * The gap between the top of the scrolling view and the bottom of the header layout in pixels.
     */
    final int getVerticalLayoutGap() {
        return mVerticalLayoutGap;
    }

    /**
     * Set the distance that this view should overlap any {@link AppBarLayout}.
     *
     * @param overlayTop the distance in px
     */
    public final void setOverlayTop(int overlayTop) {
        mOverlayTop = overlayTop;
    }

    /**
     * Returns the distance that this view should overlap any {@link AppBarLayout}.
     */
    public final int getOverlayTop() {
        return mOverlayTop;
    }

}
```

这个基类的代码还是很好理解的，因为之前就说过了，正常来说被依赖的 View 会优先于依赖它的 View 处理，所以需要依赖的 View 可以在 measure/layout 的时候，找到依赖的 View 并获取到它的测量/布局的信息，然后根据依赖的 View 的信息来测量和对 child 进行布局，最后的效果就是把 Child 置于 Header 之下，且如果高度为 `WRAP_CONTENT`，高度就是沾满剩余的空间

我们的实现类，需要重写的除了抽象方法 `findFirstDependency` 外，还需要重写 `getScrollRange`，我们把 Header 的 Id `id_uc_news_header_pager` 定义在 `ids.xml` 资源文件内，方便依赖的判断；至于缩放的高度，根据 **结果图** 得知是 `Header高度 - Title高度 - Tab高度`，把 Title 高度 `uc_news_header_title_height` 和 Tab 视图的高度 `uc_news_tabs_height` 也定义在 `dimens.xml`，得出如下代码

```java
public class UcNewsContentBehavior extends HeaderScrollingViewBehavior {
    //省略构造信息
    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return isDependOn(dependency);
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        //省略，还未讲到
    }
    //通过ID判读，找到第一个依赖
    @Override
    protected View findFirstDependency(List<View> views) {
        for (int i = 0, z = views.size(); i < z; i++) {
            View view = views.get(i);
            if (isDependOn(view))
                return view;
        }
        return null;
    }

    @Override
    protected int getScrollRange(View v) {
        if (isDependOn(v)) {
            return Math.max(0, v.getMeasuredHeight() - getFinalHeight());
        } else {
            return super.getScrollRange(v);
        }
    }

    private int getFinalHeight() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_tabs_height) + DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_title_height);
    }
    //依赖的判断
    private boolean isDependOn(View dependency) {
        return dependency != null && dependency.getId() == R.id.id_uc_news_header_pager;
    }
}
```

好了，列表页初始状态完成了，接着列表页需要根据 Header 的上移而上移，上移使用 `TranslationY` 属性来控制即可，在 `dimens.xml` 中定义好 Header 的偏移范围值 `uc_news_header_pager_offset` ，当 Header 偏移了 `uc_news_header_pager_offset` 的时候，列表页的向上偏移值应该是 `getScrollRange()` 方法计算出的结果，那么，在接受到 `onDependentViewChanged` 的时候，列表页的 `TranslationY` 计算公式为：header.getTranslationY() / H(uc_news_header_pager_offset) * getScrollRange

列表页的`Behavior`最终代码如下：

```java
//列表页的Behavior
public class UcNewsContentBehavior extends HeaderScrollingViewBehavior {
    //...省略构造信息
    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return isDependOn(dependency);
    }
    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        offsetChildAsNeeded(parent, child, dependency);
        return false;
    }

    private void offsetChildAsNeeded(CoordinatorLayout parent, View child, View dependency) {
        child.setTranslationY((int) (-dependency.getTranslationY() / (getHeaderOffsetRange() * 1.0f) * getScrollRange(dependency)));

    }

    @Override
    protected View findFirstDependency(List<View> views) {
        for (int i = 0, z = views.size(); i < z; i++) {
            View view = views.get(i);
            if (isDependOn(view))
                return view;
        }
        return null;
    }

    @Override
    protected int getScrollRange(View v) {
        if (isDependOn(v)) {
            return Math.max(0, v.getMeasuredHeight() - getFinalHeight());
        } else {
            return super.getScrollRange(v);
        }
    }

    private int getHeaderOffsetRange() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_pager_offset);
    }

    private int getFinalHeight() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_tabs_height) + DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_title_height);
    }
    //依赖的判断
    private boolean isDependOn(View dependency) {
        return dependency != null && dependency.getId() == R.id.id_uc_news_header_pager;
    }
}
```

第一个难啃的骨头终于搞定，接着是来自 Header 的挑战

Header 的滚动事件来源于列表页中的 `NestedScrollingChild`，所以 Header 的 `Behavior` 需要重写于 NestedScroll 相关的方法，不仅仅需要拦截 Scroll 事件还需要拦截 Fling 事件，通过改变 `TranslationY` 值来"消费"掉这些事件，另外需要为该 `Behavior` 定义两种状态，打开和关闭，而如果在滑动中途手指离开（ ACTION_UP 或者 ACTION_CANCEL ），需要根据偏移量来判断进入打开还是关闭状态，这里我使用 Scroller + Runnalbe 来进行动画效果，因为直接使用 `ViewPropertyAnimator` 得到的结果不太理想，具体可以看代码的注释，就不细讲了

```java
public class UcNewsHeaderPagerBehavior extends ViewOffsetBehavior {
    private static final String TAG = "UcNewsHeaderPager";
    public static final int STATE_OPENED = 0;
    public static final int STATE_CLOSED = 1;
    public static final int DURATION_SHORT = 300;
    public static final int DURATION_LONG = 600;

    private int mCurState = STATE_OPENED;

    private OverScroller mOverScroller;

    //...省略构造信息

    private void init() { //构造器中调用
        mOverScroller = new OverScroller(DemoApplication.getAppContext());
    }

    @Override
    protected void layoutChild(CoordinatorLayout parent, View child, int layoutDirection) {
        super.layoutChild(parent, child, layoutDirection);
    }

    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
        //拦截垂直方向上的滚动事件且当前状态是打开的并且还可以继续向上收缩
        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0 && canScroll(child, 0) && !isClosed(child);
    }

    private boolean canScroll(View child, float pendingDy) {
        int pendingTranslationY = (int) (child.getTranslationY() - pendingDy);
        if (pendingTranslationY >= getHeaderOffsetRange() && pendingTranslationY <= 0) {
            return true;
        }
        return false;
    }

    @Override
    public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, View child, View target, float velocityX, float velocityY) {
        // consumed the flinging behavior until Closed
        return !isClosed(child);
    }

    private boolean isClosed(View child) {
        boolean isClosed = child.getTranslationY() == getHeaderOffsetRange();
        return isClosed;
    }

    public boolean isClosed() {
        return mCurState == STATE_CLOSED;
    }

    private void changeState(int newState) {
        if (mCurState != newState) {
            mCurState = newState;
        }
    }

    @Override
    public boolean onInterceptTouchEvent(CoordinatorLayout parent, final View child, MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_UP && !isClosed()) {
            handleActionUp(parent, child);
        }
        return super.onInterceptTouchEvent(parent, child, ev);
    }

    @Override
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dx, int dy, int[] consumed) {
        super.onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed);
        //dy>0 scroll up;dy<0,scroll down
        float halfOfDis = dy / 4.0f; //消费掉其中的4分之1，不至于滑动效果太灵敏
        if (!canScroll(child, halfOfDis)) {
            child.setTranslationY(halfOfDis > 0 ? getHeaderOffsetRange() : 0);
        } else {
            child.setTranslationY(child.getTranslationY() - halfOfDis);
        }
        //只要开始拦截，就需要把所有Scroll事件消费掉
        consumed[1] = dy;
    }

    //Header偏移量
    private int getHeaderOffsetRange() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_pager_offset);
    }


    private void handleActionUp(CoordinatorLayout parent, final View child) {
        if (mFlingRunnable != null) {
            child.removeCallbacks(mFlingRunnable);
            mFlingRunnable = null;
        }
        mFlingRunnable = new FlingRunnable(parent, child);
        if (child.getTranslationY() < getHeaderOffsetRange() / 3.0f) {
            mFlingRunnable.scrollToClosed(DURATION_SHORT);
        } else {
            mFlingRunnable.scrollToOpen(DURATION_SHORT);
        }

    }
    //结束动画的时候调用，并改变状态
    private void onFlingFinished(CoordinatorLayout coordinatorLayout, View layout) {
        changeState(isClosed(layout) ? STATE_CLOSED : STATE_OPENED);
    }


    private FlingRunnable mFlingRunnable;

    /**
     * For animation , Why not use {@link android.view.ViewPropertyAnimator } to play animation is of the
     * other {@link android.support.design.widget.CoordinatorLayout.Behavior} that depend on this could not receiving the correct result of
     * {@link View#getTranslationY()} after animation finished for whatever reason that i don't know
     */
    private class FlingRunnable implements Runnable {
        private final CoordinatorLayout mParent;
        private final View mLayout;

        FlingRunnable(CoordinatorLayout parent, View layout) {
            mParent = parent;
            mLayout = layout;
        }

        public void scrollToClosed(int duration) {
            float curTranslationY = ViewCompat.getTranslationY(mLayout);
            float dy = getHeaderOffsetRange() - curTranslationY;
            //这里做了些处理，避免有时候会有1-2Px的误差结果，导致最终效果不好
            mOverScroller.startScroll(0, Math.round(curTranslationY - 0.1f), 0, Math.round(dy + 0.1f), duration);
            start();
        }

        public void scrollToOpen(int duration) {
            float curTranslationY = ViewCompat.getTranslationY(mLayout);
            mOverScroller.startScroll(0, (int) curTranslationY, 0, (int) -curTranslationY, duration);
            start();
        }

        private void start() {
            if (mOverScroller.computeScrollOffset()) {
                ViewCompat.postOnAnimation(mLayout, mFlingRunnable);
            } else {
                onFlingFinished(mParent, mLayout);
            }
        }

        @Override
        public void run() {
            if (mLayout != null && mOverScroller != null) {
                if (mOverScroller.computeScrollOffset()) {
                    ViewCompat.setTranslationY(mLayout, mOverScroller.getCurrY());
                    ViewCompat.postOnAnimation(mLayout, this);
                } else {
                    onFlingFinished(mParent, mLayout);
                }
            }
        }
    }
}
```

### 实现标题视图和 Tab 视图跟随头部的实时移动

剩下 Title 和 Tab 的 Behavior ，相对上两个来说是比较简单的，都只需要子在 `onDependentViewChanged` 方法中，根据 Header 的变化而改变 `TranslationY` 值即可

Title 的 Behavior，为了简单，Title 直接设置 TopMargin 来使得初始状态完全偏移出父容器

```java
public class UcNewsTitleBehavior extends CoordinatorLayout.Behavior<View> {
    //...构造信息

    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, View child, int layoutDirection) {
        // FIXME: 16/7/27 不知道为啥在XML设置-45dip,解析出来的topMargin少了1个px,所以这里用代码设置一遍
        ((CoordinatorLayout.LayoutParams) child.getLayoutParams()).topMargin = -getTitleHeight();
        parent.onLayoutChild(child, layoutDirection);
        return true;
    }

    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return isDependOn(dependency);
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        offsetChildAsNeeded(parent, child, dependency);
        return false;
    }

    private void offsetChildAsNeeded(CoordinatorLayout parent, View child, View dependency) {
        int headerOffsetRange = getHeaderOffsetRange();
        int titleOffsetRange = getTitleHeight();
        if (dependency.getTranslationY() == headerOffsetRange) {
            child.setTranslationY(titleOffsetRange); //直接设置终值，避免出现误差
        } else if (dependency.getTranslationY() == 0) {
            child.setTranslationY(0); //直接设置初始值
        } else {
            //根据Header的TranslationY值来改变自身的TranslationY
            child.setTranslationY((int) (dependency.getTranslationY() / (headerOffsetRange * 1.0f) * titleOffsetRange));
        }
    }
    //Header偏移值
    private int getHeaderOffsetRange() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_pager_offset);
    }
    //标题高度
    private int getTitleHeight() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_title_height);
    }
    //依赖判断
    private boolean isDependOn(View dependency) {
        return dependency != null && dependency.getId() == R.id.id_uc_news_header_pager;
    }
}
```

Tab初始状态需要放置在Header之下，所以还是继承自`HeaderScrollingViewBehavior`，因为指定的高度，所以LayoutParams得Mode为EXACTLY，所以在测量的时候不会被特殊处理

```java
public class UcNewsTabBehavior extends HeaderScrollingViewBehavior {

    //..省略构造信息
    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return isDependOn(dependency);
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        offsetChildAsNeeded(parent, child, dependency);
        return false;
    }

    private void offsetChildAsNeeded(CoordinatorLayout parent, View child, View dependency) {
        float offsetRange = dependency.getTop() + getFinalHeight() - child.getTop();
        int headerOffsetRange = getHeaderOffsetRange();
        if (dependency.getTranslationY() == headerOffsetRange) {
            child.setTranslationY(offsetRange);  //直接设置终值，避免出现误差
        } else if (dependency.getTranslationY() == 0) {
            child.setTranslationY(0);//直接设置初始值
        } else {
            child.setTranslationY((int) (dependency.getTranslationY() / (getHeaderOffsetRange() * 1.0f) * offsetRange));
        }
    }

    @Override
    protected View findFirstDependency(List<View> views) {
        for (int i = 0, z = views.size(); i < z; i++) {
            View view = views.get(i);
            if (isDependOn(view))
                return view;
        }
        return null;
    }

    private int getHeaderOffsetRange() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_pager_offset);
    }

    private int getFinalHeight() {
        return DemoApplication.getAppContext().getResources().getDimensionPixelOffset(R.dimen.uc_news_header_title_height);
    }
    private boolean isDependOn(View dependency) {
        return dependency != null && dependency.getId() == R.id.id_uc_news_header_pager;
    }
}
```

最后布局代码就贴了，代码已经上传到 **[GITHUB](https://github.com/BCsl/UcMainPagerDemo)** ，可以上去看看且顺便给个 star 吧

## 写在最后

目前来说，Demo 还可以有更进一步的完善，例如在打开模式的情况下，禁止列表页 ViewPager 的左右滑动，且设置选中的 Pager 位置为 0 并列表滚动到第一个位置，每个列表还可以增加下拉刷新功能等...但是这些都和主题 `Behavior` 无关，所以就不再去实现了

如果你看完了文章且觉得有用，那么我希望你能顺手点个推荐/喜欢/收藏，写一篇用心的技术分享文章的确不容易（能抽这么多时间来写这篇文章，其实主要是因为这几天公寓断网了、网了、了...这一断就一星期,所以也拖延了发布时间）

## 那些有用的参考资料

- [UC浏览器首页滑动动画实现](http://ittiger.cn/2016/05/26/UC%E6%B5%8F%E8%A7%88%E5%99%A8%E9%A6%96%E9%A1%B5%E6%BB%91%E5%8A%A8%E5%8A%A8%E7%94%BB%E5%AE%9E%E7%8E%B0/)
- [Android NestedScrolling 实战](http://www.race604.com/android-nested-scrolling/)
- [拦截一切的CoordinatorLayout Behavior](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0224/3991.html)
- [SwipeDismissBehavior用法及实现原理](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1103/3650.html)
