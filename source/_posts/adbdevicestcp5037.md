---
title: 连接genymotion模拟器错误 cannot bind 'tcp:5037'
date: 2017-10-18 16:50:11
tags: [genymotion,Android]
---

Android 模拟器常用命令

查看当前正在运行模拟器列表

```shell
adb devices  
```

android studio 无法识别模拟器及真机,使用 adb devices 命令查看,发现以下错误

```shell
adb devices
List of devices attached
adb server is out of date.  killing...
cannot bind 'tcp:5037': Address already in use
ADB server didn't ACK
* failed to start daemon *
```

<!--more-->

问题原因分析**

​	gnymotion 与 系统adb 调用的不是同一个sdk中的adb．所以可能是其中一个启动了一个adb,之后再次启动的时候就提示端口占用了．（两个adb使用了同一端口）

**解决方案**

​	1.查看自己系统adb的路径

```shell
which adb
/Users/indeed/Library/Android/sdk/platform-tools/adb
```

​	2.设置genymotion的sdk路径为上述的sdk路径．

打开genymotin->Settings->ADB->Use custom Android SDK tools->Browse->选择目标sdk（注意copy到sdk目录就可以了:/Users/indeed/Library/Android/sdk）

​	3:重启模拟器,再次查询

```shell
adb devices
List of devices attached
192.168.56.101:5555	device
```

