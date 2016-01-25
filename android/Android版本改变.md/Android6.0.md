# Android6.0（api23）改变
[Android6.0-Changes](http://developer.android.com/intl/zh-cn/about/versions/marshmallow/android-6.0-changes.html#behavior-power)

## 新权限检测模型

## 睡眠和待机（新的电池优化模型）

## HttpClient的API移除
解决方案，在应用的`build.gradle`文件添加
```java
android {
  useLibrary 'org.apache.http.legacy'
}

```

## BoringSSL

## 硬件标识符的获取
以前的一些获取硬件标识的方法如`WifiInfo.getMacAddress()`，`BluetoothAdapter.getAddress()`会返回常量`02:00:00:00:00:00`

## 通知
`Notification.setLatestEventInfo()`方法的移除，使用`Notification.Builder`

## AudioManager改变
设置音量和静音方法的改变

## 文字选择

## Keystore
不再支持`DES`

## Wifi和网络连接
