# Notification机制浅析（基于SDK23）

最近在使用自定义内容的`Notification`的时候遇到一个bug，`RemoteServiceException: Bad notification posted from package xxx, Couldn't expand RemoteViews for....`可把我玩坏了，而且不知是不是Instant Run的问题有些情况下可以正确运行，有时又不可以了，而且都是卸载安装，搞了大半天，google上也没找到合适的解决方案，后来想起公司分享提到过这个坑，解决办法就是每次更新`Notification`的时候都新建`RemoteView`，问题也的确是解决了。。。后来自己新建一个项目DEMO想验证下到底哪里有问题，进行类似操作也没出现问题。。实在是一脸懵逼，既然这样，就`Read the fucking source code`吧，顺便简单了解下Notification的机制

如果用SDK25来编译，那么Notification#contentView会显示"deprecated"，因为可能为null

## 先来说原因和结论

因为文章会涉及`Binder`机制，并不是每个人都了解，如果你只需要解决`Couldn't expand RemoteViews for...`，可以直接看原因和解决方案

### 原因

- `Notification#contentView`为null

- `Notification#[contentView|bigContentView|headsUpContentView].apply()`异常，contentView、bigContentView、headsUpContentView都是`RemoteView`，所以是`RemoteViews#apply`方法运行异常

### 解决方案

首先是`Notification#contentView`不能为null，了解下`contentView`是怎么赋值/创建的，下面是平时用于构建通知的代码：

```java
RemoteViews contentView = new RemoteViews(getPackageName(), R.layout.notify); //取决于是否需要自定义显示内容
contentView.setTextViewText(R.id.notify_title, getString(R.string.title));
//...
mNotification = new NotificationCompat.Builder(getApplicationContext())
        .setContent(contentView) //setTitle...等
        .setTicker("")
        .setSmallIcon(getNotificationIcon())
        .setWhen(System.currentTimeMillis())
        .setPriority(NotificationCompat.PRIORITY_MAX)
        .setOngoing(true)
        .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
        .setContentIntent(PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT))
        .setCategory(NotificationCompat.CATEGORY_PROMO)
        .build();
mNotificationManager.notify(MY_ID, mNotification);
```

以下是`NotificationCompat.Builder#build()`这个方法运行的流程图

![Notification contentView的创建]()

其中`makeContentView`方法，判断`mContentView`是否不为null，是就直接返回，这个`mContentView`是通过`NotificationCompat.Builder#setContent()`赋值的

```java
Notification.java

private RemoteViews makeContentView() {
    if (mContentView != null) {
        return mContentView;
    } else {
        return applyStandardTemplate(getBaseLayoutResource());
    }
}

即使没有调用`NotificationCompat.Builder#setContent()`方法，最后还是会创建一个新的指定样式的`RemoteViews`并返回

/**
 * @param hasProgress whether the progress bar should be shown and set
 */
private RemoteViews applyStandardTemplate(int resId, boolean hasProgress) {
    RemoteViews contentView = new BuilderRemoteViews(mContext.getApplicationInfo(), resId);

    resetStandardTemplate(contentView);
    //....
    if (mContentTitle != null) {
        contentView.setTextViewText(R.id.title, processLegacyText(mContentTitle));
    }
    if (mContentText != null) {
        contentView.setTextViewText(R.id.text, processLegacyText(mContentText));
        showLine3 = true;
    }
    //....
    return contentView;
}
```

所以可以看到，**`contentView`一般不会为null，所以这没什么需要注意的**

再来看，`RemoteViews#apply`方法运行异常，从上面可以知道，`Notification`的内容区域是用`RemoteViews`来"填充"的，但`RemoteViews`并不是一个View，不继承自`View`，只是保存了需要显示的布局文件ID`mLayoutId`和如何解析并初始化成`View`对象等，具体看`RemoteViews#apply()`方法

```java
public View apply(Context context, ViewGroup parent, OnClickHandler handler) {
    RemoteViews rvToApply = getRemoteViewsToApply(context);//一般为this，除非你指定横屏或竖屏时候的样式
    View result;
    final Context contextForResources = getContextForResources(context);
    Context inflationContext = new ContextWrapper(context) {
      //...
    };
    LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    inflater = inflater.cloneInContext(inflationContext);
    inflater.setFilter(this);
    result = inflater.inflate(rvToApply.getLayoutId(), parent, false);
    rvToApply.performApply(result, parent, handler);

    return result;
}
```

接着调用`RemoteViews#performApply()`，遍历`Action`列表并执行其`Action#apply`方法

```java
private void performApply(View v, ViewGroup parent, OnClickHandler handler) {
    if (mActions != null) {
        handler = handler == null ? DEFAULT_ON_CLICK_HANDLER : handler;
        final int count = mActions.size();
        for (int i = 0; i < count; i++) {
            Action a = mActions.get(i);
            a.apply(v, parent, handler);
        }
    }
}
```

这个`mActions`是什么时候构造并初始化的？其实际类型是`ReflectionAction`，当我们调用`RemoteViews#setXXX`来初始化其控件的内容时就会新建一个`ReflectionAction`并添加到`mActions`

如:

```java
contentView.setTextViewText(R.id.notify_title, getString(R.string.title));
```

添加一个`ReflectionAction`

``` `java
public void setTextViewText(int viewId, CharSequence text) {
    setCharSequence(viewId, "setText", text);
}

//帮我们预设为`setText`方法名
public void setCharSequence(int viewId, String methodName, CharSequence value) {
    addAction(new ReflectionAction(viewId, methodName, ReflectionAction.CHAR_SEQUENCE, value));
}
```

最后看`ReflectionAction`的实现，就是通过反射来设值

```java
@Override
public void apply(View root, ViewGroup rootParent, OnClickHandler handler) {
    final View view = root.findViewById(viewId);
    if (view == null) return;

    Class<?> param = getParameterType();
    if (param == null) {
        throw new ActionException("bad type: " + this.type);
    }

    try {
        getMethod(view, this.methodName, param).invoke(view, wrapArg(this.value));
    } catch (ActionException e) {
        throw e;
    } catch (Exception ex) {
        throw new ActionException(ex);
    }
}
```

所以为了正确运行，**必须保证你对自己的`RemoteView`某个`viewId`所代表的`View`的初始化用合适`RemoteViews#setter`的方法**，另外`RemoteViews`支持的控件也是有限制的，可以看[官方文档](https://developer.android.com/guide/topics/appwidgets/index.html)，所以文章开始所说的没次更新通知的时候新建`RemoteView`在我目前看来并没什么依据

## Binder类结构和流程

Notification的机制简单的分位两大角色，`NotificationManager`和`NotificationListener`，前一个用来发布、更新、管理通知，后一个用于监听通知的变化，典型的观察者模式

![NotificationManager和NotificationListener的类图]()

## NotificationListener的注册流程

![NotificationListener的注册流程]()

`NotificationListener`是SDK18+以上才有的，因为SDK18开始提供了[NotificationListenerService](https://developer.android.com/reference/android/service/notification/NotificationListenerService.html)可以用来读取应用通知，其实现就是通过`NotificationListener`来实现的，其内部有一个`INotificationListener.Stub`类型的Binder本地对象，把其Binder代理对象注册到`NotificationManager`，监听通知的变更，而系统通知栏改也为了用`NotificationListenerService`来实现，通知栏启动的时候启动其内部的`NotificationListenerService`并注册，注册的Binder代理对象会以`ManagerServiceInfo`的形式记录在`NotificationManagerService`的`NotificationListener`列表，并且会在`NotificationListener`的Binder对象销毁的时候自动从列表中移除

## Notification的发布流程

![Notificaion的发布流程]()

用户通过`NotificationManager.notify()`方法发送、更新一个通知，通过Binder进行通信，最终由`NotificationManagerService`来统一处理，每个`Notificaion`都会在`NotificationManagerService`中已`NotificationRecord`的形式为记录，`NotificationManagerService`并没有UI相关的逻辑

## 参考

[NotificationManagerService笔记](http://blog.csdn.net/guoqifa29/article/details/44133905)

[Android NotificationListenerService原理简介](http://blog.csdn.net/yueqian_scut/article/details/51417547)

[Heads-Up Notification](http://blog.csdn.net/wds1181977/article/details/49783787)

[Android dev Notification](https://developer.android.com/guide/topics/ui/notifiers/notifications.html)
