---
title: Android Emulator 下载失败修复
author: nhsoft.lsd
date: 2025-11-28
categories: [Java]
pin: false
---

日志
```
Packages to install: - Android Emulator (emulator)


Preparing "Install Android Emulator v.36.2.12".
Downloading https://dl.google.com/android/repository/emulator-darwin_aarch64-14214601.zip
An error occurred while preparing SDK package Android Emulator: Not in GZIP format.
"Install Android Emulator v.36.2.12" failed.
Failed packages:
- Android Emulator (emulator)
```

Android Emulator 下载失败修复
```shell
rm -rf ~/Library/Android/sdk/.downloadIntermediates/
```

执行完以后重新下载

# Android 模拟器设置翻墙

```shell
emulator -avd Pixel9a -http-proxy http://10.0.2.2:7890

// 执行后如果执行一下方法返回是null
adb shell settings get global http_proxy

// 执行以下命令设置翻墙
adb shell settings put global http_proxy 10.0.2.2:7897
```






![weixin.png](/assets/img/nhsoft_lsd/weixin.png)


<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
