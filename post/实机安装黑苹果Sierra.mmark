+++
categories = ["operating_system"]
date = "2017-06-15T23:36:03+08:00"
draft = false
keywords = ["黑苹果", "实机安装"]
tags = ["黑苹果","实机安装"]
title = "实机安装黑苹果Sierra"

+++

虚拟机里面不论是kvm还是virtualbox都安装过Sierra了，即使是在SSD上使用，其流畅性和舒适度还是无法和实机安装相比的，
因此最终还是安装了黑苹果，用起来很舒服。

### 首先是主机参数：
cpu:       i5 4590  
主板:      B85M-d2v  
显卡:      hd4600集显，AMD R9 285  
声卡:      RealTek ACL 887  
网卡:      rtl8111  
<!--more-->

由于macOS换到x86平台已经很长时间了，而且都是intel的cpu，所以黑苹果安装比较容易，主要解决了hd4600的完整驱动，
以及声卡网卡的驱动，一切就搞定了。

现在一般都是采用Clover + UEFI + GPT的方式安装了，我这次的安装同样是这个方案，clover实在是做了太多的工作，极大
的简化了黑苹果的安装和使用。

## 1. 在虚拟机的macOS的app Store中下载安装app
下载之后会在/Applications文件夹中，我这里下载的是Sierra的安装包，路径为
`/Applications/Install macOS Sierra.app`然后
```
/Applications/Install macOS Sierra.app/contents/Resources/createinstallmedia --volume /Volumes/{usb} \
--applicationpath /Applications/Install macOS Sierra.app --nointeraction
```
其中{usb}是通过host分配给virtualbox虚拟机的U盘的名称，mac自动挂载，会给它一个命名，比如一个label为“哈哈的U盘”，
那么就是`--volume /Volumes/哈哈的U盘`。创建安装盘完成后，就可以进行第二步，clover的设置了。

## 2. clover设置
[github](https://github.com/linuxhenhao/hackintosh)上有clover的efi文件夹和config.plist文件
由于采用的是UEFI+CLOVER+GPT的方式安装系统，是不需要像mbr那样在分区表那块写入第一阶段引导代码的，uefi会直接找efi分区，
然后找该分区下EFI文件夹下的boot/boot.efi，也可以不是这个文件名，那就需要用譬如windows平台下的easyEFI添加一下clover
的`/EFI/CLOVER/CLOVERX64.EFI`。

### 2.1 clover 安装
首先需要新建，或者已有一个fat32的分区，大小500M或者2、300M都可以，需要添加esp和boot的标签，这样主板的uefi固件才能知道
这个分区是存放efi文件的分区。然后在该分区下新建一个`EFI`文件夹，将上面的github里的CLOVER文件夹放到该文件夹下，新建一个
`Boot`文件夹在`EFI`文件夹下，然后把CLOVER文件夹下的CLOVERX64.EFI复制过来，文件名改为`Bootx64.efi`，这个是选择以EFI
方式启动后的默认加载文件的文件名，不用写入机器固件中。
或者从CLOVER的[SourceForge](https://sourceforge.net/projects/cloverefiboot/files/Bootable_ISO/)页面下载
iso文件，复制iso里的boot和clover文件夹到新建的EFI文件夹下就OK。![clover efi](/images/bootableEFIClover.png)

### 2.2 clover的配置
#### 2.2.1 efi驱动文件
安装好clover之后接下来就是配置了，由于clover开发者们的great works，我们现在可以很简单的就安装好黑苹果，对于EFI方式
安装，需要的efi驱动文件都应该放在EFI/CLOVER/drivers64UEFI文件夹下，有几个必须的efi文件，如果缺少的话一定要下载放好，
否者安装过程都进不了，直接卡死。

hfsplus.efi  # 用来读取hfsplus分区的内容，做出来的安装U盘就是这个格式，所以这是必须的
partitiondxe.efi  # 不知道干什么的，clover的wiki上说这个是必须的
osxaptiofix2drv.efi  # 做一些仿冒工作，然内核能够启动（好像是这样）
fsinject.efi  # inject kext必须的驱动

#### 2.2.2 kext文件
clover可以加载内核扩展，当然对于新的macOS，由于SIP的存在，必须在clover的配置文件里配置了CsrActiveConfig才可以加载
和更改系统/S/L/E文件夹下面的kext。kext放置的位置在`/EFI/CLOVER/kexts/版本号/`，如果列出的版本号中没有你的macOS版本，
kexts放到Other文件夹下就可以。
只有一个扩展是必须的*FakeSMC.kext*，最新的[下载地址](https://bitbucket.org/RehabMan/os-x-fakesmc-kozlek/downloads/)
我这里还有一个rtl8111.kext的网卡驱动,[下载地址](https://bitbucket.org/RehabMan/os-x-realtek-network/downloads/)
还有Lilu.kext [地址](https://github.com/vit9696/Lilu)，这是一个提供方便patch内核扩展的扩展，可以减少很多重复工作，不需要修
该系统硬盘上的kext文件而热修改内容，从而实现一些功能，比如我的声卡就是用依赖于Lilu的AppleALC.kext驱动的，[地址](https://github.com/vit9696/AppleALC)
当然，AppleALC.kext还需要在config.plist进行layout的指定，不然出不了声音。

#### 2.2.3 config.plist文件
这是配置的关键，可以仿冒硬件id，让硬件使用原生驱动，可以patch系统的内核扩展，让其支持我们的硬件，只有正确的配置文件才能够达到
比较好的使用体验，当然第一步是一个能够让系统启动起来的配置文件，可以参考我的[github](https://github.com/linuxhenhao/hackintosh)
中的配置文件。主要是显卡问题，不论是intel的集显还是amd、nv的独显，一般都可以直接用bing搜索相应的显卡型号+hackintosh或者clover得到
我们需要的配置。一些具体细节的东西需要参考clover的[wiki](https://clover-wiki.zetam.org/Configuration)。
这些kext都是放在/EFI/CLOVER/kexts/10.12或者others里面。

##### 显卡
这里我的独显启动很蛋疼，在Yosemite的时候还可以免配置使用，但是之后的所有版本，都需要开启集显，并且把集显设置为primary的情况下，盲着
在clover的界面（因为只有一个显示器，接的独显）进入mac，然后系统进来后，login界面会让R9 285显示出来。很麻烦，所以最后只用HD4600的集显了。

hd4600的配置，需要配置两个地方，一个是Graphics选项下的Inject部分的Intel设置为True，还需要设置ig-platfor-id.
```
<key>Graphics</key>
	<dict>
		<key>Inject</key>
		<dict>
			<key>ATI</key>
			<false/>
			<key>Intel</key>
			<true/>
			<key>NVidia</key>
			<false/>
		</dict>
		<key>ig-platform-id</key>
		<string>0x0a260006</string>
		<key>NvidiaSingle</key>
		<false/>
	</dict>
```
只有Inject intel设置为True了，clover才会应用后面的ig-platform-id，ig-platform-id对应的是显卡不同的端口情况，需要找到合适的，只能搜索
然后一个个测试。有一个必须提出的问题是，由于苹果早就不支持VGA接口了，所以接在VGA接口上是不能够正常驱动的，原来在不inject intel的情况下，系统
可以进入图形界面，但是不识别显卡型号，并且显存只有7M。inject之后，还接到VGA口上开机黑屏，而接到DVI接口上正常工作，显存和型号都正常。当然
hdmi和dp接口也都是可以工作的。

##### 声卡
```
<key>Audio</key>
		<dict>
			<key>Inject</key>
			<string>11</string>
		</dict>
```
由于Lilu和AppleALC的存在，只要对audio inject就行了，具体情况的话由于接口的不同，这个值不同，对于B85M-d2v主板自带的ALC887, 设置为11就可以了。
这个id和上面的显卡id一样，即使成功调用了mac的系统驱动，但是也需要找到对应的layout id，才能够使前面版、后面版的声音都能够正常控制。开始直接
inject 1，后面版的输出没有问题，但是前面板没有声音，改成11就解决了。只能通过一个一个的试，找到合适的id值。具体型号的驱动有哪些layout id，可以在
AppleALC的github repo的[Resources](https://github.com/vit9696/AppleALC/tree/master/Resources)文件夹里找到对应的显卡型号文件夹，里面包含
了所有目前被支持的ID值。

## 总结
其实有了CLover，找到合适的配置文件之后，安装过程是比较简单的，驱动只要有了网卡驱动声卡驱动，显卡驱动，那么最基本的问题也都解决了，安装之后可以
正常更新、安装软件，体验很好。

