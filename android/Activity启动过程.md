# Activity的启动过程

## setp1
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
## setp2
```java
Launcher3.java

boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);//intent的FLAG_ACTIVITY_NEW_TASK置为1，以为在新的TASK启动Activity
        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
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
## setp3
```java
Activity.java

public void startActivity(Intent intent, Bundle options) {
       if (options != null) {
           startActivityForResult(intent, -1, options);
       } else {
           // Note we want to go through this call for compatibility with applications that may have overridden the method.
           startActivityForResult(intent, -1);
       }
   }
```
## setp4
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
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
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
## setp5
```java
Instrumentation.java
//requestCode=-1 ,option=null,token:Activity的成员变量mToken,contextThread:mMainThread.getApplicationThread()

public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, Fragment target,
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
## setp6
```java
ActivityManagerProxy.java
//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,caller:mMainThread.getApplicationThread()

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
## setp7

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

//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,caller:mMainThread.getApplicationThread()

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

##  setp9

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
## setp10

```java

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
            stack.mConfigWillChange = config != null
                    && mService.mConfiguration.diff(config) != 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG,"Starting activity when config will change = " + stack.mConfigWillChange);

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
## setp11

```java

//requestCode=-1 ,option=null,resultTo:Activity的成员变量mToken(IApplicationTokenStub)，profileFd=null,profileFile=null,startFlags=0
//resultWho:Activity(Fragment)的成员变量mWho,caller:mMainThread.getApplicationThread()
//outResult=null,config=null,ainfo:from resolveActivity,outActivity:null,

final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo, IBinder resultTo,
            String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity) {
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
                Slog.w(TAG, "Unable to find app for caller " + caller
                      + " (pid=" + callingPid + ") when starting: "
                      + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        if (err == ActivityManager.START_SUCCESS) {
            final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from pid " + (callerApp != null ? callerApp.pid : callingPid));
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
          //ActivityStackSupervisor有多个ActivityStack,检测IApplicationTokenStub类型参数是否在其中一个ActivityStack中
          //这里的sourceRecord应该是mHomeStack？
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(
                TAG, "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
		//Will send result to，上面可以知 resultRecord应该是空
        ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;

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
            // We couldn't find a class that can handle the given Intent.
            // That's the end of that!
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // We couldn't find the specific class specified in the Intent.
            // Also the end of the line.
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }

        if (err != ActivityManager.START_SUCCESS) {
            //...
            return err;
        }

        final int startAnyPerm = mService.checkPermission(
                START_ANY_ACTIVITY, callingPid, callingUid);
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
                || stack.mResumedActivity.info.applicationInfo.uid != callingUid) {
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
		//里面也会调用startActivityUncheckedLocked
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
__ReadMore：FLAG_ACTIVITY_FORWARD_RESULT的处理__
