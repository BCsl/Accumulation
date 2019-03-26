# [VpnService](https://developer.android.com/reference/android/net/VpnService)

[官方实例](https://android.googlesource.com/platform/development/+/master/samples/ToyVpn/src/com/example/android/toyvpn/ToyVpnConnection.java)

Android 上用来扩展实现 Vpn（Virtual Private Network） 方案的工具类

建立一个 Vpn 接口，可以配置地址和路由规则并返回一个文件描述符（实际为 ParcelFileDescriptor 类）

通过该文件描述符可以读取 Vpn 接口接受到远程服务器的 IP 数据包（FileOutputStream）和向远程服务器发送的 IP 数据包（FileInputStream）

**运行在 IP 协议之上**，应用程序可以通过 `tunnel` 和与远程服务器交换数据包

需要在 manitest 上注册

```java
<service android:name=".ExampleVpnService"
        android:permission="android.permission.BIND_VPN_SERVICE">
    <intent-filter>
        <action android:name="android.net.VpnService"/>
    </intent-filter>
</service>
```
