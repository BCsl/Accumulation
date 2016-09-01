# RecyclView各种效果的实现思路

## ItemDecoration

`ItemDecoration`可以用来实现间隔线、StickHeader等效果 需要关注该类的三个方法

- onDraw(Canvas c, RecyclerView parent, State state) //先于ItemView绘制
- onDrawOver(Canvas c, RecyclerView parent, State state) //再ItemView绘制后绘制，所以可以绘制StickHeader在ItemView之上
- getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) //Decoration尺寸，会被计入了RecyclerView 每个ItemView的padding中（widthUsed = outRect.left + outRect.right，hegihtUsed = outRect.top + outRect.bottom）

[官方例子](https://android.googlesource.com/platform/development/+/master/samples/Support7Demos/src/com/example/android/supportv7/widget/decorator/DividerItemDecoration.java#101)是最简单的使用教程

[深入理解ItemDecoration](http://blog.piasy.com/2016/03/26/Insight-Android-RecyclerView-ItemDecoration/)

## ItemAnimator

[深入理解ItemAnimator](http://blog.piasy.com/2016/04/04/Insight-Android-RecyclerView-ItemAnimator/)

## 滑动删除和拖拽排序

`ItemTouchHelper`

## 添加Header和Footer

主要实现，**装饰者模式**，扩展原`Adapter`的功能，

因为不管是Header或者Footer都是占用一整行的，所以不同的`LayoutManager`有不同的处理

`GridLayoutManager`可以通过重写`SpanSizeLookup`对象来实现某个position的子View占用的网格数量，调用时机`onViewAttachedToWindow`:

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

`StaggeredGridLayoutManager`则需要修改其`StaggeredGridLayoutManager.LayoutParams`对象，所以调用时机上也不同，在`onViewAttachedToWindow`方法：

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
