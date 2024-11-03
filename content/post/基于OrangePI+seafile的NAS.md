+++
title = "基于OrangePi+seafile的NAS"
date = 2018-03-30T14:12:25+08:00
tags = ["armhf", "linux"]
categories = ['storage']
keywords = ['orangepi','seafile', 'NAS']
markup = "goldmark"
+++
现在网上的公有云很多，比如百度网盘，以及之前的360网盘等。但是陆续关闭的公有云，百度网盘
的限速问题，隐私问题等，是许多人使用网盘的痛点。自己搭建自用的私有云成了很多人的一个选择，
但是直接购买比较大型的NAS设备一方面占地方，另一方面也是确实用不上。这里用seafile+OrangePi Plus2
搭建了一个物美价廉的私有云盘，通过小米智能插座还可以远程控制开关，十分好用。
<!--more-->
## 1. 硬件
OrangePi Plus2：[官网链接][1]，类似于树梅派的板子，性能强劲，2G内存，10/100/1000 M 自适应网卡，
带板载emmc存储。

当然也可以选其它品牌的板子，比如直接选最新的树莓派，或者是国内的香蕉派、cubieboard 等。需要注意
的除了网卡最好支持千兆以保证在家使用的速度之外，还有板子在 mainline kernel 上的支持程度。国内
的板子一般都使用全志的soc芯片，可以在购买之前上全志的[wiki][2]上查看下该芯片和板子是否被 mainline
kernel 支持。目前H3的芯片已经支持得很好了，所以购买 OrangePi Plus2 是稳妥之举，如果不在 mainline
 支持列表内，那么很可能面临的就是仍然使用 3.xx 版本的内核，无法使用最新的 systemd ，无法体验 docker，
总之会产生一些麻烦。有 mainline kernel 支持，就可以更新到各个包含 armhf 或者 arm64 架构支持的发行版，
比如 Ubuntu、Debian 的最新版本，并且可以一直更新。

SD 卡一张：4G或者以上的SD卡一张。这张卡用来启动系统，将自编译的内核、Uboot、Uboot 配置脚本写入板载 emmc。
之后就不需要了。如果购买的板子版本无板载存储，那么就需要将SD卡一直插在卡槽上，用来引导系统启动。
需要注意的是， SD 卡在用作系统分区时被频繁读写是十分容易损坏的（之前坏过两张），所以只从它上面读取内容
启动，其余的根文件系统都放置在磁盘上（ SSD 太贵，而且没有必要）。

HD硬盘+硬盘座：根文件系统以及保存个人文件的分区都放在这个硬盘上。由于是拿来作NAS的，直接买2.5寸的
台式机硬盘就行，加上一个USB接口的硬盘座用来和 OrangePi 连接。

 RAID1 冗余存储[可选]： 虽然HD硬盘的寿命都还可以，但是万一哪天突然坏了，这上面放的都是比较有纪念意义
的东西，丢了实在可惜，所以可以再买两个大小一样的HD硬盘（我使用的是 1 T 大小的），然后加上 RAID1 的硬盘
盒，组建一个 RAID1 冗余。这里推荐一款物美价廉的 RAID1 硬盘盒——[GODO][3]，买的时候注意和商家说明，需要
买断电后接通电直接处于打开状态的盒子。这个主要源于使用智能插座远程控制的需求。默认情况下，无论之前该
盒子是打开还是关闭状态，断电之后再接通，该盒子都处于关闭状态。这样是没有办法实现以插座作为开关控制
整个系统启动的。幸运的是，该盒子已经有断电再通电保持开启状态的版本，购买时和客服沟通即可。特别感谢
一下他们的技术支持，指导我把原先买的盒子用焊锡改造成保持开启版本。

## 2. 软件
### 2.1 制作可启动 SD 卡
需要 OrangePi plus2 的镜像，写入 SD 卡用来写入~~编译的内核文件、Uboot 等到 emmc 存储~~镜像。OrangePi plus2 的硬件配置
和 OrangePi plus 的一样，只有内存大小不一样，因此这两者的镜像通用。

本来是需要自己编译内核的，但是打开 armbian 网站一看，已经有 mainline 内核的系统镜像了！！！省了很多事情。
从[下载地址][4]下载镜像文件，将 SD 卡用读卡器接入电脑。Windows 下用 Win32diskimager，linux下用dd，
将下载来的 7z 压缩包解压后得到 img 后缀的镜像文件，在 linux 下用以下命令写入 SD 卡， 卡上原有内容
将被全部覆盖。

    dd if=./Armbian_5.38_Orangepiplus_Debian_stretch_next_4.14.14.img of=/dev/sdx

### 2.2 准备硬盘
采用 2 T 单硬盘 + 2个 1 T 硬盘组的RAID1  的存储方案，分区方案：

硬盘 | 分区类型 | 分区大小
-------|---------|-----------
2 T 硬盘 | ext4 | 10 G
2 T 硬盘 | btrfs | 剩余所有空间
------|---------|--------
1 T RAID | btrfs | 所有空间

*以下称 2 T 硬盘为主硬盘， RAID1 硬盘组为备份盘，其中 2 T 硬盘包含根分区和数据分区*。   

将数据存储区单独使用 btrfs 独立分区， RAID1 硬盘组完全采用 btrfs 分区，这样就可以使用
btrfs 的 snapshot 功能创建快照，并使用 btrfs send | btrfs recieve 将快照增量备份至
RAID1 实现冗余备份。可以使用 fdisk/parted/gparted 完成上述分区，当然需要在 Linux 下完成。
gparted图形界面比较简单直观。

* 复制镜像根分区到 2 T 硬盘根分区    
然后，需要将 Armbian 镜像中的根分区复制到主硬盘的根分区：

```bash
    >sudo parted Armbian_xxx.img  # 使用parted查看镜像中的分区情况
    >(parted) unit B  # 将parted的显示单位设置为byte
    >(parted) print
    Model:  (file)
    Disk /home/huangyu/Downloads/Armbian_5.38_Orangepiplus_Debian_stretch_next_4.14.14.img: 1396703232B
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags:

    Number  Start     End          Size         Type     File system  Flags
     1      4194304B  1396703231B  1392508928B  primary  ext4

    # 可以看到img文件内包含一个ext4的分区，起始位置是4194304B
    >sudo mount -t ext4 -o offset=4194304 Armbian_xxx.img /mnt
    >sudo mount /dev/sdx1 /media/root  # 将主硬盘的根分区也就是10G的ext4分区挂载
    >sudo cp -Rp /mnt/* /media/root  # 将Armbian镜像根分区中的内容复制到主硬盘根分区
```
将镜像 Armbian_xxx.img 整个放到主硬盘根分区中，用于后续将其写入 emmc 存储。

### 2.3 将镜像写入板载 emmc 存储
将 SD 卡插入 OrangePi plus2 的插槽，2 T 硬盘USB接入USB插孔，插上网线然后接通电源（似乎这个板子的电源开关按钮不
起作用，只能通过通电或者断电控制开关）。启动后等一段时间，会自动通过 dhcp 获取 IP 地址，
通过路由器查看板子的 IP 地址，然后从电脑上 ssh 过去，默认的用户名是 root 密码是 1234。

挂载 2 T 硬盘，然后用 dd 命令将主硬盘中的 Armbian 镜像 dd 到 /dev/mmcblk2p1， 不用理会
/dev/mmcblk2boot0 和 /dev/mmcblk2boot1 分区。

    # mount /dev/sdx /mnt
    # dd if=/mnt/Armbian_xxx.img of=/dev/mmcblk2p1

### 2.4 修改配置，从主硬盘启动
采用 Armbian 有个好处，不用自己编译 uboot 和 uboot 配置脚本了，修改配置只需要修改 /boot/armbianEnv.txt
即可， 载入该文本的动作的 uboot 脚本已经被配置好了。

2.3 中在 dd 完成之后，就可以 mount /dev/mmcblk2p1 了，其下内容就是 Armbian 镜像的内容，
除了 boot 文件夹包含启动配置文件、内核文件等不能删除外，其他文件夹要是不喜欢都可以删除，
因为我们仅仅需要从板载 emmc 启动，根分区放在了 2 T 硬盘上。在修改 boot/armbianEnv.txt
改为如下内容：

    verbosity=1
    logo=disabled
    console=both
    disp_mode=1920x1080p60
    overlay_prefix=sun8i-h3
    rootdev=UUID=主硬盘根分区的uuid
    rootfstype=主硬盘根分区的类型
通过`blkid`命令查看主硬盘根分区的uuid，将其放在对应位置，分区的类型这里就是ext4，这样
指定之后，如果拔掉 SD 卡， 插上 2 T硬盘 USB 启动系统，那么就会从 emmc 分区读取uboot，
uboot 读取 boot/armbianEnv.txt 的内容，从而找到 USB 连接的硬盘上 UUID 对应的分区，
以 rootfstype 挂载为根分区，从而实现从外部 HD硬盘启动，以增长整个板子的使用寿命。

需要更改 2 T 硬盘根分区中的 /etc/fstab 以适应此更改。如果不改fstab，内核载入开始运行后，
读取 2 T 硬盘根分区中的内容，读取 /etc/fstab 进行挂载就会出错，从而导致系统无法启动。
完整的包含 RAID1 冗余盘挂载项的 fstab 如下：

    LABEL=pi-root /    ext4 defaults,noatime,commit=60 0 0
    LABEL=swap swap swap  default 0 0
    LABEL=data-raid1 /media/raid1 btrfs defaults,noatime,compress=lzo,autodefrag 0 0
    LABEL=pi-data /media/normal btrfs defaults,noatime,compress=lzo,autodefrag,subvol=normal-subvol 0 0
    LABEL=pi-data /media/u-sda1 btrfs defaults,noatime,compress=lzo,autodefrag,subvol=important-subvol 0 0
    LABEL=pi-data /var/lib/docker btrfs defaults,noatime,compress=lzo,autodefrag,subvol=docker-subvol 0 0
用 LABEL 指定挂载点能少写点字，label 也是之前在接入电脑进行分区时设置的。其中最后一行是为了
方便玩 docker 用的，因为使用了 ext4 作为根分区，所以要专门挂载一个 subvolume 到 /var/lib/docker用于
docker的 btrfs storage backend。挂载点的文件夹可以随意，但是需要确保其存在，否则fstab挂载失败
会导致系统启动过程卡死。

### 2.5 安装seafile、配置samba等
上面的步骤完成之后，就实现了接通电源后，从 emmc 读取内核，以 2 T 硬盘中的根分区为 root，
并且挂载好数据分区、RAID1 冗余分区到指定的挂载点了。之后就可以安装seafile、安装samba
用于局域网共享视频什么的，可以节约手机、平板的空间。更可以安装个Aria2c，使用AriaNG前端，
用作远程下载，下载后直接回去可以通过 samba 贡献在各种设备上观看。 如果没有公网 IP 还可以
租个 vps， 用 frproxy 配置好暴露的端口，使得我们可以通过 vps 访问内网的 AriaNG 和 seafile。
这些服务如果是 debian 自带的，那么只需要使用 systemd enable就可以了，如果是 seafile，
AriaNG 这种，就需要自己写 Python 脚本或者是 systemd 的 service 文件来实现启动了，我采用
的是 supervisord 来控制。
还可以有更多的玩法，比如加个音响定时报个天气预报什么的，只有想不到没有做不到啊。具体的配置
等天再说：）

### 2.6 定时备份到 RAID1 冗余
btrfs 除了方便的 snapshot 之外， send 和 receive 功能也是非常的好用。由于所有的数据都
放在数据分区，定时对数据分区作 snapshot， 然后 send receive 到 RAID1 分区就实现了增量
备份。

使用三个文件实现[cron.py][5],[bakseafile.sh][6],[btrfs-backup.py][7]。

其中cron.py 执行定时任务，执行成功之后会写入 /var/log/ 文件夹作记录，实现一个相对 Linux
自带 cron 有一定区别的功能。 Linux 自带的 cron， 定期执行任务，是用时间点作为判据的，
比如每天7点执行一次， 如果这一天7点刚好没开机，八点开机，那么这一天就不会执行这个任务。
cron.py 使用时间间隔作为判据， 每次成功执行写log记录执行的时间，在启动时检查，如果距离
上次执行成功间隔大于要求的间隔，则执行任务。

bakseafile.sh是一个写好btrfs-backup.py执行所需参数的简单bash脚本。btrfs-backup.py是
在 github 上找到的一个别人写的可以保留固定数量的 snapshot 的 btrfs 备份脚本。

通过 supervisord 执行 cron.py, cron.py 中配置好定时执行 bakseafile.sh, 在 bakseafile.sh
中修改保留备份的数量、备份的源文件夹， snapshot相对源文件夹的位置， 目标文件夹。


[1]:http://www.orangepi.cn/orangepiplus2/index_cn.html
[2]:https://linux-sunxi.org/H3
[3]:https://detail.tmall.com/item.htm?spm=a230r.1.14.13.b31051815IU1QZ&id=35338661217&ns=1&abbucket=4
[4]:https://www.armbian.com/orange-pi-plus-2/
[5]:https://raw.githubusercontent.com/linuxhenhao/config-files/master/orangepi/cron.py
[6]:https://raw.githubusercontent.com/linuxhenhao/config-files/master/orangepi/bakseafile.sh
[7]:https://raw.githubusercontent.com/linuxhenhao/config-files/master/orangepi/btrfs-backup.py
