# 自定义Behavior的艺术探索-仿UC浏览器主页

## 前言&效果预览

最近几个周末基本在研究CoordinatorLayout控件和自定义Behavior当中，这期间看了不少这方面的知识，有关于CoordinatorLayout使用的文章，CoordinatorLayout的源码分析文章等等，轻轻松松入门虽然简单，无耐于网上介绍的一些例子实在是太简单，很多东西都是草草带过，尤其是关于NestedScroll效果这方面的，最后发现自己到头来其实还是一头雾水，当然，自己在周末的时候效率的确不高，干扰因素也多。但无意中发现了一篇通过自定义View的方式实现的仿UC浏览器主页的文章，顿时有了使用自定义Behavior实现这样的效果的想法，而且这种方式在我看来应该会更简单，这过程看了很多这方面的源码CoordinatorLayout、NestedScrollView、SwipeDismissBehavior、FloatingActionButton.Behavior、AppBarLayout.Behavior等，于是有了今天的这篇文章。忆当年，自己也曾经在UC浏览器实习过大半年的时间，UC也是自己一直除了QQ从塞班时代至今一直使用的APP了，只怪自己当时有点作死。。。。咳咳，扯多了，还是直接来看效果吧，因为文章比较长，不先放个效果图，估计没多少人能耐心看完。

## 揭开NestedScroll的原理

网上不少写文章写到自定义Behavior的实现方式有两种形式，其中一种是实现NestedScrolling效果的，需要关注重写`onStartNestedScroll`和`onNestedPreScroll`等一系列带`Nested`字段的方法，当你一看这样的一个方法`onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed)`是有多少个参数的时候，你通常会**一脸懵逼图**，就算你搞懂了这里的每个参数的意思，你还是会有所疑问，这样的一大堆方法是在什么时候调用的，这个时候，你首先需要弄懂的是Android5.0开始提供一套支持嵌套滑动效果的机制

NestedScrolling提供了一套父View和子View滑动交互机制。要完成这样的交互，父View需要实现`NestedScrollingParent`接口，而子View需要实现`NestedScrollingChild`接口，系统提供的`NestedScrollView`控件就实现了这两个接口，千万不要被这两个接口这么多的方法唬住了，这两个接口都有一个`NestedScrolling[Parent,Children]Helper`辅助类来帮助处理的大部分逻辑，它们之间关系如下，

### 实现NestedScrollingChild接口

实际上`NestedScrollingChildHelper`辅助类已经实现好了Child和Parent交互的逻辑。原来的View的处理Touch事件，并实现滑动的逻辑大体上不需要改变。

需要做的就是，如果要准备开始滑动了，需要告诉Parent，Child要准备进入滑动状态了，调用`startNestedScroll()`。Child在滑动之前，先问一下你的Parent是否需要滑动，也就是调用 `dispatchNestedPreScroll()`。如果父类消耗了部分滑动事件，Child需要重新计算一下父类消耗后剩下给Child的滑动距离余量。然后，Child自己进行余下的滑动。最后，如果滑动距离还有剩余，Child就再问一下，Parent是否需要在继续滑动你剩下的距离，也就是调用 `dispatchNestedScroll()`，大概就是这么一回事，当然还还会有和`scroll`类似的`fling`系列方法，但我们这里可以先忽略一下

来看看`NestedScrollView`的`NestedScrollingChild`接口实现都是交给辅助类`NestedScrollingChildHelper`来处理的，是否需要进行额外的一些操作要根据实际情况来定

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

//在初始化滚动操作的时候调用，一般在MotionEvent#ACTION_DOWN的时候调用
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

#### startNestedScroll和stopNestedScroll的调用

一般配合`stopNestedScroll`使用，`startNestedScroll`会再接收到`ACTION_DOWN`的时候调用，`stopNestedScroll`会在接收到`ACTION_UP|ACTION_CANCEL`的时候调用 伪代码应该是这样

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

`NestedScrollingChildHelper`处理`startNestedScroll`方法，可以看出可能会调用Parent的`onStartNestedScroll`和`onNestedScrollAccepted`方法，只要Parent愿意优先处理这次的滑动事件，在介绍的时候Parent还会收到`onStopNestedScroll`回调

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

#### dispatchNestedPreScroll的调用

在消费滚动事件之前调用，提供一个让Parent实现联合滚动的机会，因此Parent可以消费一部分或者全部的滑动事件，注意参数`consumed`会记录了Parent所消费掉的事件

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

`NestedScrollingChildHelper`处理`dispatchNestedPreScroll`方法，会调用到上一步里记录的希望优先处理Scroll事件的Parent的`onNestedPreScroll`方法

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

#### dispatchNestedScroll的调用

这个方法是在Child自己消费完Scroll事件后调用的

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

`NestedScrollingChildHelper`处理`dispatchNestedScroll`方法，会调用到上一步里记录的希望优先处理Scroll事件的Parent的`onNestedScroll`方法

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

### 实现NestedScrollingParent接口

同样，也有一个`NestedScrollingParentHelper`辅助类来帮助Parent实现和Child交互的逻辑。**滑动动作是Child主动发起**，Parent就受滑动回调并作出响应。从上面的Child分析可知，滑动开始的调用`startNestedScroll()`，Parent收到 `onStartNestedScroll()`回调，决定是否需要配合Child一起进行处理滑动，如果需要配合，还会回调`onNestedScrollAccepted()`

每次滑动前，Child先询问Parent是否需要滑动，即`dispatchNestedPreScroll()`，这就回调到Parent的`onNestedPreScroll()`，Parent可以在这个回调中消费掉Child的Scroll事件，也就是优先于Child滑动

Child滑动以后，会调用`dispatchNestedScroll()`，回调到Parent的`onNestedScroll()`，这里就是Child滑动后，剩下的给Parent处理，也就是后于Child滑动

最后，滑动结束Child调用`stopNestedScroll`，回调Parent的`onStopNestedScroll()`表示本次处理结束

现在我们来看看`NestedScrollingParent`的实现细节，这里以`CoordinatorLayout`来分析而不再是`NestedScrollView`，因为它才是这篇文章的主角

在这之前，首先简单介绍下`Behavior`这个对象，你可以在XML中定义它就会在`CoordinaryLayout`中解析实例化到目标子View的`LayoutParams`或者获取到`CoordinaryLayout`子View的`LayoutParams`对象通过setter方法注入，如果你自定义的`Behavior`希望实现NestedScroll效果，那么你需要关注重写以下这些方法

- onStartNestedScroll ： boolean
- onStopNestedScroll ： void
- onNestedScroll ： void
- onNestedPreScroll ： void
- onNestedFling ： void
- onNestedPreFling ： void

你会发现以上这些方法对应了`NestedScrollingParent`接口的方法，只是在参数上有所增加，且都会在`CoordiantorLayout`实现`NestedScrollingParent`接口的每个方法中作出相应回调，下面来简单走读下这部分代码

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

你会发现`CoordiantorLayout`收到来自`NestedScrollingChild`的各种回调后，都是交由需要响应的`Behavior`来处理的，所以这里可以得出一个结论，`CoordiantorLayout`是`Behavior`的一个代理类，所以`Behavior`实际上也是一个`NestedScrollingParent`，另外结合`NestedScrollingChild`实现的部分来看，你很容就能搞懂这些方法参数的实际含义

### NestedScroll小结

NestedScroll的机制的简版是这样的，当子View在处理滑动事件之前，先告诉自己的父View是否需要先处理这次滑动事件，父View处理完之后，告诉子View它处理的多少滑动距离，剩下的还是交给子View自己来处理

你也可以实现这样的一套机制，父View拦截所有事件，然后分发给需要的子View来处理，然后剩余的自己来处理。但是这样就做会使得逻辑处理更复杂，因为事件的传递本来就由外先内传递到子View，处理机制是由内向外，由子View先来处理事件本来就是遵守默认规则的，这样更自然且坑更少，不知道自己说得对不对，欢迎打脸(￣ε(#￣)☆╰╮(￣▽￣///)

## CoordinatorLayout的源码走读和如何自定义Behavior

上面在分析`NestedScrollingParent`接口的时候已经简单提到了`CoordinatorLayout`这个控件，至于这个控件是用来做什么的？`CoordinatorLayout`内部有个`Behavior`对象，这个`Behavior`对象可以通过外部setter或者在xml中指定的方式注入到`CoordinatorLayout`的某个子View的`LayoutParams`，`Behavior`对象定义了特定类型的视图交互逻辑，譬如`FloatingActionButton`的`Behavior`实现类，只要`FloatingActionButton`是`CoordinatorLayout`的子View，且设置的该`Behavior`（默认已经设置了），那么，这个FAB就会在`Snackbar`出现的时候上浮，而不至于被遮挡，而这种通过定义`Behavior`的方式就可以控制View的某一类的行为，通常会比自定义View的方式更解耦更轻便，由此可知，`Behavior`是`CoordinatorLayout`的精髓所在

### Behavior的解析和实例化

简单来看看`Behavior`是如何从xml中解析的，通过检测`xxx:behavior`属性，通过全限定名或者相对路径的形式指定路径，最后通过反射来新建实例，默认的构造器是`Behavior(Context context, AttributeSet attrs)`,如果你需要配置额外的参数，可以在外部构造好`Behavior`并通过setter的方式注入到`LayoutParams`或者获取到解析好的`Behavior`进行额外的参数设定

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

#### View之间的依赖关系

`CoordinatorLayout`的子View可以扮演着不同角色，一种是被依赖的，而另外一种则是主动寻找依赖的View，被依赖的View并不会感知到自己被依赖，被依赖的View也有可能是寻找依赖的View

这种依赖关系的建立由`CoordinatorLayout#LayoutParam`来指定，假设此时有两个View: A 和B，那么有两种情况会导致依赖关系

- A的anchor是B
- A的behavior对B有依赖

`LayoutParams`中关于依赖的判断的依据的代码如下

```java
LayoutParams.class

boolean dependsOn(CoordinatorLayout parent, View child, View dependency) {
    return dependency == mAnchorDirectChild|| (mBehavior != null && mBehavior.layoutDependsOn(parent, child, dependency));
}
```

依赖判断通过两个条件判断，一个生效即可，最容易理解的是根据`Behavior#layoutDependsOn`方法指定，例如FAB依赖`Snackbar`

```java
Behavior.java

@Override
public boolean layoutDependsOn(CoordinatorLayout parent, FloatingActionButton child, View dependency) {
    return Build.VERSION.SDK_INT >= 11 && dependency instanceof Snackbar.SnackbarLayout;
}
```

另外一个可以看到是通过`mAnchorDirectChild`来判断，首先要知道AnchorView的ID是通过setter或者xml的anchor属性形式指定，但是为了不需要每次都根据ID通过`findViewById`去解析出AnchorView，所以会使用`mAnchorView`变量缓存好，需要注意的是这个AnchorView不可以是`CoordinatorLayout`，另外也不可以是当前View的一个子View，变量`mAnchorDirectChild`记录的就是AnchorView的所属的`ViewGroup`或自身（当它直接ViewParent是`CoordinatorLayout`的时候），关于AnchorView的作用，也可以在FAB配合`AppBarLayout`使用的时候，`AppBarLayout`会作为FAB的AnchorView，就可以在`AppBarLayout`打开或者收缩状态的时候显示或者隐藏FAB，自己这方面的实践比较少，在这也可以先忽略并不影响后续分析，大家感兴趣的可以通过看相关代码一探究竟

根据这种依赖关系，`CoordinatorLayout`中维护了一个`mDependencySortedChildren`列表，里面含有所有的子View，按依赖关系排序，被依赖者排在前面，会在每次测量前重新排序，确保处理的顺序是 **被依赖的View会先被measure和layout**

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

`selectionSort`方法使用的就是`mLayoutDependencyComparator`来处理，list参数是所有子View的集合，这里使用了_选择排序法_，递增的方式，所以最后被依赖的View会排在最前

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

这里有个疑问？为什么不直接使用`Collections#sort(List<T> list, Comparator<? super T> comparator)`的方式来排序呢？我的想法是考虑到可能会出现这样的一种情况，A依赖B，B依赖C，C依赖A，这时候Comparator比较的时候，A>B，B>C，C>A，这就违背了Comparator所要求的传递性（根据传递性原则，A应该大于C），所以没有使用`sort`方法来排序，不知道自己说得是否正确，有知道的一定要告诉我，经过选择排序的方式的结果是[C，B，A]，所以虽然C依赖A，但也可能先处理了C，这就如果你使用到这样的依赖关系的时候就需要谨慎且注意了，例如你在C处理`onMeasureChild`的时候，你并不能得到C依赖的A的测量结果，因为C先于A处理了

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

### 自定义Behavior的两种目的

我们可以按照两种目的来实现自己的`Behavior`，当然也可以两种都实现啦

- 某个view监听另一个view的状态变化，例如大小、位置、显示状态等

- 某个view监听`CoordinatorLayout`内的`NestedScrollingChild`的接口实现类的滑动状态

第一种情况需要重写`layoutDependsOn`和`onDependentViewChanged`方法

第二种情况需要重写`onStartNestedScroll`和`onNestedPreScroll`系列方法（上面已经提到了哦）

对于第一种情况，我们之前分析依赖的监听的时候相关回调细节已经说完了，`Behavior`只需要在`onDependentViewChanged`做相应的处理就好

对于第二种情况，我们在NestedScoll的那节也已经把相关回调细节说了

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
        final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
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

## UC主页实现分析

### 确定依赖关系

### 说得再多也不如直接写代码

## 那些有用的参考资料
