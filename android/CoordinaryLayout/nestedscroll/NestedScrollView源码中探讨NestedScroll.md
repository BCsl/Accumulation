# NestedScrollView源码中探讨NestedScroll

## 实现NestedScrollingChild

实际上`NestedScrollingChildHelper`辅助类已经实现好了Child和 Parent交互的逻辑。原来的View的处理Touch事件，并实现滑动的逻辑大体上不需要改变。

需要做的就是，如果要准备开始滑动了，需要告诉Parent，你要准备进入滑动状态了（`MotionEvent#ACTION_DOWN`），调用`startNestedScroll()`。你在滑动之前，先问一下你的 Parent 是否需要滑动，也就是调用 `dispatchNestedPreScroll()`。如果父类滑动了一定距离，你需要重新计算一下父类滑动后剩下给你的滑动距离余量。然后，你自己进行余下的滑动。最后，如果滑动距离还有剩余，你就再问一下，Parent是否需要在继续滑动你剩下的距离，也就是调用 `dispatchNestedScroll()`。

来看看`NestedScrollView`是如何实现`NestedScrollingChild`接口的

```java
// NestedScrollingChild

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

### startNestedScroll的调用

一般配合`stopNestedScroll`使用，`startNestedScroll`会再接收到`ACTION_DOWN`的时候调用，接收到`ACTION_UP|ACTION_CANCEL`的时候调用 伪代码应该是这样

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

### dispatchNestedPreScroll的调用

在消费滚动事件之前调用，提供一个让ViewParent实现联合滚动的机会，因此ViewParent可以消费一部分或者全部的滑动事件，参数`consumed`会记录了ViewParent所消费掉的事件

```java
onTouchEvent (MotionEvent ev){
    //...
   case MotionEvent.ACTION_MOVE:
   //...
     final int y = (int) MotionEventCompat.getY(ev, activePointerIndex);
     int deltaY = mLastMotionY - y; //计算偏移量
     if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
       deltaY -= mScrollConsumed[1]; //减去被消费掉的事件
       vtev.offsetLocation(0, mScrollOffset[1]); //重新调整事件的位置
       mNestedYOffset += mScrollOffset[1];
   }
   //...
   break;
}
```

### dispatchNestedScroll的调用

```java
onTouchEvent (MotionEvent ev){
    //...
   case MotionEvent.ACTION_MOVE:
   //...
   final int scrolledDeltaY = getScrollY() - oldY; //计算这个View消费掉的事件
   final int unconsumedY = deltaY - scrolledDeltaY; //计算的是这个Vdew没有消费掉的事件
   if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
       mLastMotionY -= mScrollOffset[1];
       vtev.offsetLocation(0, mScrollOffset[1]);//重新调整事件的位置
       mNestedYOffset += mScrollOffset[1];
   }
   //...
   break;
}
```

## 实现NestedScrollingChild

同样，也有一个`NestedScrollingParentHelper`辅助类来帮助你实现和Child交互的逻辑。滑动动作是Child主动发起，Parent就受滑动回调并作出响应。从上面的Child分析可知，滑动开始的调用`startNestedScroll()`，Parent收到 `onStartNestedScroll()`回调，决定是否需要配合Child一起进行处理滑动，如果需要配合，还会回调`onNestedScrollAccepted()`

每次滑动前，Child先询问Parent是否需要滑动，即`dispatchNestedPreScroll()`，这就回调到Parent的`onNestedPreScroll()`，Parent 可以在这个回调中"劫持"掉Child的滑动，也就是先于Child滑动

Child滑动以后，会调用`dispatchNestedScroll()`，回调到Parent的`onNestedScroll()`，这里就是Child滑动后，剩下的给Parent处理，也就是后于Child滑动

最后，滑动结束Child调用`stopNestedScroll`，回调Parent的`onStopNestedScroll()`表示本次处理结束
