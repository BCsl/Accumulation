# Android广播机制小结

## 无序广播
无序广播只能通过动态的方式注册

### 动态注册
  注册过程是把`BroadcastReceiver`构造成`IIntentReceiver`类型的`Binder`本地对象并与`IntentFilter`参数通过`IPC`发送到`AMS`的过程，在`AMS`中以`KEY`为`IIntentReceiver`保存在一个`VALUE`为`ReceiverList`的`HashMap`，`ReceiverList`同样持有了宿主`IIntentReceiver`和当前进程块的引用（`ReceiverList`继承自`ArrayList`（相同的`BroadcastReceiver`类型的实例，`KEY`都默认相同，所以`Value`保存为`ArrayList`的原因是？可能重复注册两个相同类型`BroadcastReceiver`实例？），存放`BroadcastFilter`类型的`item`，而`BroadcastFilter`），在把广播接收者`IIntentReceiver`保存起来后，还需要和`IntentFilter`关联，所以还创建一个`BroadcastFilter`来把广播接收器列表`ReceiverList`和`IntentFilter`关联起来(`BroadcastFilter`继承自`IntentFilter`,只不过多了记录宿主`ReceiverList`和一些权限标识的变量)，还记得`ReceiverList`是存放`BroadcastFilter`类型的`ArrayList`，所以这个新建的`BroadcastFilter`，也需要保存在`ReceiverList`中，最后保存在`AMS`中`IntentResolver`类型的成员变量`mReceiverResolver`中去

### 发送无序广播
  通过调用`sendBroadcast`方法，都有可能对广播进行新的一轮调度，`AMS`内有两种广播队列，分别是前台广播队列`mFgBroadcastQueue`保存带`FLAG_RECEIVER_FOREGROUND`的广播，和后台广播队列`mBgBroadcastQueue`，每个广播队列内部保存了两个列表有序的广播记录和无序的广播，记录类型都是`BroadcastRecord`，`BroadcastRecord`对象记录了接收者队列（类型：动态注册的接受者`BroadcastFilter`类型和静态注册的接收者是`ResolveInfo`类型）。`AMS`所做的就是通过发送过来的广播`Intent`在在动态注册时候记录的`IntentResolver`类型的`mReceiverResolver`队列中找到匹配成功的`BroadcastFilter`队列（这过程涉及到`Intent`匹配的机制，IntentResolver#queryIntent方法），构建这样的一个`BroadcastRecord`，并添加到特定的广播队列，根据需要调用队列的调度方法`scheduleBroadcastsLocked`。

### 无序广播调度
  无序广播都是添加到无序广播队列中，由发送的过程知道，`AMS`根据发来的广播`Intent`已经找到了相匹配的过滤器`BroadcastFilter`（记录在了`BroadcastRecord`），所以在调度的过程中，只需要把无序广播队列中的接收者取出，并通知回调

### 发送有序广播

### 有序广播调度

## 静态注册

## 粘性广播
