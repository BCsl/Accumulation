# ActivityManagerService对不同的launchMode或IntentFlag的处理

当前启动一个Activity，`AMS`会新建该Activity在`AMS`中的实例`ActivityRecord`对象，具体在`ActivityStackSupervisor#startActivityUncheckedLocked`方法处理，对于使用`Intent.FLAG_ACTIVITY_NEW_TASK`、`ActivityInfo.LAUNCH_SINGLE_TASK`和`ActivityInfo.LAUNCH_SINGLE_INSTANCE`，新建之后还需要在其管理的`ActivityStack`中查找是否已经创建过了，如果是，那么对于不同的`launcheMode`或`IntentFlag`进行相应的处理

```java
//ActivityStackSupervisor

final int startActivityUncheckedLocked(ActivityRecord r,ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options){

//....

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
        if (intentActivity != null) {//false 结束953
             if (r.task == null) {
                 r.task = intentActivity.task;//记录其所在的栈，相同的affinity值或component等
             }
             targetStack = intentActivity.task.stack; //对于APPLICATION_ACTIVITY_TYPE来说，一般非HomeStack
             targetStack.mLastPausedActivity = null;
             moveHomeStack(targetStack.isHomeStack());//这里的状态待定，两种情况都可能发生
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
             final ActivityStack lastStack = getLastStack();// 这里的状态待定，两种情况都可能发生，mFocusedStack或mHomeStack
              //遍历lastStack中所有TaskRecord中的ActivityStack，找到第一个不是停止的、delayedResume、okToShow且不等于notTop的
              //notTop为NULL，于FLAG_ACTIVITY_PREVIOUS_IS_TOP标识有关，一般不用到，用于在获取栈顶ActivityRecord的时候，如果其等于该ActivityRecord，会过滤掉
             ActivityRecord curTop = lastStack == null?null : lastStack.topRunningNonDelayedActivityLocked(notTop);

             if (curTop != null && (curTop.task != intentActivity.task ||curTop.task != lastStack.topTask())) {
                 r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                 if (sourceRecord == null || (sourceStack.topActivity() != null && sourceStack.topActivity().task == sourceRecord.task)) {
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
             // If the caller has requested that the target task be reset, then do so.
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
             //Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK的搭配使用
             if ((launchFlags & (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK))
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
                     //触发onNewIntent
                     top.deliverNewIntentLocked(callingUid, r.intent);
                 } else {
                     //`r`没在所匹配的`TaskRecord`中，那就需要进一步添加到`TaskRecord
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
         }// Start795
    }
}
//...
```

<http://www.2cto.com/kf/201501/371935.html>
