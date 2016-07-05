# 从ScrollView中学习如何自定的可以滚动的View

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
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

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
- `ACTION_MOVE`通过和`mTouchSlop`来比较判断是否需要自己处理，多点触摸一般可以不考虑的

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
            /*
             * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
             * whether the user has moved far enough from his original down touch.
             */
            /*
            * Locally do absolute value. mLastMotionY is set to the y value of the down event.
            */
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
                mIsBeingDragged = true;
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

        case MotionEvent.ACTION_DOWN: { //正常情况下，都不拦截
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
