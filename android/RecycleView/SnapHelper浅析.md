# LinearSnapHelper

实现ItemView居中，类似`ViewPager`效果，但并没有居中选中的回调

## SnapHelper 简单介绍

```java
abstract class SnapHelper {
      //targetView需要在水平和垂直方向上的移动多少来达到预期效果（居中，居右等）
      public abstract int[] calculateDistanceToFinalSnap(@NonNull LayoutManager layoutManager, @NonNull View targetView);
      //找到需要进行Snap的ItemView
      public abstract View findSnapView(LayoutManager layoutManager);
      //根据滑动速度，找到目标ItemView的在Adapter中的位置
      public abstract int findTargetSnapPosition(LayoutManager layoutManager, int velocityX,int velocityY);

}
```

## 简单使用

```java
LinearLayoutManager layoutManager1 = new LinearLayoutManager(this);
  layoutManager1.setOrientation(LinearLayoutManager.HORIZONTAL);
  mMainRecycle1.setLayoutManager(layoutManager1);
  DemoAdapter demoAdapter1 = new DemoAdapter(title);
  mMainRecycle1.setAdapter(demoAdapter1);
  LinearSnapHelper snapHelper1 = new LinearSnapHelper();
```

## 实现原理

为`RecycleView`实现两个`Listener`，`OnScrollListener`,`onFlingListener`

在`RecycleView`滚动静止后，通过遍历子View找到距离中心最近的`ItemView`,计算`ItemView`和中心的距离，滚动

静止后居中：

```java
SnapHelper.java

void snapToTargetExistingView() {
    if (mRecyclerView == null) {
        return;
    }
    LayoutManager layoutManager = mRecyclerView.getLayoutManager();
    if (layoutManager == null) {
        return;
    }
    View snapView = findSnapView(layoutManager);  //抽象方法
    if (snapView == null) {
        return;
    }
    int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView); //抽象方法
    if (snapDistance[0] != 0 || snapDistance[1] != 0) {
        mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
    }
}
```

滑动居中：

```java
@Override
public boolean onFling(int velocityX, int velocityY) {
    //....
    return (Math.abs(velocityY) > minFlingVelocity || Math.abs(velocityX) > minFlingVelocity)
            && snapFromFling(layoutManager, velocityX, velocityY);
}


private boolean snapFromFling(@NonNull LayoutManager layoutManager, int velocityX, int velocityY) {
    //...
    RecyclerView.SmoothScroller smoothScroller = createSnapScroller(layoutManager);
    //..
    int targetPosition = findTargetSnapPosition(layoutManager, velocityX, velocityY); //抽象方法
    //...
    smoothScroller.setTargetPosition(targetPosition);
    layoutManager.startSmoothScroll(smoothScroller);
    return true;
}
```

可以看到`RecycleView`的居中滑动通过`smoothScrollBy`和`startSmoothScroll`方法来完成，更多细节可以自己查看源码
