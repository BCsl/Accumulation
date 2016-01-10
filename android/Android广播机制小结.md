# Android广播机制小结

## 注册
### 动态注册
  注册过程是把`BroadcastReceiver`构造成`IIntentReceiver`类型的`Binder`本地对象和`IntentFilter`参数通过`IPC`发送到`AMS`的过程，在`AMS`中以`KEY`为`IIntentReceiver`保存在一个`VALUE`为`ReceiverList`的`HashMap`，`ReceiverList`同样持有了宿主`IIntentReceiver`和当前进程块的引用（`ReceiverList`继承自`ArrayList`，存放`BroadcastFilter`类型的`item`），在把广播接收者`IIntentReceiver`保存起来后，还需要和`IntentFilter`关联，所以还创建一个`BroadcastFilter`来把广播接收器列表`ReceiverList`和`IntentFilter`关联起来(`BroadcastFilter`继承自`IntentFilter`,只不过多了记录宿主`ReceiverList`和一些权限标识的变量)，还记得`ReceiverList`是存放`BroadcastFilter`类型的`ArrayList`，所以这个新建的`BroadcastFilter`，也需要保存在`ReceiverList`中，然后保存在`AMS`中的成员变量`mReceiverResolver`中去

## 静态注册

## 发送广播和调度
通过调用`sendBroadcast`方法，都有可能对广播进行新的一轮调度，
### 有序广播


### 无序广播
