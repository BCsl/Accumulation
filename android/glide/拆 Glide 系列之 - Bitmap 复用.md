# 拆 Glide 系列之 - Bitmap 复用

## Bitmap 内存管理

Google 官方教程[Managing Bitmap Memory](https://developer.android.com/topic/performance/graphics/manage-memory.html)是这样说的

> - Android2.2（API 8）一下的时候，当 GC 工作时，应用的线程会暂停工作，同步的 GC 会影响性能。而 Android2.3 之后，GC 变成了并发的，意味着 Bitmap 没有引用的时候其占有的内存会很快被回收

> - 在Android2.3.3（API10）之前，Bitmap 的像素数据存放在 Native 内存，而 Bitmap 对象本身则存放在 Dalvik Heap 中。Native 内存中的像素数据以不可预测的方式进行同步回收，有可能会导致内存升高甚至 OOM Crash。而在 Android3.0 之后，Bitmap 的像素数据也被放在了 Dalvik Heap 中

所以根据不同的 Android 版本，对于 Bitmap 的内存处理上，也需要以不同的方式来对待

- Android2.3.3 以下

推荐使用 `Bitmap#recycle` 方法进行 Bitmap 内存回收

- Android3.0 以上

推荐的是 Bitmap 内存复用，为此引入了一个 `BitmapFactory.Options.inBitmap` 字段来设置打算复用的 Bitmap，这个字段设置之后 Bitmap 解码的时候会尝试复用这一张存在的 Bitmap。内存复用减少内存的分配操作可以避免内存抖动带来的潜在的卡顿，[Glide](https://github.com/bumptech/glide) 内部的 Bitmap 复用模块正是今天的主角

## Bitmap 复用

几点限制：

- 被复用的 Bitmap 必须为 `Mutable`（通过 `BitmapFactory.Options` 设置）
- 4.4 之前，将要解码的图像（无论是资源还是流）必须是 jpeg 或 png 格式且和被复用的 Bitmap 大小一样，其中`BitmapFactory.Options#inSampleSize` 字段必须设置为 1，要求比较严苛
- 4.4 以后，将要解码的图像的内存需要大于等于要复用的 Bitmap 的内存

![Glide Bitmap 复用模块](https://raw.githubusercontent.com/BCsl/Accumulation/master/android/glide/Bitmap%20%E5%A4%8D%E7%94%A8.png)

其中最重要的角色是 `LruPoolStrategy` ，是 Bitmap 内存复用策略的接口，有三个实现类 `SizeConfigStrategy`、`SizeStrategy` 和 `AttributeStrategy`

在分析这个具体的复用策略前，先介绍一下其中实现 LRU 的数据结构，我们都知道 SDK 提供的 `LRUCache` 类可以帮助实现 LRU 功能，且内部的数据结构是 `LinkedHashMap`。但 `Glide` 并没有使用现成的，而是定义了一个 `GroupedLinkedMap` ，其类似 `LinkedHashMap` ，其实体 `LinedEntry` 可以用于实现双向链表，并通过 `GroupedLinkedMap#makeTail`/`GroupedLinkedMap#makeHead` 方法来改变链表头尾位置，使得具有 LRU 功能，且 `LinedEntry` 的 `Value` 为一个 `List` 集合，因为可能有多个相同 `key`（例如：Bitmap Size）的 `Bitmap` 集合可以复用，而且可以控制移除队列最后一个对象，而不是整个 `List` 集合，这应该是不使用 `LRUCache` 的原因

```java
class GroupedLinkedMap<K extends Poolable, V> {
    private final LinkedEntry<K, V> head = new LinkedEntry<K, V>();
    private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<K, LinkedEntry<K, V>>();

    public void put(K key, V value) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);

        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            makeTail(entry);   //移到表尾
            keyToEntry.put(key, entry);
        } else {
            key.offer(); //如果 key 相同，那么可以把现在的回收到对象池
        }

        entry.add(value);
    }

    public V get(K key) {
        LinkedEntry<K, V> entry = keyToEntry.get(key);
        if (entry == null) {
            entry = new LinkedEntry<K, V>(key);
            keyToEntry.put(key, entry);
        } else {
            key.offer();
        }
        makeHead(entry);    //移到表头
        return entry.removeLast();
    }

    public V removeLast() {
        LinkedEntry<K, V> last = head.prev;

        while (!last.equals(head)) {
            V removed = last.removeLast();
            if (removed != null) {
                return removed;
            } else {
                removeEntry(last);  //清理已经空的 LinkedEntry
                keyToEntry.remove(last.key);
                last.key.offer();
            }

            last = last.prev;
        }

        return null;
    }

    // Make the entry the most recently used item.
    private void makeHead(LinkedEntry<K, V> entry) {
        removeEntry(entry);
        entry.prev = head;
        entry.next = head.next;
        updateEntry(entry);
    }

    // Make the entry the least recently used item.
    private void makeTail(LinkedEntry<K, V> entry) {
        removeEntry(entry);
        entry.prev = head.prev;
        entry.next = head;
        updateEntry(entry);
    }

    private static <K, V> void updateEntry(LinkedEntry<K, V> entry) {
        entry.next.prev = entry;
        entry.prev.next = entry;
    }

    private static <K, V> void removeEntry(LinkedEntry<K, V> entry) {
        entry.prev.next = entry.next;
        entry.next.prev = entry.prev;
    }

    private static class LinkedEntry<K, V> {
        private final K key;
        private List<V> values;
        LinkedEntry<K, V> next;
        LinkedEntry<K, V> prev;

        // Used only for the first item in the list which we will treat specially and which will not contain a value.
        public LinkedEntry() {
            this(null);
        }

        public LinkedEntry(K key) {
            next = prev = this;
            this.key = key;
        }
        //清理集合的最后一个元素
        public V removeLast() {
            final int valueSize = size();
            return valueSize > 0 ? values.remove(valueSize - 1) : null;
        }

        public int size() {
            return values != null ? values.size() : 0;
        }

        public void add(V value) {
            if (values == null) {
                values = new ArrayList<V>();
            }
            values.add(value);
        }
    }
}
```

### `SizeStrategy`

下面以 `SizeStrategy` 来分析，这是 4.4 以上的复用策略，只要被复用的内存比需要申请的内存小

```java
@TargetApi(Build.VERSION_CODES.KITKAT)
class SizeStrategy implements LruPoolStrategy {
    //被复用的Bitmap的内存必须大于需要申请内存的Bitmap的内存，但也要避免不要过大
    //例如分配 1M 空间的 Bitmap，确实没必要去复用以前分配的 100M 的可用 Bitmap 空间，所以这里的规定是 8 倍
    private static final int MAX_SIZE_MULTIPLE = 8;
    private final KeyPool keyPool = new KeyPool();
    private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<Key, Bitmap>();
    // 使用 TreeMap 是为了能够按照 key（这里实际上是 Bitmap 的 size） 的大小进行排序，找到大于或等于目标 Bitmap 内存的复用内存空间
    private final TreeMap<Integer, Integer> sortedSizes = new PrettyPrintTreeMap<Integer, Integer>();

    @Override
    public void put(Bitmap bitmap) {
        int size = Util.getBitmapByteSize(bitmap);
        final Key key = keyPool.get(size);

        groupedMap.put(key, bitmap);

        Integer current = sortedSizes.get(key.size);
        sortedSizes.put(key.size, current == null ? 1 : current + 1);
    }

    @Override
    public Bitmap get(int width, int height, Bitmap.Config config) {
        final int size = Util.getBitmapByteSize(width, height, config); //目标 bitmap 大小，
        Key key = keyPool.get(size);

        Integer possibleSize = sortedSizes.ceilingKey(size);  //获取一个大于等于当前 size 的 Key
        if (possibleSize != null && possibleSize != size && possibleSize <= size * MAX_SIZE_MULTIPLE) {
            keyPool.offer(key); //这个 key 的使命到此，所以存回去对象池
            key = keyPool.get(possibleSize);
        }

        // Do a get even if we know we don't have a bitmap so that the key moves to the front in the lru pool
        final Bitmap result = groupedMap.get(key);
        if (result != null) {
            result.reconfigure(width, height, config);  
            decrementBitmapOfSize(possibleSize);
        }

        return result;
    }

    @Override
    public Bitmap removeLast() {
        Bitmap removed = groupedMap.removeLast();
        if (removed != null) {
            final int removedSize = Util.getBitmapByteSize(removed);
            decrementBitmapOfSize(removedSize);
        }
        return removed;
    }

    private void decrementBitmapOfSize(Integer size) {
        Integer current = sortedSizes.get(size);
        if (current == 1) {
            sortedSizes.remove(size);
        } else {
            sortedSizes.put(size, current - 1);
        }
    }
    //..

    @Override
    public int getSize(Bitmap bitmap) {
        return Util.getBitmapByteSize(bitmap);
    }
    //..

    // 对象池，存储以 size 为标识的池对象
    static class KeyPool extends BaseKeyPool<Key> {

        public Key get(int size) {
            Key result = get();
            result.init(size);
            return result;
        }

        @Override
        protected Key create() {
            return new Key(this);
        }
    }

    // 以 size 为标识的池对象
    static final class Key implements Poolable {
        private final KeyPool pool;
        private int size;

        Key(KeyPool pool) {
            this.pool = pool;
        }

        public void init(int size) {
            this.size = size;
        }

        @Override
        public boolean equals(Object o) {
            if (o instanceof Key) {
                Key other = (Key) o;
                return size == other.size;
            }
            return false;
        }

        @Override
        public int hashCode() {
            return size;
        }

        @Override
        public String toString() {
            return getBitmapString(size);
        }

        @Override
        public void offer() {
            pool.offer(this);
        }
    }
}
```

### Bitmap 大小的计算

```java
/**
 * Returns the in memory size of the given {@link Bitmap} in bytes.
 */
@TargetApi(Build.VERSION_CODES.KITKAT)
public static int getBitmapByteSize(Bitmap bitmap) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        // Workaround for KitKat initial release NPE in Bitmap, fixed in MR1\. See issue #148.
        try {
            return bitmap.getAllocationByteCount();
        } catch (NullPointerException e) {
            // Do nothing.
        }
    }
    return bitmap.getHeight() * bitmap.getRowBytes();
}
```

## 参考

- [GlideBitmapPool](https://github.com/amitshekhariitbhu/GlideBitmapPool)
- [Glide](https://github.com/bumptech/glide)
- [Managing Bitmap Memory](https://developer.android.com/topic/performance/graphics/manage-memory.html)
- [Android性能优化（五）之细说Bitmap](http://www.jianshu.com/p/e49ec7d053b3)
