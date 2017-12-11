# ActivityManagerService 对不同的 launchMode 或 IntentFlag 的处理

当前启动一个 Activity，`AMS` 会新建该 Activity 在 `AMS` 中的实例 `ActivityRecord` 对象，具体在 `ActivityStackSupervisor#startActivityUncheckedLocked` 方法处理，对于使用 `Intent.FLAG_ACTIVITY_NEW_TASK` 、`ActivityInfo.LAUNCH_SINGLE_TASK` 和 `ActivityInfo.LAUNCH_SINGLE_INSTANCE`，新建之后还需要在其管理的 `ActivityStack` 中查找是否已经创建过了，如果是，那么对于不同的 `launcheMode` 或 `IntentFlag` 进行相应的处理

## 预处理

以下几种情况下，待启动的 Intent.FLAG 会带上 `Intent.FLAG_ACTIVITY_NEW_TASK`，代表可能需要在新的任务栈启动目标 `Activity`：

- (1)`sourceRecord=null` 说明要启动的 `Activity` 并不是由一个 `Activity` 的 `Context` 启动的，这时候我们总是启动一个新的 `TASK`
- (2)`Launcher` 是 `SingleInstance` 模式（或者说启动新的 `Activity` 的源 `Activity` 的 `launchMode` 为 `SingleInstance`）,因为 `SingleInstance` 模式的 `Activity` 不希望和你分享同一个 `TaskRecord`
- (3)需要启动的 `Activity` 的 `launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE|| r.launchMode ==ActivityInfo.LAUNCH_SINGLE_TASK`，因为表明了自己希望在新的 `TaskRecord` 中启动
- (4)`sourceRecord` 正在 `finish`
- (5) 从 Launcher 启动，Launcher 会帮你自带

代码

```java
if (sourceRecord == null) {
     //这种情况下是否和启动来自Service和BroadcastReceiver的context一致？
    if ((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
        launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
    }
} else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
} else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
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
```

## 处理流程

这里代码还是有点多，很难各种情况去考虑，可以考虑分情况分析，例子如下：

- 1 . 当前任务栈 A、B、C，C 启动 B，B 使用 `SingleTask` ，B 不指定 `affinity` 值
- 2 . 当前任务栈 A 和任务栈 B,C，A 启动 C，C 使用 `SingleTask`，B，C 指定 `affinity` 值，（ `TaskRecord` 置前台）返回操作，主要模拟下图

![SingleTask所在栈置前台](http://img.my.csdn.net/uploads/201108/23/0_13141106978dkE.gif)

- 3 . 当前任务栈 A，A 启动 B，B 使用 `SingleInstance`
- 4 . 接着 3，B 启动 B（ B 的 TaskRecord 不置前）

### 例子分析

#### 例子1

这个就不模拟了，结果明显的是，C 出栈，B 显示

#### 例子2

```java
//ActivityStackSupervisor

final int startActivityUncheckedLocked(ActivityRecord r,ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options){

//....
boolean addingToTask = false;

if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {

    if (r.resultTo == null) {//true

        ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE ? findTaskLocked(r) : findActivityLocked(intent, r.info);
        if (intentActivity != null) { //不为null，找到C，因为相同的affinity值
             if (r.task == null) {
                 r.task = intentActivity.task;//记录其所在的栈，相同的affinity值或component等
             }
             targetStack = intentActivity.task.stack; //对于APPLICATION_ACTIVITY_TYPE来说，一般非HomeStack
             targetStack.mLastPausedActivity = null;
             moveHomeStack(targetStack.isHomeStack());//ABC所处的栈mFocusedStack为前台栈，所以这里没什么改变
             if (intentActivity.task.intent == null) {
                 // 不为null，当前的TaskRecord又启动A的Intent启动
             }
             final ActivityStack lastStack = getLastStack();//STACK_STATE_HOME_IN_BACK，mFocusStack
             ActivityRecord curTop = lastStack == null ? null : lastStack.topRunningNonDelayedActivityLocked(notTop);//A

             if (curTop != null && (curTop.task != intentActivity.task ||curTop.task != lastStack.topTask())) {
                //当前A所处的TaskRecord处于前台，启动的C的TaskRecord为后台，所以需要置前
                r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                if (sourceRecord == null || (sourceStack.topActivity() != null && sourceStack.topActivity().task == sourceRecord.task)) {
                          movedHome = true;
                          targetStack.moveTaskToFrontLocked(intentActivity.task, r, options);//把C的TaskRecord放到顶层，走一遍resumeTopActivityLocked，并把当前的mResumedActivity A给暂停掉，当前的C还并没走到Resume的逻辑，所以继续
                          //搭配FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME使用，那么按返回键会返回HOME，不管你当前的
                          if ((launchFlags &
                            (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                              intentActivity.task.mOnTopOfHome = true;
                            }
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
                 }
                 //...
                 return ActivityManager.START_RETURN_INTENT_TO_CALLER;
             }
             //Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK的搭配使用
             if ((launchFlags & (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) {
                 reuseTask = intentActivity.task;
                //finish栈内所有Activity，所以目标Activity肯定是需要新建了，但TaskRecord复用
                 reuseTask.performClearTaskLocked();
                 reuseTask.setIntent(r.intent, r.info);
             } else if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                     || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                     || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                //launchMode为SingleTask或者SingleInstance，或者IntentFlag带FLAG_ACTIVITY_CLEAR_TOP标志，清除当前activity以上的activity
                //启动Activity对应的`ActivityRecord`以上的Activity都会被finish掉，并返回该`ActivityRecord`实例，如果当前实例是默认模式，且没带`FLAG_ACTIVITY_SINGLE_TOP`标识，那么当前的实例也会finish，并返回NULL
                 ActivityRecord top =intentActivity.task.performClearTaskLocked(r, launchFlags);
                 if (top != null) {
                   //当前实例以上的activity被finish掉
                     if (top.frontOfTask) {
                         top.task.setIntent(r.intent, r.info);
                     }
                     //触发onNewIntent
                     top.deliverNewIntentLocked(callingUid, r.intent);
                 } else {
                     //当前实例以上的activity和自己都被finish掉或者栈中找不到要启动的Activity，所以需要在新的任务中启动
                     addingToTask = true;
                     // Now pretend like this activity is being started
                     // by the top of its task, so it is put in the
                     // right place.
                     sourceRecord = intentActivity;
                 }
             }
             //.....
             if (!addingToTask && reuseTask == null) {
                 // We didn't do anything...  but it was needed (a.k.a., client
                 // don't use that intent!)  And for paranoia, make
                 // sure we have correctly resumed the top activity.
                 if (doResume) {
                     targetStack.resumeTopActivityLocked(null, options);//按照流程，之前在moveTaskToFrontLocked方法内部已经调用过resumeTopActivityLocked方法，等Activity Pause后，就会使获得焦点的栈顶Activity启动，按照注释，这里再调用只是为了确保之前的逻辑是否正确？实际上在这个例子流程中并没什么用，但在例子4中，却是不可缺少
                 } else {
                     ActivityOptions.abort(options);
                 }
                 //...
                 return ActivityManager.START_TASK_TO_FRONT;
             }
         }// Start795
         //...
    }
}
//...
```

#### 例子3

当前任务栈 A，A 启动 B，B 使用 `SingleInstance` ，会自动带上 `Intent.FLAG_ACTIVITY_NEW_TASK`

```java
final int startActivityUncheckedLocked(ActivityRecord r,ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options){

//....
boolean addingToTask = false;

if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
    // If bring to front is requested, and no result is requested, and we can find a task that was started with this same
    // component, then instead of launching bring that one to the front.
    if (r.resultTo == null) {//true
        //SingleInstance 会查找唯一的实例（类名、包名、启动用户），这里为null
        ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE ? findTaskLocked(r) : findActivityLocked(intent, r.info);
        if (intentActivity != null) { //为null
             //...
         }// Start795
         //上段代码的逻辑是看一下，当前在堆栈是否有要启动的Activity的ActivityRecord，如果有那么对于不同的launchMode就会要进行不同的响应
          if (r.packageName != null) {
              ActivityStack topStack = getFocusedStack();//A所在的Stack
              //启动的用户不一样，notTop为null，一般为应用内启动
              ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);
              if (top != null && r.resultTo == null) {
                //...
              }

          } else {
            //...
          }

          boolean newTask = false;
          boolean keepCurTransition = false;

          // Should this be considered a new task?
          //addingToTask为false，因为前面的intentActivity为null
          if (r.resultTo == null && !addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {//true
              targetStack = adjustStackFocus(r);//mFocusStack不为null，得到mFocusStack
              moveHomeStack(targetStack.isHomeStack());//状态修改为STACK_STATE_HOME_TO_BACK
              if (reuseTask == null) { //reuseTask为null，没有可以利用的已经存在的TaskRecord，没哟指定affinity值，所以和A所处的affinity值一样
                 //创建新的TaskRecord，如果已经记录在其他TASKRECORD，移除，记录新的TaskRecord，并把TaskRecord插入到mTaskHistory的顶部，但TaskRecord并未记录该ActivityRecord
                  r.setTask(targetStack.createTaskRecord(getNextTaskId(), r.info, intent, true),null, true);
                  //Activity会启动在新的栈中
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
            //....
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
}
```

结论，使用 `SingleInstance` 启动的 Activity，会查找全局唯一的 `ActivityRecord` 实例，不会和其他 Activity 共享 `TaskRecord`

#### 例子4

接着 3，B 启动 B，或者 B 启动 A，A 再启动 B

```java
//ActivityStackSupervisor

final int startActivityUncheckedLocked(ActivityRecord r,ActivityRecord sourceRecord, int startFlags, boolean doResume, Bundle options){

//....
boolean addingToTask = false;

if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
        || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
    if (r.resultTo == null) {//true
        //会找到实例B
        ActivityRecord intentActivity = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE ? findTaskLocked(r) : findActivityLocked(intent, r.info);
        if (intentActivity != null) { //不为null
             if (r.task == null) {
                 r.task = intentActivity.task;
             }
             targetStack = intentActivity.task.stack; //对于APPLICATION_ACTIVITY_TYPE来说，一般非HomeStack
             targetStack.mLastPausedActivity = null;
             moveHomeStack(targetStack.isHomeStack());//ABC所处的栈mFocusedStack为前台栈，所以这里没什么改变
             if (intentActivity.task.intent == null) {
                 // 不为null，当前的TaskRecord又启动A的Intent启动
             }
             final ActivityStack lastStack = getLastStack();//B所属的Stack，mFocusStack
             ActivityRecord curTop = lastStack == null ? null : lastStack.topRunningNonDelayedActivityLocked(notTop);//找到B自己的实例

             if (curTop != null && (curTop.task != intentActivity.task ||curTop.task != lastStack.topTask())) {
                //...
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
                 }
                 //...
                 return ActivityManager.START_RETURN_INTENT_TO_CALLER;
             }
             //还是可以和Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK搭配使用的
             if ((launchFlags & (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) == (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK)) {
                 reuseTask = intentActivity.task; //记录继续使用的TaskRecord，下一步会清空栈内的Activity
                 reuseTask.performClearTaskLocked();  //finish栈内所有Activity，所以目标Activity肯定是需要新建了
                 reuseTask.setIntent(r.intent, r.info);
             } else if ((launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0
                     || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                     || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                //launchMode为SingleTask或者SingleInstance，或者IntentFlag带FLAG_ACTIVITY_CLEAR_TOP标志，清除当前activity以上的activity
                //启动Activity对应的`ActivityRecord`以上的Activity都会被finish掉，并返回该`ActivityRecord`实例，如果当前实例是默认模式，且没带`FLAG_ACTIVITY_SINGLE_TOP`标识，那么当前的实例也会finish，并返回NULL
                 ActivityRecord top =intentActivity.task.performClearTaskLocked(r, launchFlags);
                 if (top != null) {
                   //当前实例以上的activity被finish掉
                     if (top.frontOfTask) {
                         top.task.setIntent(r.intent, r.info);
                     }
                     //触发onNewIntent
                     top.deliverNewIntentLocked(callingUid, r.intent);
                 } else {
                     //当前实例以上的activity和自己都被finish掉或者栈中找不到要启动的Activity，所以需要在新的任务中启动
                     addingToTask = true;
                     // Now pretend like this activity is being started
                     // by the top of its task, so it is put in the
                     // right place.
                     sourceRecord = intentActivity;
                 }
             }
             //......

             if (!addingToTask && reuseTask == null) {
               if (doResume) {
                 targetStack.resumeTopActivityLocked(null, options);
               } else {
                 ActivityOptions.abort(options);
               }
               new RuntimeException("here").fillInStackTrace());
               return ActivityManager.START_TASK_TO_FRONT;
             }

         }
         //.....
    }
}
//...
```

结论，SingleInstance 模式只能和 `Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK` 搭配使用，其他的 `IntentFlag` 会无效，如果和 `Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK` 搭配使用，栈内所有 `Activity` 都会被清掉，且 finish，所以需要重新启动 Activity 的 `onCreate-onStart-onResume`，但是 `TaskRecord` 可以复用，如果不使用，走的流程将是 `onNewIntent-onStart(视情况，有没有走onStop)-onResume`，且注意到当前任务栈并没有置为前台

## 小结

默认模式：不带 `Intent.FLAG_ACTIVITY_NEW_TASK` 情况下，即使指定 `taskAffinity`，但由哪个 `TaskRecord` 栈的 `Activity` 启动，就会存放在哪个 `TaskRecord`，且可以启动多个实例，如果指定 `Intent.FLAG_ACTIVITY_NEW_TASK`，那么就在包名或者指定的 `taskAffinity` 中启动

`Intent.FLAG_ACTIVITY_CLEAR_TASK`:`SingleTask`和`SingleInstance`都可以默认带`Intent.FLAG_ACTIVITY_NEW_TASK`，如果再指定`Intent.FLAG_ACTIVITY_CLEAR_TASK`，且实例已经存在了，启动该`Activity`，重走`onCreate`流程

`SingleTask` : 它并不像官方文档描述的一样：`The system creates a new task and instantiates the activity at the root of the new task`，而是在跟它有相同 `affinity` 的任务中启动(默认为包名)，并且位于这个任务的堆栈顶端，并把它之上的 `Activity` 全部出栈并 finish；如果前台 `TaskRecord` 的 `Activity` 启动后台 `TaskRecord` 的使用的 `SingleTask` 的 `Activity`，如果后台任务栈中已经有该 `Activity`，其所在的 `TaskRecord` 会置前；只要实例已经存在了，启动该 `Activity`，且没带 `Intent.FLAG_ACTIVITY_NEW_TASK`，会触发 `onNewIntent` 回调

`SingleInstance`：全局单实例，不和别的 Activity 分享 TaskRecord，可以搭配 `Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_CLEAR_TASK` 使用；如果前台 `Activity` 启动后台任务的使用的 `SingleInstance` 的 `Activity`，如果后台任务栈中已经有该 `Activity`，并不像 `singleTask` 一样会改变当前栈的前后顺序（因为自己独享，不用考虑栈底下其他 Activity），会触发 `onNewIntent` 回调；只要实例已经存在了，启动该`Activity`，且没带 `Intent.FLAG_ACTIVITY_NEW_TASK`，会触发 `onNewIntent` 回调。适用场景：可以在应用外弹出 Dialog 样式的 Activity 的时候使用或全屏 Activity 广告

## 更多

- [解开Android应用程序组件Activity的"singleTask"之谜](http://blog.csdn.net/luoshengyang/article/details/6714543)

- [Activity的launchMode详细分析](http://blog.csdn.net/yuanzeyao/article/details/42809153)
