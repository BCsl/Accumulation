# Custom LayoutManager
[Refer](http://wiresareobsolete.com/2014/09/building-a-recyclerview-layoutmanager-part-1/)

## Recycler
Your LayoutManager is given access to a Recycler instance at key points in the process when you might need to recycle old views, or obtain new views from a potentially recycled previous child.

The Recycler also removes the need to directly access the view’s current adapter implementation. When your LayoutManager requires a new child view, simply call getViewForPosition() and the Recycler will return the view with the appropriate data already bound. __The Recycler takes care of determining whether a new view must be created, or if an existing scrapped view gets reused.__ Your responsibility, inside your LayoutManager, is to ensure that views which are no longer visible get passed to the Recycler in a timely manner; this will keep the Recycler from creating more view objects than is necessary.

简单的说，LayoutManager这个对象不需要关注子View的创建和复用，这部分只要交给Recycler对象就可以了，当需要一个子View的时候，你只需要调用Recycler对象的`getViewForPosition()`方法即可，另外还需要把不再可见的子View交给Recycler对象回收处理。

## Detach vs. Remove
There are two ways to handle existing child views during a layout update: detach and remove. Detach is meant to be a lightweight operation for reordering views. Views that are detached are expected to be re-attached before your code returns. This can be used to modify the indices of attached child views without re-binding or re-creating those views through the Recycler.

Remove is meant for views that are no longer needed. Any view that is permanently removed should be placed in the Recycler for later re-use, but the API does not enforce this. It is up to you whether the views you remove also get recycled.

RecyclerView更新的时候有两种方式来更新子View，detach和remove。Detach是一个记录视图的轻量的操作，被detached子视图一般预计会在同一次layout操作中重新的attached到RecyclerView。这样就可以通过Recycler在不重新绑定/重新创建子视图的情况下修改已连attached的子视图的索引。
Remove意味着这个子视图不再需要被使用了。任何被移除的子视图都应该回收到Recycler对象等待下次的使用。

## Scrap vs. Recycle
Recycler has a two-level view caching system: the scrap heap and the recycle pool. The scrap heap represents a lighter weight collection where views can be returned to the LayoutManager directly without passing through the adapter again. Views are typically placed here when they are temporarily being detached, but will be re-used within the same layout pass. The recycle pool consists of views that are assumed to have incorrect data (data from a different position), so they will always be passed through the adapter to have data re-bound before they are returned to the LayoutManager.

When attempting to supply the LayoutManager with a new view, a Recycler will first check the scrap heap for a matching position/id; if one exists, it will be returned without re-binding to the adapter data. If no matching view is found, the Recycler will instead pull a suitable view from the recycle pool and bind the necessary data to it from the adapter (i.e. RecyclerView.Adapter.bindViewHolder() is invoked) before returning it. In cases where no valid views exist in the recycle pool, a new view will be created instead (i.e. RecyclerView.Adapter.createViewHolder() is invoked) before being bound, and returned.

RecyclerView有两种缓存系统：Scrap heap（垃圾堆？）和Recycle pool（回收池）。Scrap heap 代表一个保存一些可以直接而不需要通过适配器就可以返回给LayoutManager的子视图的轻量集合。通常被detached的子视图就存放在这里，并在一次layout操作的过程中等待被重新attached（这部分的子视图并没有真正的移除出父视图，所以如果翻译成垃圾堆似乎有些不妥）。Recycle pool 存放的是那些假定并没有得到正确数据(相应位置的数据)的视图，因此它们都要经过适配器重新绑定后才能返回给LayoutManager。
当要给LayoutManager提供一个新视图时，Recycler首先会检查Scrap heap有没有对应的position/id；如果有对应的内容，就直接返回数据不需要通过适配器重新绑定。如果没有的话，Recycler就会从Recycle pool里弄一个合适的视图出来，然后用Adapter给它绑定必要的数据 (就是调用RecyclerView.Adapter.bindViewHolder())再返回。如果Recycle pool中也不存在有效视图，就会在绑定数据前创建新的视图(就是 RecyclerView.Adapter.createViewHolder())，最后返回数据。

## Rule of Thumb
The LayoutManager API lets you do pretty much all of these tasks independently if you wish, so the possible combinations can be a bit numerous. In general, use detachAndScrapView() for views you want to temporarily reorganize and expect to re-attach inside the same layout pass. Use removeAndRecycleView() for views you no longer need based on the current layout.
通常来说，如果你想要临时整理并且希望稍后在同一layout操作过程重新使用某个视图的话，可以对它调用detachAndScrapView()。如果基于当前布局你不再需要某个子视图的话，对其调用removeAndRecycleView()。

## 需要重写的方法
### generateDefaultLayoutParams()
这个不解释
### onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state)
Lay out all relevant child views from the given adapter.
The LayoutManager is in charge of the behavior of item animations. By default,RecyclerView has a non-null {@link #getItemAnimator() ItemAnimator}, and simple item animations are enabled. This means that add/remove operations on the adapter will result in animations to add new or appearing items, removed or disappearing items, and moved items. If a LayoutManager returns false from  {@link #supportsPredictiveItemAnimations()}, which is the default, and runs a normal layout operation during {@link #onLayoutChildren(Recycler, State)}, the RecyclerView will have enough information to run those animations in a simple fade views in and out, whether they are actually added/removed or whether they are moved on or off the screen due to other add/remove operations.
LayoutManager会管理子视图的动画效果。默认情况下，RecyclerView的`getItemAnimator()`方法返回一个不为null的ItemAnimator对象，且子视图的动画效果是打开的。这意味着Adapter对子视图的添加或移除操作都会展示子视图的添加子视图的出现或者移除子视图的消失动画效果。如果LayoutManager的`supportsPredictiveItemAnimations()`方法返回`false`，默认就是，并在`onLayoutChildren(Recycler, State)`方法运行的过程中进行正常的布局操作，不过是添加或移除或者其实的操作导致重屏幕视线中出现或消失，RecyclerView将会有足够的信息去展现淡入或淡出的动画效果效果。

A LayoutManager wanting a better item animation experience, where items can be animated onto and off of the screen according to where the items exist when they are not on screen, then the LayoutManager should return true from {@link #supportsPredictiveItemAnimations()} and add additional logic to {@link #onLayoutChildren(Recycler, State)}. Supporting predictive animations means that {@link #onLayoutChildren(Recycler, State)} will be called twice; once as a "pre" layout step to determine where items would have been prior to a real layout, and again to do the "real" layout. In the pre-layout phase,items will remember their pre-layout positions to allow them to be laid out appropriately. Also, {@link LayoutParams#isItemRemoved() removed} items will be returned from the scrap to help determine correct placement of other items. These removed items should not be added to the child list, but should be used to help calculate correct positioning of other views, including views that were not previously onscreen (referred to as APPEARING views), but whose pre-layout offscreen position can be determined given the extra information about the pre-layout removed views.

为了有更好的动画体验，LayoutManager应该在`supportsPredictiveItemAnimations()`方法中返回`true`并需要在`onLayoutChildren(Recycler, State)`方法中做额外的逻辑。支持预言动画意味`onLayoutChildren(Recycler, State)`方法将会被调用两次。第一次的预布局layout的目的是决定子视图在真正布局之前的位置然后第二次才是真正的layout。在预布局阶段，子视图会记住它们预布局阶段合理设置的位置，另外{@link LayoutParams#isItemRemoved()}方法返回`true`代表子视图从相应的数据集中被移除，被移除的子视图可以从Scrap中返回来帮助决定其他子视图的恰当位置。

The second layout pass is the real layout in which only non-removed views will be used. The only additional requirement during this pass is, if {@link #supportsPredictiveItemAnimations()} returns true, to note which views exist in the child list prior to layout and which are not there after  layout (referred to as DISAPPEARING views), and to position/layout those views  appropriately, without regard to the actual bounds of the RecyclerView. This allows  the animation system to know the location to which to animate these disappearing views.

第二次的layout会真正的为已经添加的子视图进行布局。如果`supportsPredictiveItemAnimations()`方法返回`true`，在layout的过程就需要进行额外操作如下，记录在layout之前和layout之后的消失的子视图名单，并合适地为这些视图进行布局，并不需要考虑RecyclerView的边界。这使得动画系统知道这些需要进行消失的子视图的位置，并来进行消失动画效果。

它会在RecyclerView需要初始化布局时调用，当适配器的数据改变时(或者整个适配器被换掉时)会再次调用。__注意！这个方法不是在每次你对布局作出改变时调用的。__ 它是初始化布局或者在数据改变时重置子视图布局的好位置。
我们要做的就是根据当前可见的视图元素进行布局。

通常来说，在这类方法之中你需要完成的主要步骤如下：
* 在滚动事件结束后检查所有attached视图当前的偏移位置。
* 判断是否需要添加新视图填充由滚动屏幕产生的空白部分，并从Recycler中获取子视图
* 判断当前视图是否不再显示，移除并放置到Recycler中。
* 判断剩余视图是否需要整理，发生上述变化后可能需要你修改视图的子索引来更好地和它们的适配器位置校准。
