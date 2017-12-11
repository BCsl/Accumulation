# ActivityRecord的查找方式

当启动一个 Activity 请求发送到 Ams 后，AMS 为该 Activity 新建一个 `ActivityRecord` 对象，但 AMS可 能已经启动过这个 Activity，也就是 `ActivityStack` 内有一个相同的 `ActivityRecord` ，它记录了其所在的 `ActivityStack` 和 `TaskRecord` ，对于 launcheMode 是否为 `ActivityInfo.LAUNCH_SINGLE_INSTANCE` 有两种方式来查找

## 非 ActivityInfo.LAUNCH_SINGLE_INSTANCE，判断栈顶有效的 Activity

匹配 `TaskRecord` 栈顶的第一个非 finishing 状态的 `ActivityRecord`

```java
ActivityStackSupervisor.java

ActivityRecord findTaskLocked(ActivityRecord r) {
    //mStacks，正常只有两个，mHomeStack和mFocusedStack
    for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
        final ActivityStack stack = mStacks.get(stackNdx);
        //不是普通的 Activity，而当前栈又不是 HomeStack 就过滤，也就是非 `APPLICATION_ACTIVITY_TYPE` 类型的都存放在 HomeStack?
        //而 `APPLICATION_ACTIVITY_TYPE` 只能放到非 HomeStack
        if (!r.isApplicationActivity() && !stack.isHomeStack()) {
            continue;
        }
        final ActivityRecord ar = stack.findTaskLocked(r);
        if (ar != null) {
            return ar;
        }
    }
    return null;
}
```

接着调用了 `ActivityStack#findTaskLocked`，遍历 `TaskRecord`，找到栈顶（非finish状态不考虑），userId 相同、非`SINGLE_INSTANCE`、affinity 值相同（默认为包名）或者 `component`（包名和类名）相同的 Activity

```java
ActivityStack.java

ActivityRecord findTaskLocked(ActivityRecord target) {
    Intent intent = target.intent;
    ActivityInfo info = target.info;
    ComponentName cls = intent.getComponent();
    if (info.targetActivity != null) {
        cls = new ComponentName(info.packageName, info.targetActivity);
    }
    //userId比较
    final int userId = UserHandle.getUserId(info.applicationInfo.uid);

    for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
        final TaskRecord task = mTaskHistory.get(taskNdx);
        if (task.userId != userId) {
            continue;
        }
        //和TaskRecord的栈顶Activity状态比较
        final ActivityRecord r = task.getTopActivity(); //栈顶向下查找第一个非finish的ActivityRecord
        if (r == null || r.finishing || r.userId != userId || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {  //非 LAUNCH_SINGLE_INSTANCE
            continue;
        }
        //taskAffinity比较
        if (task.affinity != null) {
            if (task.affinity.equals(info.taskAffinity)) {
                return r;
            }
        } else if (task.intent != null && task.intent.getComponent().equals(cls)) {
            return r;
        } else if (task.affinityIntent != null  && task.affinityIntent.getComponent().equals(cls)) {
            return r;
        }
    }

    return null;
}
```

```java
TaskRecord.java

ActivityRecord getTopActivity() {
    for (int i = mActivities.size() - 1; i >= 0; --i) {
        final ActivityRecord r = mActivities.get(i);
        if (r.finishing) {
            continue;
        }
        return r;
    }
    return null;
}
```

## 带ActivityInfo.LAUNCH_SINGLE_INSTANCE，查找唯一的实例

主要是根据`ActivityInfo`来查找，对于`LAUNCH_SINGLE_INSTANCE`模式的，只会有一个唯一的`ActivityRecord`实例在特定的`TaskRecord`中，匹配包名和Activity类名和UserId

```java
ActivityStackSupervisor.java

ActivityRecord findActivityLocked(Intent intent, ActivityInfo info) {
    for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
        final ActivityRecord ar = mStacks.get(stackNdx).findActivityLocked(intent, info);
        if (ar != null) {
            return ar;
        }
    }
    return null;
}
```

```java
ActivityStack.java
/**
 * Returns the first activity (starting from the top of the stack) that
 * is the same as the given activity.  Returns null if no such activity
 * is found.
 */
ActivityRecord findActivityLocked(Intent intent, ActivityInfo info) {
    ComponentName cls = intent.getComponent();
    if (info.targetActivity != null) {
        cls = new ComponentName(info.packageName, info.targetActivity);
    }
    final int userId = UserHandle.getUserId(info.applicationInfo.uid);
    for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
        TaskRecord task = mTaskHistory.get(taskNdx);
        if (task.userId != mCurrentUser) {
            return null;
        }
        final ArrayList<ActivityRecord> activities = task.mActivities;
        for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
            ActivityRecord r = activities.get(activityNdx);
            if (!r.finishing && r.intent.getComponent().equals(cls) && r.userId == userId) {
                return r;
            }
        }
    }

    return null;
}
```

## 另外的一些操作

### topRunningNonDelayedActivityLocked

找到栈顶的`ActivityRecord`，这里的`r != notTop`仅仅是地址的判断

```java
final ActivityRecord topRunningNonDelayedActivityLocked(ActivityRecord notTop) {
    for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
        final TaskRecord task = mTaskHistory.get(taskNdx);
        final ArrayList<ActivityRecord> activities = task.mActivities;
        for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
            ActivityRecord r = activities.get(activityNdx);
            if (!r.finishing && !r.delayedResume && r != notTop && okToShow(r)) {
                return r;
            }
        }
    }
    return null;
}

//需要带FLAG_SHOW_ON_LOCK_SCREEN 或者由相同的用户启动
boolean okToShow(ActivityRecord r) {
    return r.userId == mCurrentUser || (r.info.flags & ActivityInfo.FLAG_SHOW_ON_LOCK_SCREEN) != 0;
}
```
