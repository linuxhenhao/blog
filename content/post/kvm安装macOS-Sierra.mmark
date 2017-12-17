+++
date = "2017-06-04T20:51:18+08:00"
draft = false
title = "kvm安装macOS Sierra"
tags = ["kvm", "linux", "macOS Sierra"]
+++

通过修改进行一些修改可以使用virtualbox安装macOS，但是virtualbox跟进内核更新往往存在一定的落后，而kvm不存在这个问题，可以使用最新的内核，而且virt-manager的管理使用起来也很方便。因此采用kvm来安装macOS Sierra。
<!--more-->

使用这个[github repo](https://github.com/kholia/OSX-KVM) ，这个repo当中已经将kvm安装macOS的相关配置以及可能遇到的问题都搞定了，直接使用就可以，十分方便

```
git clone --depth=1 https://github.com/kholia/OSX-KVM
```

在clone过来之后的文件夹里，首先需要创建安装系统用的iso，使用`create_install_iso.sh` 可以创建所需的iso，但是这个脚本只能在osx系统里面运行。首先在osx系统的apple store中下载安装所需要的app，然后再运行该脚本，就能在当前文件夹下创建好所需的iso文件。我使用的时macOS Sierra 10.12.1。

* 安装qemu和virt-manager

```
sudo apt-get install qemu uml-utilities virt-manager
```

* 将virtualbox中创建的iso文件用ftp或者共享文件夹的方式将其放到host环境当中

* qemu-image create然后在macOS-libvirt.xml中修改iso文件的位置， bootloader enorh的位置
```
qemu-img create -f qcow2 mac_hdd.img 64G
virsh --connect qemu:///system macOS-libvirt.xml #在virt-manager中添加具备安装macOS的配置的虚拟机
```

* 使用virt-manager启动该虚拟机

遇到了问题，启动后需要使用键盘时发现键盘无效，打开配置，和之前virt-manager中添加的虚拟机对比，发现虚拟键盘和虚拟鼠标有两套，而正常工作的配置中都只有一套PS2的，因此删掉通用虚拟键盘和通用鼠标，只留PS系的鼠标键盘，果然正常工作了。
启动了安装界面之后，在选择安装位置时需要首先格式化添加的虚拟硬盘，但是上方的menubar并没有正常显示，根据[github repo](https://github.com/kholia/OSX-KVM),用Alt-F2和方向键来使所需要的disk utility启动，然后在disk utility
当中格式化硬盘，之后就可以安装了。

## 更新后问题以及解决方式
### 问题
安装之后可以正常启动，第一件事就是更新系统，从10.12.1更新到10.12.5,再次开机之后启动出错，出现了don't steal mac的提示，经过搜索，问题出在qemu的applesmc相关代码上，
但是目前发行版中自带的qemu是没有应用这些更新的，因此只能不更新了。虚拟机还有一个好处就是可以方便的建立snapshot，应该在安装完成之后就建立相应的snapshot，更新后出问题
了还可以回滚，但是开始并没有这么干，只能重新安装了。10.12.4之后都有这个问题，详细信息[qemu-devel邮件列表](https://lists.nongnu.org/archive/html/qemu-devel/2017-03/msg06366.html)
[介绍post](https://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/)

### 另一种解决方案
可能使用fakesmc的方式可以解决问题，直接使用virt-manager加ovmf的方式，用uefi的方式启动，使用黑苹果常用的Clover引导，添加fakeSMC应该可以解决问题。
但是下载了Clover的bootable的iso之后，直接添加到相应的虚拟机无法从其启动，挂载qcow2之后，在mac_hdd.img中的efi分区中复制加入Clover的文件。
uefi方式启动后进入了一个uefi shell，使用map命令可以看到硬盘的alias，输入alias:可以切换到相应分区，这里的命令和windows的dos命令很像。
```
map #查看设备和别名的映射关系
别名: #切换到相应分区，和dos命令一样，如FS0 映射为我们的efi分区，输入FS0:，切换到该分区
```
可以直接输入efi文件所在的位置，运行该efi，启动。
```
FS0:\EFI\boot\bootx64.efi
```
但是实际运行clover启动时，提示load error，所以暂时没有解决。：（

## 挂载hfsplus分区
有了Sierra虚拟机之后，有时候需要在host主机中挂载分区进行文件的操作，记录以下
### 首先需要挂载qcow2文件
```
sudo modprobe nbd # 挂在qcow2所需的内核模块
sudo qemu-nbd -c /dev/nbd0 /path2qcow2/mac_hdd.img
sudo kpartx /dev/nbd0 # apt-get install kpartx可以安装，会根据硬盘设备文件创建相应的mapper设备
sudo fdisk -l /dev/nbd0
  Device          Start       End   Sectors   Size Type
  /dev/nbd0p1        40    409639    409600   200M EFI System
  /dev/nbd0p2    409640 132948151 132538512  63.2G Apple HFS/HFS+
  /dev/nbd0p3 132948152 134217687   1269536 619.9M Apple boot
```

#### 关闭journal然后挂载
需要挂载`/dev/mapper/nbd0p2`，但是在挂载当中发现只能只读模式挂载，通过搜索发现原因在于需要关掉journal
有两种方案

* 编译一个能够在linux中运行的关闭journal的程序[pastbin](https://pastebin.com/W8pfgHRe)
* 使用安装iso使用diskutil命令关闭
由于安装时的iso还在，只需在启动虚拟机时选择从iso启动即可，因此选择使用第二种方式，其中同样需要使用Alt-F2和方向键启动terminal，在其中运行
`diskutil disableJournal <disk>`，这里的disk是/dev/disk0s2，之后还需要启用journal功能
```
sudo mount -t hfsplus -o force,rw /dev/mapper/nbd0p2 /mnt
```
从[这里](https://bitbucket.org/RehabMan/os-x-fakesmc-kozlek/downloads/)下载最新的fakeSMC，将fakeSMC.kext放入/S/L/E/当中，
```
chmod 755 fakeSMC.kext #设置好权限
chown root:wheel fakeSMC.kext #设置好owner， 权限和owner必须设置，不然无法加载
rm -R /System/Library/Caches/com.apple.kext.caches # 必须删除生成的kext的缓存，才能使系统启动时使用添加的fakeSMC.kext
```
结果发现放入后系统出现`can't perform kext scan .....`，直接放fakeSMC这个思路行不通

## 总结
使用[github repo](https://github.com/kholia/OSX-KVM)，使用10.12.1制作的iso文件可以在kvm中成功安装和使用，由于qemu的appleSMC相关代码
的问题，10.12.4及之后的系统无法使用，因此安装完成后不要更新，等qemu修复相关问题后再更新。

### 使用中遇到的问题
使用apple store 安装软件时，登陆出错，提示`your device or computer could not be verified `，经搜索，[apple论坛](https://discussions.apple.com/thread/3175514?start=0)
上有解决方法
```
Solution

1) Delete the file Macintosh HD/Library/Preferences/SystemConfiguration/NetworkInterfaces.plist

2) Restart OSX

3) Try logging into Mac App Store again.
```
但是该方法只对由于默认联网网卡的名称不是en0导致的问题有效，并非我遇到的问题的解决方案。

该安装方案利用的时变色龙bootloader在bios模式下启动的系统，因此之前的摸索方向错误，无须使用uefi+clover的方式，直接使用enoch就可以。

当然首先是解决apple store无法登陆的问题，使用变色龙作为bootloader，因此使用osx-kvm repo中的配置文件就直接解决了这个问题。
其中起作用的部分应该是`ethernetBuiltIn`，再次创建快照，测试添加fakeSMC之后更新到最新10.12.5的问题。

osx-kvm相应的git repo中已经有了配置好的plist文件。在`/Extra`中放入该文件即可，变色龙也可以加载嵌入kext，解决steal macos的问题。
具体来说，在系统的`/Extra/`文件夹中放入一个org.chameleon.Boot.plist，里面有ethernetBuiltIn选项，设置为Yes，
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>Timeout</key>
<string>10</string>
<key>EthernetBuiltIn</key>
<string>Yes</string>
<key>PCIRootUID</key>
<string>1</string>
<key>KernelBooter_kexts</key>
<string>Yes</string>
<key>CsrActiveConfig</key>
<string>103</string>
</dict>
</plist>
```

添加额外的kext文件，在sierra中由于SIP的存在，会有一致性验证，直接往/S/L/E中添加kext会导致系统无法启动，需要让bootloader把它的SIP关掉，
配置文件中的CSRactiveConfig 103就是这个目的。
kernelBooter_kexts设置成yes，使系统在启动时会从`/Extra/Extensions`文件夹中加载kext文件，因此将下载来的最新的fakeSMC.kext放到
该文件夹内（没有的话创建一个）。

结果发现即使成功使用了fakeSMC.kext，仍然在更新之后出现Don't  steal mac os 的提示，确实是qemu本身就存在问题。根据从[qemu官网](http://www.qemu.org)
下载的源码，即使到了qemu 2.9.0这个问题仍然没有被修复。问题源于qemu的appleSMC.c中相关实现的问题。对于macOS sierra 10.12.4以及之后的系统，都需要使用
patch后的qemu才能够正常的安装使用。相关详情见[mail-archive](https://www.mail-archive.com/qemu-devel@nongnu.org/msg441562.html)。

这是我整理的[patch](/static/uploads/qemu-sierra10.12.4upper.patch)，由于appleSMC.c很多个版本没变过了，所以可以直接patch到2.8-2.9版本的qemu
源码然后编译，这样就可以更新到最新的10.12.5也不会出现问题。
