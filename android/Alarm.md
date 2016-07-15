# 执行重复的操作

<http://hukai.me/android-training-course-in-chinese/background-jobs/scheduling/alarms.html>

## 概述

`Alarms`基于`AlarmManager.class`，提供了一种在应用生命周期外的基于时间的操作。你可以使用闹钟初始化一个长时间的操作，例如每天开启一次后台服务，下载当日的天气预报。

## 特性

- 允许你通过预设时间或者设定某个时间间隔，来处理Intent；
- 你可以将它与BroadcastReceiver相结合，来启动服务并执行其他操作
- 可在应用范围之外执行，所以你可以在你的应用没有运行或设备处于睡眠状态的情况下，使用它来触发事件或行为
- 帮助你的应用最小化资源需求，你可以使用闹钟调度你的任务，来替代计时器或者长时间持续运行的后台服务

## 注意

对于那些需要确保在应用生命周期内发生的定时操作，可以使用闹钟替代使用Handler结合Timer与Thread的方法。因为它可以让Android系统更好地统筹系统资源。

## 权衡利弊

重复闹钟的机制比较简单，没有太多的灵活性。它对于你的应用来说或许不是一种最好的选择，特别是当你想要触发网络操作的时候。设计不佳的闹钟会导致电量快速耗尽，而且会对服务端产生巨大的负荷。

### 最佳实践

在设计重复闹钟过程中，你所做出的每一个决定都有可能影响到你的应用将会如何使用系统资源。例如，我们假想一个会从服务器同步数据的应用。同步操作基于的是系统时钟时间，具体来说，每一个应用的实例会在下午十一点整进行同步，巨大的服务器负荷会导致服务器响应时间变长，甚至拒绝服务。因此在我们使用闹钟时，请牢记下面的最佳实践建议

- 对任何由重复闹钟触发的网络请求添加一定的随机性 1.在闹钟触发时做一些本地任务。"本地任务"指的是任何不需要访问服务器或者从服务器获取数据的任务 2.同时对于那些包含有网络请求的闹钟，在调度时机上增加一些随机性
- 尽量让你的闹钟频率最小
- 如果不是必要的情况，不要唤醒设备（这一点与闹钟的类型有关）
- 触发闹钟的时间不必过度精确，尽量使用`setInexactRepeating()`方法替代`setRepeating()`方法。当你使用setInexactRepeating()方法时，Android系统会集中多个应用的重复闹钟同步请求，并一起触发它们。这可以减少系统将设备唤醒的总次数，以此减少电量消耗。从Android 4.4（API19）开始，所有的重复闹钟都将是非精确型的。注意虽然`setInexactRepeating()`是`setRepeating()`的改进版本，它依然可能会导致每一个应用的实例在某一时间段内同时访问服务器，造成服务器负荷过重。因此如之前所述，对于网络请求，我们需要为闹钟的触发时机增加随机性。
- 尽量避免让闹钟基于时钟时间

想要在某一个精确时刻触发重复闹钟是比较困难的。我们应该尽可能使用`ELAPSED_REALTIME`。

## 设置重复执行的闹钟

- 设置闹钟类型
- 设置触发时间。如果触发时间是过去的某个时间点，闹钟会立即被触发
- 闹钟间隔时间。例如，一天一次，每小时一次，每五秒一次，等等
- 在闹钟被触发时才被发出的`Pending Intent`。如果你为同一个`Pending Intent`设置在另一个闹钟，那么它会将第一个闹钟覆盖。

### 选择提醒类型

- `ELAPSED_REALTIME`:从设备启动之后开始算起，度过了某一段特定时间后，激活`Pending Intent`，但不会唤醒设备。其中设备睡眠的时间也会包含在内。
- `ELAPSED_REALTIME_WAKEUP`:从设备启动之后开始算起，度过了某一段特定时间后唤醒设备
- `RTC`:在某一个特定时刻激活`Pending Intent`，但不会唤醒设备。
- `RTC_WAKEUP`:在某一个特定时刻唤醒设备并激活`Pending Intent`

### [EXAMPLE](http://developer.android.com/intl/zh-cn/training/scheduling/alarms.html)

### 取消闹钟

你可能希望在应用中添加取消闹钟的功能。要取消闹钟，可以调用`AlarmManager`的`cancel()`方法，并把你不想激活的`PendingIntent`传递进去，例如：

```java
// If the alarm has been set, cancel it.
if (alarmMgr!= null) {
    alarmMgr.cancel(alarmIntent);
}
```

### 开机启动闹钟

默认情况下，所有的闹钟会在设备关闭时被取消。要防止闹钟被取消，你可以让你的应用在用户重启设备后自动重启一个重复闹钟。这样可以让`AlarmManager`继续执行它的工作，且不需要用户手动重启闹钟

## 步骤

### 监听启动广播

```java
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
```

### 接受广播

```java
public class SampleBootReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent.getAction().equals("android.intent.action.BOOT_COMPLETED")) {
            // Set the alarm here.
        }
    }
}
```

### 添加到清单文件

```java
<receiver android:name=".SampleBootReceiver"
        android:enabled="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"></action>
    </intent-filter>
</receiver>
```

注意`Manifest`文件中，对接收器设置了`android:enabled="false"`属性。这意味着除非应用显式地启用它，不然该接收器将不被调用。这可以防止接收器被不必要地调用。你可以像下面这样启动接收器（比如用户设置了一个闹钟）：

```java
ComponentName receiver = new ComponentName(context, SampleBootReceiver.class);
PackageManager pm = context.getPackageManager();

pm.setComponentEnabledSetting(receiver,
        PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
        PackageManager.DONT_KILL_APP);
```

一旦你像上面那样启动了接收器，它将一直保持启动状态，即使用户重启了设备也不例外。换句话说，通过代码设置的启用配置将会覆盖掉Manifest文件中的现有配置，即使重启也不例外。接收器将保持启动状态，直到你的应用将其禁用。你可以像下面这样禁用接收器（比如用户取消了一个闹钟）：

```java
ComponentName receiver = new ComponentName(context, SampleBootReceiver.class);
PackageManager pm = context.getPackageManager();

pm.setComponentEnabledSetting(receiver,PackageManager.COMPONENT_ENABLED_STATE_DISABLED,PackageManager.DONT_KILL_APP);
```
