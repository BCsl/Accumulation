# 扭转思维，如何实现ThreadLocal

大部分翻译自[How is ThreadLocal implemented?](http://www.javacodegeeks.com/2013/11/how-is-threadlocal-implemented.html)

## ThreadLocal介绍

> Implements a thread-local storage, that is, a variable for which each thread has its own value. All threads share the same {@code ThreadLocal} object ,but each sees a different value when accessing it, and changes made by one thread do not affect the other threads. The implementation supports {@code null} values.

大概意思是说，一个`ThreadLocal`变量可以为不同的线程保存不同的值

## 实现

通过介绍可知，为线程保存线程本地变量，那么我们可能需要一个`Map`对象，即`ThreadLocal<T>`变量内部维护一个`HashMap<Thread,T>`对象，其中，`KEY`可以通过`Thread.currentThread()`来获取，但是`HashMap`不是线程安全的，我们可以用`Hashtable`（相当于为每个方法加了同步原语synchronize的HashMap，效率低）或者`ConcurrentHashMap`(锁分段技术，效率高点)代替，一旦你使用这样的方式实现，那么需要面对的问题有两：

- 1、内存泄漏分析，因为KEY为`Thread`
- 2、同步的开销

为了解决第一个问题可以考虑使用弱应用替代Thread的强引用，但关于同步的开销明显是没有更好的方法解决了？

**这是一个关于跳脱固有框架思考的例子**

但实际上JDK中的实现并不是这样的且更加优雅，前面我们考虑以`Thread`作为KEY记录在`ThreadLocal`来为`Thread`保存线程独有的变量，而JDK的实现则是刚好与我们相反，用`Thread`来记录这样的一个以`ThreadLocal`-`Object`的映射关系（不一定是用JAVA中的`Map`容器来实现）

以Android23的为例子

```java
public class Thread implements Runnable {
    ThreadLocal.Values localValues;
    // cut for brevity
}
```

```java
public class ThreadLocal<T> {
    // cut for brevity
    static class Value {        
        private Object[] table;

        void put(ThreadLocal<?> keys, Object value) {
            cleanUp();
            int firstTombstone = -1;
            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }
              }
            }//end put

            //........

        }//end Value

    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }

    Values initializeValues(Thread current) {
        return current.localValues = new Values();
    }

    // ...
}//end ThreadLocal
```

`Thread`记录了一个`ThreadLocal.Value`实例，用来记录了[`ThreadLocal`实例-线程本地变量]的映射，Key为hash值，不用担心内存泄漏问题，并且避免了同步所带来的开销问题，的确是十分优雅的实现

## 扩展阅读

- [聊聊并发（四）----深入分析ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap/)
- [深度剖析ConcurrentHashMap](http://qifuguang.me/2015/09/10/[Java%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E5%85%AB]%E6%B7%B1%E5%BA%A6%E5%89%96%E6%9E%90ConcurrentHashMap/)
