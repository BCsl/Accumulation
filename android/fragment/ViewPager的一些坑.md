# ViewPager 的一些坑

## ViewPager 的刷新

调用链 `PagerAdapter#notifyDataSetChanged` -> `DataSetObserver#onChanged` -> `ViewPager#dataSetChanged`

`mItems` 记录的是当前加载到内存中的 pager ，`notifyDataSetChanged` 的调用会遍历 `mItems`，通过 `mAdapter.getItemPosition` 方法确定是否需要改变当前的 pager，

```java
void dataSetChanged() {
    // This method only gets called if our observer is attached, so mAdapter is non-null.

    final int adapterCount = mAdapter.getCount();
    mExpectedAdapterCount = adapterCount;
    boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1 && mItems.size() < adapterCount;
    int newCurrItem = mCurItem;

    boolean isUpdating = false;
    for (int i = 0; i < mItems.size(); i++) {
        final ItemInfo ii = mItems.get(i);
        final int newPos = mAdapter.getItemPosition(ii.object); // 默认即返回PagerAdapter.POSITION_UNCHANGED

        if (newPos == PagerAdapter.POSITION_UNCHANGED) {
            continue;
        }

        if (newPos == PagerAdapter.POSITION_NONE) { // 如果当前 pager 需要返回 PagerAdapter.POSITION_NONE
            mItems.remove(i);
            i--;

            if (!isUpdating) {
                mAdapter.startUpdate(this);
                isUpdating = true;
            }

            mAdapter.destroyItem(this, ii.position, ii.object); // 销毁，不用担心 view/fragment 泄漏
            needPopulate = true;

            if (mCurItem == ii.position) {
                // Keep the current item in the valid range
                newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));  //限制选中item
                needPopulate = true;
            }
            continue;
        }

        if (ii.position != newPos) {
            if (ii.position == mCurItem) {
                // Our current item changed position. Follow it.
                newCurrItem = newPos;
            }

            ii.position = newPos;
            needPopulate = true;
        }
    }

    if (isUpdating) {
        mAdapter.finishUpdate(this);
    }

    if (needPopulate) {
        //...
        setCurrentItemInternal(newCurrItem, false, true); //会回调 onPageSelected
        requestLayout();
    }
}
```

> 另外注意，`notifyDataSetChanged` 后可能会回调 `onPageSelected` ，这个 position 是否自己想要的或是否需要处理这个回调，需要根据情况来处理

所以为了 `notifyDataSetChanged` 生效，请根据情况修改 `getItemPosition` 返回值，其中 `mChildCount` 的具体值，应该是之前 `Adapter#getCount` 的数量，或者说比之前的数量要大，不然会出现 pager 移除不彻底的情况！！！

```java
//XXXPagerAdapter.java

private int mChildCount;

@Override
public void notifyDataSetChanged() {
    mChildCount = &{large than before}; //这里的 mChildCount 需要注意了！必须必之前的数量值要大！！
    super.notifyDataSetChanged();
}

@Override
public int getItemPosition(Object object) {
    if (mChildCount > 0) {
        mChildCount--;
        return POSITION_NONE;
    }
    return super.getItemPosition(object);
}
```

## notifyDataSetChanged 后 PageTransformer 不生效

如果的 `ViewPager` 使用了 `PageTransformer`，且一屏显示多个 pager ，那么你会发现你的 `PageTransformer` 没生效，除非你滑动下你的 `ViewPager` 才会正确的显示

如果出现这种情况，那么强制调用 `scrollToItem` 方法刷新一次吧

```java
try {
    //主要解决notifyDataSetChanged后 PageTransformer 不生效的问题
    MethodUtils.invokeDeclaredMethod(mCustomViewPager, "scrollToItem", new Object[]{index, false, 0, true}, new Class[]{int.class, boolean.class, int.class, boolean.class});
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```

## 初始化的当前 pager 为 0 的时候，不会调 OnPageChangeListener

因为 `ViewPager` 默认就是 0 ，所以就算你调用 `ViewPager.setCurrentItem(0)` ，因为前后位置没变，所以不会回调，所以自己手动回调

```java
mViewPager.setCurrentItem(index ,/**false/true**/);
mViewPager.addOnPageChangeListener(this);
mViewPager.post(new Runnable() {
    @Override
    public void run() {
        if (...) {
            //注意，有多少个 OnPageChangeListener 就要回调多少个
            mPageChangeListener.onPageSelected(index);
        }
    }
});
```

另外一种实现则如下，则不需要自己手动回调 `OnPageChangeListener`但却需要用到反射，并没有孰优孰劣之分

```java
mViewPager.setCurrentItem(index ,/**false/true**/);
mViewPager.addOnPageChangeListener(this);
mViewPager.post(new Runnable() {
    @Override
    public void run() {
        if (...) {
            try {
                //主要解决 PageTransformer 不生效或者 OnPageChangeListener 不回调问题
                MethodUtils.invokeDeclaredMethod(mCustomViewPager, "scrollToItem", new Object[]{index, false, 0, true}, new Class[]{int.class, boolean.class, int.class, boolean.class});
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }
    }
});
```

## 源码浅析

### onMeasure

`ViewPager` 默认宽高都为 0 ，不支持使用 `wrap_content` 的，使用 `match_parent` 或指定特定宽高值。其中比较重要且复杂的是 `populate` 方法

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // For simple implementation, our internal size is always 0.
    // We depend on the container to specify the layout size of
    // our view.  We can't really know what it is since we will be
    // adding and removing different arbitrary views and do not
    // want the layout to change as this happens.
    setMeasuredDimension(getDefaultSize(0, widthMeasureSpec), getDefaultSize(0, heightMeasureSpec));  

    final int measuredWidth = getMeasuredWidth();
    final int maxGutterSize = measuredWidth / 10;
    mGutterSize = Math.min(maxGutterSize, mDefaultGutterSize);

    // Children are just made to fill our space.
    int childWidthSize = measuredWidth - getPaddingLeft() - getPaddingRight();
    int childHeightSize = getMeasuredHeight() - getPaddingTop() - getPaddingBottom();

    /*
     * Make sure all children have been properly measured. Decor views first.
     * Right now we cheat and make this less complicated by assuming decor
     * views won't intersect. We will pin to edges based on gravity.
     */
    int size = getChildCount();
    for (int i = 0; i < size; ++i) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (lp != null && lp.isDecor) {
                //decor view 忽略
            }
        }
    }

    mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(childWidthSize, MeasureSpec.EXACTLY);
    mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(childHeightSize, MeasureSpec.EXACTLY);

    // Make sure we have created all fragments that we need to have shown.
    mInLayout = true;
    populate();
    mInLayout = false;

    // Page views next.
    size = getChildCount();
    for (int i = 0; i < size; ++i) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (lp == null || !lp.isDecor) {
                final int widthSpec = MeasureSpec.makeMeasureSpec((int) (childWidthSize * lp.widthFactor), MeasureSpec.EXACTLY);
                child.measure(widthSpec, mChildHeightMeasureSpec); //子View高度默认填充 ViewPager，宽度跟lp.widthFactor有关
            }
        }
    }
}
```

### onLayout

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    final int count = getChildCount();
    int width = r - l;
    int height = b - t;
    int paddingLeft = getPaddingLeft();
    int paddingTop = getPaddingTop();
    int paddingRight = getPaddingRight();
    int paddingBottom = getPaddingBottom();
    final int scrollX = getScrollX();

    int decorCount = 0;

    // First pass - decor views. We need to do this in two passes so that
    // we have the proper offsets for non-decor views later.
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            int childLeft = 0;
            int childTop = 0;
            if (lp.isDecor) {
              //decorView ,忽略
            }
        }
    }

    final int childWidth = width - paddingLeft - paddingRight;
    // Page views. Do this once we have the right padding offsets from above.
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            ItemInfo ii;
            if (!lp.isDecor && (ii = infoForChild(child)) != null) {
                int loff = (int) (childWidth * ii.offset);
                int childLeft = paddingLeft + loff;
                int childTop = paddingTop;
                if (lp.needsMeasure) {
                    // This was added during layout and needs measurement.
                    // Do it now that we know what we're working with.
                    lp.needsMeasure = false;
                    final int widthSpec = MeasureSpec.makeMeasureSpec((int) (childWidth * lp.widthFactor), MeasureSpec.EXACTLY);
                    final int heightSpec = MeasureSpec.makeMeasureSpec((int) (height - paddingTop - paddingBottom), MeasureSpec.EXACTLY);
                    child.measure(widthSpec, heightSpec);
                }

                child.layout(childLeft, childTop,childLeft + child.getMeasuredWidth(), childTop + child.getMeasuredHeight());
            }
        }
    }
    mTopPageBounds = paddingTop;
    mBottomPageBounds = height - paddingBottom;
    mDecorChildCount = decorCount;

    if (mFirstLayout) {
        scrollToItem(mCurItem, false, 0, false);
    }
    mFirstLayout = false;
}
```

### populate

`ItemInfo` 的数据结构来描述一个 pager，`mItems` 的 `ItemInfo` 列表则记录着需要加载到 `ViewPager` 中的 pager，一般通过 `addNewItem(int position, int index)` 方法新建，又调用到 `mAdapter.instantiateItem` 方法来新建 pager（View 或者 Fragment）， `populate` 就是用来确定当前需要加载的 `ItemInfo`，并销毁额外的 `ItemInfo`

```java
void populate() {
    populate(mCurItem);
}


void populate(int newCurrentItem) {
    ItemInfo oldCurInfo = null;
    if (mCurItem != newCurrentItem) {
        oldCurInfo = infoForPosition(mCurItem);
        mCurItem = newCurrentItem;
    }

    if (mAdapter == null) {
        sortChildDrawingOrder();
        return;
    }

    if (mPopulatePending) {
        if (DEBUG) Log.i(TAG, "populate is pending, skipping for now...");
        sortChildDrawingOrder();
        return;
    }

    if (getWindowToken() == null) {
        return;
    }

    mAdapter.startUpdate(this);
    //mOffscreenPageLimit 的意义在于加载当前 item 的前、后mOffscreenPageLimit个 Pager
    final int pageLimit = mOffscreenPageLimit;
    final int startPos = Math.max(0, mCurItem - pageLimit);
    final int N = mAdapter.getCount();
    final int endPos = Math.min(N - 1, mCurItem + pageLimit);

    //...

    // Locate the currently focused item or add it if needed.
    int curIndex = -1;
    ItemInfo curItem = null;
    for (curIndex = 0; curIndex < mItems.size(); curIndex++) {  //curIndex = 0
        final ItemInfo ii = mItems.get(curIndex);
        // mItems一般记录的是连续的 position，所以需要判断第一个大于当前 position 的进行判断即可知道是否存在
        if (ii.position >= mCurItem) {
            if (ii.position == mCurItem) curItem = ii;
            break;
        }
    }

    if (curItem == null && N > 0) {
        curItem = addNewItem(mCurItem, curIndex); //addNewItem(int position, int index):ItemInfo
    }

    if (curItem != null) {
        float extraWidthLeft = 0.f;
        //curItem 左边的 pager 的 index
        int itemIndex = curIndex - 1;
        ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
        final int clientWidth = getClientWidth();
        //算出左边页面需要的宽度，一般可简化为 1.0f + (float) getPaddingLeft() / (float) clientWidth;，代表了前一个 pager 距离当前 pager 有多少个 clientWidth 单位，一般则为1
        // 代表离屏的宽度，默认最小也会加载当前 pager 左右两个离屏 pager
        final float leftWidthNeeded = clientWidth <= 0 ? 0 : 2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
        //这个 for 循环的目的是确定 curItem 左边的 pager 的 itemInfo ，并清理需要 destory 的 pager
        for (int pos = mCurItem - 1; pos >= 0; pos--) {
            if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
              //超过 OffscreenPageLimit 指定的值，且离屏超过一屏了，那么这个 pager 就可能需要 destory 了
                if (ii == null) {
                    break;
                }
                if (pos == ii.position && !ii.scrolling) {
                    mItems.remove(itemIndex);
                    mAdapter.destroyItem(this, pos, ii.object);
                    itemIndex--;
                    curIndex--;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                }
            } else if (ii != null && pos == ii.position) {
                //已经存在的，继续向左遍历
                extraWidthLeft += ii.widthFactor;
                itemIndex--;
                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            } else {
                //不存在的，新建
                ii = addNewItem(pos, itemIndex + 1);
                extraWidthLeft += ii.widthFactor;
                curIndex++;
                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            }
        }
        // 确定 curItem 的右边的 pager ，基本逻辑和上面一样
        float extraWidthRight = curItem.widthFactor;
        itemIndex = curIndex + 1;
        if (extraWidthRight < 2.f) {
          //...
        }

        calculatePageOffsets(curItem, curIndex, oldCurInfo);
    }

    mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);

    mAdapter.finishUpdate(this);

    // 根据 ItemInfo 提供的信息修改LayoutParams的属性
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View child = getChildAt(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        lp.childIndex = i;
        if (!lp.isDecor && lp.widthFactor == 0.f) {
            // 0 means requery the adapter for this, it doesn't have a valid width.
            final ItemInfo ii = infoForChild(child);
            if (ii != null) {
                lp.widthFactor = ii.widthFactor;
                lp.position = ii.position;
            }
        }
    }
    sortChildDrawingOrder();

    if (hasFocus()) {
      //焦点处理
    }
}
```
