# Android广播发送

## Step1
同样，不管是`Activity`、`Service`上发送广播，最终实现都在`ContextImpl`，最后发送到`ActivityManagerService`处理

```java
@Override
 public void sendBroadcast(Intent intent) {
     warnIfCallingFromSystemProcess();
     String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
     try {
         intent.prepareToLeaveProcess();
         ActivityManagerNative.getDefault().broadcastIntent(
             mMainThread.getApplicationThread(), intent, resolvedType, null,
             Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, false, false,
             getUserId());
     } catch (RemoteException e) {
     }
 }
```
## Step2
由`ActivityManagerProxy`发送到`AMS`

```java
//resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
//appOp：-1，serialized：false，sticky：false
public int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType,  IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle map,
        String requiredPermission, int appOp, boolean serialized,
        boolean sticky, int userId) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
    data.writeInt(resultCode);
    data.writeString(resultData);
    data.writeBundle(map);
    data.writeString(requiredPermission);
    data.writeInt(appOp);
    data.writeInt(serialized ? 1 : 0);
    data.writeInt(sticky ? 1 : 0);
    data.writeInt(userId);
    mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
    reply.readException();
    int res = reply.readInt();
    reply.recycle();
    data.recycle();
    return res;
}
```

## Step3
`AMS`处理`BROADCAST_INTENT_TRANSACTION`
```java
//resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
//appOp：-1，serialized：false，sticky：false
case BROADCAST_INTENT_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app =b != null ? ApplicationThreadNative.asInterface(b) : null;//依然是Binder本地对象
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            b = data.readStrongBinder();//null
            IIntentReceiver resultTo =b != null ? IIntentReceiver.Stub.asInterface(b) : null;//null
            int resultCode = data.readInt();//-1
            String resultData = data.readString();//null
            Bundle resultExtras = data.readBundle();//null
            String perm = data.readString();//null
            int appOp = data.readInt();//-1
            boolean serialized = data.readInt() != 0;//0
            boolean sticky = data.readInt() != 0;//0
            int userId = data.readInt();
            int res = broadcastIntent(app, intent, resolvedType, resultTo,
                    resultCode, resultData, resultExtras, perm, appOp,
                    serialized, sticky, userId);
            reply.writeNoException();
            reply.writeInt(res);
            return true;
        }

```
首先检验`Intent`的有效性，是否带文件描述符，如果进程未启动完成，带`FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT`，表示可以在进程未启动完之前接收广播，没有该标识则抛异常，接着调用方法`broadcastIntentLocked`处理
```java
//resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
//appOp：-1，serialized：false，sticky：false
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle map,
        String requiredPermission, int appOp, boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
    //intent有效性检查，不能带文件描述符
        intent = verifyBroadcastLocked(intent);
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo,
                resultCode, resultData, map, requiredPermission, appOp, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}

```

## Step4 获取目标Receivers
首先处理一些特殊的广播，如：接收到`PackageManager`发来的应用包移除广播，就会把所有属于该包下的`ActivityRecord`出栈等，`AMS`内有两种广播队列，分别是前台广播队列`mFgBroadcastQueue`保存带`FLAG_RECEIVER_FOREGROUND`的广播，和后台广播队列`mBgBroadcastQueue`，先会尝试向并行`receivers`递送广播，此时会调用到`queue.scheduleBroadcastsLocked()`，简单地说就是，新建一个`BroadcastRecord`节点，并插入对应的`BroadcastQueue`，最后发起实际的广播调度（`scheduleBroadcastsLocked()`），不光并行处理部分需要一个`BroadcastRecord`节点，串行处理部分也需要`BroadcastRecord`节点。也就是说，要激发一次广播，`AMS`必须构造一个或两个`BroadcastRecord`节点，并将之插入合适的广播队列。插入成功后，再执行队列的`scheduleBroadcastsLocked()`动作
`BroadcastRecord`节点内部的`receivers`列表，记录着和这个广播动作相关的目标`receiver`信息，该列表内部的子节点可能是`ResolveInfo`类型的，也可能是`BroadcastFilter`类型的。`ResolveInfo`是从`PKMS`处查到的静态`receiver`的描述信息，它的源头是`PKMS`解析的那些`AndroidManifest.xml`文件。而`BroadcastFilter`来自于动态注册`receiver`时，保存在`AMS`成员变量`mReceiverResolver`
```java
//resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
//appOp：-1，serialized：false，sticky：false
private final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle map, String requiredPermission, int appOp,
        boolean ordered, boolean sticky, int callingPid, int callingUid,
        int userId) {
    intent = new Intent(intent);

    // By default broadcasts do not go to stopped apps.
    //在Android3.1之后，PKMS加强了对“处于停止状态的”应用的管理。如果一个应用在安装后从来没有启动过，或者已经被用户强制停止了，那么这个应用就处于停止状态（stoppedstate）。为了达到精细调整的目的，Android增加了2个flag：FLAG_INCLUDE_STOPPED_PACKAGES和FLAG_EXCLUDE_STOPPED_PACKAGES，以此来表示intent是否要激活“处于停止状态的”应用。而默认则是不激活
    intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

    //....

    userId = handleIncomingUser(callingPid, callingUid, userId,true, false, "broadcast", callerPackage);//权限检测

    // Make sure that the user who is receiving this broadcast is started.
    // If not, we will just skip it.
    if (userId != UserHandle.USER_ALL && mStartedUsers.get(userId) == null) {
        if (callingUid != Process.SYSTEM_UID || (intent.getFlags()
                & Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
            Slog.w(TAG, "Skipping broadcast of " + intent+ ": user " + userId + " is stopped");
            return ActivityManager.BROADCAST_SUCCESS;
        }
    }

    /*
     * Prevent non-system code (defined here to be non-persistent processes) from sending protected broadcasts.
     */
    int callingAppId = UserHandle.getAppId(callingUid);
    if (callingAppId == Process.SYSTEM_UID || callingAppId == Process.PHONE_UID
        || callingAppId == Process.SHELL_UID || callingAppId == Process.BLUETOOTH_UID ||
        callingUid == 0) {
        // Always okay.
    } else if (callerApp == null || !callerApp.persistent) {
        //...
    }

    // Handle special intents: if this broadcast is from the package
    // manager about a package being removed, we need to remove all of
    // its activities from the history stack.
    final boolean uidRemoved = Intent.ACTION_UID_REMOVED.equals(intent.getAction());
    if (Intent.ACTION_PACKAGE_REMOVED.equals(intent.getAction())
            || Intent.ACTION_PACKAGE_CHANGED.equals(intent.getAction())
            || Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE.equals(intent.getAction())
            || uidRemoved) {
        if (checkComponentPermission(android.Manifest.permission.BROADCAST_PACKAGE_REMOVED,callingPid, callingUid, -1, true)
                == PackageManager.PERMISSION_GRANTED) {
            if (uidRemoved) {
                final Bundle intentExtras = intent.getExtras();
                final int uid = intentExtras != null
                        ? intentExtras.getInt(Intent.EXTRA_UID) : -1;
                if (uid >= 0) {
                    BatteryStatsImpl bs = mBatteryStatsService.getActiveStatistics();
                    synchronized (bs) {
                        bs.removeUidStatsLocked(uid);
                    }
                    mAppOpsService.uidRemoved(uid);
                }
            } else {
                // If resources are unavailable just force stop all
                // those packages and flush the attribute cache as well.
                if (Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE.equals(intent.getAction())) {
                    String list[] = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
                    if (list != null && (list.length > 0)) {
                        for (String pkg : list) {
                            forceStopPackageLocked(pkg, -1, false, true, true, false, userId,
                                    "storage unmount");
                        }
                        sendPackageBroadcastLocked(
                                IApplicationThread.EXTERNAL_STORAGE_UNAVAILABLE, list, userId);
                    }
                } else {
                    Uri data = intent.getData();
                    String ssp;
                    if (data != null && (ssp=data.getSchemeSpecificPart()) != null) {
                        boolean removed = Intent.ACTION_PACKAGE_REMOVED.equals(
                                intent.getAction());
                        if (!intent.getBooleanExtra(Intent.EXTRA_DONT_KILL_APP, false)) {
                            forceStopPackageLocked(ssp, UserHandle.getAppId(
                                    intent.getIntExtra(Intent.EXTRA_UID, -1)), false, true, true,
                                    false, userId, removed ? "pkg removed" : "pkg changed");
                        }
                        if (removed) {
                            sendPackageBroadcastLocked(IApplicationThread.PACKAGE_REMOVED,
                                    new String[] {ssp}, userId);
                            if (!intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {
                                mAppOpsService.packageRemoved(
                                        intent.getIntExtra(Intent.EXTRA_UID, -1), ssp);

                                // Remove all permissions granted from/to this package
                                removeUriPermissionsForPackageLocked(ssp, userId, true);
                            }
                        }
                    }
                }
            }
        } else {
            //"Permission Denial: "
            throw new SecurityException(msg);
        }

    // Special case for adding a package: by default turn on compatibility mode.
    } else if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())) {
        Uri data = intent.getData();
        String ssp;
        if (data != null && (ssp=data.getSchemeSpecificPart()) != null) {
            mCompatModePackages.handlePackageAddedLocked(ssp,
                    intent.getBooleanExtra(Intent.EXTRA_REPLACING, false));
        }
    }

    /*
     * If this is the time zone changed action, queue up a message that will reset the timezone
     * of all currently running processes. This message will get queued up before the broadcast
     * happens.
     */
    if (intent.ACTION_TIMEZONE_CHANGED.equals(intent.getAction())) {
        mHandler.sendEmptyMessage(UPDATE_TIME_ZONE);
    }

    if (intent.ACTION_CLEAR_DNS_CACHE.equals(intent.getAction())) {
        mHandler.sendEmptyMessage(CLEAR_DNS_CACHE_MSG);
    }

    if (Proxy.PROXY_CHANGE_ACTION.equals(intent.getAction())) {
        ProxyProperties proxy = intent.getParcelableExtra("proxy");
        mHandler.sendMessage(mHandler.obtainMessage(UPDATE_HTTP_PROXY_MSG, proxy));
    }

    // Add to the sticky list if requested.
    if (sticky) {//false，先忽略
        if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,callingPid, callingUid)
                != PackageManager.PERMISSION_GRANTED) {
                  //...
            throw new SecurityException(msg);
        }
        if (requiredPermission != null) {
            Slog.w(TAG, "Can't broadcast sticky intent " + intent+ " and enforce permission " + requiredPermission);
            return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
        }
        if (intent.getComponent() != null) {
            throw new SecurityException("Sticky broadcasts can't target a specific component");
        }
        // We use userId directly here, since the "all" target is maintained
        // as a separate set of sticky broadcasts.
        if (userId != UserHandle.USER_ALL) {
            // But first, if this is not a broadcast to all users, then
            // make sure it doesn't conflict with an existing broadcast to
            // all users.
            ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(
                    UserHandle.USER_ALL);
            if (stickies != null) {
                ArrayList<Intent> list = stickies.get(intent.getAction());
                if (list != null) {
                    int N = list.size();
                    int i;
                    for (i=0; i<N; i++) {
                        if (intent.filterEquals(list.get(i))) {
                            throw new IllegalArgumentException(
                                    "Sticky broadcast " + intent + " for user "
                                    + userId + " conflicts with existing global broadcast");
                        }
                    }
                }
            }
        }
        ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
        if (stickies == null) {
            stickies = new ArrayMap<String, ArrayList<Intent>>();
            mStickyBroadcasts.put(userId, stickies);
        }
        ArrayList<Intent> list = stickies.get(intent.getAction());
        if (list == null) {
            list = new ArrayList<Intent>();
            stickies.put(intent.getAction(), list);
        }
        int N = list.size();
        int i;
        for (i=0; i<N; i++) {
            if (intent.filterEquals(list.get(i))) {
                // This sticky already exists, replace it.
                list.set(i, new Intent(intent));
                break;
            }
        }
        if (i >= N) {
            list.add(new Intent(intent));
        }
    }

    int[] users;
    if (userId == UserHandle.USER_ALL) {
        // Caller wants broadcast to go to all started users.
        users = mStartedUserArray;
    } else {
        // Caller wants broadcast to go to one specific user.
        users = new int[] {userId};
    }

    // Figure out who all will receive this broadcast.
    List receivers = null;
    List<BroadcastFilter> registeredReceivers = null;
    // Need to resolve the intent to interested receivers...
    //FLAG_RECEIVER_REGISTERED_ONLY标识标识的是动态注册的广播，至于静态注册的广播是通过PackageManager获得
    if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)== 0) {
        receivers = collectReceiverComponents(intent, resolvedType, users);
    }
    if (intent.getComponent() == null) {
        //找到符合的接受者（详细之后再看），返回结果按照优先级排序
        registeredReceivers = mReceiverResolver.queryIntent(intent,resolvedType, false, userId);
    }
    //是否替代
    final boolean replacePending =(intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;


    int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
    if (!ordered && NR > 0) {
        // If we are not serializing this broadcast, then send the
        // registered receivers separately so they don't wait for the
        // components to be launched.
        final BroadcastQueue queue = broadcastQueueForIntent(intent);
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType, requiredPermission,
                appOp, registeredReceivers, resultTo, resultCode, resultData, map,
                ordered, sticky, false, userId);

        final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
        if (!replaced) {
            queue.enqueueParallelBroadcastLocked(r);
            queue.scheduleBroadcastsLocked();//
        }
        //上面完成了无序广播的调度
        registeredReceivers = null;
        NR = 0;
    }
    //看来是先处理动态注册的，之后才是静态注册的
    // Merge into one list.
    int ir = 0;
    //receiver记录的是在清单文件夹静态注册的广播
    if (receivers != null) {
        // A special case for PACKAGE_ADDED: do not allow the package
        // being added to see this broadcast.  This prevents them from
        // using this as a back door to get run as soon as they are
        // installed.  Maybe in the future we want to have a special install
        // broadcast or such for apps, but we'd like to deliberately make
        // this decision.
        //防止监听自己的应用安装后自己收到广播而自动启动
        String skipPackages[] = null;
        if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
            Uri data = intent.getData();
            if (data != null) {
                String pkgName = data.getSchemeSpecificPart();
                if (pkgName != null) {
                    skipPackages = new String[] { pkgName };
                }
            }
        } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
            skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
        }
        //移除特定的广播任务
        if (skipPackages != null && (skipPackages.length > 0)) {
            for (String skipPackage : skipPackages) {
                if (skipPackage != null) {
                    int NT = receivers.size();
                    for (int it=0; it<NT; it++) {
                        ResolveInfo curt = (ResolveInfo)receivers.get(it);
                        if (curt.activityInfo.packageName.equals(skipPackage)) {
                            receivers.remove(it);
                            it--;
                            NT--;
                        }
                    }
                }
            }
        }
        //如果是无序广播，在上面NR已置为了0，下面是用于合并有序和静态注册的广播
        int NT = receivers != null ? receivers.size() : 0;//静态注册的广播数量
        int it = 0;
        ResolveInfo curt = null;
        BroadcastFilter curr = null;
        while (it < NT && ir < NR) {
            if (curt == null) {
                curt = (ResolveInfo)receivers.get(it);
            }
            if (curr == null) {
                curr = registeredReceivers.get(ir);
            }
            if (curr.getPriority() >= curt.priority) {
                // Insert this broadcast record into the final list.
                receivers.add(it, curr);
                ir++;
                curr = null;
                it++;
                NT++;
            } else {
                // Skip to the next ResolveInfo in the final list.
                it++;
                curt = null;
            }
        }
    }
    while (ir < NR) {
        if (receivers == null) {
            receivers = new ArrayList();
        }
        receivers.add(registeredReceivers.get(ir));
        ir++;
    }
    //下面的操作和处理无序广播一样
    if ((receivers != null && receivers.size() > 0)|| resultTo != null) {
        BroadcastQueue queue = broadcastQueueForIntent(intent);
        //resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
        //appOp：-1，serialized：false，sticky：false
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType,
                requiredPermission, appOp, receivers, resultTo, resultCode,
                resultData, map, ordered, sticky, false, userId);
        }
        boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
        if (!replaced) {
            queue.enqueueOrderedBroadcastLocked(r);
            queue.scheduleBroadcastsLocked();
        }
    }

    return ActivityManager.BROADCAST_SUCCESS;
}

```
## Step5

以上不管是处理有序广播还是无序广播，最重要的无疑是`scheduleBroadcastsLocked`方法调用
```java
BroadcastQueue.java

public void scheduleBroadcastsLocked() {
       if (mBroadcastsScheduled) {
           return;
       }
       mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
       mBroadcastsScheduled = true;
   }

```
处理`BROADCAST_TIMEOUT_MSG`消息

```java
BroadcastQueue.java

case BROADCAST_INTENT_MSG: {
    processNextBroadcast(true);
} break;
```

## Step6
所有的静态`receiver`都是串行处理的，而动态`receiver`则会按照发广播时指定的方式，进行“并行”或“串行”处理。能够并行处理的广播，其对应的若干`receiver`一定都已经存在了，不会牵扯到启动新进程的操作，所以可以在一个`while`循环中，一次性全部`deliver`。而有序广播，则需要一个一个地处理，其滚动处理的手段是发送事件，也就是说，在一个`receiver`处理完毕后，会利用广播队列（`BroadcastQueue`）的`mHandler`，发送一个`BROADCAST_INTENT_MSG`事件，从而执行下一次的`processNextBroadcast`的调度

### Step6.1 处理无序广播
遍历并行列表（`mParallelBroadcasts`）的每一个`BroadcastRecord`以及其中的`receivers`列表。对于无序广播而言，`receivers`列表中的每个子节点是个`BroadcastFilter`。我们直接通过方法`deliverToRegisteredReceiverLocked`将广播递送出去即可

```java

final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        BroadcastRecord r;

        mService.updateCpuStats();

        if (fromMsg) {
          //表示BROADCAST_INTENT_MSG消息已经处理完了
            mBroadcastsScheduled = false;
        }

        // First, deliver any non-serialized broadcasts right away.
        while (mParallelBroadcasts.size() > 0) {
            //BroadcastRecord类型的r内部记录了该广播的所有接收者
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            final int N = r.receivers.size();//接收者数量
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                //把非有序队列中各个广播发送给广播接收者
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
            }
            addBroadcastToHistoryLocked(r);
        }
        //.......
      }
    }
```

### Step6.2 BroadcastQueue#deliverToRegisteredReceiverLocked

先进行的权限判断、操作的检测、目标进程是否启动等操作，如果都OK，前面可知`BroadcastFilter`用来关联了动态注册的`IIntentReceiver`和`IntentFilter`,所以拿到一个`BroadcastFilter`接可以知道宿主`IIntentReceiver`，最后调用`performReceiveLocked`
```java
//order=false；
private final void deliverToRegisteredReceiverLocked(BroadcastRecord r,BroadcastFilter filter, boolean ordered) {
    boolean skip = false;
    if (filter.requiredPermission != null) {
        int perm = mService.checkComponentPermission(filter.requiredPermission,r.callingPid, r.callingUid, -1, true);
        if (perm != PackageManager.PERMISSION_GRANTED) {
            //Permission Denial: broadcasting
            skip = true;
        }
    }
    if (!skip && r.requiredPermission != null) {
        int perm = mService.checkComponentPermission(r.requiredPermission,
                filter.receiverList.pid, filter.receiverList.uid, -1, true);
        if (perm != PackageManager.PERMISSION_GRANTED) {
            //Permission Denial: receiving
            skip = true;
        }
    }
    if (r.appOp != AppOpsManager.OP_NONE) {
        int mode = mService.mAppOpsService.noteOperation(r.appOp,
                filter.receiverList.uid, filter.packageName);
        if (mode != AppOpsManager.MODE_ALLOWED) {
            if (DEBUG_BROADCAST)  Slog.v(TAG,
                    "App op " + r.appOp + " not allowed for broadcast to uid "
                    + filter.receiverList.uid + " pkg " + filter.packageName);
            skip = true;
        }
    }
    if (!skip) {
        skip = !mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                r.callingPid, r.resolvedType, filter.receiverList.uid);
    }

    if (filter.receiverList.app == null || filter.receiverList.app.crashing) {
        Slog.w(TAG, "Skipping deliver [" + mQueueName + "] " + r+ " to " + filter.receiverList + ": process crashing");
        skip = true;
    }

    if (!skip) {
        // If this is not being sent as an ordered broadcast, then we
        // don't want to touch the fields that keep track of the current
        // state of ordered broadcasts.
        if (ordered) {//false
            r.receiver = filter.receiverList.receiver.asBinder();//记录的是IIntentReceiver的Binder本地对象
            r.curFilter = filter;
            filter.receiverList.curBroadcast = r;
            r.state = BroadcastRecord.CALL_IN_RECEIVE;
            if (filter.receiverList.app != null) {
                // Bump hosting application to no longer be in background scheduling class.  Note that we can't do that if there
                // isn't an app...  but we can only be in that case for things that directly call the IActivityManager API, which
                // are already core system stuff so don't matter for this.
                r.curApp = filter.receiverList.app;
                filter.receiverList.app.curReceiver = r;
                mService.updateOomAdjLocked(r.curApp, true);
            }
        }
        try {
          //filter.receiverList.app:ApplicationThread
          //filter.receiverList.receiver：
            performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                new Intent(r.intent), r.resultCode, r.resultData,
                r.resultExtras, r.ordered, r.initialSticky, r.userId);
            if (ordered) {
                r.state = BroadcastRecord.CALL_DONE_RECEIVE;
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Failure sending broadcast " + r.intent, e);
            if (ordered) {
                r.receiver = null;
                r.curFilter = null;
                filter.receiverList.curBroadcast = null;
                if (filter.receiverList.app != null) {
                    filter.receiverList.app.curReceiver = null;
                }
            }
        }
    }
}

```

### 6.3 BroadcastQueue#performReceiveLocked

接着如果当前进程是存在的且已经启动，通过`ApplicationThread`来进行回调但实际上还是通过`IIntentReceiver`来回调，
```java
BroadcastQueue.java

private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null && app.thread != null) {
        // If we have an app thread, do the call through that so it is correctly ordered with other one-way calls.
        app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                data, extras, ordered, sticky, sendingUser, app.repProcState);
    } else {
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}

```

### 6.4 IIntentReceiver#performReceive

```java
LoadedApk#ReceiverDispatcher#IIntentReceiver

public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                //获取宿主ReceiverDispatcher
                LoadedApk.ReceiverDispatcher rd = mDispatcher.get();

                if (rd != null) {
                    rd.performReceive(intent, resultCode, data, extras,ordered, sticky, sendingUser);
                } else {
                    // The activity manager dispatched a broadcast to a registered
                    // receiver in this process, but before it could be delivered the
                    // receiver was unregistered.  Acknowledge the broadcast on its
                    // behalf so that the system's broadcast sequence can continue.
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,"Finishing broadcast to unregistered receiver");
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

```
### 6.5 ReceiverDispatcher#performReceive
构造`Args`封装成一个消息,`Args`继承自`PendingResult`实现了`Runnable`接口
```java
LoadedApk#ReceiverDispatcher

public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {

    Args args = new Args(intent, resultCode, data, extras, ordered,sticky, sendingUser);
    if (!mActivityThread.post(args)) {
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManagerNative.getDefault();

            args.sendFinished(mgr);
        }
    }
}

```

### 6.6 Args#run
但是构造一个`Args`对象又有什么用？或者说`PendingResult`的作用是什么？这个`PendingResult`可以在`BroadcastReceiver#onReceive`方法中通过`goAsync`方法返回，表示的结果的状态，在一个广播处理完之后必须调用其`finish`方法（这个在回调`onReceiver`方法后会自动回调）,`finish`方法用来完成某个广播，对于一个已经处理完的广播，如果是有序广播，接收完之后需要向`AMS`发一个回包，以便`AMS`可以将这个有序广播发送给下一个广播接收者。

```java
final class Args extends BroadcastReceiver.PendingResult implements Runnable {
    private Intent mCurIntent;
    private final boolean mOrdered;

    public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
            boolean ordered, boolean sticky, int sendingUser) {
        super(resultCode, resultData, resultExtras,
                mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED,
                ordered, sticky, mIIntentReceiver.asBinder(), sendingUser);
        mCurIntent = intent;
        mOrdered = ordered;
    }

    public void run() {
        final BroadcastReceiver receiver = mReceiver;//具体注册的BroadcastReceiver，保存在ReceiverDispatcher成员变量
        final boolean ordered = mOrdered;


        final IActivityManager mgr = ActivityManagerNative.getDefault();
        final Intent intent = mCurIntent;
        mCurIntent = null;

        if (receiver == null || mForgotten) {
            if (mRegistered && ordered) {
                sendFinished(mgr);
            }
            return;
        }

        try {
            ClassLoader cl =  mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            setExtrasClassLoader(cl);
            receiver.setPendingResult(this);
            receiver.onReceive(mContext, intent);//回调具体的Receiver
        } catch (Exception e) {
            if (mRegistered && ordered) {
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,"Finishing failed broadcast to " + mReceiver);
                sendFinished(mgr);
            }
              //...
        }
        if (receiver.getPendingResult() != null) {
            finish();
        }
    }
}

```
