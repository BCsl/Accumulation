# RecyclView各种效果的实现思路

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

## StickHeader的实现

## 下拉刷新和自动加载

## EmptyView

## 通用的Adapter
