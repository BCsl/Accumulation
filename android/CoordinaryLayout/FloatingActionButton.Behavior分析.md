# FloatingActionButton.Behavior分析

`FloatingActionButton.Behavior`依赖于`Snackbar.SnackbarLayout`，会在`Snackbar.SnackbarLayout`出现的时候上浮，从成员变量可以指定是通过属性动画实现的，相对来说比较简单，直接上代码

```java
/**
 * Behavior designed for use with {@link FloatingActionButton} instances. It's main function
 * is to move {@link FloatingActionButton} views so that any displayed {@link Snackbar}s do
 * not cover them.
 */
public static class Behavior extends CoordinatorLayout.Behavior<FloatingActionButton> {
    // We only support the FAB <> Snackbar shift movement on Honeycomb and above. This is
    // because we can use view translation properties which greatly simplifies the code.
    private static final boolean SNACKBAR_BEHAVIOR_ENABLED = Build.VERSION.SDK_INT >= 11;

    private ValueAnimatorCompat mFabTranslationYAnimator;
    private float mFabTranslationY;
    private Rect mTmpRect;

    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent,
            FloatingActionButton child, View dependency) {
        // 依赖于Snackbar.SnackbarLayout
        return SNACKBAR_BEHAVIOR_ENABLED && dependency instanceof Snackbar.SnackbarLayout;
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, FloatingActionButton child,View dependency) {
        if (dependency instanceof Snackbar.SnackbarLayout) {
            updateFabTranslationForSnackbar(parent, child, dependency);
        } else if (dependency instanceof AppBarLayout) {
            // If we're depending on an AppBarLayout we will show/hide it automatically
            // if the FAB is anchored to the AppBarLayout
            updateFabVisibility(parent, (AppBarLayout) dependency, child);
        }
        return false;
    }
    //跟AppBarLayout有关
    private boolean updateFabVisibility(CoordinatorLayout parent,AppBarLayout appBarLayout, FloatingActionButton child) {
        final CoordinatorLayout.LayoutParams lp = (CoordinatorLayout.LayoutParams) child.getLayoutParams();
        if (lp.getAnchorId() != appBarLayout.getId()) {
            // The anchor ID doesn't match the dependency, so we won't automatically
            // show/hide the FAB
            return false;
        }

        if (child.getUserSetVisibility() != VISIBLE) {
            // The view isn't set to be visible so skip changing it's visibility
            return false;
        }

        if (mTmpRect == null) {
            mTmpRect = new Rect();
        }

        // First, let's get the visible rect of the dependency
        final Rect rect = mTmpRect;
        ViewGroupUtils.getDescendantRect(parent, appBarLayout, rect);

        if (rect.bottom <= appBarLayout.getMinimumHeightForVisibleOverlappingContent()) {
            // If the anchor's bottom is below the seam, we'll animate our FAB out
            child.hide(null, false);
        } else {
            // Else, we'll animate our FAB back in
            child.show(null, false);
        }
        return true;
    }
    //更新FloatingActionButton位置
    private void updateFabTranslationForSnackbar(CoordinatorLayout parent, final FloatingActionButton fab, View snackbar) {
        final float targetTransY = getFabTranslationYForSnackbar(parent, fab);
        if (mFabTranslationY == targetTransY) {
            // We're already at (or currently animating to) the target value, return...
            return;
        }

        final float currentTransY = ViewCompat.getTranslationY(fab); //可知，snackbar是通过`TranslationY`的位移做动画效果

        // Make sure that any current animation is cancelled
        if (mFabTranslationYAnimator != null && mFabTranslationYAnimator.isRunning()) {
            mFabTranslationYAnimator.cancel();
        }

        if (fab.isShown() && Math.abs(currentTransY - targetTransY) > (fab.getHeight() * 0.667f)) {
            // If the FAB will be travelling by more than 2/3 of it's height, let's animate it instead
            // 如果fab
            if (mFabTranslationYAnimator == null) {
                mFabTranslationYAnimator = ViewUtils.createAnimator();
                mFabTranslationYAnimator.setInterpolator( AnimationUtils.FAST_OUT_SLOW_IN_INTERPOLATOR);
                mFabTranslationYAnimator.setUpdateListener(new ValueAnimatorCompat.AnimatorUpdateListener() {
                            @Override
                            public void onAnimationUpdate(ValueAnimatorCompat animator) {
                                ViewCompat.setTranslationY(fab,animator.getAnimatedFloatValue());
                            }
                        });
            }
            mFabTranslationYAnimator.setFloatValues(currentTransY, targetTransY);
            mFabTranslationYAnimator.start();
        } else {
            // Now update the translation Y
            ViewCompat.setTranslationY(fab, targetTransY);
        }

        mFabTranslationY = targetTransY;
    }
    //获取和SnackbarLayout之间的偏移量，这里一般都是0吧
    private float getFabTranslationYForSnackbar(CoordinatorLayout parent,  FloatingActionButton fab) {
        float minOffset = 0;
        final List<View> dependencies = parent.getDependencies(fab); //获取到所有依赖FloatingActionButton的子View
        for (int i = 0, z = dependencies.size(); i < z; i++) {
            final View view = dependencies.get(i);
            if (view instanceof Snackbar.SnackbarLayout && parent.doViewsOverlap(fab, view)) { //如果已经和SnackbarLayout相交的情况下，计算偏移量，前提是SnackbarLayout依赖Fab
                minOffset = Math.min(minOffset,ViewCompat.getTranslationY(view) - view.getHeight());
            }
        }

        return minOffset;
    }
    //Fab可以依赖AppBarLayout，且FAB的可见受AppBarLayout的状态影响
    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, FloatingActionButton child, int layoutDirection) {
        // First, lets make sure that the visibility of the FAB is consistent
        final List<View> dependencies = parent.getDependencies(child); //返回Fab所依赖的所有的View
        for (int i = 0, count = dependencies.size(); i < count; i++) {
            final View dependency = dependencies.get(i);
            if (dependency instanceof AppBarLayout && updateFabVisibility(parent, (AppBarLayout) dependency, child)) {
                break;
            }
        }
        // Now let the CoordinatorLayout lay out the FAB
        parent.onLayoutChild(child, layoutDirection);
        // Now offset it if needed
        offsetIfNeeded(parent, child);
        return true;
    }

    /**
     * Pre-Lollipop we use padding so that the shadow has enough space to be drawn. This method
     * offsets our layout position so that we're positioned correctly if we're on one of
     * our parent's edges.
     */
    private void offsetIfNeeded(CoordinatorLayout parent, FloatingActionButton fab) {
        final Rect padding = fab.mShadowPadding;

        if (padding != null && padding.centerX() > 0 && padding.centerY() > 0) {
            final CoordinatorLayout.LayoutParams lp =
                    (CoordinatorLayout.LayoutParams) fab.getLayoutParams();

            int offsetTB = 0, offsetLR = 0;

            if (fab.getRight() >= parent.getWidth() - lp.rightMargin) {
                // If we're on the left edge, shift it the right
                offsetLR = padding.right;
            } else if (fab.getLeft() <= lp.leftMargin) {
                // If we're on the left edge, shift it the left
                offsetLR = -padding.left;
            }
            if (fab.getBottom() >= parent.getBottom() - lp.bottomMargin) {
                // If we're on the bottom edge, shift it down
                offsetTB = padding.bottom;
            } else if (fab.getTop() <= lp.topMargin) {
                // If we're on the top edge, shift it up
                offsetTB = -padding.top;
            }

            fab.offsetTopAndBottom(offsetTB);
            fab.offsetLeftAndRight(offsetLR);
        }
    }
}
```
