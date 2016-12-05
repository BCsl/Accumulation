# 应用提取.apk反编译分析小结

功能点 Apk提取、分享、禁用

## 应用列表显示(第三方/系统应用的区分)

通过执行`pm list packages [-e] [-d] [-3]`命令获取包名列表，根据包名即可得到`PackageInfo`，该命令不需要root，另外也可以使用SDK API来获取，获取到的是`ApplicationInfo`列表，但似乎并不能直接区分第三方和系统的获取，需要根据`ApplicationInfo`提供的一些参数来判断

```java
List<ApplicationInfo> apps = getPackageManager().getInstalledApplications(PackageManager.GET_SIGNATURES);
for (ApplicationInfo info : apps) {  
        if ((info.flags & ApplicationInfo.FLAG_SYSTEM) == 0) {  
          //...
        }
      }
```

## 应用APK存档路径

使用命令`pm path packageName`，同样的，SDK也提供了API来获取，但是两个获取到的路径似乎不太一样，不影响

```java
ApplicationInfo app = getContext().getApplicationContext().getApplicationInfo();
String originFilePath = app.sourceDir;
```

## 应用禁用

开始是对这个功能感兴趣，实际上很简单，通过adb的`pm disable/enable packageName`命令即可，需要root，项目使用了[libsuperuser](https://github.com/Chainfire/libsuperuser)库来简化root权限的处理和命令的执行

## 用到的一些ADB命令附录

- `pm disable/enable packageName` 禁用或者使用某个App

- `pm uninstall packageName` 卸载APK

- `pm path packageName` 列出某个APK存档文件路劲

- `pm list packages [-e] [-d] [-3]` ，打印出系统安装的应用包名，其中-3表示第三方，-e列出没被禁用的包名，-d列出禁用的包名，[更多](http://www.cnblogs.com/08shiyan/p/3480048.html)

## 第三方库

- [libsuperuser](https://github.com/Chainfire/libsuperuser)
