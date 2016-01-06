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
          //intent的FLAG_ACTIVITY_NEW_TASK置为1，以为在新的TASK启动Activity
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
`Instrumentation`对象，用来辅助管理`Activity`生命周期，应用启动的时候新建，保存在`Activity`变量`mInstrumentation`
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
使用`ActivityManagerProxy`的代理对象通知`AMS`去启动一个`Activity`
```java
Instrumentation.java
//contextThread:mMainThread.getApplicationThread(),是一个binder对象
//token:Activity的成员变量mToken（IApplicationToken.Stub类型）
//requestCode=-1 ,option=null,

public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Fragment target,
    Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    //...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mWho : null,
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
//caller:mMainThread.getApplicationThread()，是一个binder本地对象
//resultTo:Activity的成员变量mToken，IApplicationToken.Stub，应用标识保存了其在AMS中的ActivityRecord
//requestCode=-1 ,option=null,，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,

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
`Ams`处理`START_ACTIVITY_TRANSACTION`请求
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
//resultTo:Activity的成员变量mToken，是一个binder本地对象
//caller:mMainThread.getApplicationThread()
//requestCode=-1 //option=null，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho

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
__question1:userid是如何确认的?__

##  Step9
`ActivityManagerService`调用`ActivityStackSupervisor`处理，ActivityStackSupervisor用来辅助管理ActivityStack，ActivityStack里面有多个TaskRecord，即通常说的Activity栈，TaskRecord有多个ActivityRecord，是Activity在内核空间的表示；
ActivityStackSupervisor内部有两种ActivityStack，mHomeStack和mFocusedStack,mHomeStack保存launcher app的Activity，mFocusedStack保存当前接受输入事件和正启动的下一个Activity
```java
ActivityManagerService.java

//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,caller:mMainThread.getApplicationThread()

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

`AMS`中对`Activity`的管理是通过任务栈的形式来管理的，也就是利用`TaskRecord`代表Task，系统中有很多Task，所以就出现了Task栈——`ActivityStack`，按理说只要一个`ActivityStack`就OK了，但是Android4.0+有两个`ActivityStack`，并且还可以有多个。
`ActivityStackSupervisor`是一个管理`ActivityStack`类，里面的函数应该可以给出上面的答案。相关函数主要有`adjustStackFocus()`、`getFocusedStack()`、`setFocusedStack()`等。[ReadMore](http://blog.csdn.net/guoqifa29/article/details/39341931)


```java
ActivityStackSupervisor.java
//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,caller:mMainThread.getApplicationThread()
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
        ActivityInfo aInfo = resolveActivity(intent, resolvedType, startFlags,
                profileFile, profileFd, userId);

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
            final ActivityStack stack = getFocusedStack();//这里会是mHomeStack
			      //config为null，从launcher启动
            stack.mConfigWillChange = config != null&& mService.mConfiguration.diff(config) != 0;

            final long origId = Binder.clearCallingIdentity();

            if (aInfo != null &&(aInfo.applicationInfo.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {
                // This may be a heavy-weight process!  Check to see if we already
                // have another, different heavy-weight process running.
                //....
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
                outResult.result = res;
                if (res == ActivityManager.START_SUCCESS) {
                    mWaitingActivityLaunched.add(outResult);
                    do {
                        try {
                            mService.wait();
                        } catch (InterruptedException e) {
                        }
                    } while (!outResult.timeout && outResult.who == null);
                } else if (res == ActivityManager.START_TASK_TO_FRONT) {
                    ActivityRecord r = stack.topRunningActivityLocked(null);
                    if (r.nowVisible) {
                        outResult.timeout = false;
                        outResult.who = new ComponentName(r.info.packageName, r.info.name);
                        outResult.totalTime = 0;
                        outResult.thisTime = 0;
                    } else {
                        outResult.thisTime = SystemClock.uptimeMillis();
                        mWaitingActivityVisible.add(outResult);
                        do {
                            try {
                                mService.wait();
                            } catch (InterruptedException e) {
                            }
                        } while (!outResult.timeout && outResult.who == null);
                    }
                }
            }

            return res;
        }
    }

```
__READMORE:ActivityInfo的解析__

## Step11
用户启动组件权限的检测，并会新建相应的`ActivityRecord`,
```java
ActivityStackSupervisor.java
//requestCode=-1,option=null
//resultTo:Activity的成员变量mToken(IApplicationTokenStub)，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho
//caller:mMainThread.getApplicationThread()，保存在ProcessRecord#thread
//outResult=null,config=null,ainfo:from resolveActivity,outActivity:null,

final int startActivityLocked(IApplicationThread caller,Intent intent, String resolvedType, ActivityInfo aInfo,
        IBinder resultTo,String resultWho, int requestCode,int callingPid, int callingUid, String callingPackage,
        int startFlags, Bundle options,boolean componentSpecified, ActivityRecord[] outActivity) {

        int err = ActivityManager.START_SUCCESS;
        ProcessRecord callerApp = null;
        if (caller != null) {
          //ActivityManagerService内部保存着Processrecord的队列记录在成员变量mLruProcesses中
          //从成员mLruProcesses中获取与参数IApplicationThread对应的Processrecord
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
          //ActivityStackSupervisor有多个ActivityStack,检测IApplicationToken.Stub类型参数是否在其中一个ActivityStack中
          //这里的sourceRecord是mHomeStack
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG, "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
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
        //resultWho:launcher的成员变量mWho,
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
`ActivityRecord`的构造

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
      resultTo = _resultTo;//null
      resultWho = _resultWho;//launcher的成员变量mWho
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
          if (nonLocalizedLabel == null && labelRes == 0) {
              ApplicationInfo app = aInfo.applicationInfo;
              nonLocalizedLabel = app.nonLocalizedLabel;
              labelRes = app.labelRes;
          }
          icon = aInfo.getIconResource();
          logo = aInfo.getLogoResource();
          theme = aInfo.getThemeResource();
          realTheme = theme;
          if (realTheme == 0) {
              realTheme = aInfo.applicationInfo.targetSdkVersion
                      < Build.VERSION_CODES.HONEYCOMB
                      ? android.R.style.Theme
                      : android.R.style.Theme_Holo;
          }
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

          AttributeCache.Entry ent = AttributeCache.instance().get(packageName,
                  realTheme, com.android.internal.R.styleable.Window, userId);
          fullscreen = ent != null && !ent.array.getBoolean(
                  com.android.internal.R.styleable.Window_windowIsFloating, false)
                  && !ent.array.getBoolean(
                  com.android.internal.R.styleable.Window_windowIsTranslucent, false);
          noDisplay = ent != null && ent.array.getBoolean(
                  com.android.internal.R.styleable.Window_windowNoDisplay, false);

          if ((!_componentSpecified || _launchedFromUid == Process.myUid()
                  || _launchedFromUid == 0) &&
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
              mActivityType = APPLICATION_ACTIVITY_TYPE;
          }

          immersive = (aInfo.flags & ActivityInfo.FLAG_IMMERSIVE) != 0;
      } else {
          realActivity = null;
          taskAffinity = null;
          stateNotNeeded = false;
          baseDir = null;
          resDir = null;
          dataDir = null;
          processName = null;
          packageName = null;
          fullscreen = true;
          noDisplay = false;
          mActivityType = APPLICATION_ACTIVITY_TYPE;
          immersive = false;
      }
  }

```
__ReadMore：FLAG_ACTIVITY_FORWARD_RESULT的处理__

__question2:mPendingActivityLaunches的作用？__

## Step12

新建或重用`ActivityStack`,`TaskRecord`，__各种launcerMode和IntentFlag的处理__，最后调用目标`ActivityStack`的`startActivityLocked`方法
下面是三种情况下，其启动的Intent.FLAG会带上Intent.FLAG_ACTIVITY_NEW_TASK，代表需要在新的任务栈启动目标`Activity`：
* （1）`sourceRecord=null`说明要启动的`Activity`并不是由一个`Activity`的`Context`启动的，这时候我们总是启动一个新的`TASK`
* （2）`Launcher`是`SingleInstance`模式（或者说启动新的`Activity`的源`Activity`的`launchMode`为`SingleInstance`）
* （3）需要启动的`Activity`的`launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE|| r.launchMode ==ActivityInfo.LAUNCH_SINGLE_TASK`
其中可以发现:__`SingleTask`模式，但是它并不像官方文档描述的一样：`The system creates a new task and instantiates the activity at the root of the new task`，而是在跟它有相同`taskAffinity`的任务中启动，并且位于这个任务的堆栈顶端，所以如果指定特定的`taskAffinity`，就可以在新的`TaskRecord`启动__

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
       //下面是三种需要创建新的TASK的情况：
       //（1）sourceRecord=null说明要启动的Activity并不是由一个Activity的Context启动的，这时候我们总是启动一个新的TASK。
       //（2）Launcher是SingleInstance模式（或者说启动新的Activity的源Activity的launchMode为SingleInstance）
       //（3）需要启动的Activity的launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE|| r.launchMode == //ActivityInfo.LAUNCH_SINGLE_TASK（SingleTask也需要NEW_TASK？）
       if (sourceRecord == null) {
            //这种情况下是否和启动来自Service和BroadcastReceiver的context一致？
           // This activity is not being started from another...  in this case we -always- start a new task.
           if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
               Slog.w(TAG, "startActivity called from non-Activity context; forcing " +"Intent.FLAG_ACTIVITY_NEW_TASK for: " + intent);
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
               //如果sourceRecord已经处于finishing状态，那么他所在的Task就很有可能已经为空或者即将为空
               if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                   Slog.w(TAG, "startActivity called from finishing " + sourceRecord
                           + "; forcing " + "Intent.FLAG_ACTIVITY_NEW_TASK for: " + intent);
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
      //如果新的Activity要新创建Task而启动，而且原先的Activity需要返回结果，这种情况下，调用函数r.resultTo.task.stack.sendA//ctivityResultLocke//d（）将 //Activity.RESULT_CANCELED结果返回给launcher,并将resultTo置为null，resultTo的使命结束了。
       if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
           // For whatever reason this activity is being launched into a new
           // task...  yet the caller has requested a result back.  Well, that
           // is pretty messed up, so instead immediately send back a cancel
           // and let the new task continue launched as normal without a
           // dependency on its originator.
           Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
           r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho, r.requestCode,
               Activity.RESULT_CANCELED, null);
           r.resultTo = null;
       }

       boolean addingToTask = false;
       boolean movedHome = false;
       TaskRecord reuseTask = null;
       ActivityStack targetStack;
       //FLAG_ACTIVITY_NEW_TASK标志位为1，FLAG_ACTIVITY_MULTIPLE_TASK会为0
       //FLAG_ACTIVITY_MULTIPLE_TASK：Do not use this flag unless you are implementing your own top-level application //launcher.
       if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
           // If bring to front is requested, and no result is requested, and we can find a task that was started with this same
           // component, then instead of launching bring that one to the front.
           if (r.resultTo == null) {//true
               // See if there is a task to bring to the front.  If this is a SINGLE_INSTANCE activity, there can be one and only one
               // instance of it in the history, and it is always in its own unique task, so we do a special search.
               //对于LAUNCH_SINGLE_INSTANCE模式的，只会有一个唯一的实例在历史ActivityStack,且运行在特定的TaskRecord中，所以先在存在;LAUNCH_SINGLE_TASK也是只有一个唯一是ActivityStack
               //中的寻找，否则才新建TASK
               //findTaskLocked：从最新的记录开始遍历mStacks,在每个ActivityStack中查找出与ActivityRecord类型参数r对应的ActivityRecord，什么才是相对应的？从最新的Taskrecord开始遍历mTaskHistory找,找出处于栈顶且状态不为finishing的ActivityRecord，该ActivityRecord记
               //录的userid和目标的userid不等，或者栈顶activity的launchmode为LAUNCH_SINGLE_INSTANCE都找失败，从下一个TaskRecord中找，否则优先比较当前TaskRecord和目标ActivityRecord的affinity值，component值，affinityIntent的component值，其中一个匹配就返回
               //findActivityLocked:从最新的记录开始遍历mStacks,在每个ActivityStack中查找出与ActivityRecord类型参数r对应的ActivityRecord，///什么才是相对应的？从最新的Taskrecord开始遍历mTaskHistory,然后遍历TaskRecord中的mActivity，找到不处于finishing状态，
               //componnent和目标相等且userid相等的ActivityRecord
               //区别：findActivityLocked的查找是从所有Task中的ActivityRecord的比较查找，findTaskLocked只从每个Task的栈顶的ActivityRecord
               //来进行比较查找
               ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                       ? findTaskLocked(r)
                       : findActivityLocked(intent, r.info);
               //intentActivity 所在的TASK就是要启动的Activity所在的Task，要启动的Activity所在的Stack //targetStack就是intentActivity所在的Stack。
               //这里为NULL,先忽略(这里有关于ActivityInfo.LAUNCH_SINGLE_TASK是否在新栈启动的逻辑).
               if (intentActivity != null) {//false
                    if (r.task == null) {
                        r.task = intentActivity.task;
                    }
                    targetStack = intentActivity.task.stack;
                    targetStack.mLastPausedActivity = null;
                    if (DEBUG_TASKS) Slog.d(TAG, "Bring to front target: " + targetStack+ " from " + intentActivity);
                    moveHomeStack(targetStack.isHomeStack());//修改栈状态
                    if (intentActivity.task.intent == null) {
                        // This task was started because of movement of
                        // the activity based on affinity...  now that we
                        // are actually launching it, we can assign the
                        // base intent.
                        intentActivity.task.setIntent(intent, r.info);
                    }
                    // If the target task is not in the front, then we need
                    // to bring it to the front...  except...  well, with
                    // SINGLE_TASK_LAUNCH it's not entirely clear.  We'd like
                    // to have the same behavior as if a new instance was
                    // being started, which means not bringing it to the front
                    // if the caller is not itself in the front.
                    final ActivityStack lastStack = getLastStack();
                    	//遍历lastStack中所有TaskRecord中的ActivityStack，找到第一个不是停止的、delayedResume、okToShow且不等于notTop的
                      //notTop为NULL，因为FLAG_ACTIVITY_PREVIOUS_IS_TOP标识不为1
                    ActivityRecord curTop = lastStack == null?null : lastStack.topRunningNonDelayedActivityLocked(notTop);
                    //通常curTop等于intentActivity，
                    if (curTop != null && (curTop.task != intentActivity.task ||curTop.task != lastStack.topTask())) {
                        r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                        if (sourceRecord == null || (sourceStack.topActivity() != null &&
                                sourceStack.topActivity().task == sourceRecord.task)) {
                            // We really do want to push this one into the
                            // user's face, right now.
                            movedHome = true;
                            if ((launchFlags &
                                    (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                                    == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                                // Caller wants to appear on home activity.
                                intentActivity.task.mOnTopOfHome = true;
                            }
                            targetStack.moveTaskToFrontLocked(intentActivity.task, r, options);
                            options = null;
                        }
                    }
                    // If the caller has requested that the target task be
                    // reset, then do so.
                    if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                        intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                    }
                    if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                        // We don't need to start a new activity, and
                        // the client said not to do anything if that
                        // is the case, so this is it!  And for paranoia, make
                        // sure we have correctly resumed the top activity.
                        if (doResume) {
                            resumeTopActivitiesLocked(targetStack, null, options);
                        } else {
                            ActivityOptions.abort(options);
                        }
                        if (r.task == null)  Slog.v(TAG,
                                "startActivityUncheckedLocked: task left null",
                                new RuntimeException("here").fillInStackTrace());
                        return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                    }
                    if ((launchFlags &
                            (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK))
                            == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) {
                        // The caller has requested to completely replace any
                        // existing task with its new activity.  Well that should
                        // not be too hard...
                        reuseTask = intentActivity.task;
                        reuseTask.performClearTaskLocked();
                        reuseTask.setIntent(r.intent, r.info);
                    } else if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                        // In this situation we want to remove all activities
                        // from the task up to the one being started.  In most
                        // cases this means we are resetting the task to its
                        // initial state.
                        //像Intent#FLAG_ACTIVITY_CLEAR_TOP一样触发清理操作，查找当前栈中是否存在r,如果存在，r以上的Activity都会被finiish掉，并返回该ActivityRecord实例，如果当前实例是默认模式，那么当前的实例也会finish，并返回NULL
                        ActivityRecord top =intentActivity.task.performClearTaskLocked(r, launchFlags);
                        if (top != null) {
                            if (top.frontOfTask) {
                                // Activity aliases may mean we use different
                                // intents for the top activity, so make sure
                                // the task now has the identity of the new
                                // intent.
                                top.task.setIntent(r.intent, r.info);
                            }
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT,
                                    r, top.task);
                            top.deliverNewIntentLocked(callingUid, r.intent);
                        } else {
                            // A special case: we need to
                            // start the activity because it is not currently
                            // running, and the caller has asked to clear the
                            // current task to have this activity at the top.
                            addingToTask = true;
                            // Now pretend like this activity is being started
                            // by the top of its task, so it is put in the
                            // right place.
                            sourceRecord = intentActivity;
                        }
                    } else if (r.realActivity.equals(intentActivity.task.realActivity)) {
                        // In this case the top activity on the task is the
                        // same as the one being launched, so we take that
                        // as a request to bring the task to the foreground.
                        // If the top activity in the task is the root
                        // activity, deliver this new intent to it if it
                        // desires.
                        if (((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP)
                                && intentActivity.realActivity.equals(r.realActivity)) {
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r,
                                    intentActivity.task);
                            if (intentActivity.frontOfTask) {
                                intentActivity.task.setIntent(r.intent, r.info);
                            }
                            intentActivity.deliverNewIntentLocked(callingUid, r.intent);
                        } else if (!r.intent.filterEquals(intentActivity.task.intent)) {
                            // In this case we are launching the root activity
                            // of the task, but with a different intent.  We
                            // should start a new instance on top.
                            addingToTask = true;
                            sourceRecord = intentActivity;
                        }
                    } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                        // In this case an activity is being launched in to an
                        // existing task, without resetting that task.  This
                        // is typically the situation of launching an activity
                        // from a notification or shortcut.  We want to place
                        // the new activity on top of the current task.
                        addingToTask = true;
                        sourceRecord = intentActivity;
                    } else if (!intentActivity.task.rootWasReset) {
                        // In this case we are launching in to an existing task
                        // that has not yet been started from its front door.
                        // The current task has been brought to the front.
                        // Ideally, we'd probably like to place this new task
                        // at the bottom of its stack, but that's a little hard
                        // to do with the current organization of the code so
                        // for now we'll just drop it.
                        intentActivity.task.setIntent(r.intent, r.info);
                    }
                    if (!addingToTask && reuseTask == null) {
                        // We didn't do anything...  but it was needed (a.k.a., client
                        // don't use that intent!)  And for paranoia, make
                        // sure we have correctly resumed the top activity.
                        if (doResume) {
                            targetStack.resumeTopActivityLocked(null, options);
                        } else {
                            ActivityOptions.abort(options);
                        }
                        if (r.task == null)  Slog.v(TAG,
                            "startActivityUncheckedLocked: task left null",
                            new RuntimeException("here").fillInStackTrace());
                        return ActivityManager.START_TASK_TO_FRONT;
                    }
                }
           }
       }
      //上段代码的逻辑是看一下，当前在堆栈顶端的Activity是否就是即将要启动的Activity，有些情况下，如果
      //即将要启动的Activity就在堆栈的顶端，那么，就不会重新启动这个Activity的别一个实例了，具体可以参考官方网站
      //http://developer.android.com/reference/android/content/pm/ActivityInfo.html
       if (r.packageName != null) {
           // If the activity being launched is the same as the one currently
           // at the top, then we need to check if it should only be launched
           // once.
           ActivityStack topStack = getFocusedStack();//mHomeStack
           //从最新的Taskrecord开始遍历mTaskHistory,然后遍历TaskRecord中的mActivity，找到第一个状态不为finishing，
           //不为delayedResume，不等于参数r，且okToShow方法返回true的ActivityRecord，这里的top会为null
           ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);//notTop为null
           if (top != null && r.resultTo == null) {
               if (top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                   if (top.app != null && top.app.thread != null) {
                       if ((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                           || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP
                           || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
                           // For paranoia, make sure we have correctly
                           // resumed the top activity.
                           topStack.mLastPausedActivity = null;
                           if (doResume) {
                               resumeTopActivitiesLocked();
                           }
                           ActivityOptions.abort(options);
                           if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                               // We don't need to start a new activity, and
                               // the client said not to do anything if that
                               // is the case, so this is it!
                               if (r.task == null)  Slog.v(TAG,
                                   "startActivityUncheckedLocked: task left null",
                                   new RuntimeException("here").fillInStackTrace());
                               return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                           }
                           top.deliverNewIntentLocked(callingUid, r.intent);
                           if (r.task == null)  Slog.v(TAG,
                               "startActivityUncheckedLocked: task left null",
                               new RuntimeException("here").fillInStackTrace());
                           return ActivityManager.START_DELIVERED_TO_TOP;
                       }
                   }
               }
           }

       } else {
           if (r.resultTo != null) {
               r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho,
                       r.requestCode, Activity.RESULT_CANCELED, null);
           }
           ActivityOptions.abort(options);
           if (r.task == null)  Slog.v(TAG,
               "startActivityUncheckedLocked: task left null",
               new RuntimeException("here").fillInStackTrace());
           return ActivityManager.START_CLASS_NOT_FOUND;
       }

       boolean newTask = false;
       boolean keepCurTransition = false;

       // Should this be considered a new task?
       //addingToTask为false，因为前面的intentActivity为null
       if (r.resultTo == null && !addingToTask&& (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {//true
           targetStack = adjustStackFocus(r);//获取当前或新建的mFocusedStack
           moveHomeStack(targetStack.isHomeStack());//状态修改为STACK_STATE_HOME_TO_BACK
           if (reuseTask == null) {//同addingToTask，reuseTask为null
              //创建新的TaskRecord，如果已经记录在其他TASKRECORD，移除，记录新的TaskRecord，并把TaskRecord插入到mTaskHistory的顶部，但TaskRecord并未记录该ActivityRecord
               r.setTask(targetStack.createTaskRecord(getNextTaskId(), r.info, intent, true),null, true);
               if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r + " in new task " +r.task);
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
       } else if (sourceRecord != null) {
            TaskRecord sourceTask = sourceRecord.task;
            targetStack = sourceTask.stack;
            moveHomeStack(targetStack.isHomeStack());
            if (!addingToTask &&
                    (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
                // In this case, we are adding the activity to an existing
                // task, but the caller has asked to clear that task if the
                // activity is already running.
                //...
            } else if (!addingToTask &&
                    (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                // In this case, we are launching an activity in our own task
                // that may already be running somewhere in the history, and
                // we want to shuffle it to the front of the stack if so.
                //...
            }
            // An existing activity is starting this new activity, so we want
            // to keep the new one in the same task as the one that is starting
            // it.
            r.setTask(sourceTask, sourceRecord.thumbHolder, false);
            if (DEBUG_TASKS) Slog.v(TAG, "Starting new activity " + r
                    + " in existing task " + r.task + " from source " + sourceRecord);

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

__ReadMore:创建新的ActivityStack的过程，包括了WindowManagerService的处理__
```java
ActivityStackSupervisor.java
//可以看出对于ApplicationActivity类型的ActivityRecord总添加在mFocusedStack的Task中

ActivityStack adjustStackFocus(ActivityRecord r) {
        final TaskRecord task = r.task;//task为null，因为当前的r并未添加到任何TaskRecord
        //根据Step11ActivityRecord的构造，可以知道r.isApplicationActivity为true
        if (r.isApplicationActivity() || (task != null && task.isApplicationTask())) {
            if (task != null) {//false
                if (mFocusedStack != task.stack) {
                    if (DEBUG_FOCUS || DEBUG_STACK) Slog.d(TAG,"adjustStackFocus: Setting focused stack to r=" + r + " task=" + task);
                    mFocusedStack = task.stack;
                } else {
                    if (DEBUG_FOCUS || DEBUG_STACK) Slog.d(TAG,"adjustStackFocus: Focused stack already=" + mFocusedStack);
                }
                return mFocusedStack;
            }
            if (mFocusedStack != null) {
                return mFocusedStack;
            }
            for (int stackNdx = mStacks.size() - 1; stackNdx > 0; --stackNdx) {
                ActivityStack stack = mStacks.get(stackNdx);
                if (!stack.isHomeStack()) {
                    if (DEBUG_FOCUS || DEBUG_STACK) Slog.d(TAG,"adjustStackFocus: Setting focused stack=" + stack);
                    mFocusedStack = stack;
                    return mFocusedStack;
                }
            }
            // Time to create the first app stack for this user.
            int stackId = mService.createStack(-1, HOME_STACK_ID,StackBox.TASK_STACK_GOES_OVER, 1.0f);
            if (DEBUG_FOCUS || DEBUG_STACK)
            Slog.d(TAG, "adjustStackFocus: New stack r=" + r +" stackId=" + stackId);
            mFocusedStack = getStack(stackId);
            return mFocusedStack;
        }
        //非ApplicationActivity类型的ActivityRecord才会到这里
        return mHomeStack;
    }


ActivityManagerService.java

//taskId=-1，relativeStackBoxId=HOME_STACK_ID=0,position=6,weight=1
@Override
public int createStack(int taskId, int relativeStackBoxId, int position, float weight) {
    enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,"createStack()");
    synchronized (this) {
        long ident = Binder.clearCallingIdentity();
        try {
            int stackId = mStackSupervisor.createStack();
            mWindowManager.createStack(stackId, relativeStackBoxId, position, weight);
            if (taskId > 0) {//taskId=-1，还未绑定相应的额TaskRecord
                moveTaskToStack(taskId, stackId, true);//移动到特定的TaskRecord到特定的ActivityStack的顶端或底端
            }
            return stackId;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
}

int createStack() {
     while (true) {
         if (++mLastStackId <= HOME_STACK_ID) {
             mLastStackId = HOME_STACK_ID + 1;
         }
         if (getStack(mLastStackId) == null) {
             break;
         }
     }
     mStacks.add(new ActivityStack(mService, mContext, mLooper, mLastStackId));//ActivityStack的新建
     return mLastStackId;
 }

```
__READMORE:创建新的TaskRecord__

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

   private void insertTaskAtTop(TaskRecord task) {
        // If this is being moved to the top by another activity or being launched from the home activity, set mOnTopOfHome accordingly.
        ActivityStack lastStack = mStackSupervisor.getLastStack();//mForceStack
        final boolean fromHome = lastStack == null ? true : lastStack.isHomeStack();//false
        if (!isHomeStack() && (fromHome || topTask() != task)) {//true,topTask==null,因为当前并没有ActivityRecord记录在`TaskRecord`
            task.mOnTopOfHome = fromHome;//如果标记为fromHome，当前Task退出之后就显示的是Home（Launch the home activity when leaving this task.）
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
        if (!newTask) {//false
          //不是新的任务
        //...
        }
        // Place a new activity at top of stack, so it is next to interact with the user.
        // If we are not placing the new activity front most, we do not want
        // to deliver the onUserLeaving callback to the actual frontmost activity
        if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {//false,task=null
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG,"startActivity() behind front, mUserLeaving=false");
        }

        task = r.task;
        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        task.addActivityToTop(r);//添加到`TaskRecord`中`mActivities`的顶部

        r.putInHistory();//inHistory标识为true
        r.frontOfTask = newTask;//true
        if (!isHomeStack() || numActivities() > 0) {//true
            // We want to show the starting preview window if we are switching to a new task, or the next activity's process is
            // not currently running.
            boolean showStartingIcon = newTask;//true
            ProcessRecord proc = r.app;//null
            if (proc == null) {//true
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {//false
                showStartingIcon = true;
            }

            if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                mWindowManager.prepareAppTransition(newTask
                        ? AppTransition.TRANSIT_TASK_OPEN
                        : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            r.updateOptionsLocked(options);//pendingOptions=null
            mWindowManager.addAppToken(task.mActivities.indexOf(r),
                    r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_ON_LOCK_SCREEN) != 0, r.userId);
            boolean doShow = true;
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
                ActivityRecord prev = mResumedActivity;
                if (prev != null) {
                    // We don't want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.task != r.task) {
                        prev = null;
                    }
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                mWindowManager.setAppStartingWindow(
                        r.appToken, r.packageName, r.theme,
                        mService.compatibilityInfoForPackageLocked(
                                r.info.applicationInfo), r.nonLocalizedLabel,
                        r.labelRes, r.icon, r.logo, r.windowFlags,
                        prev != null ? prev.appToken : null, showStartingIcon);
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            //...
        }
        if (VALIDATE_TOKENS) {
            validateAppTokensLocked();
        }

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

该函数是将处于栈顶的`Activity`组件激活，这个`Activity`正好就是即将要启动的`Activity`组件，但在启动之前需要先暂停其他栈中的`mResumedActivity`h,接着就是当前栈的`mResumedActivity`,由于从`Launcher`启动，那么就需要先暂停`mHomeStack`中的`Launcher`组件

```java
ActivityStack.java

//prev=null,options=null
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (ActivityManagerService.DEBUG_LOCKSCREEN) mService.logLockScreen("");

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
        //分别暂停所有`mStacks`中不是前台`Stack`中的所有`mResumedActivity`和暂停当前栈的`mResumedActivity`
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving);//返回true
        if (mResumedActivity != null) {//mResumedActivity为NULL，当前`ActivityStack`在Step12新建
            pausing = true;
            startPausingLocked(userLeaving, false);//暂停栈正在显示的Activity，见Step17
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
* [ActivityStackSupervisor.StartActivityUncheckedLocked()函数分析](http://www.aiuxian.com/article/p-1880233.html)
* [Activity管理机制](http://blog.csdn.net/guoqifa29/article/details/39341931)
* [http://blog.csdn.net/guoqifa29/article/details/40015127](http://blog.csdn.net/guoqifa29/article/details/40015127)
* [Activity生命周期的回调，你应该知道得更多](http://blog.csdn.net/yalinfendou/article/details/46909173)
