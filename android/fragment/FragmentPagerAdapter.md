# FragmentPagerAdapter 和 FragmentStatePagerAdapter 的对比

按照官方的说法 `FragmentPagerAdapter` 会尽量保存 `Fragment` 在内存中，并不适合大量 `Fragment` 的场景，在有比较多 `Fragment` 的情况下，推荐使用 `FragmentStatePagerAdapter`，当 `Fragment` 不可见的时候，就可能会被 `destroy` 掉，只保存其 `Fragment` 状态，用于后续的恢复，这样可以减轻内存的负担

## 显示和移除的处理

### `FragmentPagerAdapter`

对于 `FragmentPagerAdapter` 来说，并不保存 `Fragment` 状态，会用 `attach` 配合 `detach` 来显示和轻量地移除 `Fragment`，所以并不需要保存状态

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
    //...
    // 如果已经在
    String name = makeFragmentName(container.getId(), itemId);  //"android:switcher:" + viewId + ":" + id
    Fragment fragment = mFragmentManager.findFragmentByTag(name); //内存恢复回来，也会优先找到内存中的Fragment，所以每次新建Fragment 的ArrayList来初始化也不需要担心？
    if (fragment != null) {
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        mCurTransaction.add(container.getId(), fragment, makeFragmentName(container.getId(), itemId));
    }
    //...

    return fragment;
}
```

```java
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    //...
    mCurTransaction.detach((Fragment)object);
}
```

### `FragmentStatePagerAdapter`

通过 `add` 和 `remove` 的方式来添加和移除 `Fragment`，所以需要保存 `Fragment` 状态，在下次显示的时候恢复，保存的信息和 `Activity` 对 `Fragment` 的处理一样，可见 [内存回收/恢复](./内存回收.png)

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
    // 内存恢复的时候恢复`mFragments`，所以已经存在就不需要进行额外操作了
    if (mFragments.size() > position) {
        Fragment f = mFragments.get(position);
        if (f != null) {
            return f;
        }
    }
    //...
    Fragment fragment = getItem(position);
    if (mSavedState.size() > position) {
        Fragment.SavedState fss = mSavedState.get(position);
        if (fss != null) {
            fragment.setInitialSavedState(fss); //状态恢复
        }
    }
    while (mFragments.size() <= position) {
        mFragments.add(null);
    }
    //....
    mFragments.set(position, fragment);
    mCurTransaction.add(container.getId(), fragment);

    return fragment;
}
```

```java
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    Fragment fragment = (Fragment) object;
    //...
    while (mSavedState.size() <= position) {
        mSavedState.add(null);
    }
    mSavedState.set(position, fragment.isAdded() ? mFragmentManager.saveFragmentInstanceState(fragment) : null);  //状态保存见 ./内存回收.png
    mFragments.set(position, null);
    mCurTransaction.remove(fragment);
}
```

## 内存恢复

### `FragmentPagerAdapter`

不保留状态，另外，根据 `instantiateItem` 的逻辑来看，内存恢复回来，也会优先找到内存中的 `Fragment`，所以每次新建 `Fragment` 的列表来初始化也不需要担心，因为之前已经添加进去了的会复用，相应的 `getItem` 也不会被调用到的，所以在使用 `ViewPager` + `FragmentPagerAdapter` 搭配使用的时候，不需要考虑内存回收后 `Fragment` 重叠、泄漏问题

```java
@Override
public Parcelable saveState() {
    return null;
}

@Override
public void restoreState(Parcelable state, ClassLoader loader) {
}
`
```

### `FragmentStatePagerAdapter`

保存 `mSavedState` 数组，里面保存了被移除的 `Fragment` 的状态和保存当前已经添加的 `Fragment`，`key` 为「 `f` + `index` 」

```java
@Override
public Parcelable saveState() {
    Bundle state = null;
    if (mSavedState.size() > 0) {
        state = new Bundle();
        Fragment.SavedState[] fss = new Fragment.SavedState[mSavedState.size()];
        mSavedState.toArray(fss);
        state.putParcelableArray("states", fss);
    }
    for (int i=0; i<mFragments.size(); i++) {
        Fragment f = mFragments.get(i);
        if (f != null && f.isAdded()) {
            if (state == null) {
                state = new Bundle();
            }
            String key = "f" + i;
            mFragmentManager.putFragment(state, key, f);
        }
    }
    return state;
}
```

```java
@Override
public void restoreState(Parcelable state, ClassLoader loader) {
    if (state != null) {
        Bundle bundle = (Bundle)state;
        bundle.setClassLoader(loader);
        Parcelable[] fss = bundle.getParcelableArray("states");
        mSavedState.clear();
        mFragments.clear();
        if (fss != null) {
            for (int i=0; i<fss.length; i++) {
                mSavedState.add((Fragment.SavedState)fss[i]);
            }
        }
        Iterable<String> keys = bundle.keySet();
        for (String key: keys) {
            if (key.startsWith("f")) {
                int index = Integer.parseInt(key.substring(1));
                Fragment f = mFragmentManager.getFragment(bundle, key);
                if (f != null) {
                    while (mFragments.size() <= index) {
                        mFragments.add(null);
                    }
                    //...
                    mFragments.set(index, f);
                } else {
                    Log.w(TAG, "Bad fragment at key " + key);
                }
            }
        }
    }
}
```
