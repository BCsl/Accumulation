# ListView源码分析

需要结合AbsListView
## 属性
```java
* @attr ref android.R.styleable#ListView_entries
* @attr ref android.R.styleable#ListView_divider
* @attr ref android.R.styleable#ListView_dividerHeight
* @attr ref android.R.styleable#ListView_headerDividersEnabled
* @attr ref android.R.styleable#ListView_footerDividersEnabled
```
## 构造

并没有什么特别
```java
public ListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        //....
        CharSequence[] entries = a.getTextArray(
                com.android.internal.R.styleable.ListView_entries);
        if (entries != null) {
            setAdapter(new ArrayAdapter<CharSequence>(context,
                    com.android.internal.R.layout.simple_list_item_1, entries));
        }
            // If a divider is specified use its intrinsic height for divider height
            setDivider(d);
            //...
            setOverscrollHeader(osHeader);
            //...
            setOverscrollFooter(osFooter);
            //...
            // Use the height specified, zero being the default
            setDividerHeight(dividerHeight);
            //...
        mHeaderDividersEnabled = a.getBoolean(R.styleable.ListView_headerDividersEnabled, true);
        mFooterDividersEnabled = a.getBoolean(R.styleable.ListView_footerDividersEnabled, true);
        //...
    }
```
## 测量

这里可能的疑问是高度的确定。

```java
@Override
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       // Sets up mListPadding
       super.onMeasure(widthMeasureSpec, heightMeasureSpec);

       int widthMode = MeasureSpec.getMode(widthMeasureSpec);
       int heightMode = MeasureSpec.getMode(heightMeasureSpec);
       int widthSize = MeasureSpec.getSize(widthMeasureSpec);
       int heightSize = MeasureSpec.getSize(heightMeasureSpec);

       int childWidth = 0;
       int childHeight = 0;
       int childState = 0;

       mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
       if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED ||
               heightMode == MeasureSpec.UNSPECIFIED)) {
           final View child = obtainView(0, mIsScrap);
           //计算ItemView的长度，这里的计算并没有考虑外边距，而且也不支持ItemView带外边距，高度在默认情况下使用UNSPECIFIED模式
           measureScrapChild(child, 0, widthMeasureSpec);

           childWidth = child.getMeasuredWidth();
           childHeight = child.getMeasuredHeight();
           childState = combineMeasuredStates(childState, child.getMeasuredState());

           if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                   ((LayoutParams) child.getLayoutParams()).viewType)) {
               mRecycler.addScrapView(child, 0);//默认可以执行到这里
           }
       }

       if (widthMode == MeasureSpec.UNSPECIFIED) {
           widthSize = mListPadding.left + mListPadding.right + childWidth +
                   getVerticalScrollbarWidth();
       } else {
           widthSize |= (childState&MEASURED_STATE_MASK);
       }

       if (heightMode == MeasureSpec.UNSPECIFIED) {
           heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                   getVerticalFadingEdgeLength() * 2;
       }

       if (heightMode == MeasureSpec.AT_MOST) {
           // TODO: after first layout we should maybe start at the first visible position, not 0
          //  Measures the height of the given range of children (inclusive) and
          //returns the height with this ListView's padding and divider heights
          //included.If maxHeight is provided, the measuring will stop when the current height reaches maxHeight.
           heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);//这里计算的最大高度一个Item的高度
       }

       setMeasuredDimension(widthSize , heightSize);
       mWidthMeasureSpec = widthMeasureSpec;
   }

```
再设置Adapter之前和之后的两种测量机制，而我们只需要关心设置Adapter之后的。

## layoutChildren

根据数据源的情况分两种情况 1.Adapter不为空 2.Adapter为空的情况
Adapter为空比较简单，移除全部View和恢复所有默认标志位,而在Adapter不为空，但数据为空，情况大致一样
Adapter不为空的情况下，需要分第一次layout（Adapter数据>0，childCount==0）和第二次layout（Adapter数据>0，childCount>0）情况
初次次Layout的时候childCount必定为0，因为所有ItemView都是在Layout的过程添加，通过`addViewInLayout`方法来添加View，`Gallery`也是，
会从上往下`fillDown`或者从下往上'fillUp'（mStackFromBottom决定），填充满最多一屏的ItemVIew。
由于初次Layout之后，ListView已经有了足够的ItemView，再次Layout的情况下，需要把这些添加的ItemView移除，不然就重复了呗，但是移除之前，记得
把这里ItemView中需要缓存的ItemView做好缓存，以便复用。再就是填充方式与第一次不一样的是，填充使用的方法是'fillSpecific'，区别在于，先填充某
个特定位置的Item,在往上和往下来填充ItemView。具体的填充View的方法在`setupChild`,会根据ItemView的来源（是否从回收堆中取得），会有不一样的行为
，如果不是从回收堆中回收的View，通过`addViewInLayout`方法来为ListView新增一个View，否则使用`attachViewToParent`把之前被`detachViewsFromParent`
再次attach到界面。

```java
@Override
  protected void layoutChildren() {
      final boolean blockLayoutRequests = mBlockLayoutRequests;
      if (blockLayoutRequests) {
          return;
      }

      mBlockLayoutRequests = true;

      try {
          super.layoutChildren();//父类空实现

          invalidate();

          if (mAdapter == null) {
              resetList();//移除全部View、恢复默认标志位等
              invokeOnItemScrollListener();
              return;
          }

          final int childrenTop = mListPadding.top;
          final int childrenBottom = mBottom - mTop - mListPadding.bottom;//mButtom和mTop的数值?

          final int childCount = getChildCount();//在第一次Layout的时候必定为0，因为所有的ItemView都是来layout的过程添加的

          int index = 0;
          int delta = 0;

          View sel;
          View oldSel = null;
          View oldFirst = null;
          View newSel = null;

          // Remember stuff we will need down below
          switch (mLayoutMode) {
          case LAYOUT_SET_SELECTION:
              index = mNextSelectedPosition - mFirstPosition;
              if (index >= 0 && index < childCount) {
                  newSel = getChildAt(index);
              }
              break;
          case LAYOUT_FORCE_TOP:
          case LAYOUT_FORCE_BOTTOM:
          case LAYOUT_SPECIFIC:
          case LAYOUT_SYNC:
              break;
          case LAYOUT_MOVE_SELECTION:
          default://默认情况
              // Remember the previously selected view
              index = mSelectedPosition - mFirstPosition;
              if (index >= 0 && index < childCount) {
                  oldSel = getChildAt(index);
              }
              // Remember the previous first child
              oldFirst = getChildAt(0);

              if (mNextSelectedPosition >= 0) {
                  delta = mNextSelectedPosition - mSelectedPosition;
              }
              // Caution: newSel might be null
              newSel = getChildAt(index + delta);
          }
          //在Adapter调用`notifityDataSetChange`方法的时候会设为true，在`AdapterView`的`AdapterDataSetObserver`中响应。
          boolean dataChanged = mDataChanged;
          if (dataChanged) {
              handleDataChanged();
          }
          // Handle the empty set by removing all views that are visible and calling it a day
          if (mItemCount == 0) {
              resetList();
              invokeOnItemScrollListener();
              return;
          } else if (mItemCount != mAdapter.getCount()) {
              throw new IllegalStateException("The content of the adapter has changed but "
                      + "ListView did not receive a notification. Make sure the content of "
                      + "your adapter is not modified from a background thread, but only from "
                      + "the UI thread. Make sure your adapter calls notifyDataSetChanged() "
                      + "when its content changes. [in ListView(" + getId() + ", " + getClass()
                      + ") with Adapter(" + mAdapter.getClass() + ")]");
          }

          setSelectedPositionInt(mNextSelectedPosition);
          //...

          // Remember which child, if any, had accessibility focus. This must
          // occur before recycling any views, since that will clear
          // accessibility focus.
          //...
          //...

          // Take focus back to us temporarily to avoid the eventual call to
          // clear focus when removing the focused child below from messing
          // things up when ViewAncestor assigns focus back to someone else.
          //...

          // Pull all children into the RecycleBin.These views will be reused if possible
          final int firstPosition = mFirstPosition;
          final RecycleBin recycleBin = mRecycler;
          if (dataChanged) {
              for (int i = 0; i < childCount; i++) {
                  recycleBin.addScrapView(getChildAt(i), firstPosition+i);
              }
          } else {
              recycleBin.fillActiveViews(childCount, firstPosition);//activeViews存储的是ListView中所有的View
          }
          // Clear out old views，结合上下文，应该是把当前ChildView 丢到RecycleBin中缓存，之后移除全部，这样才不会出现重复的View
          detachAllViewsFromParent();
          recycleBin.removeSkippedScrap();

          switch (mLayoutMode) {
          case LAYOUT_SET_SELECTION:
              if (newSel != null) {
                  sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
              } else {
                  sel = fillFromMiddle(childrenTop, childrenBottom);
              }
              break;
          case LAYOUT_SYNC:
              sel = fillSpecific(mSyncPosition, mSpecificTop);
              break;
          case LAYOUT_FORCE_BOTTOM:
              sel = fillUp(mItemCount - 1, childrenBottom);
              adjustViewsUpOrDown();
              break;
          case LAYOUT_FORCE_TOP:
              mFirstPosition = 0;
              sel = fillFromTop(childrenTop);
              adjustViewsUpOrDown();
              break;
          case LAYOUT_SPECIFIC:
              sel = fillSpecific(reconcileSelectedPosition(), mSpecificTop);
              break;
          case LAYOUT_MOVE_SELECTION:
              sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
              break;
          default://默认
              if (childCount == 0) {
                //在还没有数据的情况下，以下进行了把Adapter中的数据填充到布局中，填充的界限根据最大高度(mBottom - mTop)或者Adapter数量决定
                  if (!mStackFromBottom) {//mStackFromBottom默认为false
                    //从上到下填充View
                      final int position = lookForSelectablePosition(0, true);//Find a position that can be selected (i.e., is not a separator).
                      setSelectedPositionInt(position);//记录seelctedPosition和rowId
                      //Fills the list from top to bottom(childrenTop), starting with mFirstPosition
                      //fillFromTop主要是对FirstPosition的取值进行判断，
                      sel = fillFromTop(childrenTop);
                  } else {
                    //从下到上
                      final int position = lookForSelectablePosition(mItemCount - 1, false);
                      setSelectedPositionInt(position);
                      sel = fillUp(mItemCount - 1, childrenBottom);
                  }
                  //进行以上步骤后，ListView这个容器已经有了足够填充一屏或者Adapter最大数量的ItemView，下次的layout就不会再进入到这里的逻辑了
              } else {
                  if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                      sel = fillSpecific(mSelectedPosition,
                              oldSel == null ? childrenTop : oldSel.getTop());
                  } else if (mFirstPosition < mItemCount) {
                      sel = fillSpecific(mFirstPosition,
                              oldFirst == null ? childrenTop : oldFirst.getTop());
                  } else {
                      sel = fillSpecific(0, childrenTop);//Put a specific item at a specific location on the screen and then buildup and down from there.这里和fillDown一致
                  }
              }
              break;
          }

          // Flush any cached views that did not get reused above
          recycleBin.scrapActiveViews();

          if (sel != null) {
              // The current selected item should get focus if items are
              // focusable.
              if (mItemsCanFocus && hasFocus() && !sel.hasFocus()) {
                  final boolean focusWasTaken = (sel == focusLayoutRestoreDirectChild &&
                          focusLayoutRestoreView != null &&
                          focusLayoutRestoreView.requestFocus()) || sel.requestFocus();
                  if (!focusWasTaken) {
                      // Selected item didn't take focus, but we still want to
                      // make sure something else outside of the selected view
                      // has focus.
                      final View focused = getFocusedChild();
                      if (focused != null) {
                          focused.clearFocus();
                      }
                      positionSelector(INVALID_POSITION, sel);
                  } else {
                      sel.setSelected(false);
                      mSelectorRect.setEmpty();
                  }
              } else {
                  positionSelector(INVALID_POSITION, sel);
              }
              mSelectedTop = sel.getTop();
          } else {
              final boolean inTouchMode = mTouchMode == TOUCH_MODE_TAP
                      || mTouchMode == TOUCH_MODE_DONE_WAITING;
              if (inTouchMode) {
                  // If the user's finger is down, select the motion position.
                  final View child = getChildAt(mMotionPosition - mFirstPosition);
                  if (child != null) {
                      positionSelector(mMotionPosition, child);
                  }
              } else if (mSelectorPosition != INVALID_POSITION) {
                  // If we had previously positioned the selector somewhere,
                  // put it back there. It might not match up with the data,
                  // but it's transitioning out so it's not a big deal.
                  final View child = getChildAt(mSelectorPosition - mFirstPosition);
                  if (child != null) {
                      positionSelector(mSelectorPosition, child);
                  }
              } else {
                  // Otherwise, clear selection.
                  mSelectedTop = 0;
                  mSelectorRect.setEmpty();
              }

              // Even if there is not selected position, we may need to
              // restore focus (i.e. something focusable in touch mode).
              if (hasFocus() && focusLayoutRestoreView != null) {
                  focusLayoutRestoreView.requestFocus();
              }
          }

          // Attempt to restore accessibility focus, if necessary.
          //...

          // Tell focus view we are done mucking with it, if it is still in
          // our view hierarchy.
          if (focusLayoutRestoreView != null
                  && focusLayoutRestoreView.getWindowToken() != null) {
              focusLayoutRestoreView.onFinishTemporaryDetach();
          }

          mLayoutMode = LAYOUT_NORMAL;
          mDataChanged = false;
          if (mPositionScrollAfterLayout != null) {
              post(mPositionScrollAfterLayout);
              mPositionScrollAfterLayout = null;
          }
          mNeedSync = false;
          setNextSelectedPositionInt(mSelectedPosition);

          updateScrollIndicators();

          if (mItemCount > 0) {
              checkSelectionChanged();
          }

          invokeOnItemScrollListener();
      } finally {
          if (!blockLayoutRequests) {
              mBlockLayoutRequests = false;
          }
      }
  }
```

`fillDown`，从上到下填充View,填充的界限根据最大高度(mBottom - mTop)或者Adapter数量决定
结合下面的`makeAndAddView`方法很好理解。
```java
/**
   * Fills the list from pos down to the end of the list view.
   *
   * @param pos The first position to put in the list
   *
   * @param nextTop The location where the top of the item associated with pos
   *        should be drawn
   *
   * @return The view that is currently selected, if it happens to be in the
   *         range that we draw.
   */
private View fillDown(int pos, int nextTop) {
    View selectedView = null;

    int end = (mBottom - mTop);
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
        end -= mListPadding.bottom;
    }
    while (nextTop < end && pos < mItemCount) {
        // is this the selected item?
        boolean selected = pos == mSelectedPosition;
        View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);
        nextTop = child.getBottom() + mDividerHeight;
        if (selected) {
            selectedView = child;
        }
        pos++;
    }
    setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);//removeView相关  忽略
    return selectedView;
}
```

`makeAndAddView`，根据位置从RecyleBind中的ActiveViews中获取View,ItemView的来源可能是一个刚新建的没有用过的View（因为刚开始的时候
我们也没缓存）也可能是从RecyleBin中获取，ActiveViews记录的是上一次布局之前的在ListView中所有可缓存的ItemView。
`obtainView`方法在AbsListView，根据位置获取ItemView，首先从TransientStateView中获取，之后才从scrapView中查找，在没有的情况下，或者
缓存中的View和Adapter.getView返回来View不一致（通常也是新建），这种情况下需要进行缓存

```java
/**
    * Obtain the view and add it to our list of children. The view can be made
    * fresh, converted from an unused view, or used as is if it was in the
    * recycle bin.
    *
    * @param position Logical position in the list
    * @param y Top or bottom edge of the view to add
    * @param flow If flow is true, align top edge to y. If false, align bottom
    *        edge to y.
    * @param childrenLeft Left edge where children should be positioned
    * @param selected Is this position selected?
    * @return View that was added
    */
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
         boolean selected) {
     View child;
     if (!mDataChanged) {
         // Try to use an existing view for this position
         child = mRecycler.getActiveView(position);
         if (child != null) {
             // Found it -- we're using an existing child
             // This just needs to be positioned
             setupChild(child, position, y, flow, childrenLeft, selected, true);
             return child;
         }
     }

     // 因为在ActiveViews没找到可用的废弃的ItemView，就尝试在transientView和ScrapView中寻找，或者新建
     child = obtainView(position, mIsScrap);

     // This needs to be positioned and measured
     setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

     return child;
 }

```

`makeAndAddView`方法只是从RecyleBin中获取某个位置的ItemView，而__该方法才是把ItemView添加到ListView中，并排版（layout）__
在ItemView不是从RecyleBin中取出，或者Selected状态不一致或者ItemView请求layoutRequest的情况下都需要重新测量
另外注意到最后一个参数 recyled，标识当前设置的View是否从RecyleBin缓存中取得，如果不是，__调用的是`attachViewToParent`__ 方式
把新的View添加到ListView，如果是，只是把之前被`attachViewToParent`的View__重新`attachViewToParent`__ 到ListView上。

```java
/**
    * Add a view as a child and make sure it is measured (if necessary) and
    * positioned properly.
    * @param child The view to add
    * @param position The position of this child
    * @param y The y position relative to which this view will be positioned
    * @param flowDown If true, align top edge to y. If false, align bottom
    *        edge to y.是否从上往下布局排版
    * @param childrenLeft Left edge where children should be positioned
    * @param selected Is this position selected?
    * @param recycled Has this view been pulled from the recycle bin? If so it
    *        does not need to be remeasured.
    */
   private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
           boolean selected, boolean recycled) {
       //setupListItem
       final boolean isSelected = selected && shouldShowSelector();
       final boolean updateChildSelected = isSelected != child.isSelected();
       final int mode = mTouchMode;
       final boolean isPressed = mode > TOUCH_MODE_DOWN && mode < TOUCH_MODE_SCROLL &&
               mMotionPosition == position;
       final boolean updateChildPressed = isPressed != child.isPressed();
       final boolean needToMeasure = !recycled || updateChildSelected || child.isLayoutRequested();

       // Respect layout params that are already in the view. Otherwise make some up...
       // noinspection unchecked
       AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
       if (p == null) {
           p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
       }
       p.viewType = mAdapter.getItemViewType(position);

       if ((recycled && !p.forceAdd) || (p.recycledHeaderFooter &&
               p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
          //与'addView'方法不同，把view加入到children集合中，而'addView'方法在加入children集合中之前进行了
          //`requestLayout`和`invalidate`,并且可以接收到`OnHierarchyChangeListener`回调
           attachViewToParent(child, flowDown ? -1 : 0, p);
       } else {
           p.forceAdd = false;//跳到这里，有可能是因为forceAdd为true
           if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
               p.recycledHeaderFooter = true;
           }
           addViewInLayout(child, flowDown ? -1 : 0, p, true);//adds a view during layout
       }

       if (updateChildSelected) {
           child.setSelected(isSelected);
       }

       if (updateChildPressed) {
           child.setPressed(isPressed);
       }

       if (mChoiceMode != CHOICE_MODE_NONE && mCheckStates != null) {
           if (child instanceof Checkable) {
               ((Checkable) child).setChecked(mCheckStates.get(position));
           } else if (getContext().getApplicationInfo().targetSdkVersion
                   >= android.os.Build.VERSION_CODES.HONEYCOMB) {
               child.setActivated(mCheckStates.get(position));
           }
       }

       if (needToMeasure) {
           int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                   mListPadding.left + mListPadding.right, p.width);
           int lpHeight = p.height;
           int childHeightSpec;
           if (lpHeight > 0) {
               childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
           } else {
               childHeightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
           }
           child.measure(childWidthSpec, childHeightSpec);
       } else {
           cleanupLayoutState(child);// Prevents the specified child to be laid out during the next layout pass.
       }

       final int w = child.getMeasuredWidth();
       final int h = child.getMeasuredHeight();
       final int childTop = flowDown ? y : y - h;

       if (needToMeasure) {
           final int childRight = childrenLeft + w;
           final int childBottom = childTop + h;
           child.layout(childrenLeft, childTop, childRight, childBottom);
       } else {
         //Offset this view's horizontal（vertical） location by the specified amount of pixels.
           child.offsetLeftAndRight(childrenLeft - child.getLeft());
           child.offsetTopAndBottom(childTop - child.getTop());
       }
       if (mCachingStarted && !child.isDrawingCacheEnabled()) {
           child.setDrawingCacheEnabled(true);
       }

       if (recycled && (((AbsListView.LayoutParams)child.getLayoutParams()).scrappedFromPosition)
               != position) {
           child.jumpDrawablesToCurrentState();
       }
   }

```

## setAdapter

会根据是否有Header和Footer来对Adapter进行了改造，出现了新的Adapter类型`HeaderViewListAdapter`,这里就先忽略呗，
并在这里初始化RecyleBin的ViewType数组,设置完之后需要重新布局，再次调用`layoutChildren`:

```java
private ArrayList<FixedViewInfo> mHeaderViewInfos = Lists.newArrayList();
private ArrayList<FixedViewInfo> mFooterViewInfos = Lists.newArrayList();

@Override
public void setAdapter(ListAdapter adapter) {
    if (mAdapter != null && mDataSetObserver != null) {
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
    }

    resetList();//移除layout中所有View，清楚各种标志位，包括footer和header的状态
    mRecycler.clear();//清楚RecyleBind中的所有View，

    if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
        mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
    } else {
        mAdapter = adapter;
    }

    mOldSelectedPosition = INVALID_POSITION;
    mOldSelectedRowId = INVALID_ROW_ID;

    // AbsListView#setAdapter will update choice mode states.
    super.setAdapter(adapter);
    if (mAdapter != null) {
        mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();//默认为true
        mOldItemCount = mItemCount;
        mItemCount = mAdapter.getCount();
        checkFocus();

        mDataSetObserver = new AdapterDataSetObserver();
        mAdapter.registerDataSetObserver(mDataSetObserver);

        mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());//

        int position;
        if (mStackFromBottom) {//默认false，从上往下布局
            position = lookForSelectablePosition(mItemCount - 1, false);//Find a position that can be selected
        } else {
            position = lookForSelectablePosition(0, true);
        }
        setSelectedPositionInt(position);//Utility to keep mSelectedPosition and mSelectedRowId in sync
        setNextSelectedPositionInt(position);//position Intended value for mSelectedPosition the next time we go through layout
        if (mItemCount == 0) {
            // Nothing selected
            checkSelectionChanged();
        }
    } else {
        mAreAllItemsSelectable = true;
        checkFocus();
        // Nothing selected
        checkSelectionChanged();
    }
    requestLayout();
}
```

## 手势滑动

通过`layoutChildren`的分析，我们已经成功地填充了一屏的数据，那之后的数据又是如何填充到界面上的？我们都知道当一个Item移除界面，就会被重新进入
界面的Item所复用，那么这个过程到底是怎么处理的？
__AbsListView#onTouchEvent__

```java
@Override
    public boolean onTouchEvent(MotionEvent ev) {
        if (!isEnabled()) {
            // A disabled view that is clickable still consumes the touch events, it just doesn't respond to them.
            return isClickable() || isLongClickable();
        }

        if (mPositionScroller != null) {
            mPositionScroller.stop();//andles scrolling between positions within the list.
        }

        if (mIsDetaching || !isAttachedToWindow()) {
            // Something isn't right.
            // Since we rely on being attached to get data set change notifications,
            // don't risk doing anything where we might try to resync and find things
            // in a bogus state.
            return false;
        }

        startNestedScroll(SCROLL_AXIS_VERTICAL);//Begin a nestable scroll operation along the given axes.

        if (mFastScroll != null) {//Helper object that renders and controls the fast scroll thumb.忽略
            boolean intercepted = mFastScroll.onTouchEvent(ev);
            if (intercepted) {
                return true;
            }
        }

        initVelocityTrackerIfNotExists();
        final MotionEvent vtev = MotionEvent.obtain(ev);

        final int actionMasked = ev.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            mNestedYOffset = 0;
        }
        vtev.offsetLocation(0, mNestedYOffset);
        switch (actionMasked) {
            case MotionEvent.ACTION_DOWN: {
                onTouchDown(ev);
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                onTouchMove(ev, vtev);
                break;
            }

            case MotionEvent.ACTION_UP: {
                onTouchUp(ev);
                break;
            }

            case MotionEvent.ACTION_CANCEL: {
                onTouchCancel();
                break;
            }

            case MotionEvent.ACTION_POINTER_UP: {
                onSecondaryPointerUp(ev);
                final int x = mMotionX;
                final int y = mMotionY;
                final int motionPosition = pointToPosition(x, y);
                if (motionPosition >= 0) {
                    // Remember where the motion event started
                    final View child = getChildAt(motionPosition - mFirstPosition);
                    mMotionViewOriginalTop = child.getTop();
                    mMotionPosition = motionPosition;
                }
                mLastY = y;
                break;
            }

            case MotionEvent.ACTION_POINTER_DOWN: {
                // New pointers take over dragging duties
                final int index = ev.getActionIndex();
                final int id = ev.getPointerId(index);
                final int x = (int) ev.getX(index);
                final int y = (int) ev.getY(index);
                mMotionCorrection = 0;
                mActivePointerId = id;
                mMotionX = x;
                mMotionY = y;
                final int motionPosition = pointToPosition(x, y);
                if (motionPosition >= 0) {
                    // Remember where the motion event started
                    final View child = getChildAt(motionPosition - mFirstPosition);
                    mMotionViewOriginalTop = child.getTop();
                    mMotionPosition = motionPosition;
                }
                mLastY = y;
                break;
            }
        }
        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(vtev);
        }
        vtev.recycle();
        return true;
    }
```
主要来分析一个滑动的过程__down,move,up__

`onTouchDown`

```java
private void onTouchDown(MotionEvent ev) {
    mActivePointerId = ev.getPointerId(0);
    //默认TOUCH_MODE_REST
    if (mTouchMode == TOUCH_MODE_OVERFLING) {
        // Stopped the fling. It is a scroll.
        mFlingRunnable.endFling();
        if (mPositionScroller != null) {
            mPositionScroller.stop();
        }
        mTouchMode = TOUCH_MODE_OVERSCROLL;
        mMotionX = (int) ev.getX();
        mMotionY = (int) ev.getY();
        mLastY = mMotionY;
        mMotionCorrection = 0;
        mDirection = 0;
    } else {
        final int x = (int) ev.getX();
        final int y = (int) ev.getY();
        int motionPosition = pointToPosition(x, y);//Maps a point to a position in the list.

        if (!mDataChanged) {
            if (mTouchMode == TOUCH_MODE_FLING) {
                // Stopped a fling. It is a scroll.
                createScrollingCache();
                mTouchMode = TOUCH_MODE_SCROLL;
                mMotionCorrection = 0;
                motionPosition = findMotionRow(y);
                mFlingRunnable.flywheelTouch();
            } else if ((motionPosition >= 0) && getAdapter().isEnabled(motionPosition)) {
                // User clicked on an actual view (and was not stopping a fling). It might be a click or a scroll.
                //Assume it is a click until proven otherwise.
                mTouchMode = TOUCH_MODE_DOWN;

                // FIXME Debounce
                if (mPendingCheckForTap == null) {
                    mPendingCheckForTap = new CheckForTap();
                }

                mPendingCheckForTap.x = ev.getX();
                mPendingCheckForTap.y = ev.getY();
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
            }
        }

        if (motionPosition >= 0) {
            // Remember where the motion event started
            final View v = getChildAt(motionPosition - mFirstPosition);
            mMotionViewOriginalTop = v.getTop();
        }

        mMotionX = x;
        mMotionY = y;
        mMotionPosition = motionPosition;
        mLastY = Integer.MIN_VALUE;
    }

    if (mTouchMode == TOUCH_MODE_DOWN && mMotionPosition != INVALID_POSITION
            && performButtonActionOnTouchDown(ev)) {
        removeCallbacks(mPendingCheckForTap);
    }
}
```

`onTouchMove`

```java
    private void onTouchMove(MotionEvent ev, MotionEvent vtev) {
        int pointerIndex = ev.findPointerIndex(mActivePointerId);
        if (pointerIndex == -1) {
            pointerIndex = 0;
            mActivePointerId = ev.getPointerId(pointerIndex);
        }

        if (mDataChanged) {
            // Re-sync everything if data has been changed since the scroll operation can query the adapter.
            layoutChildren();
        }

        final int y = (int) ev.getY(pointerIndex);

        switch (mTouchMode) {
            case TOUCH_MODE_DOWN:
            case TOUCH_MODE_TAP:
            case TOUCH_MODE_DONE_WAITING:
                // Check if we have moved far enough that it looks more like a
                // scroll than a tap. If so, we'll enter scrolling mode.
                if (startScrollIfNeeded((int) ev.getX(pointerIndex), y, vtev)) {
                    break;
                }
                // Otherwise, check containment within list bounds. If we're
                // outside bounds, cancel any active presses.
                final float x = ev.getX(pointerIndex);
                if (!pointInView(x, y, mTouchSlop)) {
                    setPressed(false);
                    final View motionView = getChildAt(mMotionPosition - mFirstPosition);
                    if (motionView != null) {
                        motionView.setPressed(false);
                    }
                    removeCallbacks(mTouchMode == TOUCH_MODE_DOWN ?
                            mPendingCheckForTap : mPendingCheckForLongPress);
                    mTouchMode = TOUCH_MODE_DONE_WAITING;
                    updateSelectorState();
                }
                break;
            case TOUCH_MODE_SCROLL:
            case TOUCH_MODE_OVERSCROLL:
                scrollIfNeeded((int) ev.getX(pointerIndex), y, vtev);
                break;
        }
    }
```

`onTouchUp`

```java
private void onTouchUp(MotionEvent ev) {
    switch (mTouchMode) {
    case TOUCH_MODE_DOWN:
    case TOUCH_MODE_TAP:
    case TOUCH_MODE_DONE_WAITING:
        final int motionPosition = mMotionPosition;
        final View child = getChildAt(motionPosition - mFirstPosition);
        if (child != null) {
            if (mTouchMode != TOUCH_MODE_DOWN) {
                child.setPressed(false);
            }

            final float x = ev.getX();
            final boolean inList = x > mListPadding.left && x < getWidth() - mListPadding.right;
            if (inList && !child.hasFocusable()) {
                if (mPerformClick == null) {
                    mPerformClick = new PerformClick();
                }

                final AbsListView.PerformClick performClick = mPerformClick;
                performClick.mClickMotionPosition = motionPosition;
                performClick.rememberWindowAttachCount();

                mResurrectToPosition = motionPosition;

                if (mTouchMode == TOUCH_MODE_DOWN || mTouchMode == TOUCH_MODE_TAP) {
                    removeCallbacks(mTouchMode == TOUCH_MODE_DOWN ?
                            mPendingCheckForTap : mPendingCheckForLongPress);
                    mLayoutMode = LAYOUT_NORMAL;
                    if (!mDataChanged && mAdapter.isEnabled(motionPosition)) {
                        mTouchMode = TOUCH_MODE_TAP;
                        setSelectedPositionInt(mMotionPosition);
                        layoutChildren();
                        child.setPressed(true);
                        positionSelector(mMotionPosition, child);
                        setPressed(true);
                        if (mSelector != null) {
                            Drawable d = mSelector.getCurrent();
                            if (d != null && d instanceof TransitionDrawable) {
                                ((TransitionDrawable) d).resetTransition();
                            }
                            mSelector.setHotspot(x, ev.getY());
                        }
                        if (mTouchModeReset != null) {
                            removeCallbacks(mTouchModeReset);
                        }
                        mTouchModeReset = new Runnable() {
                            @Override
                            public void run() {
                                mTouchModeReset = null;
                                mTouchMode = TOUCH_MODE_REST;
                                child.setPressed(false);
                                setPressed(false);
                                if (!mDataChanged && !mIsDetaching && isAttachedToWindow()) {
                                    performClick.run();
                                }
                            }
                        };
                        postDelayed(mTouchModeReset,
                                ViewConfiguration.getPressedStateDuration());
                    } else {
                        mTouchMode = TOUCH_MODE_REST;
                        updateSelectorState();
                    }
                    return;
                } else if (!mDataChanged && mAdapter.isEnabled(motionPosition)) {
                    performClick.run();
                }
            }
        }
        mTouchMode = TOUCH_MODE_REST;
        updateSelectorState();
        break;
    case TOUCH_MODE_SCROLL:
        final int childCount = getChildCount();
        if (childCount > 0) {
            final int firstChildTop = getChildAt(0).getTop();
            final int lastChildBottom = getChildAt(childCount - 1).getBottom();
            final int contentTop = mListPadding.top;
            final int contentBottom = getHeight() - mListPadding.bottom;
            if (mFirstPosition == 0 && firstChildTop >= contentTop &&
                    mFirstPosition + childCount < mItemCount &&
                    lastChildBottom <= getHeight() - contentBottom) {
                mTouchMode = TOUCH_MODE_REST;
                reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
            } else {
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);

                final int initialVelocity = (int)
                        (velocityTracker.getYVelocity(mActivePointerId) * mVelocityScale);
                // Fling if we have enough velocity and we aren't at a boundary.
                // Since we can potentially overfling more than we can overscroll, don't
                // allow the weird behavior where you can scroll to a boundary then
                // fling further.
                boolean flingVelocity = Math.abs(initialVelocity) > mMinimumVelocity;
                if (flingVelocity &&
                        !((mFirstPosition == 0 &&
                                firstChildTop == contentTop - mOverscrollDistance) ||
                          (mFirstPosition + childCount == mItemCount &&
                                lastChildBottom == contentBottom + mOverscrollDistance))) {
                    if (!dispatchNestedPreFling(0, -initialVelocity)) {
                        if (mFlingRunnable == null) {
                            mFlingRunnable = new FlingRunnable();
                        }
                        reportScrollStateChange(OnScrollListener.SCROLL_STATE_FLING);
                        mFlingRunnable.start(-initialVelocity);
                        dispatchNestedFling(0, -initialVelocity, true);
                    } else {
                        mTouchMode = TOUCH_MODE_REST;
                        reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                    }
                } else {
                    mTouchMode = TOUCH_MODE_REST;
                    reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                    if (mFlingRunnable != null) {
                        mFlingRunnable.endFling();
                    }
                    if (mPositionScroller != null) {
                        mPositionScroller.stop();
                    }
                    if (flingVelocity && !dispatchNestedPreFling(0, -initialVelocity)) {
                        dispatchNestedFling(0, -initialVelocity, false);
                    }
                }
            }
        } else {
            mTouchMode = TOUCH_MODE_REST;
            reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
        }
        break;

    case TOUCH_MODE_OVERSCROLL:
        if (mFlingRunnable == null) {
            mFlingRunnable = new FlingRunnable();
        }
        final VelocityTracker velocityTracker = mVelocityTracker;
        velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
        final int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);

        reportScrollStateChange(OnScrollListener.SCROLL_STATE_FLING);
        if (Math.abs(initialVelocity) > mMinimumVelocity) {
            mFlingRunnable.startOverfling(-initialVelocity);
        } else {
            mFlingRunnable.startSpringback();
        }

        break;
    }

    setPressed(false);

    if (mEdgeGlowTop != null) {
        mEdgeGlowTop.onRelease();
        mEdgeGlowBottom.onRelease();
    }

    // Need to redraw since we probably aren't drawing the selector anymore
    invalidate();
    removeCallbacks(mPendingCheckForLongPress);
    recycleVelocityTracker();

    mActivePointerId = INVALID_POINTER;

    if (PROFILE_SCROLLING) {
        if (mScrollProfilingStarted) {
            Debug.stopMethodTracing();
            mScrollProfilingStarted = false;
        }
    }

    if (mScrollStrictSpan != null) {
        mScrollStrictSpan.finish();
        mScrollStrictSpan = null;
    }
}
```

## Header 和 Footer分析
