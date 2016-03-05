## ThreadLocal介绍
>Implements a thread-local storage, that is, a variable for which each thread has its own value. All threads share the same {@code ThreadLocal} object ,but each sees a different value when accessing it, and changes made by one thread do not affect the other threads. The implementation supports {@code null} values.

大概意思是说，一个`ThreadLocal`变量可以为不同的线程保存不同的值

## 实现

通过介绍可知，居然可以为不同的线程保存不同的值，那么我们可能需要一个`Map`对象，即`ThreadLocal<T>`变量内部维护一个`HashMap<Thread,T>`对象，其中，`KEY`可以通过`Thread.currentThread()`来获取
