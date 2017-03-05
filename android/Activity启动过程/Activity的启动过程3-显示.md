# Activity的启动过程3-显示

## Step25(Setp14)

找到当前取得焦点的`ActivityStack`,启动目标`ActivityRecord`

```java
ActivityStackSupervisor.java

//targetStack:mForceStack，setp12新建的，target:上一个暂停的`Activity`(`mHomeStack`的Laucher)，targetOptions:null
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,Bundle targetOptions) {
     if (targetStack == null) {
         targetStack = getFocusedStack();//mForceStack,当前状态STACK_STATE_HOME_TO_BACK（Step12）
     }
     boolean result = false;
     for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
         final ActivityStack stack = mStacks.get(stackNdx);
         if (isFrontStack(stack)) {//true
             if (stack == targetStack) {
                 result = stack.resumeTopActivityLocked(target,targetOptions);
             } else {
                 stack.resumeTopActivityLocked(null);
             }
         }
     }
     return result;
 }
```

## Setp26

暂停完所有`mLastStack`的`mResumedActivity`后，就需要显示当前`mFocusedStack`栈顶的Activity，通过`WindowManagerService#setAppVisibility`来进行界面的显示？并设置`WWindowManagerService`转换动画，由于所属进程并未创建，最后调用`ActivityStackSupervisor.startSpecificActivityLocked`

```java
ActivityStack.java
//prev=上一个暂停的`Activity`(`mHomeStack`的Laucher),options=null
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        // Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);//Step12新建的ActivityRecord

        // Remember how we'll process this pause/resume situation, and ensure that the state is reset however we wind up proceeding.
        final boolean userLeaving = mStackSupervisor.mUserLeaving;//true Step12,向源Activity发送用户离开事件
        mStackSupervisor.mUserLeaving = false;

        if (next == null) {//false
            // There are no more activities!  Let's just start up the Launcher...
            //...
        }

        next.delayedResume = false;

        // If the top activity is the resumed one, nothing to do.
        //mResumedActivity为null，next.state为ActivityState.INITIALIZING
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&mStackSupervisor.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            //...
            return false;
        }

        final TaskRecord nextTask = next.task;
        final TaskRecord prevTask = prev != null ? prev.task : null;//Launcher所属的ActivityStack
        if (prevTask != null && prevTask.mOnTopOfHome && prev.finishing && prev.frontOfTask) {//false
          //...
        }

        // If we are sleeping, and there is no resumed activity, and the top activity is paused, well that is the state we want.
        //如果要启动的Activity是上次终止的Activity，且系统正要进入关机或者睡眠，那么直接返回，因为没有意义
        if (mService.isSleepingOrShuttingDown()&& mLastPausedActivity == next&& mStackSupervisor.allPausedActivitiesComplete()) {
            //....
            return false;
        }

        // Make sure that the user who owns this activity is started.  If not, we will just leave it as is because someone should be bringing another //user's activities to the top of the stack.
        if (mService.mStartedUsers.get(next.userId) == null) {
            Slog.w(TAG, "Skipping resume of top activity " + next+ ": user " + next.userId + " is stopped");
            return false;
        }

        // The activity may be waiting for stop, but that is no longer appropriate for it.
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mWaitingVisibleActivities.remove(next);

        next.updateOptionsLocked(options);//null

        // If we are currently pausing an activity, then don't do anything until that is done.
        //如果还有正在暂停的`ActivityRecord`，不再继续
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG,"resumeTopActivityLocked: Skip resume: some activity pausing.");
            return false;
        }
        //...
        // We need to start pausing the current activity so the top one can be resumed...
        //分别暂停所有`mStacks`中不是前台`Stack`中的所有`mResumedActivity`和暂停当前栈的`mResumedActivity`
        //直到最后一个需要暂停Activity暂停成功，这里返回false
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving);//返回true
        if (mResumedActivity != null) {//mResumedActivity为NULL，当前`ActivityStack`在Setp12新建
            pausing = true;
            startPausingLocked(userLeaving, false);//暂停栈正在显示的Activity，见Step17
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
        }
        //如果有需要pause的Activity，则需要先暂停再显示当前需要显示的
        if (pausing) {//false
          //...
        }

        // If the most recent activity was noHistory but was only stopped rather than stopped+finished because the device went to //sleep, we need to make //sure to finish it as we're making a new activity topmost.
        if (mService.mSleeping && mLastNoHistoryActivity != null &&!mLastNoHistoryActivity.finishing) {//false
            if (DEBUG_STATES) Slog.d(TAG, "no-history finish of " + mLastNoHistoryActivity +" on new resume");
            requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,null, "no-history", false);
            mLastNoHistoryActivity = null;
        }
        //需要显示next
        if (prev != null && prev != next) {//Laucher
            //虽然Launcher已经暂停了，但是当前还是可见的！但next仍然是不可见的
            if (!prev.waitingVisible && next != null && !next.nowVisible) {//waitingVisible在Setp25置为false，true
                prev.waitingVisible = true;
                mStackSupervisor.mWaitingVisibleActivities.add(prev);
            } else {
                // The next activity is already visible, so hide the previous activity's windows right now so we can show the new one ASAP.
                // We only do this if the previous is finishing, which should mean it is on top of the one being resumed so hiding it quickly
                // is good.  Otherwise, we want to do the normal route of allowing the resumed activity to be shown so we can decide if the
                // previous should actually be hidden depending on whether the new one is found to be full-screen or not.
                //next如果已经可见了，那么现在就隐藏prev的Window，如果prev正在finishing，代表当前正在显示，所以要尽快的把它隐藏。否则，正常情况下我们希望
                //当前显示的Activity的是否真正隐藏依赖于是否找到一个全屏显示的Activity
                if (prev.finishing) {
                    mWindowManager.setAppVisibility(prev.appToken, false);
                    if (DEBUG_SWITCH)
                    Slog.v(TAG, "Not waiting for visible to hide: "+ prev + ", waitingVisible="
                            + (prev != null ? prev.waitingVisible : null)+ ", nowVisible=" + next.nowVisible);
                } else {
                    if (DEBUG_SWITCH)
                    Slog.v(TAG, "Previous already visible but still waiting to hide: "+ prev + ", waitingVisible="
                        + (prev != null ? prev.waitingVisible : null)+ ", nowVisible=" + next.nowVisible);
                }
            }
        }

        // Launching this app's activity, make sure the app is no longer considered stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(next.packageName, false, next.userId);
             /* TODO: Verify if correct userid */
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
          //...
        }

        //开始显示next，需要隐藏prev
        boolean anim = true;
        if (prev != null) {
          //..
        }
        if (anim) {
            next.applyOptionsLocked();
        } else {
            next.clearOptionsLocked();
        }

        ActivityStack lastStack = mStackSupervisor.getLastStack();//mFocusedStack
        //next是新建的，所运行进程并未创建ProcessRecord类型的app为null
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Resume running: " + next);
            //....
        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.logo, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        return true;
    }
```

## Step27

```java
//r:setp12创建的需要显示的Activity，andResume:true,checkConfig:true
void startSpecificActivityLocked(ActivityRecord r,boolean andResume, boolean checkConfig) {
    //ActivityManagerService管理者所有组件的所在的进程
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,r.info.applicationInfo.uid, true);//return null

    r.task.stack.setLaunchTime(r);

    if (app != null && app.thread != null) {
        try {
            app.addPackage(r.info.packageName, mService.mProcessStats);
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }
    }
    //如果目标Activity所在的app进程还未开启，如果是从Launcher启动一个新的APP
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,"activity", r.intent.getComponent(), false, false, true);
}
```

## Step28

应用进程的创建和启动**(详细的需要之后写一篇单独的笔记记录)** 这里不再深究，最后通过`Process.start`启动一个新的应用进程，指定该进程的入口程序函数为`ActivityThread`类的`main()`方法，那么接下来就从它开始分析新的应用程序的启动过程

```java
ActivityManagerService.java
//knownToBeDead:true,intentFlags:0,hostingType:activity,allowWhileBooting:false,isolated:flase,keepIfLarge:true
final ProcessRecord startProcessLocked(String processName,ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,boolean isolated, boolean keepIfLarge) {
        ProcessRecord app;
        if (!isolated) {
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);//先尝试从缓存中获取，NULL
        } else {
            // If this is an isolated process, it can't re-use an existing process.
            app = null;
        }
        // We don't have to do anything more if:
        // (1) There is an existing application record; and
        // (2) The caller doesn't think it is dead, OR there is no thread object attached to it so we know it couldn't have crashed; and
        // (3) There is a pid assigned to it, so it is either starting or   already running.
        if (app != null && app.pid > 0) {
            if (!knownToBeDead || app.thread == null) {
                // We already have the app running, or are waiting for it to
                // come up (we have a pid but not yet its thread), so keep it.
                if (DEBUG_PROCESSES) Slog.v(TAG, "App already running: " + app);
                // If this is a new package in the process, add the package to the list
                app.addPackage(info.packageName, mProcessStats);
                return app;
            }
            // An application record is attached to a previous process,clean it up now.
            handleAppDiedLocked(app, true, true);
        }

        String hostingNameStr = hostingName != null? hostingName.flattenToShortString() : null;

        if (!isolated) {
            if ((intentFlags&Intent.FLAG_FROM_BACKGROUND) != 0) {
                // If we are in the background, then check to see if this process is bad.  If so, we will just silently fail.
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    return null;
                }
            } else {
                // When the user is explicitly starting a process, then clear its crash count so that we won't make it bad until they see at
                // least one crash dialog again, and make the process good again if it had been bad.
                if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                        + "/" + info.processName);
                mProcessCrashTimes.remove(info.processName, info.uid);
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    mBadProcesses.remove(info.processName, info.uid);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        }

        if (app == null) {
            app = newProcessRecordLocked(info, processName, isolated);
            if (app == null) {
                return null;
            }
            mProcessNames.put(processName, app.uid, app);
            if (isolated) {
                mIsolatedProcesses.put(app.uid, app);
            }
        } else {
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, mProcessStats);
        }

        // If the system is not ready yet, then hold off on starting this process until it is.
        if (!mProcessesReady&& !isAllowedWhileBooting(info)&& !allowWhileBooting) {
            if (!mProcessesOnHold.contains(app)) {
                mProcessesOnHold.add(app);
            }
            if (DEBUG_PROCESSES) Slog.v(TAG, "System not ready, putting on hold: " + app);
            return app;
        }

        startProcessLocked(app, hostingType, hostingNameStr);
        return (app.pid != 0) ? app : null;
    }
```

创建`Processrecord`实体

```java
//isolated=false;
final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,boolean isolated) {
    String proc = customProcess != null ? customProcess : info.processName;
    BatteryStatsImpl.Uid.Proc ps = null;
    BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
    int uid = info.uid;
    if (isolated) {
        //....
    }
    synchronized (stats) {
        ps = stats.getProcessStatsLocked(info.uid, proc);
    }
    return new ProcessRecord(ps, info, proc, uid);
}
```

启动新的进程，设置超时消息（和前面的暂停超时类似）

```java
//hostingType:activity
private final void startProcessLocked(ProcessRecord app,String hostingType, String hostingNameStr) {
        if (app.pid > 0 && app.pid != MY_PID) {
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            app.setPid(0);
        }

        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG,"startProcessLocked removing on hold: " + app);
        mProcessesOnHold.remove(app);

        updateCpuStats();

        try {
            int uid = app.uid;

            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            if (!app.isolated) {
                int[] permGids = null;
                try {
                    final PackageManager pm = mContext.getPackageManager();
                    permGids = pm.getPackageGids(app.info.packageName);

                    if (Environment.isExternalStorageEmulated()) {
                        if (pm.checkPermission(
                                android.Manifest.permission.ACCESS_ALL_EXTERNAL_STORAGE,
                                app.info.packageName) == PERMISSION_GRANTED) {
                            mountExternal = Zygote.MOUNT_EXTERNAL_MULTIUSER_ALL;
                        } else {
                            mountExternal = Zygote.MOUNT_EXTERNAL_MULTIUSER;
                        }
                    }
                } catch (PackageManager.NameNotFoundException e) {
                    Slog.w(TAG, "Unable to retrieve gids", e);
                }

                /*
                 * Add shared application GID so applications can share some
                 * resources like shared libraries
                 */
                if (permGids == null) {
                    gids = new int[1];
                } else {
                    gids = new int[permGids.length + 1];
                    System.arraycopy(permGids, 0, gids, 1, permGids.length);
                }
                gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
            }
            if (mFactoryTest != SystemServer.FACTORY_TEST_OFF) {
                if (mFactoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL
                        && mTopComponent != null
                        && app.processName.equals(mTopComponent.getPackageName())) {
                    uid = 0;
                }
                if (mFactoryTest == SystemServer.FACTORY_TEST_HIGH_LEVEL
                        && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                    uid = 0;
                }
            }
            int debugFlags = 0;
            //...

            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, null);

            BatteryStatsImpl bs = app.batteryStats.getBatteryStats();
            synchronized (bs) {
                if (bs.isOnBattery()) {
                    app.batteryStats.incStartsLocked();
                }
            }
            //...
            if (app.persistent) {
                Watchdog.getInstance().processStarted(app.processName, startResult.pid);
            }
            //...
            app.setPid(startResult.pid);
            app.usingWrapper = startResult.usingWrapper;
            app.removed = false;
            synchronized (mPidsSelfLocked) {
                this.mPidsSelfLocked.put(startResult.pid, app);
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        } catch (RuntimeException e) {
            // XXX do better error recovery.
            app.setPid(0);
            Slog.e(TAG, "Failure starting process " + app.processName, e);
        }
    }
```

## Step29

新的应用进程都做了哪些操作1.在主线程创建`Looper`循环消息处理模型；2.新建`ActivityThread`，并新建`ApplicationThread`这个Binder本地对象，并调用其`attach`方法；3.调用了`AsyncTask#init()`方法,强制其静态成员`sHandler`的构造

```java
public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        Process.setArgV0("<pre-initialized>");
        //Looper.prepare的同时，并记录sMainLooper，所以主线程上默认带Looper的
        Looper.prepareMainLooper();
        //
        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

主要来看`ActivityThrad#attach`方法，把`ApplicationThread`这个本地对象发送到`ActivityManagerService`，`ActivityManagerService`在调度`Activity`的时候，就可以使用到该`ApplicationThread`

```java
ActivityThread.java
//System：false
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;//false
    if (!system) {
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                ensureJitEnabled();
            }
        });
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());//ApplicationThreadNative的binder对象
        IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            // Ignore
        }
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        //...
    }

    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());

    ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
        @Override
        public void onConfigurationChanged(Configuration newConfig) {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources
                // immediately, because upon returning the view
                // hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                    // This actually changed the resources!  Tell
                    // everyone about it.
                    if (mPendingConfiguration == null || mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                        mPendingConfiguration = newConfig;

                        queueOrSendMessage(H.CONFIGURATION_CHANGED, newConfig);
                    }
                }
            }
        }
        @Override
        public void onLowMemory() {
        }
        @Override
        public void onTrimMemory(int level) {
        }
    });
}
```

## setp30

发送`ApplicationThread`这个Binder本地对象到`ActivityManagerService`(`ApplicationThread`是用来调度`Activity`的各种生命周期的，从Setp19处理`Activity`的暂停操作便可知道)

```java
Application

public void attachApplication(IApplicationThread app) throws RemoteException
  {
      Parcel data = Parcel.obtain();
      Parcel reply = Parcel.obtain();
      data.writeInterfaceToken(IActivityManager.descriptor);
      data.writeStrongBinder(app.asBinder());
      mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
      reply.readException();
      data.recycle();
      reply.recycle();
  }
```

## step31

`ActivityManagerService`处理`ATTACH_APPLICATION_TRANSACTION`事件

```java
ActivityManagerService.java

case ATTACH_APPLICATION_TRANSACTION: {
         data.enforceInterface(IActivityManager.descriptor);
         IApplicationThread app = ApplicationThreadNative.asInterface(data.readStrongBinder());
         if (app != null) {
             attachApplication(app);
         }
         reply.writeNoException();
         return true;
     }


@Override
  public final void attachApplication(IApplicationThread thread) {
      synchronized (this) {
          int callingPid = Binder.getCallingPid();
          final long origId = Binder.clearCallingIdentity();
          attachApplicationLocked(thread, callingPid);
          Binder.restoreCallingIdentity(origId);
      }
  }
```

对当前的`Processrecord`初始化，另外在Step28发送的进程启动超时消息也会在这一步移除，另外最重要的一步是`ApplicationThread#bindApplication`调用

```java
ActivityManagerService.java

private final boolean attachApplicationLocked(IApplicationThread thread,int pid)
    // Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);//现在不再为空了
        }
    } else {
        app = null;
    }

    if (app == null) {
    //...
    }
    // If this application record is still attached to a previous
    // process, clean it up now.
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }

    final String processName = app.processName;
    try {
        AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);//绑定死亡通知
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        startProcessLocked(app, "link fail", processName);
        return false;
    }
    app.makeActive(thread, mProcessStats);//主要是thread变量的赋值
    app.curAdj = app.setAdj = -100;
    app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
    app.forcingToForeground = null;
    app.foregroundServices = false;
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;

    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

    if (!normalMode) {
        Slog.i(TAG, "Launching preboot mode app: " + app);
    }
    //...
    try {
    //...
        String profileFile = app.instrumentationProfileFile;
        ParcelFileDescriptor profileFd = null;
        boolean profileAutoStop = false;
        if (mProfileApp != null && mProfileApp.equals(processName)) {
            mProfileProc = app;
            profileFile = mProfileFile;
            profileFd = mProfileFd;
            profileAutoStop = mAutoStopProfiler;
        }
        //...
        // If the app is being launched for restore or full backup, set it up specially
        //..

        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        app.compat = compatibilityInfoForPackageLocked(appInfo);
        if (profileFd != null) {
            profileFd = profileFd.dup();
        }
        thread.bindApplication(processName, appInfo, providers,
                app.instrumentationClass, profileFile, profileFd, profileAutoStop,
                app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                mCoreSettingsObserver.getCoreSettingsLocked());
        updateLruProcessLocked(app, false, false);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        // todo: Yikes!  What should we do?  For now we will try to
        // start another process, but that could easily get us in
        // an infinite loop of restarting processes...
        //..
    }

    // Remove this record from the list of starting applications.
    mPersistentStartingProcesses.remove(app);
    mProcessesOnHold.remove(app);

    boolean badApp = false;
    boolean didSomething = false;

    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app, mHeadless)) {
                didSomething = true;
            }
        } catch (Exception e) {
            badApp = true;
        }
    }

    // Find any services that should be running in this process...
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            badApp = true;
        }
    }

    // Check if a next-broadcast receiver is in this process...
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            // If the app died trying to launch the receiver we declare it 'bad'
            badApp = true;
        }
    }

    // Check whether the next backup agent is in this process...

    if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
        if (DEBUG_BACKUP) Slog.v(TAG, "New app is backup target, launching agent for " + app);
        ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
        try {
            thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                    compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                    mBackupTarget.backupMode);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    if (badApp) {
        // todo: Also need to kill application to deal with all
        // kinds of exceptions.
        handleAppDiedLocked(app, false, true);
        return false;
    }

    if (!didSomething) {
        updateOomAdjLocked();
    }

    return true;
}
```

## step32

BIND_APPLICATION

```java
ApplicationThread.java

public final void bindApplication(String processName,
        ApplicationInfo appInfo, List<ProviderInfo> providers,
        ComponentName instrumentationName, String profileFile,
        ParcelFileDescriptor profileFd, boolean autoStopProfiler,
        Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
        Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
        Bundle coreSettings) {

    if (services != null) {
        // Setup the service cache in the ServiceManager
        ServiceManager.initServiceCache(services);
    }

    setCoreSettings(coreSettings);

    AppBindData data = new AppBindData();
    data.processName = processName;
    data.appInfo = appInfo;
    data.providers = providers;
    data.instrumentationName = instrumentationName;
    data.instrumentationArgs = instrumentationArgs;
    data.instrumentationWatcher = instrumentationWatcher;
    data.instrumentationUiAutomationConnection = instrumentationUiConnection;
    data.debugMode = debugMode;
    data.enableOpenGlTrace = enableOpenGlTrace;
    data.restrictedBackupMode = isRestrictedBackupMode;
    data.persistent = persistent;
    data.config = config;
    data.compatInfo = compatInfo;
    data.initProfileFile = profileFile;
    data.initProfileFd = profileFd;
    data.initAutoStopProfiler = false;
    queueOrSendMessage(H.BIND_APPLICATION, data);
}
```

ActivityThrad的成员H处理

```java
ActivityThread.java

case BIND_APPLICATION:
    AppBindData data = (AppBindData)msg.obj;
    handleBindApplication(data);
    break;
```

各种程序配置的初始化（时区，地区，HTTP代理等信息），主要的是`Instrumentation`的初始化、Application的创建和`Instrumentation#callApplicationOnCreate`方法的调用（回调Application#onCreate）

```java
ActivityThread.java

private void handleBindApplication(AppBindData data) {
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);

    mProfiler = new Profiler();
    mProfiler.profileFile = data.initProfileFile;
    mProfiler.profileFd = data.initProfileFd;
    mProfiler.autoStopProfiler = data.initAutoStopProfiler;

    // send up app name; do this *before* waiting for debugger
    //...

    //可以看到，在2.3以前,AsyncTask使用的是多线程并行线程池，2.3以后则默认使用顺序单个执行的线程池
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }

    /*
     * Before spawning a new process, reset the time zone to be the system time zone.
     * This needs to be done because the system time zone could have changed after the
     * the spawning of this process. Without doing this this process would have the incorrect
     * system time zone.
     */
    TimeZone.setDefault(null);

    /*
     * Initialize the default locale in this process for the reasons we set the time zone.
     */
    Locale.setDefault(data.config.locale);

    /*
     * Update the system configuration since its preloaded and might not
     * reflect configuration changes. The configuration object passed
     * in AppBindData can be safely assumed to be up to date
     */
    mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
    mCurDefaultDisplayDpi = data.config.densityDpi;
    applyCompatConfiguration(mCurDefaultDisplayDpi);

    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

    /**
     * Switch this process to density compatibility mode if needed.
     */
    if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)== 0) {
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    updateDefaultDensity();

    final ContextImpl appContext = new ContextImpl();
    appContext.init(data.info, null, this);
    if (!Process.isIsolated()) {
        final File cacheDir = appContext.getCacheDir();

        if (cacheDir != null) {
            // Provide a usable directory for temporary files
            System.setProperty("java.io.tmpdir", cacheDir.getAbsolutePath());

            setupGraphicsSupport(data.info, cacheDir);
        } else {
            Log.e(TAG, "Unable to setupGraphicsSupport due to missing cache directory");
        }
    }
    /**
     * For system applications on userdebug/eng builds, log stack
     * traces of disk and network access to dropbox for analysis.
     */
     //.

    /**
     * For apps targetting SDK Honeycomb or later, we don't allow
     * network usage on the main event loop / UI thread.
     * Note to those grepping:  this is what ultimately throws
     * NetworkOnMainThreadException ...
     *这就是2.3以后不可以在主线程进行网络请请求的原因
     */
    if (data.appInfo.targetSdkVersion > 9) {
        StrictMode.enableDeathOnNetwork();
    }

    if (data.debugMode != IApplicationThread.DEBUG_OFF) {
        // XXX should have option to change the port.
        Debug.changeDebugPort(8100);
        //....
    }

    // Enable OpenGL tracing if required
    //..

    // Allow application-generated systrace messages if we're debuggable.
    boolean appTracingAllowed = (data.appInfo.flags&ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    Trace.setAppTracingAllowed(appTracingAllowed);

    /**
     * Initialize the default http proxy in this process for the reasons we set the time zone.
     */
     //...

  //Instrumentation的初始化,一个应用程序中只有一个Instrumentation对象，每个Activity内部都有一个该对象的引用,初始化完成之后，//帮助管理Activity生命周期的回调
    if (data.instrumentationName != null) {//true
        InstrumentationInfo ii = null;
        try {
            ii = appContext.getPackageManager().getInstrumentationInfo(data.instrumentationName, 0);
        } catch (PackageManager.NameNotFoundException e) {
        }
        if (ii == null) {
            throw new RuntimeException(
                "Unable to find instrumentation info for: "
                + data.instrumentationName);
        }

        mInstrumentationAppDir = ii.sourceDir;
        mInstrumentationAppLibraryDir = ii.nativeLibraryDir;
        mInstrumentationAppPackage = ii.packageName;
        mInstrumentedAppDir = data.info.getAppDir();
        mInstrumentedAppLibraryDir = data.info.getLibDir();

        ApplicationInfo instrApp = new ApplicationInfo();
        instrApp.packageName = ii.packageName;
        instrApp.sourceDir = ii.sourceDir;
        instrApp.publicSourceDir = ii.publicSourceDir;
        instrApp.dataDir = ii.dataDir;
        instrApp.nativeLibraryDir = ii.nativeLibraryDir;
        LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true);
        ContextImpl instrContext = new ContextImpl();
        instrContext.init(pi, null, this);

        try {
            java.lang.ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
          //...
        }

        mInstrumentation.init(this, instrContext, appContext,
               new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
               data.instrumentationUiAutomationConnection);
          //....

    } else {
        mInstrumentation = new Instrumentation();
    }

    if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
    }

    // Allow disk access during application and provider setup. This could
    // block processing ordered broadcasts, but later processing would
    // probably end up doing the same disk access.
    final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        //..

        // Do this after providers, since instrumentation tests generally start their
        // test thread at this point, and we don't want that racing.
        try {
            mInstrumentation.onCreate(data.instrumentationArgs);//这里是空实现
        }
        catch (Exception e) {
            throw new RuntimeException(
                "Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
        }

        try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```

## Step33

在`Step32`创建和初始化了`Instrumentation`，用来辅助管理`Activity`生命周期，和创建了应用的`Application`对象，并回调了其`onCreate`方法，会到`step31`,`bindApplication`之后，接着`mStackSupervisor.attachApplicationLocked`方法的执行，找到要启动的`AActivityRecord`信息

```java
boolean attachApplicationLocked(ProcessRecord app, boolean headless) throws Exception {
    boolean didSomething = false;
    final String processName = app.processName;
    for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
        final ActivityStack stack = mStacks.get(stackNdx);
        if (!isFrontStack(stack)) {
            continue;
        }
        ActivityRecord hr = stack.topRunningActivityLocked(null);
        if (hr != null) {
            if (hr.app == null && app.uid == hr.info.applicationInfo.uid&& processName.equals(hr.processName)) {
                try {
                    if (headless) {
                        Slog.e(TAG, "Starting activities not supported on headless device: "+ hr);
                    } else if (realStartActivityLocked(hr, app, true, true)) {//正常返回true
                        didSomething = true;
                    }
                } catch (Exception e) {
                    throw e;
                }
            }
        }
    }
    if (!didSomething) {
        ensureActivitiesVisibleLocked(null, 0);
    }
    return didSomething;
}
```

## Step34

保存正要启动的`ActivityRecord`记录的进程信息，重要的是`app.thread.scheduleLaunchActivity`的调用

```java
ActivityStackSupervisor.java

//andResume:true,checkConfig:true
final boolean realStartActivityLocked(ActivityRecord r,ProcessRecord app, boolean andResume, boolean checkConfig)
        throws RemoteException {
    r.startFreezingScreenLocked(app, 0);
    mWindowManager.setAppVisibility(r.appToken, true);

    // schedule launch ticks to collect information about slow apps.
    r.startLaunchTickingLocked();

    // Have the window manager re-evaluate the orientation of the screen based on the new activity order.  Note that
    // as a result of this, it can call back into the activity manager with a new orientation.  We don't care about //that, because the activity is not currently running so we are just restarting it anyway.
    if (checkConfig) {
        Configuration config = mWindowManager.updateOrientationFromAppTokens(
                mService.mConfiguration,
                r.mayFreezeScreenLocked(app) ? r.appToken : null);
        mService.updateConfigurationLocked(config, r, false, false);
    }

    r.app = app;//记录进程信息
    app.waitingToKill = null;
    r.launchCount++;
    r.lastLaunchTime = SystemClock.uptimeMillis();

    int idx = app.activities.indexOf(r);
    if (idx < 0) {
        app.activities.add(r);
    }
    mService.updateLruProcessLocked(app, true, true);

    final ActivityStack stack = r.task.stack;
    try {
        if (app.thread == null) {
            throw new RemoteException();
        }
        List<ResultInfo> results = null;
        List<Intent> newIntents = null;
        if (andResume) {//true
            results = r.results;
            newIntents = r.newIntents;
        }
        //...
        if (r.isHomeActivity() && r.isNotResolverActivity()) {
            // Home process is the root process of the task.
            mService.mHomeProcess = r.task.mActivities.get(0).app;
        }
        mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
        r.sleeping = false;
        r.forceNewConfig = false;
        mService.showAskCompatModeDialogLocked(r);
        r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
        String profileFile = null;
        ParcelFileDescriptor profileFd = null;
        boolean profileAutoStop = false;
        if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
            if (mService.mProfileProc == null || mService.mProfileProc == app) {
                mService.mProfileProc = app;
                profileFile = mService.mProfileFile;
                profileFd = mService.mProfileFd;
                profileAutoStop = mService.mAutoStopProfiler;
            }
        }
        app.hasShownUi = true;
        app.pendingUiClean = true;
        if (profileFd != null) {
            try {
                profileFd = profileFd.dup();
            } catch (IOException e) {
                if (profileFd != null) {
                    try {
                        profileFd.close();
                    } catch (IOException o) {
                    }
                    profileFd = null;
                }
            }
        }
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_TOP);
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,
                new Configuration(mService.mConfiguration), r.compat,
                app.repProcState, r.icicle, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profileFile, profileFd,
                profileAutoStop);

        if ((app.info.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {
            // This may be a heavy-weight process!  Note that the package
            // manager will ensure that only activity can run in the main
            // process of the .apk, which is the only thing that will be
            // considered heavy-weight.
            //..
        }

    } catch (RemoteException e) {
      //...
    }

    r.launchFailed = false;
    if (stack.updateLRUListLocked(r)) {
        Slog.w(TAG, "Activity " + r+ " being launched, but already in LRU list");
    }

    if (andResume) {
        // As part of the process of launching, ActivityThread also performs a resume.
        stack.minimalResumeActivityLocked(r);
    } else {
        // This activity is not starting in the resumed state... which
        // should look like we asked it to pause+stop (but remain visible),
        // and it has done so and reported back the current icicle and
        // other state.
        r.state = ActivityState.STOPPED;
        r.stopped = true;
    }

    // Launch the new version setup screen if needed.  We do this -after-
    // launching the initial activity (that is, home), so that it can have
    // a chance to initialize itself while in the background, making the
    // switch back to it faster and look better.
    if (isFrontStack(stack)) {
        mService.startSetupActivityLocked();
    }

    return true;
}
```

## Step35

```java
ApplicationThread.java
//notResumed:false
// we use token to identify this activity without having to send the
// activity itself back to the activity manager. (matters more with ipc)
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
        int procState, Bundle state, List<ResultInfo> pendingResults,
        List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
        String profileName, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    r.token = token;//ActivityRecord的标识
    r.ident = ident;
    r.intent = intent;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;

    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;

    r.startsNotResumed = notResumed;//false
    r.isForward = isForward;

    r.profileFile = profileName;
    r.profileFd = profileFd;
    r.autoStopProfiler = autoStopProfiler;

    updatePendingConfiguration(curConfig);

    queueOrSendMessage(H.LAUNCH_ACTIVITY, r);
}
```

## Step36

处理`LAUNCH_ACTIVITY`消息

```java
case LAUNCH_ACTIVITY: {
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                    r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                } break;
```

## Setp37

```java
ActivityThread.java

private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well we are back active so skip it.
    unscheduleGcIdler();

    if (r.profileFd != null) {
        mProfiler.setProfiler(r.profileFile, r.profileFd);
        mProfiler.startProfiling();
        mProfiler.autoStopProfiler = r.autoStopProfiler;
    }

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);

        if (!r.activity.mFinished && r.startsNotResumed) {
            // The activity manager actually wants this one to start out
            // paused, because it needs to be visible but isn't in the
            // foreground.  We accomplish this by going through the
            // normal startup (because activities expect to go through
            // onResume() the first time they run, before their window
            // is displayed), and then pausing it.  However, in this case
            // we do -not- need to do the full pause cycle (of freezing
            // and such) because the activity manager assumes it can just
            // retain the current state it has.
            try {
                r.activity.mCalled = false;
                mInstrumentation.callActivityOnPause(r.activity);
                // We need to keep around the original state, in case
                // we need to be created again.  But we only do this
                // for pre-Honeycomb apps, which always save their state
                // when pausing, so we can not have them save their state
                // when restarting from a paused state.  For HC and later,
                // we want to (and can) let the state be saved as the normal
                // part of stopping the activity.
                if (r.isPreHoneycomb()) {
                    r.state = oldState;
                }
                if (!r.activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPause()");
                }

            } catch (SuperNotCalledException e) {
                throw e;

            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                            "Unable to pause activity "
                            + r.intent.getComponent().toShortString()
                            + ": " + e.toString(), e);
                }
            }
            r.paused = true;
        }
    } else {
        // If there was an error, for any reason, tell the activity
        // manager to stop us.
        try {
            ActivityManagerNative.getDefault().finishActivity(r.token, Activity.RESULT_CANCELED, null);
        } catch (RemoteException ex) {
            // Ignore
        }
    }
}
```

## Step37

创建`Activity`，`Context`上下文的创建，这样`Activity`就能访问到特定的应用资源以及启动其他应用程序组件，`attach`初始化、`onCreate`、`onStart`还有`onRestoreInstanceState`回调

```java
ActivityThread.java

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,r.activityInfo.targetActivity);
    }

    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);//创建一个Activity对象
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component  + ": " + e.toString(), e);
        }
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);//前面已经创建过，所以这里不会再创建

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);

        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);

            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;//确保onCreate的执行
            mInstrumentation.callActivityOnCreate(activity, r.state);
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component
                + ": " + e.toString(), e);
        }
    }

    return activity;
}
```

`Activity`的`attach`初始化，包括`Window`的创建

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config) {
    attachBaseContext(context);

    mFragments.attachActivity(this, mContainer, null);

    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

`Activity`的`onCreate`回调

```java
Instrumentation.java

public void callActivityOnCreate(Activity activity, Bundle icicle) {
    if (mWaitingActivities != null) {
        synchronized (mSync) {
            final int N = mWaitingActivities.size();
            for (int i=0; i<N; i++) {
                final ActivityWaiter aw = mWaitingActivities.get(i);
                final Intent intent = aw.intent;
                if (intent.filterEquals(activity.getIntent())) {
                    aw.activity = activity;
                    mMessageQueue.addIdleHandler(new ActivityGoing(aw));
                }
            }
        }
    }

    activity.performCreate(icicle);

    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                am.match(activity, activity, activity.getIntent());
            }
        }
    }
}

Activity.java

final void performCreate(Bundle icicle) {
       onCreate(icicle);
       mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
               com.android.internal.R.styleable.Window_windowNoDisplay, false);
       mFragments.dispatchActivityCreated();
   }

protected void onCreate(Bundle savedInstanceState) {

   if (mLastNonConfigurationInstances != null) {
       mAllLoaderManagers = mLastNonConfigurationInstances.loaders;
   }
   if (mActivityInfo.parentActivityName != null) {
       if (mActionBar == null) {
           mEnableDefaultActionBarUp = true;
       } else {
           mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
       }
   }
   if (savedInstanceState != null) {
       Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
       mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
               ? mLastNonConfigurationInstances.fragments : null);
   }
   mFragments.dispatchCreate();
   getApplication().dispatchActivityCreated(this, savedInstanceState);
   mCalled = true;
}
```

`Activity`的`onStart`回调

```java
final void performStart() {
    mFragments.noteStateNotSaved();
    mCalled = false;
    mFragments.execPendingActions();
    mInstrumentation.callActivityOnStart(this);
    if (!mCalled) {
        throw new SuperNotCalledException(
            "Activity " + mComponent.toShortString() +
            " did not call through to super.onStart()");
    }
    mFragments.dispatchStart();
    if (mAllLoaderManagers != null) {
        final int N = mAllLoaderManagers.size();
        LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];
        for (int i=N-1; i>=0; i--) {
            loaders[i] = mAllLoaderManagers.valueAt(i);
        }
        for (int i=0; i<N; i++) {
            LoaderManagerImpl lm = loaders[i];
            lm.finishRetain();
            lm.doReportStart();
        }
    }
  }

    protected void onStart() {
        mCalled = true;
        if (!mLoadersStarted) {
            mLoadersStarted = true;
            if (mLoaderManager != null) {
                mLoaderManager.doStart();
            } else if (!mCheckedForLoaderManager) {
                mLoaderManager = getLoaderManager("(root)", mLoadersStarted, false);
            }
            mCheckedForLoaderManager = true;
        }

        getApplication().dispatchActivityStarted(this);
    }
```

`onRestoreInstanceState`方法回调

```java
public void callActivityOnRestoreInstanceState(Activity activity, Bundle savedInstanceState) {
        activity.performRestoreInstanceState(savedInstanceState);
    }

    final void performRestoreInstanceState(Bundle savedInstanceState) {
        onRestoreInstanceState(savedInstanceState);
        restoreManagedDialogs(savedInstanceState);
    }

    protected void onRestoreInstanceState(Bundle savedInstanceState) {
    if (mWindow != null) {
        Bundle windowState = savedInstanceState.getBundle(WINDOW_HIERARCHY_TAG);
        if (windowState != null) {
            mWindow.restoreHierarchyState(windowState);
        }
    }
}
```

## Step38

Step37的`performLaunchActivity`后还执行了`handleResumeActivity`，进行`onResume`回调

```java
//clearHide:false,reallyResume:true
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward,
        boolean reallyResume) {
    unscheduleGcIdler();

    ActivityClientRecord r = performResumeActivity(token, clearHide);

    if (r != null) {
        final Activity a = r.activity;

        if (localLOGV) Slog.v(
            TAG, "Resume " + r + " started activity: " +
            a.mStartedActivity + ", hideForNow: " + r.hideForNow
            + ", finished: " + a.mFinished);

        final int forwardBit = isForward ?
                WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        // If the window hasn't yet been added to the window manager,
        // and this guy didn't finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
            try {
                willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                        a.getActivityToken());
            } catch (RemoteException e) {
            }
        }
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(
                TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                        + r.activityInfo.name + " with newConfig " + r.newConfig);
                performConfigurationChanged(r.activity, r.newConfig);
                freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.newConfig));
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                    + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }
            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        if (!r.onlyLocalRequest) {
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            if (localLOGV) Slog.v(
                TAG, "Scheduling idle handler for " + r);
            Looper.myQueue().addIdleHandler(new Idler());
        }
        r.onlyLocalRequest = false;

        // Tell the activity manager we have resumed.
        if (reallyResume) {
            try {
                ActivityManagerNative.getDefault().activityResumed(token);
            } catch (RemoteException ex) {
            }
        }

    } else {
        // If an exception was thrown when trying to resume, then
        // just end this activity.
        try {
            ActivityManagerNative.getDefault()
                .finishActivity(token, Activity.RESULT_CANCELED, null);
        } catch (RemoteException ex) {
        }
    }
}
```

`onRestare`和`onResume`回调

```java
//clearHide:false
public final ActivityClientRecord performResumeActivity(IBinder token,boolean clearHide) {
    ActivityClientRecord r = mActivities.get(token);//不为空

    if (r != null && !r.activity.mFinished) {
        if (clearHide) {
          //...
        }
        try {
            r.activity.mFragments.noteStateNotSaved();
            if (r.pendingIntents != null) {
                deliverNewIntents(r, r.pendingIntents);
                r.pendingIntents = null;
            }
            if (r.pendingResults != null) {
                deliverResults(r, r.pendingResults);
                r.pendingResults = null;
            }
            r.activity.performResume();

            EventLog.writeEvent(LOG_ON_RESUME_CALLED,
                    UserHandle.myUserId(), r.activity.getComponentName().getClassName());

            r.paused = false;
            r.stopped = false;
            r.state = null;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                    "Unable to resume activity "
                    + r.intent.getComponent().toShortString()
                    + ": " + e.toString(), e);
            }
        }
    }
    return r;
}


final void performResume() {
    performRestart();

    mFragments.execPendingActions();

    mLastNonConfigurationInstances = null;

    mCalled = false;
    // mResumed is set by the instrumentation
    mInstrumentation.callActivityOnResume(this);
    if (!mCalled) {
        throw new SuperNotCalledException(
            "Activity " + mComponent.toShortString() +
            " did not call through to super.onResume()");
    }

    // Now really resume, and install the current status bar and menu.
    mCalled = false;

    mFragments.dispatchResume();
    mFragments.execPendingActions();

    onPostResume();
    if (!mCalled) {
        throw new SuperNotCalledException(
            "Activity " + mComponent.toShortString() +
            " did not call through to super.onPostResume()");
    }
}
```
