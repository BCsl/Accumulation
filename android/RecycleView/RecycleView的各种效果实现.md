# RecyclView各种效果的实现思路

[Android RecyclerView 使用完全解析](http://blog.csdn.net/lmj623565791/article/details/45059587)

## ItemDecoration

`ItemDecoration`可以用来实现间隔线、StickHeader等效果,需要关注该类的**三个方法**

- onDraw(Canvas c, RecyclerView parent, State state) //先于ItemView绘制
- onDrawOver(Canvas c, RecyclerView parent, State state) //再ItemView绘制后绘制，所以可以绘制StickHeader在ItemView之上
- getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) //Decoration尺寸，会被计入了RecyclerView 每个ItemView的padding中（widthUsed = outRect.left + outRect.right, outRect的4个值代表这ItemView的padding值）

[官方例子](https://android.googlesource.com/platform/development/+/master/samples/Support7Demos/src/com/example/android/supportv7/widget/decorator/DividerItemDecoration.java#101)是最简单的使用教程

[深入理解ItemDecoration](http://blog.piasy.com/2016/03/26/Insight-Android-RecyclerView-ItemDecoration/)

### 实现StickHeader

#### getItemOffests为分组的ItemView设置padding

首先需要注意的没一个分组下的第一个ItemView需要为它设置好padding，以不至于被Header覆盖，可以在`getItemOffsets`中处理

```java
@Override
public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
    int position = parent.getChildAdapterPosition(view);

    int headerHeight = 0;
    if (position != RecyclerView.NO_POSITION && hasHeader(position)) { //hasHeader用来判断当前位置和上一个位置之间的HeaderId是否一致，则可以知道是否需要Header
        View header = getHeader(parent, position).itemView; //getHeader用来获取Header View，由于我们没有真正添加到RecycleView中，我们需要为它做测量操作
        headerHeight = getHeaderHeightForLayout(header);
    }

    outRect.set(0, headerHeight, 0, 0);
}
```

Header的measure和layout模拟

```java
private RecyclerView.ViewHolder getHeader(RecyclerView parent, int position) {
    final long key = mAdapter.getHeaderId(position);
    if (mHeaderCache.containsKey(key)) {
        return mHeaderCache.get(key);
    } else {
        final RecyclerView.ViewHolder holder = mAdapter.onCreateHeaderViewHolder(parent);
        final View header = holder.itemView;
        mAdapter.onBindHeaderViewHolder(holder, position);
        int widthSpec = View.MeasureSpec.makeMeasureSpec(parent.getWidth(), View.MeasureSpec.EXACTLY);
        int heightSpec = View.MeasureSpec.makeMeasureSpec(parent.getHeight(), View.MeasureSpec.UNSPECIFIED);

        int childWidth = ViewGroup.getChildMeasureSpec(widthSpec,
                parent.getPaddingLeft() + parent.getPaddingRight(), header.getLayoutParams().width);
        int childHeight = ViewGroup.getChildMeasureSpec(heightSpec,
                parent.getPaddingTop() + parent.getPaddingBottom(), header.getLayoutParams().height);
        header.measure(childWidth, childHeight);
        header.layout(0, 0, header.getMeasuredWidth(), header.getMeasuredHeight());
        mHeaderCache.put(key, holder);
        return holder;
    }
}
```

#### onDrawOver为Header进行绘制

前面为每一个分组的地一个ItemView设置好了padding，这一步则是在padding上进行绘制操作，通过`View#draw`方法来处理，而不是直接添加一个View到RecycleView

```java
public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
    final int count = parent.getChildCount();

    for (int layoutPos = 0; layoutPos < count; layoutPos++) {
        final View child = parent.getChildAt(layoutPos);

        final int adapterPos = parent.getChildAdapterPosition(child);

        if (adapterPos != RecyclerView.NO_POSITION && hasHeader(adapterPos)) {
            View header = getHeader(parent, adapterPos).itemView;
            c.save();
            final int left = child.getLeft();
            final int top = getHeaderTop(parent, child, header, adapterPos, layoutPos); //需要为第一个View的Header进行偏移量的调整
            c.translate(left, top);
            header.setTranslationX(left);
            header.setTranslationY(top);
            header.draw(c);
            c.restore();
        }
    }
}
```

#### 处理Header的点击

因为Header并不是真正View，所以处理点击事件只能通过触摸点来判断是否属于某个Header（RecycleView#OnItemTouchListener），这种方式不太完美

## ItemAnimator

[深入理解ItemAnimator](http://blog.piasy.com/2016/04/04/Insight-Android-RecyclerView-ItemAnimator/)

## 滑动删除和拖拽排序

`ItemTouchHelper`

## 添加Header和Footer

主要实现，**装饰者模式**，扩展原`Adapter`的功能，

因为不管是Header或者Footer都是占用一整行的，所以不同的`LayoutManager`有不同的处理

`GridLayoutManager`可以通过重写`SpanSizeLookup`对象来实现某个position的子View占用的网格数量，调用时机`RecyclerView#onAttachedToRecyclerView`:

```java
if (manager instanceof GridLayoutManager) {
    final GridLayoutManager gridManager = ((GridLayoutManager) manager);
    gridManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
        @Override
        public int getSpanSize(int position) {
            return (isHeader(position) || isFooter(position)) ? gridManager.getSpanCount() : 1;
        }
    });
}
```

`StaggeredGridLayoutManager`则需要修改其`StaggeredGridLayoutManager.LayoutParams`对象，所以调用时机上也不同，在`RecyclerView#onViewAttachedToWindow`方法：

```java
ViewGroup.LayoutParams lp = holder.itemView.getLayoutParams();
if (lp != null && lp instanceof StaggeredGridLayoutManager.LayoutParams) {
    if(isHeader(holder.getLayoutPosition()) || isFooter(holder.getLayoutPosition())) {
        StaggeredGridLayoutManager.LayoutParams p = (StaggeredGridLayoutManager.LayoutParams) lp;
        p.setFullSpan(true);
    }
}
```

还需要需要注意的是`RecyclerView.AdapterDataObserver`的回调，因为添加了Header，被装饰的Adapter是并不感知且不需要理会，所有逻辑还是自身的那一套，所以`onItemRangeChanged`的一些回调也需要在Index上做相应的改变

目前，很多库添加HeaderView和FooterView都是通过传递一个初始化好的View对象到Adapter，强引用直接保存在Adapter，并没考虑到View的创建和数据的绑定的分离和**缓存**，这对于内存和RecycleView的数据刷新上带来了不必要的麻烦[问题分析和解决](http://blog.csdn.net/zxt0601/article/details/52267325)

```java
private RecyclerView.AdapterDataObserver mDataObserver = new RecyclerView.AdapterDataObserver() {
    //....

    @Override
    public void onItemRangeInserted(int positionStart, int itemCount) {
        notifyItemRangeInserted(positionStart + getHeaderViewsCount() + 1, itemCount);
    }
    //....
};
```

## StickHeader、分组的实现

### 普通的分组

普通分组不需要实现StickHeader效果，同样可以使用 **装饰者模式**，扩展原`Adapter`的功能

分组数据整理

```java
private List<Section> findSections() {

    List<Section> listSection = new ArrayList<>();

    int n = mListValues.size();
    LinkedHashMap<String, Integer> sections = new LinkedHashMap<>();

    for(int i=0; i<n; i++) {
        String sectionName = mSectionizer.getSectionTitle(mListValues.get(i));  //Sectionizer从外部注入，实现解偶

        if(!sections.containsKey(sectionName)) {
            sections.put(sectionName, i );
            listSection.add(new Section(i ,sectionName)); //记录出现位置和标题，后续还需要根据位置进行排序

        }
    }

    return listSection ;
}
```

需要注意的是Section的排序和Section的位置还需要根据其前面出现的Section数目进行调整（如一个Section1的初步插入位置为10,但在这个Scetion1之前还有一个Section0，初步插入位置为0，那么Section1调整后的位置应该为11，Section0也还是0），另外普通的Item的位置也要进行必要的处理（如Item属于第二个Section，最终位置为10，真实位置为10-2 = 8）

### StickHeader

通过`ItemDecoration`来实现

## 下拉刷新和自动加载

## EmptyView

## 通用的Adapter

## 灵活方便处理多Type

[MultiType](https://github.com/drakeet/MultiType)

### 实现思路

全局的多Type类型池，应用初始化的时候以`Class`类型为`Index`为所有的类型进行一次注册，同时提供一个对应的`Provider`，用于绑定和设置对应类型的ViewHolder，之后`RecycleView#Adapter`根据具体的类型数据类型找到对应的`Provider`进行初始化和数据的绑定，就是这么简单，唯一的争议在于全局类型池的使用上，但作者也是给出了自己的解析，对于性能的损耗也是有限的，但对于代码的解偶的帮助却是巨大的，有机会还是可以试试
