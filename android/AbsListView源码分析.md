# AbsListView源码分析
## 属性介绍
在分析一个View源码之前，首先需要的是对View本身的使用的了解，包括它特有的属性：
* android.R.styleable#AbsListView_listSelector
* android.R.styleable#AbsListView_drawSelectorOnTop

默认false
* android.R.styleable#AbsListView_stackFromBottom

默认false
* android.R.styleable#AbsListView_scrollingCache

默认true
* android.R.styleable#AbsListView_textFilterEnabled

默认false
* android.R.styleable#AbsListView_transcriptMode

默认TRANSCRIPT_MODE_DISABLED，其他选择TRANSCRIPT_MODE_NORMAL，TRANSCRIPT_MODE_ALWAYS_SCROLL
* android.R.styleable#AbsListView_cacheColorHint

Indicates that this list will always be drawn on top of solid, single-color opaque background.
* android.R.styleable#AbsListView_fastScrollEnabled

默认false，[fastScroll介绍][fastScroll]
* android.R.styleable#AbsListView_smoothScrollbar

默认true

* android.R.styleable#AbsListView_choiceMode

Defines the choice behavior for the view. 有单选，多选等模式,默认无。

## 构造

```java
public AbsListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
       super(context, attrs, defStyleAttr, defStyleRes);
       initAbsListView();

       mOwnerThread = Thread.currentThread();
       //...
       //....省略若干属性的获取代码....
       //...

       setFastScrollAlwaysVisible(
               a.getBoolean(R.styleable.AbsListView_fastScrollAlwaysVisible, false));
       a.recycle();
   }

   private void initAbsListView() {
      // Setting focusable in touch mode will set the focusable property to true
      setClickable(true);
      setFocusableInTouchMode(true);
      setWillNotDraw(false);
      setAlwaysDrawnWithCacheEnabled(false);
      setScrollingCacheEnabled(true);

      final ViewConfiguration configuration = ViewConfiguration.get(mContext);
      mTouchSlop = configuration.getScaledTouchSlop();
      mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
      mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
      mOverscrollDistance = configuration.getScaledOverscrollDistance();
      mOverflingDistance = configuration.getScaledOverflingDistance();

      mDensityScale = getContext().getResources().getDisplayMetrics().density;
  }
```

## AbsListView#LayoutParms

比较特别的是，记录了ViewType,Id,回收堆中的位置等。
```java
/**
     * AbsListView extends LayoutParams to provide a place to hold the view type.
     */
    public static class LayoutParams extends ViewGroup.LayoutParams {
        /**
         * View type for this view, as returned by
         * {@link android.widget.Adapter#getItemViewType(int) }
         */
        int viewType;

        /**
         * When this boolean is set, the view has been added to the AbsListView
         * at least once. It is used to know whether headers/footers have already
         * been added to the list view and whether they should be treated as
         * recycled views or not.
         */
        boolean recycledHeaderFooter;

        /**
         * When an AbsListView is measured with an AT_MOST measure spec, it needs
         * to obtain children views to measure itself. When doing so, the children
         * are not attached to the window, but put in the recycler which assumes
         * they've been attached before. Setting this flag will force the reused
         * view to be attached to the window rather than just attached to the
         * parent.
         */
        boolean forceAdd;

        /**
         * The position the view was removed from when pulled out of the
         * scrap heap.
         * @hide
         */
        int scrappedFromPosition;

        long itemId = -1;
        //....
    }

```
### 在`setItemViewLayoutParams`方法中设置
```java
private void setItemViewLayoutParams(View child, int position) {
        final ViewGroup.LayoutParams vlp = child.getLayoutParams();
        LayoutParams lp;
        if (vlp == null) {
            lp = (LayoutParams) generateDefaultLayoutParams();
        } else if (!checkLayoutParams(vlp)) {
            lp = (LayoutParams) generateLayoutParams(vlp);
        } else {
            lp = (LayoutParams) vlp;
        }

        if (mAdapterHasStableIds) {
            lp.itemId = mAdapter.getItemId(position);
        }
        lp.viewType = mAdapter.getItemViewType(position);
        child.setLayoutParams(lp);
    }
```
### 获取一Item View的过程中更新LayoutParams

根据位置获取ItemView，首先从TransientStateView中获取，之后才从scrapView中查找，在没有的情况下，或者
缓存中的View和Adapter.getView返回来View不一致（通常也是新建），这种情况下需要进行缓存
```java
/**
     * Get a view and have it show the data associated with the specified
     * position. This is called when we have already discovered that the view is
     * not available for reuse in the recycle bin. The only choices left are
     * converting an old view or making a new one.
     *
     * @param position The position to display
     * @param isScrap Array of at least 1 boolean, the first entry will become true if
     *                the returned view was taken from the scrap heap, false if otherwise.
     *
     * @return A view displaying the data associated with the specified position
     */
    View obtainView(int position, boolean[] isScrap) {
        isScrap[0] = false;//用来标识是否是从scrapView中获取ItemView
        // Check whether we have a transient state view. Attempt to re-bind the
        // data and discard the view if we fail.
        final View transientView = mRecycler.getTransientStateView(position);
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            // If the view type hasn't changed, attempt to re-bind the data.
            if (params.viewType == mAdapter.getItemViewType(position)) {
                final View updatedView = mAdapter.getView(position, transientView, this);
                // If we failed to re-bind the data, scrap the obtained view.
                if (updatedView != transientView) {
                    setItemViewLayoutParams(updatedView, position);
                    mRecycler.addScrapView(updatedView, position);
                }
            }
            // Scrap view implies temporary detachment.
            isScrap[0] = true;
            return transientView;
        }

        final View scrapView = mRecycler.getScrapView(position);
        //一般情况下，我们从RecyleBin中获取缓存的View，再通过Adapter的getView方法，
        //scrapView为空，也就是converView为空，需要我们新建ItemView,否则只需要我们刷新数据即可，避免重复创建
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else {
                isScrap[0] = true;//标识从RecyleBin获取的ItemView

                child.dispatchFinishTemporaryDetach();
            }
        }

        if (mCacheColorHint != 0) {
            child.setDrawingCacheBackgroundColor(mCacheColorHint);
        }
        //...
        setItemViewLayoutParams(child, position);
        //....
        return child;
    }

```

## RecycleBin

在分析Gallery源码的时候已经分析到过一次`AbsSpinner#RecycleBin`，不同的是`AbsSpinner#RecycleBin`中并没有缓存机制，每次获取ITEM都是一次View的新建
AbsListView#RecycleBin拥有两种层次的存储机制：ActiveViews and ScrapViews（有废弃的意思）。ActiveViews是那些在开始layout的时候展现在屏幕界面上的view，在布局完
之后，ActiveViews就会“贬为”ScrapViews。ScrapViews相对来说是旧的View，为了避免不必要的创建View，它们有可能会被Adapter重复使用。

```java
public static interface RecyclerListener {
       /**
        * Indicates that the specified View was moved into the recycler's scrap heap.
        * The view is not displayed on screen any more and any expensive resource
        * associated with the view should be discarded.
        *
        * @param view
        */
       void onMovedToScrapHeap(View view);
   }

class RecycleBin {
    private RecyclerListener mRecyclerListener;

    /**
     * The position of the first view stored in mActiveViews.
     */
    private int mFirstActivePosition;

    /**
     * Views that were on screen at the start of layout. This array is populated at the start of
     * layout, and at the end of layout all view in mActiveViews are moved to mScrapViews.
     * Views in mActiveViews represent a contiguous range of Views, with position of the first
     * view store in mFirstActivePosition.
     */
    private View[] mActiveViews = new View[0];

    /**
     * Unsorted views that can be used by the adapter as a convert view.
     */
    private ArrayList<View>[] mScrapViews;

    private int mViewTypeCount;

    private ArrayList<View> mCurrentScrap;

    private ArrayList<View> mSkippedScrap;
    //下面两个字段应该和`transcriptMode`相关，`transcriptMode`默认关闭，大可先忽略
    private SparseArray<View> mTransientStateViews;
    private LongSparseArray<View> mTransientStateViewsById;

    public void setViewTypeCount(int viewTypeCount) {
        if (viewTypeCount < 1) {
            throw new IllegalArgumentException("Can't have a viewTypeCount < 1");//viewTypeCount
        }
        //重这里可以看出，会为每种类型的子View创建对应的scrapViews，而且index是按顺序的，这也是为什么当我们的的viewType不
        //连续的时候，会出现IndexOutOfBoundException
        ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
        for (int i = 0; i < viewTypeCount; i++) {
            scrapViews[i] = new ArrayList<View>();
        }
        mViewTypeCount = viewTypeCount;
        mCurrentScrap = scrapViews[0];
        mScrapViews = scrapViews;
    }
    //对子View调用forceLayout
    public void markChildrenDirty() {
        if (mViewTypeCount == 1) {
            final ArrayList<View> scrap = mCurrentScrap;
            final int scrapCount = scrap.size();
            for (int i = 0; i < scrapCount; i++) {
                scrap.get(i).forceLayout();
            }
        } else {
            final int typeCount = mViewTypeCount;
            for (int i = 0; i < typeCount; i++) {
                final ArrayList<View> scrap = mScrapViews[i];
                final int scrapCount = scrap.size();
                for (int j = 0; j < scrapCount; j++) {
                    scrap.get(j).forceLayout();
                }
            }
        }
        if (mTransientStateViews != null) {
            final int count = mTransientStateViews.size();
            for (int i = 0; i < count; i++) {
                mTransientStateViews.valueAt(i).forceLayout();
            }
        }
        if (mTransientStateViewsById != null) {
            final int count = mTransientStateViewsById.size();
            for (int i = 0; i < count; i++) {
                mTransientStateViewsById.valueAt(i).forceLayout();
            }
        }
    }

    public boolean shouldRecycleViewType(int viewType) {
        return viewType >= 0;
    }

    /**
     * Clears the scrap heap.
     */
    void clear() {
        if (mViewTypeCount == 1) {
            final ArrayList<View> scrap = mCurrentScrap;
            clearScrap(scrap);
        } else {
            final int typeCount = mViewTypeCount;
            for (int i = 0; i < typeCount; i++) {
                final ArrayList<View> scrap = mScrapViews[i];
                clearScrap(scrap);
            }
        }

        clearTransientStateViews();
    }

    /**
     *仅仅把当前AbsListView的所有child中需要缓存的View缓存到ActivViews，
     * Fill ActiveViews with all of the children of the AbsListView.
     *
     * @param childCount The minimum number of views mActiveViews should hold
     * @param firstActivePosition The position of the first view that will be stored in
     *        mActiveViews
     */
    void fillActiveViews(int childCount, int firstActivePosition) {
        if (mActiveViews.length < childCount) {///默认长度为0，在layout之前扩展
            mActiveViews = new View[childCount];
        }
        mFirstActivePosition = firstActivePosition;

        //noinspection MismatchedReadAndWriteOfArray
        final View[] activeViews = mActiveViews;
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
            // Don't put header or footer views into the scrap heap
            if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                // Note:  We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in active views.
                //        However, we will NOT place them into scrap views.
                activeViews[i] = child;
            }
        }
    }

    /**
     * Get the view corresponding to the specified position. The view will be removed from
     * mActiveViews if it is found.
     *
     * @param position The position to look up in mActiveViews
     * @return The view if it is found, null otherwise
     */
    View getActiveView(int position) {
        int index = position - mFirstActivePosition;
        final View[] activeViews = mActiveViews;
        if (index >=0 && index < activeViews.length) {
            final View match = activeViews[index];
            activeViews[index] = null;
            return match;
        }
        return null;
    }

    View getTransientStateView(int position) {
        if (mAdapter != null && mAdapterHasStableIds && mTransientStateViewsById != null) {
            long id = mAdapter.getItemId(position);
            View result = mTransientStateViewsById.get(id);
            mTransientStateViewsById.remove(id);
            return result;
        }
        if (mTransientStateViews != null) {
            final int index = mTransientStateViews.indexOfKey(position);
            if (index >= 0) {
                View result = mTransientStateViews.valueAt(index);
                mTransientStateViews.removeAt(index);
                return result;
            }
        }
        return null;
    }

    /**
     * Dumps and fully detaches any currently saved views with transient
     * state.
     */
    void clearTransientStateViews() {
        final SparseArray<View> viewsByPos = mTransientStateViews;
        if (viewsByPos != null) {
            final int N = viewsByPos.size();
            for (int i = 0; i < N; i++) {
                removeDetachedView(viewsByPos.valueAt(i), false);
            }
            viewsByPos.clear();
        }

        final LongSparseArray<View> viewsById = mTransientStateViewsById;
        if (viewsById != null) {
            final int N = viewsById.size();
            for (int i = 0; i < N; i++) {
                removeDetachedView(viewsById.valueAt(i), false);
            }
            viewsById.clear();
        }
    }

    /**
     *从缓存中获取ItemView,会有考虑到ViewType，只取ViewType一致的
     * @return A view from the ScrapViews collection. These are unordered.
     */
    View getScrapView(int position) {
        if (mViewTypeCount == 1) {
            return retrieveFromScrap(mCurrentScrap, position);
        } else {
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap >= 0 && whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
        }
        return null;
    }

    /**
     * Puts a view into the list of scrap views.
     * <p>
     * If the list data hasn't changed or the adapter has stable IDs, views
     * with transient state will be preserved for later retrieval.
     *
     * @param scrap The view to add
     * @param position The view's position within its parent
     */
    void addScrapView(View scrap, int position) {
        final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
        if (lp == null) {
            return;
        }

        lp.scrappedFromPosition = position;//position仅仅记录在LayoutParms,与在ScrapView中的存放位置无关

        // Remove but don't scrap header or footer views, or views that
        // should otherwise not be recycled.
        final int viewType = lp.viewType;
        if (!shouldRecycleViewType(viewType)) {
            return;
        }

        scrap.dispatchStartTemporaryDetach();

        // The the accessibility state of the view may change while temporary
        // detached and we do not allow detached views to fire accessibility
        // events. So we are announcing that the subtree changed giving a chance
        // to clients holding on to a view in this subtree to refresh it.
        notifyViewAccessibilityStateChangedIfNeeded(
                AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);

        // Don't scrap views that have transient state.
        final boolean scrapHasTransientState = scrap.hasTransientState();
        if (scrapHasTransientState) {
            if (mAdapter != null && mAdapterHasStableIds) {
                // If the adapter has stable IDs, we can reuse the view for
                // the same data.
                if (mTransientStateViewsById == null) {
                    mTransientStateViewsById = new LongSparseArray<View>();
                }
                mTransientStateViewsById.put(lp.itemId, scrap);
            } else if (!mDataChanged) {
                // If the data hasn't changed, we can reuse the views at
                // their old positions.
                if (mTransientStateViews == null) {
                    mTransientStateViews = new SparseArray<View>();
                }
                mTransientStateViews.put(position, scrap);
            } else {
                // Otherwise, we'll have to remove the view and start over.
                if (mSkippedScrap == null) {
                    mSkippedScrap = new ArrayList<View>();
                }
                mSkippedScrap.add(scrap);
            }
        } else {
            if (mViewTypeCount == 1) {
                mCurrentScrap.add(scrap);
            } else {
                mScrapViews[viewType].add(scrap);
            }

            if (mRecyclerListener != null) {
                mRecyclerListener.onMovedToScrapHeap(scrap);
            }
        }
    }

    /**
     * Finish the removal of any views that skipped the scrap heap.
     */
    void removeSkippedScrap() {
        if (mSkippedScrap == null) {
            return;
        }
        final int count = mSkippedScrap.size();
        for (int i = 0; i < count; i++) {
            removeDetachedView(mSkippedScrap.get(i), false);
        }
        mSkippedScrap.clear();
    }

    /**
     * Move all views remaining in mActiveViews to mScrapViews.
     */
    void scrapActiveViews() {
        final View[] activeViews = mActiveViews;
        final boolean hasListener = mRecyclerListener != null;
        final boolean multipleScraps = mViewTypeCount > 1;

        ArrayList<View> scrapViews = mCurrentScrap;
        final int count = activeViews.length;
        for (int i = count - 1; i >= 0; i--) {
            final View victim = activeViews[i];
            if (victim != null) {
                final AbsListView.LayoutParams lp
                        = (AbsListView.LayoutParams) victim.getLayoutParams();
                final int whichScrap = lp.viewType;

                activeViews[i] = null;

                if (victim.hasTransientState()) {
                    // Store views with transient state for later use.
                    victim.dispatchStartTemporaryDetach();

                    if (mAdapter != null && mAdapterHasStableIds) {
                        if (mTransientStateViewsById == null) {
                            mTransientStateViewsById = new LongSparseArray<View>();
                        }
                        long id = mAdapter.getItemId(mFirstActivePosition + i);
                        mTransientStateViewsById.put(id, victim);
                    } else if (!mDataChanged) {
                        if (mTransientStateViews == null) {
                            mTransientStateViews = new SparseArray<View>();
                        }
                        mTransientStateViews.put(mFirstActivePosition + i, victim);
                    } else if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                        // The data has changed, we can't keep this view.
                        removeDetachedView(victim, false);
                    }
                } else if (!shouldRecycleViewType(whichScrap)) {
                    // Discard non-recyclable views except headers/footers.
                    if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                        removeDetachedView(victim, false);
                    }
                } else {
                    // Store everything else on the appropriate scrap heap.
                    if (multipleScraps) {
                        scrapViews = mScrapViews[whichScrap];
                    }

                    victim.dispatchStartTemporaryDetach();
                    lp.scrappedFromPosition = mFirstActivePosition + i;
                    scrapViews.add(victim);

                    if (hasListener) {
                        mRecyclerListener.onMovedToScrapHeap(victim);
                    }
                }
            }
        }

        pruneScrapViews();
    }

    /**
     * Makes sure that the size of mScrapViews does not exceed the size of
     * mActiveViews, which can happen if an adapter does not recycle its
     * views. Removes cached transient state views that no longer have
     * transient state.
     */
    private void pruneScrapViews() {
        final int maxViews = mActiveViews.length;
        final int viewTypeCount = mViewTypeCount;
        final ArrayList<View>[] scrapViews = mScrapViews;
        for (int i = 0; i < viewTypeCount; ++i) {
            final ArrayList<View> scrapPile = scrapViews[i];
            int size = scrapPile.size();
            final int extras = size - maxViews;
            size--;
            for (int j = 0; j < extras; j++) {
                removeDetachedView(scrapPile.remove(size--), false);
            }
        }

        final SparseArray<View> transViewsByPos = mTransientStateViews;
        if (transViewsByPos != null) {
            for (int i = 0; i < transViewsByPos.size(); i++) {
                final View v = transViewsByPos.valueAt(i);
                if (!v.hasTransientState()) {
                    removeDetachedView(v, false);
                    transViewsByPos.removeAt(i);
                    i--;
                }
            }
        }

        final LongSparseArray<View> transViewsById = mTransientStateViewsById;
        if (transViewsById != null) {
            for (int i = 0; i < transViewsById.size(); i++) {
                final View v = transViewsById.valueAt(i);
                if (!v.hasTransientState()) {
                    removeDetachedView(v, false);
                    transViewsById.removeAt(i);
                    i--;
                }
            }
        }
    }

    /**
     * Puts all views in the scrap heap into the supplied list.
     */
    void reclaimScrapViews(List<View> views) {
        if (mViewTypeCount == 1) {
            views.addAll(mCurrentScrap);
        } else {
            final int viewTypeCount = mViewTypeCount;
            final ArrayList<View>[] scrapViews = mScrapViews;
            for (int i = 0; i < viewTypeCount; ++i) {
                final ArrayList<View> scrapPile = scrapViews[i];
                views.addAll(scrapPile);
            }
        }
    }

    /**
     * Updates the cache color hint of all known views.
     * 对保存的View调用setDrawingCacheBackgroundColor方法
     * @param color The new cache color hint.
     */
    void setCacheColorHint(int color) {
        if (mViewTypeCount == 1) {
            final ArrayList<View> scrap = mCurrentScrap;
            final int scrapCount = scrap.size();
            for (int i = 0; i < scrapCount; i++) {
                scrap.get(i).setDrawingCacheBackgroundColor(color);
            }
        } else {
            final int typeCount = mViewTypeCount;
            for (int i = 0; i < typeCount; i++) {
                final ArrayList<View> scrap = mScrapViews[i];
                final int scrapCount = scrap.size();
                for (int j = 0; j < scrapCount; j++) {
                    scrap.get(j).setDrawingCacheBackgroundColor(color);
                }
            }
        }
        // Just in case this is called during a layout pass
        final View[] activeViews = mActiveViews;
        final int count = activeViews.length;
        for (int i = 0; i < count; ++i) {
            final View victim = activeViews[i];
            if (victim != null) {
                victim.setDrawingCacheBackgroundColor(color);
            }
        }
    }
    //根据id或者位置，从scrapViews中回收废弃的View，id是我们通过Adapter#getItemId传递的id，且Adapter#hasStableIds要返回true才会通过id来匹配
    private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
        final int size = scrapViews.size();
        if (size > 0) {
            // See if we still have a view for this position or ID.
            for (int i = 0; i < size; i++) {
                final View view = scrapViews.get(i);
                final AbsListView.LayoutParams params =
                        (AbsListView.LayoutParams) view.getLayoutParams();

                if (mAdapterHasStableIds) {
                    final long id = mAdapter.getItemId(position);
                    if (id == params.itemId) {
                        return scrapViews.remove(i);
                    }
                } else if (params.scrappedFromPosition == position) {
                    final View scrap = scrapViews.remove(i);
                    clearAccessibilityFromScrap(scrap);
                    return scrap;
                }
            }//end for
            //如果没找到匹配的，取最后一个
            final View scrap = scrapViews.remove(size - 1);
            clearAccessibilityFromScrap(scrap);
            return scrap;
        } else {
            return null;
        }
    }

    private void clearScrap(final ArrayList<View> scrap) {
        final int scrapCount = scrap.size();
        for (int j = 0; j < scrapCount; j++) {
            removeDetachedView(scrap.remove(scrapCount - 1 - j), false);
        }
    }

    private void clearAccessibilityFromScrap(View view) {
        if (view.isAccessibilityFocused()) {
            view.clearAccessibilityFocus();
        }
        view.setAccessibilityDelegate(null);
    }

    private void removeDetachedView(View child, boolean animate) {
        child.setAccessibilityDelegate(null);
        AbsListView.this.removeDetachedView(child, animate);// Finishes the removal of a detached view. This method will dispatch the detached from window event and notify the hierarchy change listener.
    }
}
```
## 测量

主要是对整个ViewGroup的内边距的记录，因为AbsListView是抽象类，具体测量也应该是在实现类

```java
@Override
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
     if (mSelector == null) {
         useDefaultSelector();// com.android.internal.R.drawable.list_selector_background
     }
     final Rect listPadding = mListPadding;
     listPadding.left = mSelectionLeftPadding + mPaddingLeft;//mSelectionXXXPadding等初始值都是0，
     listPadding.top = mSelectionTopPadding + mPaddingTop;
     listPadding.right = mSelectionRightPadding + mPaddingRight;
     listPadding.bottom = mSelectionBottomPadding + mPaddingBottom;

     // Check if our previous measured size was at a point where we should scroll later.
     if (mTranscriptMode == TRANSCRIPT_MODE_NORMAL) {
         final int childCount = getChildCount();
         final int listBottom = getHeight() - getPaddingBottom();
         final View lastChild = getChildAt(childCount - 1);
         final int lastBottom = lastChild != null ? lastChild.getBottom() : listBottom;
         mForceTranscriptScroll = mFirstPosition + childCount >= mLastHandledItemCount &&
                 lastBottom <= listBottom;
     }
 }
```
## 布局

```java
/**
    * Subclasses should NOT override this method but
    *  {@link #layoutChildren()} instead.
    */
   @Override
   protected void onLayout(boolean changed, int l, int t, int r, int b) {
       super.onLayout(changed, l, t, r, b);

       mInLayout = true;

       final int childCount = getChildCount();
       if (changed) {
           for (int i = 0; i < childCount; i++) {
               getChildAt(i).forceLayout();
           }
           mRecycler.markChildrenDirty();//对RecyleBind中存储的View也进行forceLayout
       }

       layoutChildren();//由实现类实现
       mInLayout = false;

       mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;//OVERSCROLL_LIMIT_DIVISOR=3；

       // TODO: Move somewhere sane. This doesn't belong in onLayout().
       if (mFastScroll != null) {
           mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
       }
   }
```

## setAdapter

will update choice mode states.
```java
@Override
   public void setAdapter(ListAdapter adapter) {
       if (adapter != null) {
           mAdapterHasStableIds = mAdapter.hasStableIds();
           if (mChoiceMode != CHOICE_MODE_NONE && mAdapterHasStableIds &&
                   mCheckedIdStates == null) {
              //使用CHOICE_MODE的情况下，需要mAdapter.hasStableIds();
               mCheckedIdStates = new LongSparseArray<Integer>();//mCheckedIdStates根据id来存储checkId
           }
       }

       if (mCheckStates != null) {
           mCheckStates.clear();//Running state of which positions are currently checked
       }

       if (mCheckedIdStates != null) {
           mCheckedIdStates.clear();//Running state of which IDs are currently checked.
       }
   }
```

[fastScroll]: http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/0917/1690.html
