# condom

[condom](https://github.com/oasisfeng/condom)，一个超轻超薄的Android工具库，阻止三方SDK中常见的严重影响用户体验的『链式唤醒』行为。（对应用自身的功能无影响）

## 原理

实现原理实际很简单，拦截 `Context#startServiceXXX` 和 `Context#sendBroadcastXXX` 系列方法，具体的实现是在原本的 `Context` 进行一层装饰，`CondomContext` 实际类型为 `ContextWarpper`

```java
public static
@CheckReturnValue
CondomContext wrap(final Context base, final @Nullable String tag) {
    //..
    final Context app_context = base.getApplicationContext();
    final boolean debuggable = (base.getApplicationInfo().flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    if (app_context instanceof Application) {    // The application context is indeed an Application, this should be preserved semantically.
        final Application app = (Application) app_context;
        final CondomApplication condom_app = new CondomApplication(app, tag, debuggable);
        final CondomContext condom_context = new CondomContext(base, condom_app, tag, debuggable);
        condom_app.attachBaseContext(base == app_context ? condom_context : new CondomContext(app, app, tag, debuggable));
        return condom_context;
    } else
        return new CondomContext(base, base == app_context ? null : new CondomContext(app_context, app_context, tag, debuggable), tag, debuggable);
}
```

装饰的 `Context` 对 `Context#startServiceXXX` 和 `Context#sendBroadcastXXX` 系列方法做了包名过滤操作，如 `startService`

```java
@Override
public ComponentName startService(final Intent intent) {
    return proceed(OutboundType.START_SERVICE, intent, null, new WrappedValueProcedure<ComponentName>() {
        @Override
        public ComponentName proceed(final Intent intent) {
            return CondomContext.super.startService(intent);
        }
    });
}
```

组件相关的启动操作最后由 `CondomContext#proceed` 处理，检测包名，

```java
private
@CheckReturnValue
<T> T proceed(final OutboundType type, final Intent intent, final @Nullable T negative_value, final WrappedValueProcedure<T> procedure) {
    final ComponentName component = intent.getComponent();
    final String target_pkg = component != null ? component.getPackageName() : intent.getPackage();
    if (getPackageName().equals(target_pkg))
        return procedure.proceed(intent);        // Self-targeting request is allowed unconditionally
    final int original_flags = adjustIntentFlags(intent);
    if (shouldBlockRequestTarget(type, target_pkg)) {
        if (DEBUG) Log.w(TAG, "Blocked outbound " + type + ": " + intent);
        return negative_value;
    }
    try {
        return procedure.proceed(intent);
    } finally {
        intent.setFlags(original_flags);
    }
}
```

其中 `shouldBlockRequestTarget` 调用 `OutboundJudge` 接口检测是否过滤掉这次操作，这个接口需要自己实现

```java
public interface OutboundJudge {

    boolean shouldAllow(OutboundType type, String target_pkg);
}
```

`adjustIntentFlags` 方法则是对 `flag` 做一些处理

```java
private int adjustIntentFlags(final Intent intent) {
    final int original_flags = intent.getFlags();
    if (mDryRun)
        return original_flags;

    if (mExcludeBackgroundPackages)
        intent.addFlags(SDK_INT >= N ? FLAG_RECEIVER_EXCLUDE_BACKGROUND : Intent.FLAG_RECEIVER_REGISTERED_ONLY);

    if (SDK_INT >= HONEYCOMB_MR1 && mExcludeStoppedPackages)
        intent.setFlags((intent.getFlags() & ~Intent.FLAG_INCLUDE_STOPPED_PACKAGES) | Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
    return original_flags;
}
```
