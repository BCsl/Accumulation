# Android广播注册

## Setp1
不管是`Activity`、`Service`上注册广播，最终实现都在`ContextImpl`，最后发送到`ActivityManagerService`处理

```java
@Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
    //context：一般为NULL
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
              //...
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();//ActivityThread#mH,所以可以肯定，最后还会在ActivityThread来处理回调的
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }
```
## Step2
注意到，在与`ActivityManagerService`通信之前先构造了一个`IIntentReceiver`对象，这是一个`IPC`机制的服务端对象，有两种方式来保存宿主`ReceiverDispatcher`，强引用或强、弱引用，这里只使用的是弱引用方式

```java
static final class ReceiverDispatcher {

    final static class InnerReceiver extends IIntentReceiver.Stub {
        final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
        final LoadedApk.ReceiverDispatcher mStrongRef;
        //strong:false;
        InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
            mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
            mStrongRef = strong ? rd : null;
        }
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
            //...
            if (rd != null) {
                rd.performReceive(intent, resultCode, data, extras,
                        ordered, sticky, sendingUser);
            } else {
                // The activity manager dispatched a broadcast to a registered
                // receiver in this process, but before it could be delivered the
                // receiver was unregistered.  Acknowledge the broadcast on its
                // behalf so that the system's broadcast sequence can continue.
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing broadcast to unregistered receiver");
                IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    if (extras != null) {
                        extras.setAllowFds(false);
                    }
                    mgr.finishReceiver(this, resultCode, data, extras, false);
                } catch (RemoteException e) {
                    Slog.w(ActivityThread.TAG, "Couldn't finish broadcast to unregistered receiver");
                }
            }
        }
    }

    final IIntentReceiver.Stub mIIntentReceiver;
    final BroadcastReceiver mReceiver;
    final Context mContext;
    final Handler mActivityThread;
    final Instrumentation mInstrumentation;
    final boolean mRegistered;
    final IntentReceiverLeaked mLocation;
    RuntimeException mUnregisterLocation;
    boolean mForgotten;

    final class Args extends BroadcastReceiver.PendingResult implements Runnable {
      //...
    }
    //instrumentation:false, registered:true
    ReceiverDispatcher(BroadcastReceiver receiver, Context context,Handler activityThread, Instrumentation instrumentation,
            boolean registered) {
        if (activityThread == null) {
            throw new NullPointerException("Handler must not be null");
        }
        mIIntentReceiver = new InnerReceiver(this, !registered);
        mReceiver = receiver;
        mContext = context;
        mActivityThread = activityThread;
        mInstrumentation = instrumentation;
        mRegistered = registered;
        mLocation = new IntentReceiverLeaked(null);
        mLocation.fillInStackTrace();
    }

    //...

    IIntentReceiver getIIntentReceiver() {
        return mIIntentReceiver;
    }
    //...


}

```
## Step3
接着就是`ActivityManagerProxy`处理`registerReceiver`方法

```java
public Intent registerReceiver(IApplicationThread caller, String packageName,IIntentReceiver receiver,
        IntentFilter filter, String perm, int userId) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeString(packageName);
    data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
    filter.writeToParcel(data, 0);
    data.writeString(perm);
    data.writeInt(userId);
    mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);
    reply.readException();
    Intent intent = null;
    int haveIntent = reply.readInt();
    if (haveIntent != 0) {
        intent = Intent.CREATOR.createFromParcel(reply);
    }
    reply.recycle();
    data.recycle();
    return intent;
}

```

## Step4
`ActivityManagerService`处理`REGISTER_RECEIVER_TRANSACTION`命令

```java

case REGISTER_RECEIVER_TRANSACTION:
       {
           data.enforceInterface(IActivityManager.descriptor);
           IBinder b = data.readStrongBinder();
           IApplicationThread app =b != null ? ApplicationThreadNative.asInterface(b) : null;
           String packageName = data.readString();
           b = data.readStrongBinder();
           IIntentReceiver rec= b != null ? IIntentReceiver.Stub.asInterface(b) : null;
           IntentFilter filter = IntentFilter.CREATOR.createFromParcel(data);
           String perm = data.readString();
           int userId = data.readInt();
           Intent intent = registerReceiver(app, packageName, rec, filter, perm, userId);
           reply.writeNoException();
           if (intent != null) {
               reply.writeInt(1);
               intent.writeToParcel(reply, 0);
           } else {
               reply.writeInt(0);
           }
           return true;
       }
```

这里会做权限的检测，把`IIntentReceiver`类型的广播接收器`receiver`保存一个`ReceiverList`列表中，这个列表的宿主进程是`rl.app`，这里就是注册所在的进程了，在`ActivityManagerService`中，用一个进程记录块`ProcessRecord`来表示这个应用程序进程，它里面有一个列表`receivers`，专门用来保存这个进程注册的广播接收器。接着，又把这个`ReceiverList`列表以`receiver`为`Key`值保存在`ActivityManagerService`的成员变量`mRegisteredReceivers`中，这些都是为了方便在收到广播时，快速找到对应的广播接收器的

```java

public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
    enforceNotIsolatedCaller("registerReceiver");
    int callingUid;
    int callingPid;
    synchronized(this) {
        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = getRecordForAppLocked(caller);//获取用户进程，一般不空
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when registering receiver " + receiver);
            }
            if (callerApp.info.uid != Process.SYSTEM_UID &&
                    !callerApp.pkgList.containsKey(callerPackage)) {
                throw new SecurityException("Given caller package " + callerPackage
                        + " is not running in process " + callerApp);
            }
            callingUid = callerApp.info.uid;
            callingPid = callerApp.pid;
        } else {
            callerPackage = null;
            callingUid = Binder.getCallingUid();
            callingPid = Binder.getCallingPid();
        }

        userId = this.handleIncomingUser(callingPid, callingUid, userId,true, true, "registerReceiver", callerPackage);//权限检测

        List allSticky = null;

        // Look for any matching sticky broadcasts...
        //这里先通过getStickiesLocked函数查找一下有没有对应的sticky intent列表存在。什么是Sticky Intent呢？我们在最后一次调用sendStickyBroadcast函数来发送某个Action类型的广播时，系统会把代表这个广播的Intent保存下来，这样，后来调用registerReceiver来注册相同Action类型的广播接收器，就会得到这个最后发出的广播。这就是为什么叫做Sticky Intent了，这个最后发出的广播虽然被处理完了，但是仍然被粘住在ActivityManagerService中，以便下一个注册相应Action类型的广播接收器还能继承处理，这里得到的allSticky和sticky都为null
        Iterator actions = filter.actionsIterator();
        if (actions != null) {
            while (actions.hasNext()) {
                String action = (String)actions.next();
                allSticky = getStickiesLocked(action, filter, allSticky,
                        UserHandle.USER_ALL);
                allSticky = getStickiesLocked(action, filter, allSticky,
                        UserHandle.getUserId(callingUid));
            }
        } else {
            allSticky = getStickiesLocked(null, filter, allSticky,
                    UserHandle.USER_ALL);
            allSticky = getStickiesLocked(null, filter, allSticky,
                    UserHandle.getUserId(callingUid));
        }

        // The first sticky in the list is returned directly back to
        // the client.
        Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;


        if (receiver == null) {
            return sticky;
        }
        //mRegisteredReceivers是一个HashMap，key：IIntentReceiver（相同的BroadcastReceiver，KEY都默认），value:ReceiverList
        //value保存为list的原因是？有可能重复注册两个相同类型`BroadcastReceiver`实例？
        //ReceiverList继承自`ArrayList`，其实就是把广播接收器receiver保存一个ReceiverList列表中，这个列表的宿主进程是rl.app，
        ReceiverList rl = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            rl = new ReceiverList(this, callerApp, callingPid, callingUid,userId, receiver);
            if (rl.app != null) {
                //保存了该进程所注册的广播
                rl.app.receivers.add(rl);
            } else {
                try {
                    receiver.asBinder().linkToDeath(rl, 0);
                } catch (RemoteException e) {
                    return sticky;
                }
                rl.linkedToDeath = true;
            }
            //以IIntentReceiver为key,又记录在了AMS的HashMap中
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        } else if (rl.uid != callingUid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for uid " + callingUid
                    + " was previously registered for uid " + rl.uid);
        } else if (rl.pid != callingPid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for pid " + callingPid
                    + " was previously registered for pid " + rl.pid);
        } else if (rl.userId != userId) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for user " + userId
                    + " was previously registered for user " + rl.userId);
        }
        //上面只是把广播接收器receiver保存起来了，但是还没有把它和filter关联起来，这里就创建一个BroadcastFilter来把广播接收器列表rl和filter关联起来，然后保存在ActivityManagerService中的成员变量mReceiverResolver中去
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,permission, callingUid, userId);
        rl.add(bf);

        mReceiverResolver.addFilter(bf);

        // Enqueue broadcasts for all existing stickies that match
        // this filter.
        if (allSticky != null) {
            ArrayList receivers = new ArrayList();
            receivers.add(bf);

            int N = allSticky.size();
            for (int i=0; i<N; i++) {
                Intent intent = (Intent)allSticky.get(i);
                BroadcastQueue queue = broadcastQueueForIntent(intent);
                BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                        null, -1, -1, null, null, AppOpsManager.OP_NONE, receivers, null, 0,
                        null, null, false, true, true, -1);
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        }

        return sticky;
    }
}

```

# 取消注册
