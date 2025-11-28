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
