# 从ScrollView中学习如何自定的可以滚动的View

- 触摸事件处理
- 学习如何自定的可以滚动的View
- OverScroll效果

## 初始化阶段

做一些自定义View常用的参数的初始化

```java
private void initScrollView() {
    mScroller = new OverScroller(getContext());
    setFocusable(true);
    setDescendantFocusability(FOCUS_AFTER_DESCENDANTS);
    setWillNotDraw(false);
    final ViewConfiguration configuration = ViewConfiguration.get(mContext);
    mTouchSlop = configuration.getScaledTouchSlop();
    mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
    mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
    mOverscrollDistance = configuration.getScaledOverscrollDistance();
    mOverflingDistance = configuration.getScaledOverflingDistance();
}
```

## 测量阶段

ScrollView在子View大于ScrollView的时候是可以垂直滑动的，所以它对子View的高度是没有限制的，这里可能会经历两次的测量

第一次的测量的信息一般会如下（只是单一的测试，但估计效果一样），测量模式会是0，也就是`MeasureSpec.UNSPECIFIED`，所以子View可以申请自己想要的大小

```
debugMeasure() called with: hideMode = [0], hideSize = [0]
```

是否进行第二次测量是根据`mFillViewport`参数来决定，默认为false，即不要求子View和ScrollView具有同样大小，以下则是测量的代码

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec); //第一次测量

    if (!mFillViewport) { //是否使Content填充ScrollView
        return;
    }

    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        return;
    }

    if (getChildCount() > 0) {
        final View child = getChildAt(0);
        final int height = getMeasuredHeight();
        if (child.getMeasuredHeight() < height) {
            final int widthPadding;
            final int heightPadding;
            final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
            if (targetSdkVersion >= VERSION_CODES.M) {
                widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
                heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
            } else {
                widthPadding = mPaddingLeft + mPaddingRight;
                heightPadding = mPaddingTop + mPaddingBottom;
            }

            final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec, widthPadding, lp.width);
            final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height - heightPadding, MeasureSpec.EXACTLY);
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

```java

@Override
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,  mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin+ widthUsed, lp.width);
    final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(MeasureSpec.getSize(parentHeightMeasureSpec), MeasureSpec.UNSPECIFIED);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

## 绘制阶段

是的ScrollView也有自己的绘制逻辑的，想想ScrollView当滑动到最顶端或者最底部的时候都会有一个指示效果，这个效果叫做`EdgeEffect`，这就不得不说，Android的View提供了三个与视图滚动到边缘相关的Mode参数，甚至可以用来指示实现仿照IOS弹性滚动的效果（只是Android上使用`EdgeEffect`效果来替代），但这里不细讲，可以看看[这篇文章](http://www.jianshu.com/p/834e522d02dc)，而在ScrollView中，只要参数不是`OVER_SCROLL_NEVER`,就可以带有`EdgeEffect`效果

```java
@Override
public void setOverScrollMode(int mode) {
    if (mode != OVER_SCROLL_NEVER) {
        if (mEdgeGlowTop == null) {
            Context context = getContext();
            mEdgeGlowTop = new EdgeEffect(context);
            mEdgeGlowBottom = new EdgeEffect(context);
        }
    } else {
        mEdgeGlowTop = null;
        mEdgeGlowBottom = null;
    }
    super.setOverScrollMode(mode);
}
```

ScrollView倒是没有实现`onDraw`方法，而是重写了`draw`方法，总之这里的绘制逻辑是和`EdgeEffect`相关，代码也不多，重点在于判断拖拉到边缘的处理

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    if (mEdgeGlowTop != null) {
        final int scrollY = mScrollY;
        final boolean clipToPadding = getClipToPadding();
        if (!mEdgeGlowTop.isFinished()) {
            final int restoreCount = canvas.save();
            final int width;
            final int height;
            final float translateX;
            final float translateY;
            if (clipToPadding) {
                width = getWidth() - mPaddingLeft - mPaddingRight;
                height = getHeight() - mPaddingTop - mPaddingBottom;
                translateX = mPaddingLeft;
                translateY = mPaddingTop;
            } else {
                width = getWidth();
                height = getHeight();
                translateX = 0;
                translateY = 0;
            }
            canvas.translate(translateX, Math.min(0, scrollY) + translateY);
            mEdgeGlowTop.setSize(width, height);
            if (mEdgeGlowTop.draw(canvas)) {
                postInvalidateOnAnimation();
            }
            canvas.restoreToCount(restoreCount);
        }
        if (!mEdgeGlowBottom.isFinished()) {
            final int restoreCount = canvas.save();
            final int width;
            final int height;
            final float translateX;
            final float translateY;
            if (clipToPadding) {
                width = getWidth() - mPaddingLeft - mPaddingRight;
                height = getHeight() - mPaddingTop - mPaddingBottom;
                translateX = mPaddingLeft;
                translateY = mPaddingTop;
            } else {
                width = getWidth();
                height = getHeight();
                translateX = 0;
                translateY = 0;
            }
            canvas.translate(-width + translateX,
                        Math.max(getScrollRange(), scrollY) + height + translateY);
            canvas.rotate(180, width, 0);
            mEdgeGlowBottom.setSize(width, height);
            if (mEdgeGlowBottom.draw(canvas)) {
                postInvalidateOnAnimation();
            }
            canvas.restoreToCount(restoreCount);
        }
    }
}
```

## 实现滑动

### 拦截事件

首先是在`onInterceptTouchEvent`做处理判断是否需要执行自己`onTouchEvent`，默认返回`false`。在`dispatchTouchEvent`方法内部调用，用来判断是会否拦截某个事件，如果当前`View`拦截了某个事，那么在同一个事件序列中，**此方法不会再次调用**，返回结果表示是否拦截某个事件（能调用的前提是`MotionEvent.ACTION_DOWN`事件，或者ViewGroup的`mFirstTouchTarget`不为NULL，就是说有子`View`对这次事件感兴趣）,如果不熟悉这个方法的话可以看看《Android开发艺术》或者自己看源码

这里一般的套路是：

- `ACTION_DOWN`做一些参数初始化的操作，一般并不拦截，即返回false，否在子View就不可能接收到触摸事件了（例如：点击，etc）
- `ACTION_MOVE`通过和`mTouchSlop`来比较判断是否需要自己处理，多点触摸可以先不考虑

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {

    final int action = ev.getAction();
    if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        return true;
    }
    /*
     * Don't try to intercept touch if we can't scroll anyway.
     */
    if (getScrollY() == 0 && !canScrollVertically(1)) {
        return false;
    }

    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_MOVE: { //通过和mTouchSlop比较来判断
            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                // If we don't have a valid id, the touch down wasn't on content.
                break;
            }

            final int pointerIndex = ev.findPointerIndex(activePointerId);
            if (pointerIndex == -1) {
                Log.e(TAG, "Invalid pointerId=" + activePointerId + " in onInterceptTouchEvent");
                break;
            }

            final int y = (int) ev.getY(pointerIndex);
            final int yDiff = Math.abs(y - mLastMotionY);
            if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                mIsBeingDragged = true; //需要拦截事件
                mLastMotionY = y;
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
                mNestedYOffset = 0;
                if (mScrollStrictSpan == null) {
                    mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
                }
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: { //不拦截Down，否则子View就没机会处理触摸事件
            final int y = (int) ev.getY();
            if (!inChild((int) ev.getX(), (int) y)) {
                mIsBeingDragged = false;
                recycleVelocityTracker();
                break;
            }
            /*
             * Remember location of down touch. ACTION_DOWN always refers to pointer index 0.
             */
            mLastMotionY = y;
            mActivePointerId = ev.getPointerId(0);

            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            /*
            * If being flinged and user touches the screen, initiate drag;
            * otherwise don't.  mScroller.isFinished should be false when
            * being flinged.
            */
            mIsBeingDragged = !mScroller.isFinished();
            if (mIsBeingDragged && mScrollStrictSpan == null) {
                mScrollStrictSpan = StrictMode.enterCriticalSpan("ScrollView-scroll");
            }
            startNestedScroll(SCROLL_AXIS_VERTICAL);
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            /* Release the drag */
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            recycleVelocityTracker();
            if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                postInvalidateOnAnimation();
            }
            stopNestedScroll();
            break;
        case MotionEvent.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;
    }

    /*
    * The only time we want to intercept motion events is if we are in the drag mode.
    */
    return mIsBeingDragged;
}
```

### 处理触摸事件

#### ACTION_DOWN和ACTION_MOVE

代码中会有有关Nested相关的代码，这些是5.0之后添加的，可以忽略

```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    initVelocityTrackerIfNotExists();

    MotionEvent vtev = MotionEvent.obtain(ev);

    final int actionMasked = ev.getActionMasked();

    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }
    vtev.offsetLocation(0, mNestedYOffset);

    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            if (getChildCount() == 0) {
                return false;
            }
            if ((mIsBeingDragged = !mScroller.isFinished())) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            if (!mScroller.isFinished()) {
                mScroller.abortAnimation();
                if (mFlingStrictSpan != null) {
                    mFlingStrictSpan.finish();
                    mFlingStrictSpan = null;
                }
            }
            // Remember where the motion event started
            mLastMotionY = (int) ev.getY();
            mActivePointerId = ev.getPointerId(0);
            //...
            break;
        }
        case MotionEvent.ACTION_MOVE:
            final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
            if (activePointerIndex == -1) {
                Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                break;
            }
            final int y = (int) ev.getY(activePointerIndex);
            int deltaY = mLastMotionY - y;
            //...
            if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
                mIsBeingDragged = true;
                if (deltaY > 0) {
                    deltaY -= mTouchSlop;
                } else {
                    deltaY += mTouchSlop;
                }
            }
            if (mIsBeingDragged) {
                // Scroll to follow the motion event
                mLastMotionY = y - mScrollOffset[1];

                final int oldY = mScrollY;
                final int range = getScrollRange(); //Child滚动的范围，Child高度减去ScrollView高度和内边距
                final int overscrollMode = getOverScrollMode(); //决定了是否有拉拽到边缘时的效果
                boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||(overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                // Calling overScrollBy will call onOverScrolled, which
                // calls onScrollChanged if applicable.
                if (overScrollBy(0, deltaY, 0, mScrollY, 0, range, 0, mOverscrollDistance, true) && !hasNestedScrollingParent(){  //关键点在于overScrollBy，处理了滚动事件
                    // Break our velocity if we hit a scroll barrier.
                    mVelocityTracker.clear();
                }

                final int scrolledDeltaY = mScrollY - oldY;
                final int unconsumedY = deltaY - scrolledDeltaY;
                if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) { //忽略
                    mLastMotionY -= mScrollOffset[1];
                    vtev.offsetLocation(0, mScrollOffset[1]);
                    mNestedYOffset += mScrollOffset[1];
                } else if (canOverscroll) { //EdgeEffect处理
                    final int pulledToY = oldY + deltaY;
                    if (pulledToY < 0) {
                        mEdgeGlowTop.onPull((float) deltaY / getHeight(),ev.getX(activePointerIndex) / getWidth());
                        if (!mEdgeGlowBottom.isFinished()) {
                            mEdgeGlowBottom.onRelease();
                        }
                    } else if (pulledToY > range) {
                        mEdgeGlowBottom.onPull((float) deltaY / getHeight(),1.f - ev.getX(activePointerIndex) / getWidth());
                        if (!mEdgeGlowTop.isFinished()) {
                            mEdgeGlowTop.onRelease();
                        }
                    }
                    if (mEdgeGlowTop != null && (!mEdgeGlowTop.isFinished() || !mEdgeGlowBottom.isFinished())) {
                        postInvalidateOnAnimation();
                    }
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            //...
            break;
        case MotionEvent.ACTION_CANCEL:
            //...
            break;
        case MotionEvent.ACTION_POINTER_DOWN: {
            //...
            break;
        }
        case MotionEvent.ACTION_POINTER_UP:
            //...
            break;
    }

    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(vtev);
    }
    vtev.recycle();
    return true;
}
```

使子View滚动的关键处理在`overScrollBy`方法，这个方法用来判断是否到达控件边缘，这是View类中的方法，这个方法和`scrollBy`、`scrollTo`看着有点像，`scrollBy`、`scrollTo`都是平稳地移动到目的位置，都是使用`Scroller`类来实现的，但这里`overScrollBy`也是用用`Scroller`来处理的，具体在`onOverScrolled`方法，但这个方法在View类中是空实现，需要子类来实现，`ScrollView`就实现了这个方法

```java
View.java
/**
* @param deltaX Change in X in pixels x轴方向上的改变
* @param deltaY Change in Y in pixels  y轴方向上的改变
* @param scrollX Current X scroll value in pixels before applying deltaX 当前的scrollX
* @param scrollY Current Y scroll value in pixels before applying deltaY 当前的scrollY
* @param scrollRangeX Maximum content scroll range along the X axis X轴方向上的滚动范围
* @param scrollRangeY Maximum content scroll range along the Y axis  Y轴方向上的滚动范围
* @param maxOverScrollX Number of pixels to overscroll by in either direction along the X axis. X轴方向上最大的反弹距离
* @param maxOverScrollY Number of pixels to overscroll by in either direction along the Y axis. X轴方向上最大的反弹距离
* @param isTouchEvent true if this scroll operation is the result of a touch event.
* @return true if scrolling was clamped to an over-scroll boundary along either axis, false otherwise.
*/
protected boolean overScrollBy(int deltaX, int deltaY, int scrollX, int scrollY, int scrollRangeX, int scrollRangeY,  int maxOverScrollX, int maxOverScrollY,boolean isTouchEvent) {
    final int overScrollMode = mOverScrollMode;
    final boolean canScrollHorizontal = computeHorizontalScrollRange() > computeHorizontalScrollExtent();
    final boolean canScrollVertical =  computeVerticalScrollRange() > computeVerticalScrollExtent();

    final boolean overScrollHorizontal = overScrollMode == OVER_SCROLL_ALWAYS || (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollHorizontal);

    final boolean overScrollVertical = overScrollMode == OVER_SCROLL_ALWAYS || (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollVertical);

    int newScrollX = scrollX + deltaX;
    if (!overScrollHorizontal) {
        maxOverScrollX = 0;
    }

    int newScrollY = scrollY + deltaY;
    if (!overScrollVertical) {
        maxOverScrollY = 0;
    }

    // Clamp values if at the limits and record
    final int left = -maxOverScrollX; //左边缘
    final int right = maxOverScrollX + scrollRangeX; //右边缘
    final int top = -maxOverScrollY;
    final int bottom = maxOverScrollY + scrollRangeY;

    boolean clampedX = false;
    if (newScrollX > right) { //到达右边缘
        newScrollX = right;
        clampedX = true;
    } else if (newScrollX < left) { //到达左边缘
        newScrollX = left;
        clampedX = true;
    }

    boolean clampedY = false;
    if (newScrollY > bottom) {  //到达底部边缘
        newScrollY = bottom;
        clampedY = true;
    } else if (newScrollY < top) {  //到达顶部边缘
        newScrollY = top;
        clampedY = true;
    }

    onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

    return clampedX || clampedY;
}
```

ScrollView处理OverScroll，clampedX和clampedY代表了是否是OverScroll效果，代码中使用到`OverScroller#springBack`，这个方法平时还是用得少，回弹效果？

```java
@Override
protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY) {
    // Treat animating scrolls differently; see #computeScroll() for why.
    if (!mScroller.isFinished()) {
        final int oldX = mScrollX;
        final int oldY = mScrollY;
        mScrollX = scrollX;
        mScrollY = scrollY;
        invalidateParentIfNeeded();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (clampedY) {
            mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange()); //回弹？
        }
    } else {
        super.scrollTo(scrollX, scrollY);
    }
    awakenScrollBars();
}
```

看看官方对`OverScroller#springBack`这个方法的作用的描述，当你需要实现回弹效果的时候使用该方法，startX为开始坐标值，最终X值会落在maxX和minX值中最靠近StartX的那个上

```java
/**
 * Call this when you want to 'spring back' into a valid coordinate range.
 *
 * @param startX Starting X coordinate X坐标初始值
 * @param startY Starting Y coordinate
 * @param minX Minimum valid X value   X坐标的结果的最小值
 * @param maxX Maximum valid X value   X坐标的结果的最大值
 * @param minY Minimum valid Y value  
 * @param maxY Minimum valid Y value  
 * @return true if a springback was initiated, false if startX and startY were already within the valid range.
 */
public boolean springBack(int startX, int startY, int minX, int maxX, int minY, int maxY) {
  //...
}
```

使用`Scroller`一般还需要配合`computeScroll`使用，在还没有滚动完的时候还是会继续调用`overScrollBy`方法来处理

```java
@Override
public void computeScroll() {
    if (mScroller.computeScrollOffset()) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        int x = mScroller.getCurrX();
        int y = mScroller.getCurrY();

        if (oldX != x || oldY != y) {
            final int range = getScrollRange();
            final int overscrollMode = getOverScrollMode();
            final boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS || (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

            overScrollBy(x - oldX, y - oldY, oldX, oldY, 0, range, 0, mOverflingDistance, false);
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);

            if (canOverscroll) {
                if (y < 0 && oldY >= 0) {
                    mEdgeGlowTop.onAbsorb((int) mScroller.getCurrVelocity());
                } else if (y > range && oldY <= range) {
                    mEdgeGlowBottom.onAbsorb((int) mScroller.getCurrVelocity());
                }
            }
        }

        if (!awakenScrollBars()) {
            // Keep on drawing until the animation has finished.
            postInvalidateOnAnimation();
        }
    } else {
        if (mFlingStrictSpan != null) {
            mFlingStrictSpan.finish();
            mFlingStrictSpan = null;
        }
    }
}
```

#### ACTION_UP和ACTION_CANCEL

```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    //..
    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            //...
            break;
        }
        case MotionEvent.ACTION_MOVE:
            //...
            break;
        case MotionEvent.ACTION_UP:
            if (mIsBeingDragged) {
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);

                if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                    flingWithNestedDispatch(-initialVelocity);
                } else if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0,
                        getScrollRange())) {
                    postInvalidateOnAnimation();
                }

                mActivePointerId = INVALID_POINTER;
                endDrag();
            }
            break;
        case MotionEvent.ACTION_CANCEL:
            if (mIsBeingDragged && getChildCount() > 0) {
                if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                    postInvalidateOnAnimation();
                }
                mActivePointerId = INVALID_POINTER;
                endDrag();
            }
            break;
        case MotionEvent.ACTION_POINTER_DOWN: {
            final int index = ev.getActionIndex();
            mLastMotionY = (int) ev.getY(index);
            mActivePointerId = ev.getPointerId(index);
            break;
        }
        case MotionEvent.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
            break;
    }

    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(vtev);
    }
    vtev.recycle();
    return true;
}
```

## 小结

- 在出来子View按照手指移动的时候使用的是`OverScroller#springBack`，为了可以实现OverScroll效果，如果只是普通的移动則可以使用`offsetLeftAndRight`或`offsetTopAndBottom`方法
- ViewGroup在拦截阶段，`ACTION_DOWN`做一些参数初始化的操作，一般并不拦截，即返回false，否在子View就不可能接收到触摸事件了，在`ACTION_MOVE`通过和`mTouchSlop`来比较判断是否需要拦截
