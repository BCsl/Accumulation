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
首先检验`Intent`的有效性，是否带文件描述符，
```java
//resultTo：null，resultCode：-resultData：null，map：null，requiredPermission：null
//appOp：-1，serialized：false，sticky：false
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle map,
        String requiredPermission, int appOp, boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
      //检验Intent
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

## Step4

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
        try {
            if (AppGlobals.getPackageManager().isProtectedBroadcast(
                    intent.getAction())) {
                String msg = "Permission Denial: not allowed to send broadcast "
                        + intent.getAction() + " from pid="
                        + callingPid + ", uid=" + callingUid;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            } else if (AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(intent.getAction())) {
                // Special case for compatibility: we don't want apps to send this,
                // but historically it has not been protected and apps may be using it
                // to poke their own app widget.  So, instead of making it protected,
                // just limit it to the caller.
                if (callerApp == null) {
                    String msg = "Permission Denial: not allowed to send broadcast "
                            + intent.getAction() + " from unknown caller.";
                    Slog.w(TAG, msg);
                    throw new SecurityException(msg);
                } else if (intent.getComponent() != null) {
                    // They are good enough to send to an explicit component...  verify
                    // it is being sent to the calling app.
                    if (!intent.getComponent().getPackageName().equals(
                            callerApp.info.packageName)) {
                        String msg = "Permission Denial: not allowed to send broadcast "
                                + intent.getAction() + " to "
                                + intent.getComponent().getPackageName() + " from "
                                + callerApp.info.packageName;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }
                } else {
                    // Limit broadcast to their own package.
                    intent.setPackage(callerApp.info.packageName);
                }
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Remote exception", e);
            return ActivityManager.BROADCAST_SUCCESS;
        }
    }

    // Handle special intents: if this broadcast is from the package
    // manager about a package being removed, we need to remove all of
    // its activities from the history stack.
    final boolean uidRemoved = Intent.ACTION_UID_REMOVED.equals(
            intent.getAction());
    if (Intent.ACTION_PACKAGE_REMOVED.equals(intent.getAction())
            || Intent.ACTION_PACKAGE_CHANGED.equals(intent.getAction())
            || Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE.equals(intent.getAction())
            || uidRemoved) {
        if (checkComponentPermission(
                android.Manifest.permission.BROADCAST_PACKAGE_REMOVED,
                callingPid, callingUid, -1, true)
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
            String msg = "Permission Denial: " + intent.getAction()
                    + " broadcast from " + callerPackage + " (pid=" + callingPid
                    + ", uid=" + callingUid + ")"
                    + " requires "
                    + android.Manifest.permission.BROADCAST_PACKAGE_REMOVED;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }

    // Special case for adding a package: by default turn on compatibility
    // mode.
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
    if (sticky) {
        if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,callingPid, callingUid)
                != PackageManager.PERMISSION_GRANTED) {
            String msg = "Permission Denial: broadcastIntent() requesting a sticky broadcast from pid="
                    + callingPid + ", uid=" + callingUid
                    + " requires " + android.Manifest.permission.BROADCAST_STICKY;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        if (requiredPermission != null) {
            Slog.w(TAG, "Can't broadcast sticky intent " + intent
                    + " and enforce permission " + requiredPermission);
            return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
        }
        if (intent.getComponent() != null) {
            throw new SecurityException(
                    "Sticky broadcasts can't target a specific component");
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
    if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
             == 0) {
        receivers = collectReceiverComponents(intent, resolvedType, users);
    }
    if (intent.getComponent() == null) {
        registeredReceivers = mReceiverResolver.queryIntent(intent,
                resolvedType, false, userId);
    }

    final boolean replacePending =
            (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;

    if (DEBUG_BROADCAST) Slog.v(TAG, "Enqueing broadcast: " + intent.getAction()
            + " replacePending=" + replacePending);

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
        if (DEBUG_BROADCAST) Slog.v(
                TAG, "Enqueueing parallel broadcast " + r);
        final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
        if (!replaced) {
            queue.enqueueParallelBroadcastLocked(r);
            queue.scheduleBroadcastsLocked();
        }
        registeredReceivers = null;
        NR = 0;
    }

    // Merge into one list.
    int ir = 0;
    if (receivers != null) {
        // A special case for PACKAGE_ADDED: do not allow the package
        // being added to see this broadcast.  This prevents them from
        // using this as a back door to get run as soon as they are
        // installed.  Maybe in the future we want to have a special install
        // broadcast or such for apps, but we'd like to deliberately make
        // this decision.
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

        int NT = receivers != null ? receivers.size() : 0;
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

    if ((receivers != null && receivers.size() > 0)
            || resultTo != null) {
        BroadcastQueue queue = broadcastQueueForIntent(intent);
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType,
                requiredPermission, appOp, receivers, resultTo, resultCode,
                resultData, map, ordered, sticky, false, userId);
        if (DEBUG_BROADCAST) Slog.v(
                TAG, "Enqueueing ordered broadcast " + r
                + ": prev had " + queue.mOrderedBroadcasts.size());
        if (DEBUG_BROADCAST) {
            int seq = r.intent.getIntExtra("seq", -1);
            Slog.i(TAG, "Enqueueing broadcast " + r.intent.getAction() + " seq=" + seq);
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
