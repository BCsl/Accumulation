ClipDrawable继承自DrawableWapper，从DrawableWapper的名字就知道是用来对已经存在的Drawable做了一层包裹，而ClipDrawable则是做了裁剪

ClipDrawable的通过level值来决定裁剪程度，这个值是[0,10000]，支持两个方向上的裁剪，水平和垂直，这是它的实现

```java

public void draw(Canvas canvas) {
    final Drawable dr = getDrawable();
    if (dr.getLevel() == 0) {
        return;
    }

    final Rect r = mTmpRect;
    final Rect bounds = getBounds();
    final int level = getLevel();

    int w = bounds.width();
    final int iw = 0; //mState.mDrawable.getIntrinsicWidth();
    if ((mState.mOrientation & HORIZONTAL) != 0) {
        w -= (w - iw) * (MAX_LEVEL - level) / MAX_LEVEL;
    }

    int h = bounds.height();
    final int ih = 0; //mState.mDrawable.getIntrinsicHeight();
    if ((mState.mOrientation & VERTICAL) != 0) {
        h -= (h - ih) * (MAX_LEVEL - level) / MAX_LEVEL;
    }

    final int layoutDirection = getLayoutDirection();
    Gravity.apply(mState.mGravity, w, h, bounds, r, layoutDirection);

    if (w > 0 && h > 0) {
        canvas.save();
        canvas.clipRect(r);
        dr.draw(canvas);
        canvas.restore();
    }
}
```

如果有需求需要用到自定义Drawable，完全可以参照ClipDrawable的实现方式来已存在的Drawable，但这也是它的局限性，**只是用来处理已存在的Drawable**
