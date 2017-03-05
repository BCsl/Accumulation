# Android性能检测手段

主要总结一下开源库关于性能检测的手段

## UI卡顿检测

UI单线程模型，所以UI线程上不应该处理耗时操作

### Handler

代表的开源库[AndroidPerformanceMonitor](https://github.com/markzhai/AndroidPerformanceMonitor)

`Looper`在处理`MessageQueue`中的`Message`的时候，在处理前和处理后都会用`Printer`打印日志，一般情况下，这个`Printer`为null，需要用户自己设置

```java
public static void loop() {
    final Looper me = myLooper();
    //...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        //....
    }
}
```

既然这样，**为`Looper#getMainLooper()`设置一个`Printer`，记录打印前后的时间差是否超过某个值即可**，恩，原理就这样，而且为了不影响到UI线程，最好在其他线程中来记录分析

整理后的代码：

```java
public class BlockDetectByPrinter {

    public static void start() {
        Looper.getMainLooper().setMessageLogging(new Printer() {
            public static final String MSG_START_PREFIX = ">>>>> Dispatching to";
            public static final String MSG_END_PREFIX = "<<<<< Finished to";

            @Override
            public void println(String x) {
                if (x.startsWith(MSG_START_PREFIX)) {
                    LogMonitor.getInstance().startMonitor();
                }
                if (x.startsWith(MSG_END_PREFIX)) {
                    LogMonitor.getInstance().removeMonitor();
                }
            }
        });
    }
}
```

```java
public class LogMonitor {
    private static final String TAG = "LogMonitor";
    private static LogMonitor sLogMonitor;

    private HandlerThread mLopThread = new HandlerThread("log");

    private Handler mLogHandler;

    private LogMonitor() {
        mLopThread.start();
        mLogHandler = new Handler(mLopThread.getLooper());
    }

    /**
     * 打印耗时操作的堆栈
     */
    private static Runnable mLogTask = new Runnable() {
        @Override
        public void run() {
            StringBuilder stringBuilder = new StringBuilder();
            StackTraceElement[] stackTraceElements = Looper.getMainLooper().getThread().getStackTrace();
            for (StackTraceElement s : stackTraceElements) {
                stringBuilder.append(s.toString() + "\n");
            }
            Log.e(TAG, "Block:" + stringBuilder.toString());
        }
    };

    public static LogMonitor getInstance() {
        if (sLogMonitor == null) {
            sLogMonitor = new LogMonitor();
        }
        return sLogMonitor;
    }

    public void startMonitor() {
        mLogHandler.postDelayed(mLogTask, 1000);
    }

    public void removeMonitor() {
        mLogHandler.removeCallbacks(mLogTask);
    }
}
```

### Choreographer

[Takt](https://github.com/wasabeef/Takt)

[TinyDancer](https://github.com/friendlyrobotnyc/TinyDancer)

16ms发送VSYNC进行视图更新，`Choreographer.FrameCallback`可以监听下一次触发视图的更新，理论上来说两次回调的时间周期应该在16ms，如果超过了16ms我们则认为发生了卡顿：

```java
//在下一帧View更新的时候执行
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        LogMonitor.getInstance().removeMonitor();
        LogMonitor.getInstance().startMonitor();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            Choreographer.getInstance().postFrameCallback(this);
        }
    }
});
```

## 参考

- [Android UI性能优化 检测应用中的UI卡顿](http://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650822205&idx=1&sn=6b8e78bc1d71eb79a199667cf132acf7&chksm=80b782a3b7c00bb5c12437556fca68136c75409855e9252e395b545621319edf23959942b67c&mpshare=1&scene=23&srcid=03018R47IfBW5FWwOEDIQhL2%23rd)
