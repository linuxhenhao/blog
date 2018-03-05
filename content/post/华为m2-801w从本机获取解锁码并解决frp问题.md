+++
title = "华为m2 801w从本机获取解锁码并解决frp问题"
date = 2017-12-14T00:32:13+08:00
tags = ["android", "linux"]
categories = ['others']
keywords = ['免申请解锁','android']
markup = "mmark"
+++
使用kingroot可以临时root华为m2-801w，但是重启之后root消失。利用这个临时的root获取nvme，
从而获取到解锁码.
<!--more-->
## 获取解锁码
从csdn博客上看到的方法，首先需要使用kingroot或者其他root工具root系统

    $>adb shell  # 主机上
    $su
    # cd /sdcard
    # cat if=/dev/block/platform/hi_mci.0/by-name/nvme of=nvme
    $>adb pull /sdcard/nvme # 主机上，带>都表示主机
    $>strings nvme | grep WVDEVID -B 1  # 得到16位纯数字解锁码

拔掉usb线，关机，按住音量减，插入usb线，由于供电，回启动，从而进入fastboot模式。
一般情况，此时在电脑端运行

    fastboot oem unlock <解锁码>
就可以解决问题了。但是遇到了FRP locked问题，需要先解锁FRP锁才能解锁bootloader。

FRP全称Factory Reset Protection的目的是解决丢失手机轻易被别人用的问题，直接刷机就可以当新手机用了。
但是有FRP锁，就不能直接刷机使用。更具体的情况未详细了解。

但是这个平板就是全新的系统，推测原因是国外和国内版本的rom混刷导致的。

## 搜索到的解决办法：
1\. 网上常见方法， 连续点击7次buildnumber或者叫序列号之后，开启开发者模式，然后找到
    oem unlock enable，开启，之后就可以了。

2\. 但是我在该平板的开发者选项里找不到这个选项。需要首先
    2\.1 进入recovery， 格式化cache，format factory，然后重启。
    2\.2 初始化设置，不要开启远程保护'remote protection'(实际我没看到这个选项)
    2\.3 进入设置-》账号-》google，登陆
    2\.4 进入开发者选项，应该就oem unlock选项了。

上述方法均对我的情况无效。由于猜测是混刷国内外rom导致的，由于购买的是海外版，所以决定刷回
海外版。但是EMUI4.0之后，无法在设置->系统更新中选取本地更新包更新。只能用同时按住音量+-键
的同时开机的三键强刷方式更新系统。

值得注意的是，竟然不能在连着usb线的同时强刷，否则会卡在5%正在安装。


## 具体过程：
1、刷B213退回B019中间包（花粉论坛上有）:由于是欧版刷的国内包B213，发现直接刷欧版或者国际版
的B213失败（[国外固件系列地址](https://boycracked.com/2017/05/05/official-huawei-mediapad-m2-8-0-m2-801w-stock-rom-firmware/)）
，所以刷回中文B019然后强刷国际版B213。
2、安装google play，登陆账户，然后不论是系统设置还是google设置都关闭“找回”功能。
3、安装kingroot完成root。
4、重启到bootloader，也就是fastboot界面，发现莫名其妙的就变成PHONE UNLocked了，不知道是
上述的2还是3导致的。然而frp lock
还是存在，所以没法用fastboot flash方式刷机。不过bootloader已经解了。
4、在正常系统中B019, 通过kingroot root成功， 然后使用adb shell，直接将twrp写入recovery分区，
然后由于bootloader锁没有了，就可以随意刷了。当然也就可以刷入欧版的recovery回复到我海外版
的mediapad的官方固件。

    host$>adb push twrp.img /data/local/tmp/recovery.img
    host$>adb shell
    android>$ su # 记得在界面中确认允许权限申请
    android># dd if=/data/loca/tmp/recovery.img \
                 of=/dev/block/platform/hi_mci.0/by-name/recovery

## 从官方UPDATE.APP中提取recovery.img
xda论坛上有个splitupdate.pl的脚本，也有huawei-extractor.exe的可安装软件，都可以方便的
将UPDATE.APP中的文件提取出来。通过perl脚本，提取到了RECOVERY.img。

使用file命令查看该RECOVER.img发现类型是`data`，而对于twrp或者从别的地方拿到的recovery
的类型却是`Android bootimg`，采用vim分别打开两种类型的recover，用`%!xxd`转化为16进制，
发现从UPDATE.APP中提取出来的recovery.img有一个偏移之后，和类型为`Android bootimg`的
recovery就一模一样了，具体的偏移量需要用vim打开16进制后观察得到。使用dd得到同为`android bootimg`
类型的recovery

    dd if=./RECOVERY.img of=./androidbootimg-recovery.img bs=16 skip=256
    # 小技巧，vim 用xxd转换的16进制模式，每行是16字节，看行号就能知道要跳过多少内容
    # 完成之后用file命令查看一下
    $file androidbootimg-recovery.img
    androidbootimg-recovery.img: Android bootimg xxxx # 成功

PS:

进入工程模式方法：在计算器中，横屏模式下，输入()()2846579()()=，可以进行“工厂级恢复恢复出厂设置”，
或者说“低级格式化”。

`fastboot oem frp-unlock FRPKEY`可以解锁frp，但是需要这个key，不想买所以没弄。

PPS:

果然是瞎折腾，由于混刷国内外rom导致的FRP，想刷回国外版，但是在recovery界面提示“版本信息校验失败”，
无法完成。由于刷回了b019, 所以可以root并重启后root仍然存在。其实到这里，折腾这件事的初衷就可以达到了，
通过adb shell进入系统，用dd写入想要刷如的第三方recovery，并且重启也确实可以进入了。那么就可以
刷入gapps和xposed以及supersu这些东西了。

但是不作不死，看着fastboot界面的FRP lock就不爽，于是想直接用dd将UPDATE.app里解出来的分区一个个
的写进去。但是忘了极为重要的事情——解出来的分区并不是最终写入的内容，前面有一定长度的头文件，就像
上面说的recovery.img的处理部分所说的。结果就是写入的各个分区镜像都是错误内容， 然后开不了机变砖了。
电源键按了毫无反应，也没法强刷，很可能是uboot（类似）或者主板新片坏了。网上查换主板1400, 加上屏幕
原来就摔裂了，干脆扔掉了。。。。
