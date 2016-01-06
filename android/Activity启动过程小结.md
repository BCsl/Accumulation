# 从Launcher启动Activity过程小结
## 步骤
* 1.`Launcher`向`ActivityManagerService`发送一个启动`Activit`组件的通信请求
* 2.`ActivityManagerService`将要启动的`Activity`的组件信息（`ActivityRecord`,所在的`TaskRecord`和`ActivityStack`）当前取得焦点的`mForceStack`的信息记录起来,向当前栈以外的栈通过其`Processrecord`保存的`ApplicationThread`发送暂停其`mResumedActivity`的命令，又对象的`ActivityThread`处理具体操作
* 3.处理完所有需要暂停的`Activity`后,通过`ActivityManagerProxy`代理对象发送消息告诉`AMS`处理暂停成功，可以继续启动组件
* 4.`AMS`发现当前启动的`Activity`组件的应用程序进程不存在，因此，它就会先启动一个新的应用程序进程
* 5.进程启动指定了新的应用程序的入口函数为`ActivityThread#main方法`，启动完向`AMS`发送一个启动完成的进程间请求（attach方法）
  - 1.在主线程创建`Looper`循环消息处理模型；
  - 2.新建`ActivityThread`，并新建`ApplicationThread`这个Binder本地对象，并调用其`attach`方法；
  - 3.调用了`AsyncTask#init()`方法,强制其静态成员`sHandler`的构造
* 6.`AMS`将第2步记录的`Activity`组件信息发送到第四步创建的应用程序进程，以便完成`Activity`组件启动

## 一些问题
* 1.`Activity`与`ActivityRecord`和`ActivityClientRecord`是如何关联？
  - `ActivityRecord`保存在AMS，`Activity`启动的时候新建，用来描述`Activity`，并保存到相应的`ActivityStack`的`TaskRecord`，
  - 当`ActivityRecord`所需要的信息都初始化好（`ActivityStack`,`TaskRecord`,`Processrecord`），`AMS`通过`ApplicationThread`调度启动`Activity`,会想创建`ActivityClientRecord`对象保存需要启动的`Activity`信息，保留了`ActivityRecord`的`Token`值，并以`Token`值保留在了`ActivityThread`的`mActivities`中，之后的一些回调就可以通过`ActivityClientRecord`来找到对应的`Activity`来回调，而`AMS`需要再次调度`Activity`得时候，通过其保存的`ActivityRecord`的`Token`就可以找到`ActivityClientRecord`
  - `Activity`的构造和`attach`都需要用到`ActivityClientRecord`记录的信息，新建的`Activity`也记录在了`ActivityClientRecord`中，而`ActivityClientRecord`保存在`ActivityThread`中，之后的生命周期的调度就可以通过`ActivityClientRecord`来找到对应的`Activity`

* 2.新进程的启动

* 3.发送到`AMS`的`ApplicationThread`是一个`Binder`本地对象？
  - 见Step29,记录在Processrecord#thread的应该是`binder`本地对象，并非代理对象，所以可能并没有经过进程间通信来处理`schedulePauseActivity`方法？__update:2016-01-03但如果这样暂停超时便没有了意义？,但是`ApplicationThread`各种的调度是通过发送消息到`ActivityThread`的`Handler`类型的成员变量`H`来处理！所以也是一个异步操作__，另外如果发送的是代理对象，那么会有多个`AppApplicationThread`的本地对象（在`ActivityThread`的构造的时候初始化）

* 4.`Activity`与`token`间的联系
  `Token`这个类继承自`IApplicationToken.Stub`,内部保存了所属的`ActivityRecord`的弱引用，`Token`用于`Ams`和`ApplicationThread`这个`Binder`本地对象直接通信，`ActivityClientRecord`以`Token`标识保存在`ActivityThread`，当`ActivityRecord`所需要的信息都初始化好（`ActivityStack`,`TaskRecord`,`Processrecord`），`AMS`通过`ApplicationThread`调度启动`Activity`,

* __5.各种启动模式的分析，生命周期的回调__

|     FLAG      |     出现位置      |
|---------------|------------------|
|`Intent.FLAG_ACTIVITY_FORWARD_RESULT`|`Step11`|
|`Intent.FLAG_ACTIVITY_NO_USER_ACTION`|`Step12`|
|`Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP`|`Step12`|
|`Intent.FLAG_ACTIVITY_NEW_TASK`|`Step12`|
|`Intent.FLAG_ACTIVITY_MULTIPLE_TASK`|`Step12`|
|`ActivityInfo.LAUNCH_SINGLE_TASK`|`Step12`|
|`ActivityInfo.LAUNCH_SINGLE_INSTANCE`|`Step12`|
|`ActivityInfo.LAUNCH_SINGLE_TASK`|`Step12`|
