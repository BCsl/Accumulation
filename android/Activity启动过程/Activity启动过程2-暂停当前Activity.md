# 接Activity启动过程Step15

## Step16

暂停所有`mStacks`中不是前台`Stack`中的所有`mResumedActivity`，参数`userLeaving`指示是需要要回调方法`onUserLeaving`

```java
ActivityStackSupervisor.java

//userLeaving=true
boolean pauseBackStacks(boolean userLeaving) {
     boolean someActivityPaused = false;
     for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
         final ActivityStack stack = mStacks.get(stackNdx);
         if (!isFrontStack(stack) && stack.mResumedActivity != null) {
             stack.startPausingLocked(userLeaving, false);
             someActivityPaused = true;
         }
     }
     return someActivityPaused;
 }
```

## Step17

记录标记将要暂停的当前Stack的`mResumedActivity`对象,状态标记为`PAUSING`，通过`Binder`发送异步消息到`ApplicationThread`处理暂停事务，发起暂停超时消息

```java
ActivityStack.java
//userLeaving=true,uiSleeping=false;

final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
       if (mPausingActivity != null) {
           Slog.e(TAG, "Trying to pause when pause is already pending for "
                 + mPausingActivity, new RuntimeException("here").fillInStackTrace());
       }
       ActivityRecord prev = mResumedActivity;
       if (prev == null) {
           Slog.e(TAG, "Trying to pause when nothing is resumed",new RuntimeException("here").fillInStackTrace());
           mStackSupervisor.resumeTopActivitiesLocked();//Step14
           return;
       }
       //把本来Resumed的记录Pausing和LastPaused
       mResumedActivity = null;
       mPausingActivity = prev;
       mLastPausedActivity = prev;
       mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
               || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
       prev.state = ActivityState.PAUSING;//状态记录为PAUSING，成功暂停会记录为PAUSED,Step25
       prev.task.touchActiveTime();
       clearLaunchTime(prev);
       prev.updateThumbnail(screenshotActivities(prev), null);
       stopFullyDrawnTraceIfNeeded();
       mService.updateCpuStats();
       //app变量为类型为Processrecord描述Activity组件运行的进程，app.thread类型为ApplicationThreadProxy
       if (prev.app != null && prev.app.thread != null) {
           if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending pause: " + prev);
           try {
               mService.updateUsageStats(prev, false);
               //发送终止通知，以便执行数据保存操作
               // IApplicationThread
               prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,userLeaving, prev.configChangeFlags);
           } catch (Exception e) {
               // Ignore exception, if process died other code will cleanup.
              //...
           }
       } else {
          //...
       }

       // If we are not going to sleep, we want to ensure the device is awake until the next activity is started.
       if (!mService.isSleepingOrShuttingDown()) {
           mStackSupervisor.acquireLaunchWakelock();
       }
       if (mPausingActivity != null) {
           // Have the window manager pause its key dispatching until the new
           // activity has started.  If we're pausing the activity just because
           // the screen is being turned off and the UI is sleeping, don't interrupt
           // key dispatch; the same activity will pick it up again on wakeup.
           if (!uiSleeping) {
               prev.pauseKeyDispatchingLocked();
           } else {
               if (DEBUG_PAUSE) Slog.v(TAG, "Key dispatch not paused for screen off");
           }

           // Schedule a pause timeout in case the app doesn't respond. We don't give it much time because this directly impacts the
           // responsiveness seen by the user.
           Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
           msg.obj = prev;
           prev.pauseTime = SystemClock.uptimeMillis();
           mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
           if (DEBUG_PAUSE) Slog.v(TAG, "Waiting for pause to complete...");
       } else {
           // This activity failed to schedule the pause, so just treat it as being paused now.
           if (DEBUG_PAUSE) Slog.v(TAG, "Activity not running, resuming next.");
           mStackSupervisor.getFocusedStack().resumeTopActivityLocked(null);//Step14
       }
   }
```

暂停Activity超时处理

```java
ActivityStack.java

case PAUSE_TIMEOUT_MSG: {
                  ActivityRecord r = (ActivityRecord)msg.obj;
                  // We don't at this point know if the activity is fullscreen,so we need to be conservative and assume it isn't.
                  Slog.w(TAG, "Activity pause timeout for " + r);
                  synchronized (mService) {
                      if (r.app != null) {
                          mService.logAppTooSlow(r.app, r.pauseTime, "pausing " + r);
                      }
                      activityPausedLocked(r.appToken, true);
                  }
              } break;
```

## Step18

发起的是异步的通信请求，请求暂停`token`标识的`Activity`（见Step29,记录在Processrecord#thread是binder本地对象，并非代理对象，所以可能并没有经过进程间通信来处理`schedulePauseActivity`方法？） **update:2016-01-03但如果这样暂停超时便没有了意义？,但是`ApplicationThread`各种的调度是通过发送消息到`ActivityThread`的`Handler`成语变量`H`来处理！所以也是一个异步操作** 所以这可以直接看`Step20`了

```java
ApplicationThreadProxy.java
//token:ActivityRecord#appToken（IApplicationToken.Stub类型），finished=false，userLeaving=true，configChanges=false;
public final void schedulePauseActivity(IBinder token, boolean finished,boolean userLeaving, int configChanges) {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        data.writeStrongBinder(token);
        data.writeInt(finished ? 1 : 0);
        data.writeInt(userLeaving ? 1 :0);
        data.writeInt(configChanges);
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

## Step19

```java
ApplicationThreadNative.java

public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
          throws RemoteException {
      switch (code) {
      case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION:
      {
          data.enforceInterface(IApplicationThread.descriptor);
          IBinder b = data.readStrongBinder();
          boolean finished = data.readInt() != 0;//0
          boolean userLeaving = data.readInt() != 0;//1
          int configChanges = data.readInt();//0
          schedulePauseActivity(b, finished, userLeaving, configChanges);
          return true;
      }
      //....
    }
   return super.onTransact(code, data, reply, flags);  
  }
```

## Step20

`ActivityThread`内部类 `ApplicationThread`实现了`ApplicationThreadNative`抽象类

```java
ApplicationThread.java

public final void schedulePauseActivity(IBinder token, boolean finished,boolean userLeaving, int configChanges) {
      queueOrSendMessage(finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,token,(userLeaving ? 1 : 0),configChanges);
}

  private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
       synchronized (this) {
           Message msg = Message.obtain();
           msg.what = what;//H.PAUSE_ACTIVITY
           msg.obj = obj;//ActivityRecord#appToken（IApplicationToken.Stub类型）
           msg.arg1 = arg1;//1
           msg.arg2 = arg2;//0
           mH.sendMessage(msg);
       }
   }
```

`ActivityThread`内`Handle`类型的成员变量`mH`处理`PAUSE_ACTIVITY`消息

```java
case PAUSE_ACTIVITY:
                   Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                   handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                   maybeSnapshot();
                   Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                   break;
```

## Step21

调用暂停回调

```java
ActivityThread.java

//token:ActivityRecord#appToken（IApplicationToken.Stub类型），finished=false，userLeaving=true，configChanges=false;

private void handlePauseActivity(IBinder token, boolean finished,boolean userLeaving, int configChanges) {
    ActivityClientRecord r = mActivities.get(token);//ActivityClientRecord和ActivityRecord类似
    if (r != null) {
        if (userLeaving) {
            performUserLeavingActivity(r);
        }
        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(token, finished, r.isPreHoneycomb());
        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        // Tell the activity manager we have paused.
        try {
            ActivityManagerNative.getDefault().activityPaused(token);
        } catch (RemoteException ex) {
        }
    }
}
```

## Step22

`performUserLeavingActivity`会回调`Activity`的`performUserLeaving`方法

```java
ActivityThread.java

    final void performUserLeavingActivity(ActivityClientRecord r) {
        mInstrumentation.callActivityOnUserLeaving(r.activity);
    }

Instrumentation.java

    public void callActivityOnUserLeaving(Activity activity) {
      activity.performUserLeaving();
  }

Activity.java

  final void performUserLeaving() {
    //这两个方法在Activity是空实现
       onUserInteraction();
       onUserLeaveHint();
   }
```

## Step23

`performPauseActivity`会回调`Activity`的`onSaveInstanceState`，`onPause`

```java
//token:ActivityRecord#appToken（IApplicationToken.Stub类型），saveState=Android3.0之后为false，之前为true

final Bundle performPauseActivity(IBinder token, boolean finished,boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
}

final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,boolean saveState) {
        if (r.paused) {
            if (r.activity.mFinished) {
                return null;
            }
            RuntimeException e = new RuntimeException("Performing pause of activity that is not resumed: "+r.intent.getComponent().toShortString());
        }
        Bundle state = null;
        if (finished) {
            r.activity.mFinished = true;
        }
        try {
            // Next have the activity save its current state and managed dialogs...
            if (!r.activity.mFinished && saveState) {
                state = new Bundle();
                state.setAllowFds(false);
                mInstrumentation.callActivityOnSaveInstanceState(r.activity, state);//回调onSaveInstanceState
                r.state = state;//记录state
            }
            // Now we are idle.
            r.activity.mCalled = false;//父类Acitivy的onPuase被调用后置为true，防止用户不调用super.onPause()
            mInstrumentation.callActivityOnPause(r.activity);//回调performPause，最后回调onPause
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException("Activity " + r.intent.getComponent().toShortString() +" did not call through to super.onPause()");
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to pause activity "  + r.intent.getComponent().toShortString()+ ": " + e.toString(), e);
            }
        }
        r.paused = true;
        // Notify any outstanding on paused listeners
        //...
        return state;
    }
```

`mCalled`变量用来检测`super.onPause`的执行，因为这里还有一个步骤需要处理，`Application`进行`ActivityLifecycleCallbacks`的回调

```java
Activity.java

final void performPause() {
       mDoReportFullyDrawn = false;
       mFragments.dispatchPause();
       mCalled = false;
       onPause();
       mResumed = false;
       if (!mCalled && getApplicationInfo().targetSdkVersion>= android.os.Build.VERSION_CODES.GINGERBREAD) {
           throw new SuperNotCalledException( "Activity " + mComponent.toShortString() +" did not call through to super.onPause()");
       }
       mResumed = false;
   }

   protected void onPause() {
      getApplication().dispatchActivityPaused(this);
      mCalled = true;
  }
```

## Step24

回到`Step21`,处理完`Activity`的`onPause`，还有一个步骤需要做，是否还记得`Step17`所发送的超时处理消息，`ActivityManagerNative.getDefault().activityPaused(token)`,告诉`ActivityManagerService`处理暂停成功，移除超时消息，最后添加到`mStackSupervisor`的`mStoppingActivities`，再次调用方法 `resumeTopActivitiesLocked`启动位于顶端的`Activity`组件,该方法在前面的`Step15`已经调用了，当时尚有未进入`PAUSED`状态的`Launcher`组件，所以 `ActivityStackSupervisor`调用的`pauseBackStacks`方法返回true，所以跳过了Resume步骤，先进行了Pausing步骤。

```java
ActivityManagerProxy.java

public void activityPaused(IBinder token) throws RemoteException
{
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(token);
    mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
}
```

## Step25

`ActivityManagerService`处理`Activity`暂停成功，移除`Step17`发出的超时消息，`ActivityRecord`状态记录为`PAUSED`

```java
ActivityManagerNative.java

case ACTIVITY_PAUSED_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder token = data.readStrongBinder();//ActivityRecord#appToken（IApplicationToken.Stub类型）
            activityPaused(token);
            reply.writeNoException();
            return true;
        }
```

```java
@Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            //获取ActivityRecord所在的ActivityStack，r.task.stack
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```

```java
ActivityStack.java

final void activityPausedLocked(IBinder token, boolean timeout) {
    final ActivityRecord r = isInStackLocked(token);
    if (r != null) {
        mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
        if (mPausingActivity == r) {
            r.state = ActivityState.PAUSED;
            completePauseLocked();
        } else {
          //....
        }
    }
}
```

完成当前`ActivityStack`中显示的`mResumedActivity`（实际上现在已经记录在`mPausingActivity`），接着就可以显示需要`Resume`的Activity

```java
ActivityStack.java

private void completePauseLocked() {
        ActivityRecord prev = mPausingActivity;
        if (prev != null) {
            if (prev.finishing) {//false
                if (DEBUG_PAUSE) Slog.v(TAG, "Executing finish of activity: " + prev);
                prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false);
            } else if (prev.app != null) {
                if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending stop: " + prev);
                if (prev.waitingVisible) {//false
                    prev.waitingVisible = false;
                    mStackSupervisor.mWaitingVisibleActivities.remove(prev);
                    if (DEBUG_SWITCH || DEBUG_PAUSE) Slog.v(TAG, "Complete pause, no longer waiting: " + prev);
                }
                if (prev.configDestroy) {//false
                    // The previous is being paused because the configuration
                    // is changing, which means it is actually stopping...
                    // To juggle the fact that we are also starting a new
                    // instance right now, we need to first completely stop
                    // the current instance before starting the new one.
                    if (DEBUG_PAUSE) Slog.v(TAG, "Destroying after pause: " + prev);
                    destroyActivityLocked(prev, true, false, "pause-config");
                } else {
                    mStackSupervisor.mStoppingActivities.add(prev); //添加到mStoppingActivities，
                    if (mStackSupervisor.mStoppingActivities.size() > 3 || prev.frontOfTask && mTaskHistory.size() <= 1) {
                        // If we already have a few activities waiting to stop,
                        // then give up on things going idle and start clearing
                        // them out. Or if r is the last of activity of the last task the stack
                        // will be empty and must be cleared immediately.
                        if (DEBUG_PAUSE) Slog.v(TAG, "To many pending stops, forcing idle");
                        mStackSupervisor.scheduleIdleLocked();
                    } else {
                        mStackSupervisor.checkReadyForSleepLocked();
                    }
                }
            } else {
                if (DEBUG_PAUSE) Slog.v(TAG, "App died during pause, not stopping: " + prev);
                prev = null;
            }
            mPausingActivity = null;
        }

        final ActivityStack topStack = mStackSupervisor.getFocusedStack();//mFocusedStack
        if (!mService.isSleepingOrShuttingDown()) {
            mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);//会到Step14
        } else {
          //...
        }

        if (prev != null) {
            prev.resumeKeyDispatchingLocked();
            //..
            prev.cpuTimeAtResume = 0; // reset it
        }
    }
```
