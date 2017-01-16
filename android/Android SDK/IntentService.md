## IntentService是什么？
  Service子类，但运行在子线程。以工作队列的形式处理Intent。
## 执行操作
  * 创建工作线程，用于在应用的主线程外执行`onStartCommand()`的所有Intent
  * 创建工作队列，用于将一个Intent逐一传递给`onHandleIntent`实现，这样你就不担心多线程问题
  * 处理完操作之后，自动的`stopSelf`方式来关闭Service

## 思考
  以工作队列的形式逐处理消息，这个是不是和`Handle`+`Lopper`消息机制的工作形式类似？我们往`Lopper`发送消息到消息队列，依次地
  调用特定`Handle`来处理，这个工作，消息就是逐一有序地处理。现在我们需要在子线程来处理，所以自己定义一个`Lopper`循环，当然，更简单的就是
  使用`HandlerThread`,使用`start`来启动线程，线程跑起来的时候，注册循环（`Looper.loop()`）,使用`quit`来结束线程（`Looper.quit()`）

## 源码
```java

public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            //自动的`stopSelf`方式来关闭Service
            stopSelf(msg.arg1);
        }
    }

    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public IntentService(String name) {
        super();
        mName = name;
    }

    /**
     * Sets intent redelivery preferences.  Usually called from the constructor
     * with your preferred semantics.
     *
     * <p>If enabled is true,
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_REDELIVER_INTENT}, so if this process dies before
     * {@link #onHandleIntent(Intent)} returns, the process will be restarted
     * and the intent redelivered.  If multiple Intents have been sent, only
     * the most recent one is guaranteed to be redelivered.
     *
     * <p>If enabled is false (the default),
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_NOT_STICKY}, and if the process dies, the Intent
     * dies along with it.
     */
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.
        super.onCreate();
        //创建工作线程，用于在应用的主线程外执行`onStartCommand()`的所有Intent
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        //将一个Intent逐一传递给`onHandleIntent`实现
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        //退出线程
        mServiceLooper.quit();
    }

    /**
     * Unless you provide binding for your service, you don't need to implement this
     * method, because the default implementation returns null.
     * @see android.app.Service#onBind
     */
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * This method is invoked on the worker thread with a request to process.
     * Only one Intent is processed at a time, but the processing happens on a
     * worker thread that runs independently from other application logic.
     * So, if this code takes a long time, it will hold up other requests to
     * the same IntentService, but it will not hold up anything else.
     * When all requests have been handled, the IntentService stops itself,
     * so you should not call {@link #stopSelf}.
     *
     * @param intent The value passed to {@link
     *               android.content.Context#startService(Intent)}.
     */
    protected abstract void onHandleIntent(Intent intent);
}

```
