+++
categories = ["linux"]
date = "2017-06-07T21:24:07+08:00"
draft = false
keywords = ["libimobiledevice","ios", "ifuse"]
tags = ["linux","tag2"]
title = "linux访问ios照片"

+++

之前买了个iphoneSE作为日常用机，由于常用系统是linux，在需要
导入照片时发现虽然nautilus可以发现有一个iphone，但是打不开，
看不到任何文件。这样如果需要导入照片还要重启到win，十分不便。
<!--more-->

经过搜索，发现有一个[libimobiledevice](http://www.libimobiledevice.org/)
的跨平台库，这个库的初衷就是开发出一个可以在linux上独立与ios
进行usb通讯，实现备份/还原数据，安装删除软件，当然查看照片也
不在话下。

`aptitude search`找到了libimobiledevice6的包，系统已经安装了，
但是要能够访问数据却还需要一些能够使用libimobiledevice库的软
件。`ifuse`就是这样一个工具，最简单的使用方法
```
ifuse /path/to/mount
```
ifuse不需要以root权限运行，随便找一个本用户home目录下的文件夹
挂载就行。需要注意的是，必须在*屏幕解锁*的情况下将ios设备连入
计算机，否则识别不了，会导致挂载失败。

这样挂载之后能够直接访问照片和视频数据，ifuse还支持访问某个app
的Documents文件夹，需要知道对应app的APPID，命令为
```
ifuse --appid APPID /path/to/mount
```
