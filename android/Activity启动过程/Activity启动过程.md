# Activity的启动过程

## Step1

```java
Launcher3.java

boolean startActivitySafely(View v, Intent intent, Object tag) {
      boolean success = false;
      try {
          success = startActivity(v, intent, tag);
      }
      //.....
      return success;
  }
```

## Step2

```java
Launcher3.java

boolean startActivity(View v, Intent intent, Object tag) {
          //intent的FLAG_ACTIVITY_NEW_TASK置为1，因为Launcher应用
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&!intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
            if (useLaunchAnimation) {
                ActivityOptions opts = ActivityOptions.makeScaleUpAnimation(v, 0, 0,
                        v.getMeasuredWidth(), v.getMeasuredHeight());
                startActivity(intent, opts.toBundle());
            } else {
                startActivity(intent);
            }
            return true;
        }
        //...activity not found
        return false;
    }
```

## Step3

```java
Activity.java

public void startActivity(Intent intent, Bundle options) {
       if (options != null) {
           startActivityForResult(intent, -1, options);
       } else {
           // Note we want to go through this call for compatibility with applications that may have overridden the //method.
           startActivityForResult(intent, -1);
       }
   }
```

## Step4

`Instrumentation` 对象，用来辅助管理 `Activity` 生命周期，应用启动的时候新建，保存在 `Activity` 变量 `mInstrumentation`

```java
Activity.java
//requestCode=-1 ,option=null

public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);//mInstrumentation用来监控应用程序和系统间的交互
        if (ar != null) {
            mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(),  ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        final View decor = mWindow != null ? mWindow.peekDecorView() : null;
        if (decor != null) {
            decor.cancelPendingInputEvents();
        }
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        //...
    }
}
```

## Step5

使用 `ActivityManagerProxy` 的代理对象通知 `AMS` 去启动一个 `Activity`

```java
Instrumentation.java
//contextThread:mMainThread.getApplicationThread(),是一个binder本地对象
//token:Activity的成员变量mToken（IApplicationToken.Stub类型）
//requestCode=-1 ,option=null,

public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,
    Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    //...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token,target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```

## Step6

```java
ActivityManagerProxy.java
//caller:mMainThread.getApplicationThread()，是一个binder对象
//resultTo:Activity的成员变量mToken，IBinder对象，IApplicationToken，应用标识保存了其在AMS中的ActivityRecord
//requestCode=-1
//option=null
//profileFd=null
//profileFile=null
//startFlags=0
//resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）,

public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
           String resolvedType, IBinder resultTo, String resultWho, int requestCode,
           int startFlags, String profileFile,
           ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
       Parcel data = Parcel.obtain();
       Parcel reply = Parcel.obtain();
       data.writeInterfaceToken(IActivityManager.descriptor);
       data.writeStrongBinder(caller != null ? caller.asBinder() : null);
       data.writeString(callingPackage);
       intent.writeToParcel(data, 0);
       data.writeString(resolvedType);
       data.writeStrongBinder(resultTo);
       data.writeString(resultWho);
       data.writeInt(requestCode);
       data.writeInt(startFlags);
       data.writeString(profileFile);
       if (profileFd != null) {
           data.writeInt(1);
           profileFd.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
       } else {
           data.writeInt(0);
       }
       if (options != null) {
           data.writeInt(1);
           options.writeToParcel(data, 0);
       } else {
           data.writeInt(0);
       }
       mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
       reply.readException();
       int result = reply.readInt();
       reply.recycle();
       data.recycle();
       return result;
   }
```

## Step7

`Ams` 处理 `START_ACTIVITY_TRANSACTION` 请求

```java
ActivityManagerNative.java

case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();//IApplicationThread
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            String callingPackage = data.readString();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            String profileFile = data.readString();
            ParcelFileDescriptor profileFd = data.readInt() != 0
                    ? data.readFileDescriptor() : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags,
                    profileFile, profileFd, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
```

## step8

```java
//resultTo:Activity的成员变量mToken，是一个binder本地对象，对应了ActivityRecord
//caller:mMainThread.getApplicationThread()
//requestCode=-1 //option=null，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）

ActivityManagerService.java

@Override
 public final int startActivity(IApplicationThread caller, String callingPackage,
         Intent intent, String resolvedType, IBinder resultTo,
         String resultWho, int requestCode, int startFlags,
         String profileFile, ParcelFileDescriptor profileFd, Bundle options) {
     return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
             resultWho, requestCode,
             startFlags, profileFile, profileFd, options, UserHandle.getCallingUserId());
 }
```

## Step9

`ActivityManagerService` 调用 `ActivityStackSupervisor` 处理，`ActivityStackSupervisor` 用来辅助管理 `ActivityStack`，`ActivityStack` 里面有多个 `TaskRecord`，通常会因为 Activity 的 `taskAffinity` 值的创建，而对于 `SingleInstance` 则独享一个 `TaskRecord`，也就常说的 `Activity` 栈，`TaskRecord` 有多个 `ActivityRecord`，是 `Activity` 在内核空间的表示； `ActivityStackSupervisor`内部有两种 `ActivityStack`，`mHomeStack` 和 `mFocusedStack` , `mHomeStack` 保存launcher 类型的 `Activity`，`mFocusedStack` 保存当前接受输入事件和正启动的下一个 `Activity`，这些都是非 `Launcher` 类型的 `Activity`

```java
ActivityManagerService.java

//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）
//caller:mMainThread.getApplicationThread()

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags,
            String profileFile, ParcelFileDescriptor profileFd, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, true, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profileFile, profileFd,
                null, null, options, userId);
    }
```

## Step10

`AMS` 中对 `Activity` 的管理是通过任务栈的形式来管理的，也就是利用 `TaskRecord` 代表Task，系统中有很多Task，所以就出现了Task栈---- `ActivityStack`，按理说只要一个 `ActivityStack` 就OK了，但是Android4.0+有两个`ActivityStack`，并且还可以有多个。 `ActivityStackSupervisor`是一个管理`ActivityStack`类，里面的函数应该可以给出上面的答案。相关函数主要有`adjustStackFocus()`、`getFocusedStack()`、`setFocusedStack()`等。[ReadMore](http://blog.csdn.net/guoqifa29/article/details/39341931)

```java
ActivityStackSupervisor.java
//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）
//caller:mMainThread.getApplicationThread()
//outResult=null,config=null


final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, String profileFile,
            ParcelFileDescriptor profileFd, WaitResult outResult, Configuration config,
            Bundle options, int userId) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        //是否有Component
        boolean componentSpecified = intent.getComponent() != null;

        // Don't modify the client's object!
        intent = new Intent(intent);

        // Collect information about the target of the Intent.
        ActivityInfo aInfo = resolveActivity(intent, resolvedType, startFlags, profileFile, profileFd, userId);

        synchronized (mService) {
            int callingPid;
            if (callingUid >= 0) {
                callingPid = -1;
            } else if (caller == null) {
                callingPid = Binder.getCallingPid();
                callingUid = Binder.getCallingUid();
            } else {
                callingPid = callingUid = -1;
            }
            //初始化用户ID和进程ID
            final ActivityStack stack = getFocusedStack(); //获取当前取得用户焦点的栈，这里会是mHomeStack
                  //config为null，从launcher启动
            stack.mConfigWillChange = config != null&& mService.mConfiguration.diff(config) != 0;

            final long origId = Binder.clearCallingIdentity();

            if (aInfo != null &&(aInfo.applicationInfo.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {
                // This may be a heavy-weight process!  Check to see if we already
                // have another, different heavy-weight process running.
                //....不会进来这里
            }

            int res = startActivityLocked(caller, intent, resolvedType,
                    aInfo, resultTo, resultWho, requestCode, callingPid, callingUid,
                    callingPackage, startFlags, options, componentSpecified, null);

            if (stack.mConfigWillChange) {
                // If the caller also wants to switch to a new configuration,
                // do so now.  This allows a clean switch, as we are waiting
                // for the current activity to pause (so we will not destroy
                // it), and have not yet started the next activity.
                mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                        "updateConfiguration()");
                stack.mConfigWillChange = false;
                if (DEBUG_CONFIGURATION) Slog.v(TAG,
                        "Updating to new configuration after starting activity.");
                mService.updateConfigurationLocked(config, null, false, false);
            }

            Binder.restoreCallingIdentity(origId);

            if (outResult != null) {
              //...
            }

            return res;
        }
    }
```

**READMORE:ActivityInfo的解析**

## Step11

用户启动组件权限的检测，并会新建相应的 `ActivityRecord`,

```java
ActivityStackSupervisor.java
//requestCode=-1,option=null
//resultTo:Activity的成员变量mToken(IApplicationTokenStub)，通过它可以找到起在AMS对应的ActivityRecord
//profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）
//caller:mMainThread.getApplicationThread()，保存在ProcessRecord#thread
//outResult=null,config=null,ainfo:from resolveActivity,outActivity:null,

final int startActivityLocked(IApplicationThread caller,Intent intent, String resolvedType, ActivityInfo aInfo,
        IBinder resultTo,String resultWho, int requestCode,int callingPid, int callingUid, String callingPackage,
        int startFlags, Bundle options,boolean componentSpecified, ActivityRecord[] outActivity) {

        int err = ActivityManager.START_SUCCESS;
        ProcessRecord callerApp = null;
        if (caller != null) {
          //ActivityManagerService 内部保存着 Processrecord 的队列记录在成员变量 mLruProcesses 中
          //从成员 mLruProcesses 中获取与参数 IApplicationThread 对应的 Processrecord
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        if (err == ActivityManager.START_SUCCESS) {
            final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
          //ActivityStackSupervisor 有多个 ActivityStack,检测 IApplicationToken.Stub类 型参数是否在其中一个 ActivityStack 中
          //这里的 sourceRecord 是 mHomeStack
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG, "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
              //如果requestCode大于0，且不在在finishing，那么就记录这个`ActivityRecord`，并在之后找到其所在`ActivityStack`
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
        //Will send result to，上面可以知 resultRecord应该是空
        ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;//null

        int launchFlags = intent.getFlags();
        //FLAG_ACTIVITY_FORWARD_RESULT的作用:
        //O ----startActivityForResult()----> A ----FLAG_ACTIVITY_FORWARD_RESULT----> B
        //B的setResult会在O接到结果
        if ((launchFlags&Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0
                && sourceRecord != null) {
            // Transfer the result target from the source activity to the new
            // one being started, including any failures.
            //...
        }

        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            // We couldn't find a class that can handle the given Intent.That's the end of that!
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // We couldn't find the specific class specified in the Intent. Also the end of the line.
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }

        if (err != ActivityManager.START_SUCCESS) {
            //...
            return err;
        }

        final int startAnyPerm = mService.checkPermission(START_ANY_ACTIVITY, callingPid, callingUid);
        final int componentPerm = mService.checkComponentPermission(aInfo.permission, callingPid,
                callingUid, aInfo.applicationInfo.uid, aInfo.exported);
        if (startAnyPerm != PERMISSION_GRANTED && componentPerm != PERMISSION_GRANTED) {
            //...
        }

        boolean abort = !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);
        //mController：IActivityControllerl类型
        if (mService.mController != null) {
            try {
                // The Intent we give to the watcher has the extra data
                // stripped off, since it can contain private information.
                Intent watchIntent = intent.cloneFilter();
                abort |= !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController = null;
            }
        }

        if (abort) {
            if (resultRecord != null) {
                resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
            }
            // We pretend to the caller that it was really started, but
            // they will just get a cancel result.
            setDismissKeyguard(false);
            ActivityOptions.abort(options);
            return ActivityManager.START_SUCCESS;
        }
        //新建需要启动的ActivityRecord
        //mService:ActivityManagerService
        //callerApp:Laucher对应的Processrecord
        //ainfo:from resolveActivity
        //resultRecord=null
        //resultWho:Activity的成员变量mEmbeddedID（attach的时候初始化）
        //requestCode=-1
        //componentSpecified:否指定了component属性，
        //this:
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration,
                resultRecord, resultWho, requestCode, componentSpecified, this);
        if (outActivity != null) {
            outActivity[0] = r;
        }
        //获取到取得焦点的ActivityStack(这里会是mHomeStack)
        final ActivityStack stack = getFocusedStack();
        if (stack.mResumedActivity == null
                || stack.mResumedActivity.info.applicationInfo.uid != callingUid) {//false
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid, "Activity start")) {
                PendingActivityLaunch pal =
                        new PendingActivityLaunch(r, sourceRecord, startFlags, stack);
                mService.mPendingActivityLaunches.add(pal);
                setDismissKeyguard(false);
                ActivityOptions.abort(options);
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }

        if (mService.mDidAppSwitch) {
            // This is the second allowed switch since we stopped switches,
            // so now just generally allow switches.  Use case: user presses
            // home (switches disabled, switch to home, mDidAppSwitch now true);
            // user taps a home icon (coming from home so allowed, we hit here
            // and now allow anyone to switch again).
            mService.mAppSwitchesAllowedTime = 0;
        } else {
            mService.mDidAppSwitch = true;
        }
        //里面也会调用startActivityUncheckedLocked,和变量mPendingActivityLaunches相关，先忽略
        mService.doPendingActivityLaunchesLocked(false);

        err = startActivityUncheckedLocked(r, sourceRecord, startFlags, true, options);

        if (allPausedActivitiesComplete()) {
            // If someone asked to have the keyguard dismissed on the next
            // activity start, but we are not actually doing an activity
            // switch...  just dismiss the keyguard now, because we
            // probably want to see whatever is behind it.
            dismissKeyguard();
        }
        return err;
    }
```

`ActivityRecord`的构造，有三种类型，`HOME_ACTIVITY_TYPE`,`RECENTS_ACTIVITY_TYPE`和`APPLICATION_ACTIVITY_TYPE`，普通应用都是`APPLICATION_ACTIVITY_TYPE`类型的

```java
//新建需要启动的ActivityRecord
//_service:ActivityManagerService
//_caller:Laucher对应的Processrecord
//ainfo:from resolveActivity
//_resultTo=null
//_resultWho:launcher的成员变量mWho,
//_reqCode=-1
//_componentSpecified:true
//this:

ActivityRecord(ActivityManagerService _service, ProcessRecord _caller,
          int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
          ActivityInfo aInfo, Configuration _configuration,
          ActivityRecord _resultTo, String _resultWho, int _reqCode,
          boolean _componentSpecified, ActivityStackSupervisor supervisor) {
      service = _service; //ActivityManagerService
      appToken = new Token(this);
      info = aInfo;
      launchedFromUid = _launchedFromUid;
      launchedFromPackage = _launchedFromPackage;
      userId = UserHandle.getUserId(aInfo.applicationInfo.uid);
      intent = _intent;
      shortComponentName = _intent.getComponent().flattenToShortString();
      resolvedType = _resolvedType;
      componentSpecified = _componentSpecified;
      configuration = _configuration;
      resultTo = _resultTo;//null,requestCode>0的时候才有可能不为NULL
      resultWho = _resultWho;//Launcher的成员变量mEmbeddedID（attach的时候初始化）
      requestCode = _reqCode;//-1
      state = ActivityState.INITIALIZING;
      frontOfTask = false;
      launchFailed = false;
      stopped = false;
      delayedResume = false;
      finishing = false;
      configDestroy = false;
      keysPaused = false;
      inHistory = false;
      visible = true;
      waitingVisible = false;
      nowVisible = false;
      thumbnailNeeded = false;
      idle = false;
      hasBeenLaunched = false;
      mStackSupervisor = supervisor;

      // This starts out true, since the initial state of an activity
      // is that we have everything, and we shouldn't never consider it
      // lacking in state to be removed if it dies.
      haveState = true;

      if (aInfo != null) {//true
          if (aInfo.targetActivity == null
                  || aInfo.launchMode == ActivityInfo.LAUNCH_MULTIPLE
                  || aInfo.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP) {
              realActivity = _intent.getComponent();
          } else {
              realActivity = new ComponentName(aInfo.packageName,
                      aInfo.targetActivity);
          }
          taskAffinity = aInfo.taskAffinity;
          stateNotNeeded = (aInfo.flags&
                  ActivityInfo.FLAG_STATE_NOT_NEEDED) != 0;
          baseDir = aInfo.applicationInfo.sourceDir;
          resDir = aInfo.applicationInfo.publicSourceDir;
          dataDir = aInfo.applicationInfo.dataDir;
          nonLocalizedLabel = aInfo.nonLocalizedLabel;
          labelRes = aInfo.labelRes;
          //...
          icon = aInfo.getIconResource();
          logo = aInfo.getLogoResource();
          theme = aInfo.getThemeResource();
          //...
          if ((aInfo.flags&ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
              windowFlags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
          }
          if ((aInfo.flags&ActivityInfo.FLAG_MULTIPROCESS) != 0
                  && _caller != null
                  && (aInfo.applicationInfo.uid == Process.SYSTEM_UID
                          || aInfo.applicationInfo.uid == _caller.info.uid)) {
              processName = _caller.processName;
          } else {
              processName = aInfo.processName;
          }

          if (intent != null && (aInfo.flags & ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS) != 0) {
              intent.addFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
          }

          packageName = aInfo.applicationInfo.packageName;
          launchMode = aInfo.launchMode;
          //....

          if ((!_componentSpecified || _launchedFromUid == Process.myUid() || _launchedFromUid == 0) &&
                  Intent.ACTION_MAIN.equals(_intent.getAction()) &&
                  _intent.hasCategory(Intent.CATEGORY_HOME) &&
                  _intent.getCategories().size() == 1 &&
                  _intent.getData() == null &&
                  _intent.getType() == null &&
                  (intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
                  isNotResolverActivity()) {
              // This sure looks like a home activity!
              mActivityType = HOME_ACTIVITY_TYPE;
          } else if (realActivity.getClassName().contains(RECENTS_PACKAGE_NAME)) {
              mActivityType = RECENTS_ACTIVITY_TYPE;
          } else {
              mActivityType = APPLICATION_ACTIVITY_TYPE;  //普通的Activity
          }
          immersive = (aInfo.flags & ActivityInfo.FLAG_IMMERSIVE) != 0;
      } else {
        //...
      }
  }
```

**ReadMore：FLAG_ACTIVITY_FORWARD_RESULT的处理**

**question2:mPendingActivityLaunches的作用？**

## Step12

新建或重用`ActivityStack`,`TaskRecord`，**各种launcerMode和IntentFlag的处理**，最后调用目标`ActivityStack`的`startActivityLocked`方法 下面是三种情况下，其启动的Intent.FLAG会带上`Intent.FLAG_ACTIVITY_NEW_TASK`，代表可能需要在新的任务栈启动目标`Activity`：

- （1）`sourceRecord=null`说明要启动的`Activity`并不是由一个`Activity`的`Context`启动的，这时候我们总是启动一个新的`TASK`
- （2）`Launcher`是`SingleInstance`模式（或者说启动新的`Activity`的源`Activity`的`launchMode`为`SingleInstance`）,因为`SingleInstance`模式的`Activity`不希望和你分享同一个`TaskRecord`
- （3）需要启动的`Activity`的`launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE|| r.launchMode ==ActivityInfo.LAUNCH_SINGLE_TASK`，因为表明了自己希望在新的`TaskRecord`中启动
- （4）`sourceRecord`正在`finish`

  最后可以发现:**`SingleTask`模式，但是它并不像官方文档描述的一样：`The system creates a new task and instantiates the activity at the root of the new task`，而是在跟它有相同`taskAffinity`的任务中启动，并且位于这个任务的堆栈顶端，所以如果指定特定的`taskAffinity`，就可以在新的`TaskRecord`启动**

```java
ActivityStackSupervisor.java

//r：Step11创建，sourceRecord：Launcher对应的`ActivityRecord`,startFlags=0，doResume=true，option=null
final int startActivityUncheckedLocked(ActivityRecord r,ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options){
       final Intent intent = r.intent;//Step1开始就传来的
       final int callingUid = r.launchedFromUid;
       //从launcer启动，只有Intent.FLAG_ACTIVITY_NEW_TASK被置为1
       int launchFlags = intent.getFlags();

       // We'll invoke onUserLeaving before onPause only if the launching activity did not explicitly state that this is an automated launch.
      //是否向源Activity组件发送一个用户离开事件的通知
      //onUserLeaveHint()作为activity周期的一部分，它在activity因为用户要跳转到别的activity而要退到background时使用。
      //比如,在用户按下Home键（用户的choice），它将被调用。比如有电话进来（不属于用户的choice），它就不会被调用。
      //那么系统如何区分让当前activity退到background时使用是用户的choice？
      //它是根据促使当前activity退到background的那个新启动的Activity的Intent里是否有FLAG_ACTIVITY_NO_USER_ACTION来确定的。
       mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;//true

       // If the caller has asked not to resume at this point, we make note
       // of this in the record so that we can skip it when trying to find
       // the top running activity.
       //如果laucher 没有要求在此时立即启动该Activity，那么 r.delayedResume = //true（表示可以延迟resume），当查找最上层正在运行的Activity的时候就可以跳过它了。
       if (!doResume) {
           r.delayedResume = true;
       }

       ActivityRecord notTop = (launchFlags&Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP) != 0 ? r : null;//null

       // If the onlyIfNeeded flag is set, then we can do this if the activity
       // being launched is the same as the one making the call...  or, as
       // a special case, if we do not know the caller then we count the
       // current top activity as the caller.
      //START_FLAG_ONLY_IF_NEEDED:do special start mode where a new activity is launched only if it is needed.
       if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
           ActivityRecord checkedCaller = sourceRecord;
           if (checkedCaller == null) {
               checkedCaller = getFocusedStack().topRunningNonDelayedActivityLocked(notTop);
           }
           if (!checkedCaller.realActivity.equals(r.realActivity)) {
               // Caller is not the same as launcher, so always needed.
               startFlags &= ~ActivityManager.START_FLAG_ONLY_IF_NEEDED;
           }
       }

       if (sourceRecord == null) {
            //这种情况下是否和启动来自Service和BroadcastReceiver的context一致？
           // This activity is not being started from another...  in this case we -always- start a new task.
           if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
               launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
           }
       } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
           launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
       } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
           // The activity being started is a single instance...  it always
           // gets launched into its own task.
           launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
       }
       //sourceStack表示sourceRecord所在的TASK所在的Stack
       final ActivityStack sourceStack;
       if (sourceRecord != null) {
           if (sourceRecord.finishing) {
               // If the source is finishing, we can't further count it as our source.  This
               // is because the task it is associated with may now be empty and on its way out,
               // so we don't want to blindly throw it in to that task.  Instead we will take
               // the NEW_TASK flow and try to find a task for it.
               if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                   launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
               }
               sourceRecord = null;
               sourceStack = null;
           } else {
               sourceStack = sourceRecord.task.stack;
           }
       } else {
           sourceStack = null;
       }
      //resultTo为空，request code为-1
      //如果新的Activity要新创建Task而启动，而且原先的Activity需要返回结果，这种情况下，调用函数r.resultTo.task.stack.sendActivityResultLocke()方法将 //Activity.RESULT_CANCELED结果返回给launcher,并将resultTo置为null，resultTo的使命结束了。
       if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
           // For whatever reason this activity is being launched into a new
           // task...  yet the caller has requested a result back.  Well, that
           // is pretty messed up, so instead immediately send back a cancel
           // and let the new task continue launched as normal without a
           // dependency on its originator.
           r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho, r.requestCode,Activity.RESULT_CANCELED, null);
           r.resultTo = null;
       }
       //这个变量也将决定是否要将在新的任务中启动。默认不增加到原有的任务中启动，即要在新的任务中启动
       boolean addingToTask = false;
       boolean movedHome = false;
       TaskRecord reuseTask = null;
       ActivityStack targetStack;
       //FLAG_ACTIVITY_NEW_TASK标志位为1，FLAG_ACTIVITY_MULTIPLE_TASK为默认模式
       //FLAG_ACTIVITY_MULTIPLE_TASK：Do not use this flag unless you are implementing your own top-level application //launcher.
       if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
           // If bring to front is requested, and no result is requested, and we can find a task that was started with this same
           // component, then instead of launching bring that one to the front.
           if (r.resultTo == null) {//true
               // See if there is a task to bring to the front.  If this is a SINGLE_INSTANCE activity, there can be one and only one
               // instance of it in the history, and it is always in its own unique task, so we do a special search.
               //区别：findActivityLocked的查找是从所有Task中的ActivityRecord的比较查找，findTaskLocked只从每个Task的栈顶的ActivityRecord
               //来进行比较查找
               ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                       ? findTaskLocked(r)
                       : findActivityLocked(intent, r.info);
               //intentActivity 所在的TASK就是要启动的Activity所在的Task，要启动的Activity所在的Stack
               //targetStack就是intentActivity所在的Stack。
               //这里为NULL,先忽略(这里有关于ActivityInfo.LAUNCH_SINGLE_TASK是否在新栈启动的逻辑).
               if (intentActivity != null) {
                //....launchMode和IntentFLag的处理，见“LaunchMode的处理”
                }
           }
       }
      //上段代码的逻辑是看一下，当前在堆栈是否有要启动的Activity的ActivityRecord，如果有那么对于不同的launchMode就会要进行不同的响应
       if (r.packageName != null) {
           // If the activity being launched is the same as the one currently
           // at the top, then we need to check if it should only be launched
           // once.
           //这段代码的逻辑是看一下，当前在堆栈顶端的Activity是否就是即将要启动的Activity，有些情况下，LAUNCH_SINGLE_TOP||LAUNCH_SINGLE_TASK||FLAG_ACTIVITY_SINGLE_TOP，如果即将要启动的Activity就在堆栈的顶端，那么，就不会重新启动这个Activity的别一个实例了
           ActivityStack topStack = getFocusedStack();//mHomeStack
           //启动的用户不一样，notTop为null，一般为应用内启动
           ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);
           if (top != null && r.resultTo == null) {
               if (top.realActivity.equals(r.realActivity) && top.userId == r.userId) { //应用内启动，且栈顶应用即是要启动的Activity
                  //....见“LaunchMode的处理”,Activity#onNewIntent回调，并返回
                }
           }
       } else {
           if (r.resultTo != null) {
               r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho, r.requestCode, Activity.RESULT_CANCELED, null);
           }
           ActivityOptions.abort(options);
           //...
           return ActivityManager.START_CLASS_NOT_FOUND;
       }

       boolean newTask = false;
       boolean keepCurTransition = false;

       // Should this be considered a new task?
       //addingToTask为false，因为前面的intentActivity为null
       if (r.resultTo == null && !addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
          //这种情况下，newTask为true，不进入该判断，newTask为false，代表不需要创建新的TaskRecord
           targetStack = adjustStackFocus(r);//见AMS中的两种ActivityStack，新建ActivityStack，得到mFocusStack
           moveHomeStack(targetStack.isHomeStack());//状态修改为STACK_STATE_HOME_TO_BACK
           if (reuseTask == null) { //同addingToTask，reuseTask为null
              //创建新的TaskRecord，如果已经记录在其他TASKRECORD，移除，记录新的TaskRecord，并把TaskRecord插入到mTaskHistory的顶部，但TaskRecord并未记录该ActivityRecord
               r.setTask(targetStack.createTaskRecord(getNextTaskId(), r.info, intent, true),null, true);
           } else {
               r.setTask(reuseTask, reuseTask, true);
           }
           newTask = true;
           if (!movedHome) {//true
               if ((launchFlags &
                       (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_TASK_ON_HOME))
                       == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_TASK_ON_HOME)) {
                   // Caller wants to appear on home activity, so before starting their own activity we will bring home to the front.
                   r.task.mOnTopOfHome = true;
               }
           }
       } else if (sourceRecord != null) { //应用内启动，待启动Activity和发起启动的Activity同一个栈
            TaskRecord sourceTask = sourceRecord.task;
            targetStack = sourceTask.stack;
            moveHomeStack(targetStack.isHomeStack());
            if (!addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
                // In this case, we are adding the activity to an existing
                // task, but the caller has asked to clear that task if the
                // activity is already running.
                ActivityRecord top = sourceTask.performClearTaskLocked(r, launchFlags);
                keepCurTransition = true;
                if (top != null) {
                  top.deliverNewIntentLocked(callingUid, r.intent);
                  // For paranoia, make sure we have correctly
                  // resumed the top activity.
                  targetStack.mLastPausedActivity = null;
                  if (doResume) {
                    targetStack.resumeTopActivityLocked(null);
                  }
                  ActivityOptions.abort(options);
                  //...
                  return ActivityManager.START_DELIVERED_TO_TOP;
                }
            } else if (!addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                // In this case, we are launching an activity in our own task
                // that may already be running somewhere in the history, and
                // we want to shuffle it to the front of the stack if so.
                final ActivityRecord top = sourceTask.findActivityInHistoryLocked(r);
                if (top != null) {
                    final TaskRecord task = top.task;
                    task.moveActivityToFrontLocked(top);
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, task);
                    top.updateOptionsLocked(options);
                    top.deliverNewIntentLocked(callingUid, r.intent);
                    targetStack.mLastPausedActivity = null;
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null);
                    }
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }
            // An existing activity is starting this new activity, so we want
            // to keep the new one in the same task as the one that is starting
            // it.
            r.setTask(sourceTask, sourceRecord.thumbHolder, false);

        } else {
           // This not being started from an existing activity, and not part
           // of a new task...  just put it in the top task, though these days
           // this case should never happen.
          //...
       }

       mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,intent, r.getUriPermissionsLocked());
       //...
       targetStack.mLastPausedActivity = null;//mFocusedStack
       //newTask:true,doResume=true,keepCurTransition=false,option=null
       targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
       mService.setFocusedActivityLocked(r);//记录mFocusedActivity为r，mFocusedStack
       return ActivityManager.START_SUCCESS;
   }
```

创建新的`TaskRecord`对象并插入到`mTaskHistory`栈顶

```java
ActivityStack.java
//toTop=true;
TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent, boolean toTop) {
      TaskRecord task = new TaskRecord(taskId, info, intent);
      addTask(task, toTop);
      return task;
  }

  void addTask(final TaskRecord task, final boolean toTop) {
       task.stack = this;
       if (toTop) {
           insertTaskAtTop(task);
       } else {
           mTaskHistory.add(0, task);
       }
   }
   //使用TaskRecord置顶
   private void insertTaskAtTop(TaskRecord task) {
        // If this is being moved to the top by another activity or being launched from the home activity, set mOnTopOfHome accordingly.
        ActivityStack lastStack = mStackSupervisor.getLastStack();//mHomeStack
        final boolean fromHome = lastStack == null ? true : lastStack.isHomeStack();//true
        if (!isHomeStack() && (fromHome || topTask() != task)) {//true,topTask==null,因为当前并没有ActivityRecord记录在`TaskRecord`
            task.mOnTopOfHome = fromHome;//true，如果标记为fromHome，当前Task退出之后就显示的是Home（Launch the home activity when leaving this task.）
        }
        mTaskHistory.remove(task);
        // Now put task at top.
        int stackNdx = mTaskHistory.size();
        if (task.userId != mCurrentUser) {
            // Put non-current user tasks below current user tasks.
            while (--stackNdx >= 0) {
                if (mTaskHistory.get(stackNdx).userId != mCurrentUser) {
                    break;
                }
            }
            ++stackNdx;
        }
        mTaskHistory.add(stackNdx, task);
    }
```

## Step13

前一步骤已经创建好了`ActivityRecord`实例、并绑定到与其对应的`TaskRecord`对象（但是`TaskRecord`并未将其记录在没Activity`mActivities`）和初始化好了对应的`ActivityStack`对象`mForceStack`,且该`TaskRecord`已经保存在了`ActivityStack`的`mTaskHistory`栈顶，调用目标`ActivityStack`的`startActivityLocked`方法把`ActivityRecord`记录在`TaskRecord`，之后告诉`ActivityStackSupervisor`可以继续启动一个新的应用程序

```java
ActivityStack.java

//r:Step11创建的，newTask:true，doResume=true，keepCurTransition=false，option=null
final void startActivityLocked(ActivityRecord r, boolean newTask,boolean doResume, boolean keepCurTransition, Bundle options) {
        TaskRecord rTask = r.task;//Step12赋值
        final int taskId = rTask.taskId;
        if (taskForIdLocked(taskId) == null || newTask) {//true，TaskRecord对象尚未保存到ActivityStack
            // Last activity in task had been removed or ActivityManagerService is reusing task.
            // Insert or replace.
            // Might not even be in.
            //Step12在创建的时候`TaskRecord`会调用，这里主要针对使用已存在的`TaskRecord`
            //会插入到mFocusedStack的mTaskHistory顶部，AxxSxxSxx当前状态为STACK_STATE_HOME_IN_BACK,(see Step12)
            insertTaskAtTop(rTask);
            mWindowManager.moveTaskToTop(taskId);
        }
        TaskRecord task = null;
        if (!newTask) {//如果是应用内启动，且在相同栈，进入这里的判断
          // If starting in an existing task, find where that is...
          boolean startIt = true;
          for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
              task = mTaskHistory.get(taskNdx);
              if (task == r.task) {
                //如果这个原有的Task当前对用户不可见时，这时候就不需要继续执行下去了，因为即使把这个Activity启动起来，用户也看不到，还不如先把它保存起来，等到下次这个Task对用户可见的时候，再启动不迟
                  if (!startIt) {
                      task.addActivityToTop(r);
                      r.putInHistory();
                      //....
                      ActivityOptions.abort(options);
                      return;
                  }
                  break;
              } else if (task.numFullscreen > 0) {
                  startIt = false;
              }
          }
        }
        // Place a new activity at top of stack, so it is next to interact with the user.
        // If we are not placing the new activity front most, we do not want
        // to deliver the onUserLeaving callback to the actual frontmost activity
        if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {//false,task=null
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG,"startActivity() behind front, mUserLeaving=false");
        }

        task = r.task;

        task.addActivityToTop(r);//添加到`TaskRecord`中`mActivities`的顶部

        r.putInHistory();//inHistory标识为true
        r.frontOfTask = newTask;//true
        if (!isHomeStack() || numActivities() > 0) {//true
            // We want to show the starting preview window if we are switching to a new task, or the next activity's process is
            // not currently running.
            boolean showStartingIcon = newTask;//true
            //...
            if (newTask) {//true
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            }
            if (SHOW_APP_STARTING_PREVIEW && doShow) {//true
                // Figure out if we are transitioning from another activity that is
                // "has the same starting icon" as the next one.  This allows the
                // window manager to keep the previous window it had previously
                // created, if it still had one.
                //....
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            //...
        }
        //...

        if (doResume) {//true
            mStackSupervisor.resumeTopActivitiesLocked();
        }
    }
```

## Step14

找到当前取得焦点的`ActivityStack`,启动目标`ActivityRecord`

```java
ActivityStackSupervisor.java

boolean resumeTopActivitiesLocked() {

       return resumeTopActivitiesLocked(null, null, null);
   }

boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,Bundle targetOptions) {
     if (targetStack == null) {
         targetStack = getFocusedStack();//mForceStack,当前状态STACK_STATE_HOME_IN_BACK（Step12）
     }
     boolean result = false;
     for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
         final ActivityStack stack = mStacks.get(stackNdx);
         if (isFrontStack(stack)) {
             if (stack == targetStack) {
                 result = stack.resumeTopActivityLocked(target,targetOptions);
                 //另外一种情况效果都一样，因为target和targetOptions都为NULL
             } else {
                 stack.resumeTopActivityLocked(null);
             }
         }
     }
     return result;
 }
```

## Step15

该函数是将处于栈顶的`Activity`组件激活，这个`Activity`正好就是即将要启动的`Activity`组件，但在启动之前需要先暂停其他栈中的`mResumedActivity`,接着就是当前栈的`mResumedActivity`,由于从`Launcher`启动，那么就需要先暂停`mHomeStack`中的`Launcher`组件

```java
ActivityStack.java

//prev=null,options=null
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
        //false，，next.state为ActivityState.INITIALIZING
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&mStackSupervisor.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            //...
            return false;
        }

        final TaskRecord nextTask = next.task;
        final TaskRecord prevTask = prev != null ? prev.task : null;//null
        if (prevTask != null && prevTask.mOnTopOfHome && prev.finishing && prev.frontOfTask) {//false
          //...
        }

        // If we are sleeping, and there is no resumed activity, and the top activity is paused, well that is the state we want.
        //如果要启动的Activity是上次终止的Activity，且系统正要进入关机或者睡眠，那么直接返回，因为没有意义
        if (mService.isSleepingOrShuttingDown()&& mLastPausedActivity == next&& mStackSupervisor.allPausedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Going to sleep and all paused");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        // Make sure that the user who owns this activity is started.  If not,
        // we will just leave it as is because someone should be bringing
        // another user's activities to the top of the stack.
        if (mService.mStartedUsers.get(next.userId) == null) {
            Slog.w(TAG, "Skipping resume of top activity " + next+ ": user " + next.userId + " is stopped");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        // The activity may be waiting for stop, but that is no longer appropriate for it.
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mWaitingVisibleActivities.remove(next);

        next.updateOptionsLocked(options);//null

        // If we are currently pausing an activity, then don't do anything until that is done.
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG,"resumeTopActivityLocked: Skip resume: some activity pausing.");
            return false;
        }

        // Okay we are now going to start a switch, to 'next'.  We may first
        // have to pause the current activity, but this is an important point
        // where we have decided to go to 'next' so keep track of that.
        // XXX "App Redirected" dialog is getting too many false positives
        // at this point, so turn off for now.
        if (false) {
        //...
        }

        // We need to start pausing the current activity so the top one can be resumed...
        //分别暂停所有`mStacks`中不是前台`Stack`中的所有`mResumedActivity`，也就是`mHomeStack`中的Launcher
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving);//返回true
        //上一步是暂停其他栈的mResumedActivity，这里则是暂停当前栈的mResumedActivity，当然一般只会走其中一步
        //至于下一步的场景是这样的，当前获得焦点mFocusStack，有两个TaskRecord，TaskRecord1显示在前台，显示了ActivityA，那么TaskRecord0则显示在后台
        //现在A要打开TaskRecord0中的B，而B已然存在，而且是SingleTask的，就需要把TaskRecord0置为前台，而现在的mHomeStack中的mResumedActivity就是null
        //而当前ActivityStack的mResumedActivity就是A，所以要暂停A
        if (mResumedActivity != null) {//mResumedActivity为null，当前Stack为mFocusStack
            pausing = true;
            startPausingLocked(userLeaving, false);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
        }
        //如果有需要pause的Activity，则需要先暂停再显示当前需要显示的
        if (pausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG,
                    "resumeTopActivityLocked: Skip resume: need to start pausing");
            // At this point we want to put the upcoming activity's process
            // at the top of the LRU list, since we know we will be needing it
            // very soon and it would be a waste to let it get killed if it
            // happens to be sitting towards the end.
            if (next.app != null && next.app.thread != null) {
                // No reason to do full oom adj update here; we'll let that
                // happen whenever it needs to later.
                mService.updateLruProcessLocked(next.app, false, true);
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        //....
    }
```

## Reference

- [ActivityStackSupervisor.StartActivityUncheckedLocked()函数分析](http://www.aiuxian.com/article/p-1880233.html)
- [Activity管理机制](http://blog.csdn.net/guoqifa29/article/details/39341931)
- [ActivityStackSupervisor分析](http://blog.csdn.net/guoqifa29/article/details/40015127)
- [Activity生命周期的回调，你应该知道得更多](http://blog.csdn.net/yalinfendou/article/details/46909173)
