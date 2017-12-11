# 应用置于后台点击 Launcher 进入流程

场景：当前应用的 TaskRecord 记录是 [MainActivity, SecondActivity]，从 Launcher 点击图标进入，其中应用的入口 Activity 是 MainActivity，按照经验，最后会展示 SecondActivity，点击返回后显示 MainActivity，这过程又发生了什么？

先看 `ActivityStackSupervisor#startActivityUncheckedLocked` 方法

点击应用后 Launcher 启动的 Activity 还是 `MainActivity`，所以参数 `r` 是 `MainActivity` 的 `ActivityRecord`，`sourceRecord` 则是 Launcher 的 Activity

```java
final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options) {
    final Intent intent = r.intent;
    final int callingUid = r.launchedFromUid;

    int launchFlags = intent.getFlags();

    // We'll invoke onUserLeaving before onPause only if the launching
    // activity did not explicitly state that this is an automated launch.
    mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;  //true

    // If the caller has asked not to resume at this point, we make note
    // of this in the record so that we can skip it when trying to find
    // the top running activity.
    if (!doResume) {  // doResume = true
        r.delayedResume = true;
    }

    ActivityRecord notTop = (launchFlags&Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP) != 0 ? r : null; //null

    //...
    ActivityInfo newTaskInfo = null;
    Intent newTaskIntent = null;
    final ActivityStack sourceStack;
    if (sourceRecord != null) {
        if (sourceRecord.finishing) { //false
            //....
        } else {
            sourceStack = sourceRecord.task.stack;
        }
    }
    //...

    boolean addingToTask = false;
    boolean movedHome = false;
    TaskRecord reuseTask = null;
    ActivityStack targetStack;
    //从 Launcher 启动会帮你自带 Intent.FLAG_ACTIVITY_NEW_TASK，Intent.FLAG_ACTIVITY_MULTIPLE_TASK 则不会有
    if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
            (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
            || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {

        if (r.resultTo == null) {
            // r 代表着 MainActivity，这里找到的 intentActivity 则是栈顶的 SecondActivity，因为处在同一个 ActivityStack 和 TaskRecord
            ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE ? findTaskLocked(r) : findActivityLocked(intent, r.info);
            if (intentActivity != null) {
                if (r.task == null) {
                    r.task = intentActivity.task; //确定目标 ActivityStack
                }
                targetStack = intentActivity.task.stack; //确定目标 ActivityStack
                targetStack.mLastPausedActivity = null;

                moveHomeStack(targetStack.isHomeStack()); //记录 HomeStack 置于后台
                //...
                final ActivityStack lastStack = getLastStack(); // HomeStack
                ActivityRecord curTop = lastStack == null? null : lastStack.topRunningNonDelayedActivityLocked(notTop); //Launcher Activity
                if (curTop != null && (curTop.task != intentActivity.task || curTop.task != lastStack.topTask())) {
                    r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT); //重新带回前台显示，添加 intent flag
                    if (sourceRecord == null || (sourceStack.topActivity() != null && sourceStack.topActivity().task == sourceRecord.task)) {
                        movedHome = true;
                        //这句很重要，把 SecondActivity 所在 TaskRecod 置于前台
                        targetStack.moveTaskToFrontLocked(intentActivity.task, r, options);
                        //..
                        options = null;
                    }
                }
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                }
                //..
                if ((launchFlags &
                        (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK))
                        == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) {
                  //..
                } else if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                        //...
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
                            //...
                    } //..
                }
                //...
                if (!addingToTask && reuseTask == null) {
                    // We didn't do anything...  but it was needed (a.k.a., client
                    // don't use that intent!)  And for paranoia, make
                    // sure we have correctly resumed the top activity.
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null, options);
                    } //...
                    return ActivityManager.START_TASK_TO_FRONT;
                }
            }
        }
    }
    //....
```

其中最重要的是根据 MainActivity 找到其原本所处的 `TaskRecord`，此时栈顶的是 SecondActivity，并把该 `TaskRecord` 置于用户面前，其中 `mStackSupervisor.resumeTopActivitiesLocked()` 就是把当前的 `ActivityStack`( mFocusStack ) 的 `mTaskHistory` 中的第一个 `TaskRecord` 的栈顶 `Activity` 置于前台，走一遍生命周期

```java
ActivityStack.java

final void moveTaskToFrontLocked(TaskRecord tr, ActivityRecord reason, Bundle options) {
    //...
    mStackSupervisor.moveHomeStack(isHomeStack());

    // Shift all activities with this task up to the top
    // of the stack, keeping them in the same internal order.
    insertTaskAtTop(tr);  //置于 `mTaskHistory` 的栈顶

    if (reason != null &&
            (reason.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
        mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
        ActivityRecord r = topRunningActivityLocked(null);
        if (r != null) {
            mNoAnimActivities.add(r);
        }
        ActivityOptions.abort(options);
    } else {
        updateTransitLocked(AppTransition.TRANSIT_TASK_TO_FRONT, options);
    }

    mWindowManager.moveTaskToTop(tr.taskId);
    mStackSupervisor.resumeTopActivitiesLocked();

    if (VALIDATE_TOKENS) {
        validateAppTokensLocked();
    }
}
```

## 小结

应用启动后置于后台，点击图标进入，启动 `Intent` 的 `Activity` 参数依然是在 `AndroidManifest` 指定的入口 `Activity`，但这时候该应用的栈（`TaskRecord`）已经存在，所以会把入口 `Activity` 所处的 `TaskRecord` 找到并置于前台，且栈顶的 `Activity` 则会先进行 `resume`，之后再对入口 `Activity` 的启动模式进行处理，例如 `launcheMode` 是 `singleTask` ，那么入口 `Activity` 之上的 `Activity` 就要出栈，而入口 `Activity` 显示
