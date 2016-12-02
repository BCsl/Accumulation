# Custom LayoutManager

[Refer](http://wiresareobsolete.com/2014/09/building-a-recyclerview-layoutmanager-part-1/)

## Recycler

简单的说，`LayoutManager`这个对象不需要关注子View的创建和复用，这部分只要交给`Recycler`对象就可以了，当需要一个子View的时候，你只需要调用`Recycler`对象的`getViewForPosition()`方法即可，另外还需要把不再可见的子View交给`Recycler`对象回收处理。

## Detach vs. Remove

RecyclerView更新的时候有两种方式来更新子View，detach和remove。Detach是一个记录视图的轻量的操作，被detached子视图一般预计会在同一次layout操作中重新的attached到RecyclerView。这样就可以通过Recycler在不重新绑定/重新创建子视图的情况下修改已连attached的子视图的索引。 Remove意味着这个子视图不再需要被使用了。任何被移除的子视图都应该回收到Recycler对象等待下次的使用。

## Scrap vs. Recycle

RecyclerView有两种缓存系统：Scrap heap（垃圾堆？）和Recycle pool（回收池）。Scrap heap 代表一个保存一些可以直接而不需要通过适配器就可以返回给LayoutManager的子视图的轻量集合。通常被detached的子视图就存放在这里，并在一次layout操作的过程中等待被重新attached（这部分的子视图并没有真正的移除出父视图，所以如果翻译成垃圾堆似乎有些不妥）。Recycle pool 存放的是那些假定并没有得到正确数据(相应位置的数据)的视图，因此它们都要经过适配器重新绑定后才能返回给LayoutManager。 当要给LayoutManager提供一个新视图时，Recycler首先会检查Scrap heap有没有对应的position/id；如果有对应的内容，就直接返回数据不需要通过适配器重新绑定。如果没有的话，Recycler就会从Recycle pool里弄一个合适的视图出来，然后用Adapter给它绑定必要的数据 (就是调用RecyclerView.Adapter.bindViewHolder())再返回。如果Recycle pool中也不存在有效视图，就会在绑定数据前创建新的视图(就是 RecyclerView.Adapter.createViewHolder())，最后返回数据。

## Rule of Thumb

通常来说，如果你想要临时整理并且希望稍后在同一layout操作过程重新使用某个视图的话，可以对它调用`detachAndScrapView()`。如果基于当前布局你不再需要某个子视图的话，对其调用`removeAndRecycleView()`。

## 需要重写的方法

### generateDefaultLayoutParams()

这个不解释

### onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state)

LayoutManager会管理子视图的动画效果。默认情况下，RecyclerView的`getItemAnimator()`方法返回一个不为null的ItemAnimator对象，且子视图的动画效果是打开的。这意味着Adapter对子视图的添加或移除操作都会展示子视图的添加子视图的出现或者移除子视图的消失动画效果。如果LayoutManager的`supportsPredictiveItemAnimations()`方法返回`false`，默认就是，并在`onLayoutChildren(Recycler, State)`方法运行的过程中进行正常的布局操作，不过是添加或移除或者其实的操作导致重屏幕视线中出现或消失，RecyclerView将会有足够的信息去展现淡入或淡出的动画效果效果。

为了有更好的动画体验，LayoutManager应该在`supportsPredictiveItemAnimations()`方法中返回`true`并需要在`onLayoutChildren(Recycler, State)`方法中做额外的逻辑。支持预言动画意味`onLayoutChildren(Recycler, State)`方法将会被调用两次。第一次的预布局layout的目的是决定子视图在真正布局之前的位置然后第二次才是真正的layout。在预布局阶段，子视图会记住它们预布局阶段合理设置的位置，另外{@link LayoutParams#isItemRemoved()}方法返回`true`代表子视图从相应的数据集中被移除，被移除的子视图可以从Scrap中返回来帮助决定其他子视图的恰当位置。

第二次的layout会真正的为已经添加的子视图进行布局。如果`supportsPredictiveItemAnimations()`方法返回`true`，在layout的过程就需要进行额外操作如下，记录在layout之前和layout之后的消失的子视图名单，并合适地为这些视图进行布局，并不需要考虑RecyclerView的边界。这使得动画系统知道这些需要进行消失的子视图的位置，并来进行消失动画效果。

它会在RecyclerView需要初始化布局时调用，当适配器的数据改变时(或者整个适配器被换掉时)会再次调用。**注意！这个方法不是在每次你对布局作出改变时调用的。** 它是初始化布局或者在数据改变时重置子视图布局的好位置。 我们要做的就是根据当前可见的视图元素进行布局。

[网上的一个例子1](https://github.com/HalfStackDeveloper/SwipeCardRecyclerView/blob/master/swipecardrecyclerview/src/main/java/com/wangxiandeng/swipecardrecyclerview/SwipeCardLayoutManager.java)

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    detachAndScrapAttachedViews(recycler);
    for (int i = 0; i < getItemCount(); i++) {
        View child = recycler.getViewForPosition(i);  
        measureChildWithMargins(child, 0, 0);
        addView(child);
        int width = getDecoratedMeasuredWidth(child);
        int height = getDecoratedMeasuredHeight(child);
        layoutDecorated(child, 0, 0, width, height);
        //缩放
        if (i < getItemCount() - 1) {
            child.setScaleX(0.8f);
            child.setScaleY(0.8f);
        }
      }
    }
```

[例子2](https://github.com/devunwired/recyclerview-playground/blob/master/app/src/main/java/com/example/android/recyclerplayground/layout/FixedGridLayoutManager.java)

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    //Scrap measure one child
    View scrap = recycler.getViewForPosition(0);
    addView(scrap);
    measureChildWithMargins(scrap, 0, 0);

    mDecoratedChildWidth = getDecoratedMeasuredWidth(scrap);
    mDecoratedChildHeight = getDecoratedMeasuredHeight(scrap);
    detachAndScrapView(scrap, recycler);

    updateWindowSizing();
    int childLeft;
    int childTop;

    /*
     * Reset the visible and scroll positions
     */
    mFirstVisiblePosition = 0;
    childLeft = childTop = 0;

    //Clear all attached views into the recycle bin
    detachAndScrapAttachedViews(recycler);
    //Fill the grid for the initial layout of views
    fillGrid(DIRECTION_NONE, childLeft, childTop, recycler);
}
```

两个方法的实现上的共同点在于真正布局之前需要调用`detachAndScrapAttachedViews`来暂时回收所有已经Attach的子View，该方法最后通过`removeViewAt`或者`detachViewAt`方法来暂时或者永久移除View

```java
final int childCount = getChildCount();
for (int i = childCount - 1; i >= 0; i--) {
    //....
    if (viewHolder.isInvalid() && !viewHolder.isRemoved() && !mRecyclerView.mAdapter.hasStableIds()) {
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
```

所以可以针对这个做一个**优化**，如下：

```java
detachAndScrapAttachedViews(recycler);
  //....
  for (int i = 0; i < getChildCount(); i++) {
      int position = positionOfIndex(i);
      final View child = getChildAt(i);
      viewCache.put(position, child);
    }

    for (int i = 0; i < viewCache.size(); i++) {
      detachView(viewCache.valueAt(i));
    }

    //..真正布局
    View view = viewCache.get(nextPosition);
    if (view == null) {
      view = recycler.getViewForPosition(nextPosition);
      addView(view);
      //...
    } else {
      attachView(view);
      viewCache.remove(nextPosition);
    }
    //...回收没有被重新attach的View
    for (int i = 0; i < viewCache.size(); i++) {
    final View removingView = viewCache.valueAt(i);
    recycler.recycleView(removingView);
```

通常来说，在这个方法之中你需要完成的主要步骤如下：

- 在滚动事件结束后检查所有已经attach的子View当前的偏移位置。
- 判断是否需要添加新视图填充由滚动屏幕产生的空白部分，并从`Recycler#getViewForPosition`中获取子视图
- 判断当前视图是否不再显示(通常对于支持滚动的View来说)，移除并放置到`Recycler`中
- 判断剩余视图是否需要整理，发生上述变化后可能需要你修改视图的子索引来更好地和它们的适配器位置校准
