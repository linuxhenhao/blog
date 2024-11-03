+++
title = "多功能启动U盘"
date = 2017-11-19T09:37:10+08:00
tags = ["多系统", "启动U盘"]
categories = ['operating_system']
keywords = ['多功能','安装U盘']
+++

制作一个同时具备一下四个功能的多功能启动U盘
1\. **数据存储** 2\. **windows系统安装**
3\. **linux的livecd** 4\. **mac安装**
<!--more-->
## 0. 缘由
由于经常折腾系统，安装系统修复`grub`之类的事情，所以我的U盘常常被格式化。不然就是日常的
数据和系统文件混在一起十分不方便。终于决定对U盘进行分区，将常用的功能放到后面几个分区，
前面第一个分区用作数据分区（因为windows系统只识别U盘的第一个分区）。

## 1. 目标U盘结构
U盘总大小为29G，将其规划如下

编号| 内容 | 大小
-----|-----|----|
1\. | fat32 数据区  |  14G
2\. | fat32 `grub`、`Clover`以及`winPE`和`gentoo admincd` |     8G
3\. | hfsplus `macOS High Sierra`安装文件                   |   7G

## 2. 分区
1\. 格式化

使用linux的gparted工具按照上述规划进行分区，将3个分区都格式化成`fat32`，并且命名。其中
mac安装分区由于需要使用黑苹果安装，为了能够指定安装到该分区，该分区应该具备能够被macOS挂载
的格式和方便识别的名称，简单起见，都格式化成`fat32`。

2\. 设置grub分区参数

gparted 设置`esp，boot`参数，使启动时计算机知道该分区为启动分区，找到efi文件夹的位置。
![gparted part creation](/images/post-images/gparted_multifunctional_usb.gif)

## 3. 安装grub
我们的目标是该多功能U盘能够同时在只支持`bios`启动和`UEFI`启动的电脑上均能启动。由于`grub-efi`
和`grub-i386`两种模式的安装并不冲突。`EFI`模式只是将`.efi`文件放到`EFI`文件夹内，而`BIOS`模式
将一些信息写入`mbr`，二者共用`/boot/grub`文件夹，公用`/boot/grub/grub.cfg`配置文件。

首先假设我们将grub需要安装到的分区，也就是上图中的`/dev/sdc2`挂载到了`/mnt`，然后进行
grub的安装。

3.1 grub-efi 安装

`efi-direcotry`参数必须指定，否则默认到进行grub-install所在的host的efi分区，导致无法
启动。

    grub-install --target=x86_64-efi --efi-directory /mnt --root-directory /mnt /dev/sdc

*注意*: 对于计算机内部的硬盘，grub-install会将相应的启动路径信息写入到cmos当中，但是作为移动用的U盘，
这一行为显然是无法在不同计算机间迁移的。对于未在cmos中存在信息的分区，计算机会读取默认路径，也就是带有
esp标签的分区的`EFI/boot/boot.efi和EFI/boot/bootx64.efi`。为了让多功能启动U盘在其他电脑上的UEFI
模式下也能用，我们需要将安装好之后的分区内的`EFI/debian/grubx64.efi`复制到该分区的`EFI/boot/bootx64.efi`,
为了确保能启动，可以再复制一份保存为`boot.efi`在同一文件夹下。

3.2 grub-bios 安装

    grub-install --target=i386-pc --root-directory /mnt /dev/sdc
3.3 grub调试

3.3.1 bios模式调试
使用qemu虚拟机进行grub配置的调试，默认为BIOS模式：

    sudo qemu-system-x86_64 -m 1024 /dev/sdc
3.3.2 efi模式调试

通过添加`--bios`参数指定`tinycore`的efi模式启动：

    sudo qemu-system-x86_64 --bios /usr/share/ovmf/OVMF.fd -m 1024 /dev/sdc
如果系统中没有ovmf，debian系可以采用此命令安装：

    apt-get install ovmf
但是，由于未知原因，`qemu`启动后并没有或者是无法读取到U盘的某些信息，导致其只能在第一个
分区中寻找`EFI`相关文件，而为了让数据分区干净，我们的`EFI`相关文件都在第二个分区，因此
无法使用此方式调试。

## 4. linux (gentoo admin cd)安装配置
作为不折腾不舒服斯基，常常会有弄坏引导，进不了grub等情况发生，这个时候一个体积小，功能
全的livecd就很有用。而gentoo的admincd才小于500M的体积就十分合适，而且实践发现，不需要额
外参数用于启动从iso文件复制到u盘中的内容。

[admincd下载](4)

假设下载后的cd名称为`gentoo-admincd.iso`，`grub`所在分区`/dev/sdc2`挂载于`/mnt`

    sudo mount gentoo-admincd.iso /mnt1
    sudo cp -R /mnt1/* /mnt  # 将iso内容全部复制到grub所在分区
根据cd内的`isolinux`配置，得到相应的`menuentry`需要的参数

    menuentry "gentoo admincd" {
        insmod part_gpt
        insmod btrfs
        linux /isolinux/gentoo root=/dev/ram0 init=/linuxrc dokeymap \
                  looptype=squashfs loop=/image.squashfs cdroot
        initrd /isolinux/gentoo.igz
    }

## 5. windows安装盘
5.1 windows7及以上的独立安装盘

有时候会有win7的装机需求，但是windows系的安装只用一个只有一个分区的U盘来的话比较容易，
一般不会出什么问题。需要注意的是，如果对该U盘进行过分区、调整大小等行为，可能会造成下面
的两种方式都不成功。最好是重建分区表后建立的单个的fat分区的U盘，肯定不会有莫名的问题。
5.1.1 对于mbr方式
将iso内容完全解压到fat分区，然后用grub-install给这个分区和盘安装i386-pc的grub，用

    ntldr /bootmgr
即可轻松安装。

5.1.2 对于uefi方式
这个方式更简单，直接将iso内容解压到fat分区，重启选择uefi从U盘启动即可，连grub都不需要。

## 6. 使用PE的通用型windows安装方式
由于只有一个U盘，windows系统的安装对于多分区的U盘，上述方法则要求windows安装iso的内容
必须解压到第一个分区，grub也安装到第一个分区，才能够成功启动安装。但是这样导致普通文件
和系统文件混淆在一起，很不清晰。一个通用的安装方法是使用PE，而且是同时支持uefi和mbr方式
启动的PE。这就可以将PE的文件放在后面的分区当中，启动之后加载iso进行安装。

6.1 PE文件放置以及配置

PE型的iso文件与windows安装盘的iso不同，一般都可以直接解压到非第一个分区，使用grub加载
后能够正常启动。首先挂载PE的iso文件并复制：

    mount xxPE.iso /home/user/cdrom[随意找个挂载位置即可]
    cp -R /home/user/cdrom/* /mnt  # mnt 为grub所在分区
然后配置相应的`menuentry`，对于BIOS模式的grub:

    menuentry "winpe" {
      ntldr /bootmgr
    }
对于EFI模式的grub:

    menuentry "winpe"  {
      chainloader /efi/microsoft/boot/bootx64.efi
    }

6.2 直接加载iso文件启动
某些PE不能用解压放置的方式启动，可以使用`memdisk`加载整个iso文件启动，但是性能以及通用性
不如文件解压的方法，在某些计算机上可能会失败。

使用`syslinux`项目中的`memdisk`加载iso，对于debian系的系统，才用此命令安装`memdisk`

    apt-get install grub-imageboot
装好后文件位于`/boot/memdisk`，将其复制到U盘grub所在分区根目录（此处为`/mnt/`）或者其
他目录，grub.cfg内容须做相应更改。

理论上来说，所有可以使用iso启动的iso文件都能通过`memdisk`直接加载iso文件启动，但是受制
于iso文件大小、内存大小等限制，太大的iso文件不太合适。

`memdisk`使用方式：
    linux /memdisk iso *raw*
    # 这个raw很关键，不加有些iso启动不了[memdiskwiki](1)
    initrd /xxxPE.iso
    # 需要启动的PE的iso文件，直接加载到内容，所以不宜过大
一个`menuentry`实例，iso文件大小为130M，放在grub所在分区中：

    menuentry "winpe" {
        insmod ntfs
        insmod part_gpt
        linux /memdisk iso raw
        initrd /TYx64Win8PE.iso
    }

启动进入PE了之后，U盘只有第一个分区也就是数据分区可见，其他分区是看不到的，所以需要安装的windows
系统的iso文件要么放在第一个分区，或者放在硬盘上，然后就可以用PE中的安装工具安装了。

## 7. Clover加载用于启动macOS安装
7.1 clover chainloader配置

[clover](5)是一个用于黑苹果安装的很好用的启动工具。具体的相关配置属于黑苹果安装配置的范畴，
这里就不多说。通过`grub2`的`chainloader`将`clover`启动，这样就能够启动`Installer`进行
系统安装或者是修复了。比如本次多功能U盘制作的原因－－将`macOS high Sierra`的根分区转换成
`APFS`。只能离线转换，因此不能进入`macOS`后转。

将`CLOVER`整个文件夹放到`grub`所在分区（这里的`/mnt`）的`/efi/`文件夹中。相应的`menuentry`为：

    menuentry "clover" {
        insmod part_gpt
        insmod ntfs
        chainloader /efi/clover/cloverx64.efi
    }

7.2 第三个`macOS Installer`分区的制作

这个制作在黑苹果当中进行，从`App store`中下载`macOS High Sierra 13.1.1`，然后打开文件
管理器`Finder`，点击使用`gparted`分区时分配给第三个分区的名字，挂载。假设名字为`MACUSB`，
那么挂载后位于`/Volumes/MACUSB`，下载后的安装`app`位于`/Applications/Install macOS High Sierra.app`

使用一下命令将第三个分区制作成`macOS`安装盘

    /Applications/Install macOS High Sierra.app/Contents/Resources/createinstallmedia \
    --applicationpath /Applications/Install macOS High Sierra.app --volume /Volumes/MACUSB \
    --nointeraction
完成后，从在`grub`界面选择`clover`，就会启动`clover`从而能够引导第三个分区中的`Installer`

## 8. grub blind mode修复以及BIOS|EFI模式的判定
只在grub.cfg当中写menuentry项，启动时界面能够出来，但是一旦进入某一项boot之后，就会提示
`booting in blind mode`，然后黑屏。Arch的[wiki](2)上有相应的解决办法。但是对于efi和
bios模式启动的grub，需要加载的模块不同，因此需要用到启动方式的判定。通过[`grub_platform`](3)
进行判定。然后将系统中的字体文件复制到U盘上`grub`所在分区的`boot/grub`中

    cp /usr/share/grub/unicode.pf2 /mnt/boot/grub
`grub_platform`的值，在`BIOS`模式下为`pc`，在`EFI`模式下为`efi`，均为小写，语法与`bash`相同．

相关的`grub.cfg`的内容如下，放在最上方即可，后面接上`menuentry`的内容：

    if [ x${grub_platform} == xpc ]; then
    # pc means bios mode
          insmod vbe
    else
    # efi mode
        insmod efi_gop
        insmod efi_uga
    fi
    if loadfont ${prefix}/fonts/unicode.pf2
    then
        insmod gfxterm
        set gfxmode=auto
        set gfxpayload=keep
        terminal_output gfxterm
    fi

同时，由于`clover`只支持`EFI`模式，因此将`clover`的`menuentry`放在一个`if`判定内：

        if [ x${grub_platform} == xefi ]; then
            menuentry "clover" {
                insmod part_gpt
                insmod ntfs
                chainloader /efi/clover/cloverx64.efi
            }
        fi
这样就只有是采用`EFI`模式引导的情况下才会出现`Clover`的项目．

完整的`grub.cfg`内容为：

    if [ x${grub_platform} == xpc ]; then
    # pc means bios mode
          insmod vbe
    else
    # efi mode
        insmod efi_gop
        insmod efi_uga
    fi

    if loadfont ${prefix}/fonts/unicode.pf2
    then
        insmod gfxterm
        set gfxmode=auto
        set gfxpayload=keep
        terminal_output gfxterm
    fi

    menuentry "gentoo admincd" {
        insmod part_gpt
        insmod btrfs
        linux /isolinux/gentoo root=/dev/ram0 init=/linuxrc dokeymap \
                  looptype=squashfs loop=/image.squashfs cdroot
        initrd /isolinux/gentoo.igz
    }

    if [ x${grub_platform} == xefi ]; then
      menuentry "winpe" {
        chainloader /efi/microsoft/boot/bootx64.efi
      }
    else
      # for bios
      menuentry "winpe" {
        ntldr /bootmgr
      }
    fi

    if [ x${grub_platform} == xefi ]; then
        menuentry "clover" {
            insmod part_gpt
            insmod ntfs
            chainloader /efi/clover/cloverx64.efi
        }
    fi

修改记录： 20180521： 添加关于efi模式正确运行需要的操作

[1]:http://www.syslinux.org/wiki/index.php?title=MEMDISK#Windows_NT.2F2000.2FXP.2F2003.2FVista.2F2008.2F7_.28NT_based.29
[2]:https://wiki.archlinux.org/index.php/GRUB#.22No_suitable_mode_found.22_error
[3]:https://askubuntu.com/questions/399732/how-to-check-if-grub-is-in-efi-or-bios-mode
[4]:https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-admincd-amd64/admincd-amd64-20170209.iso
[5]:https://sourceforge.net/projects/cloverefiboot/
