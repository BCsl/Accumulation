# NestedScroll的一点分析

## 以CoordinaryLayout来分析

`CoordinaryLayout`实现了`NestedScrollingParent`接口，为使得`Behavior`类能封装简化处理嵌套滚动的操作，所以在这方面实际上`CoordinaryLayout`是`Behavior`的一个代理类

`Behavior`类关于嵌套滚动的方法如下

- onStartNestedScroll ： boolean
- onNestedScrollAccepted ： void
- onStopNestedScroll ： void
- onNestedScroll ： void
- onNestedPreScroll ： void
- onNestedFling ： void
- onNestedPreFling ： void

以上这些方法对应了`NestedScrollingParent`接口的方法，只是再参数上稍有不同，且都会在`CoordinaryLayout`实现`NestedScrollingParent`接口的每个方法中作出相应回调，

```java
public class CoordinatorLayout extends ViewGroup implements NestedScrollingParent {
  //.....

//CoordinaryLayout的成员变量
private final NestedScrollingParentHelper mNestedScrollingParentHelper = new NestedScrollingParentHelper(this);

   // 参数child:当前实现`NestedScrollingParent`的ViewParent包含触发嵌套滚动的直接子view对象
   // 参数target:触发嵌套滚动的view  (在这里如果不涉及多层嵌套的话,child和target)是相同的
   // 参数nestedScrollAxes:就是嵌套滚动的滚动方向了.垂直或水平方法
   //返回参数代表当前ViewParent是否可以触发嵌套滚动操作
   //CoordinaryLayout的实现上是交由子View的Behavior来决定，并回调了各个acceptNestedScroll方法，告诉它们处理结果
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
    //按照官方文档的指示，CoordinaryLayout有一个NestedScrollingParentHelper类型的成员变量，并把这个方法交由它处理
    //同样，这里也是需要CoordinaryLayout遍历子View，对可以嵌套滚动的子View回调Behavior#onNestedScrollAccepted方法
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
