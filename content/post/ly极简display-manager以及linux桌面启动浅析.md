+++
title = "Ly极简display Manager以及linux桌面启动浅析"
date = 2018-02-06T22:23:53+08:00
tags = ["operating_system"]
categories = ['operating_system', 'linux']
keywords = ['ly', 'display manager', 'desktop startup process']
markup = "mmark"
+++
作为多年linux老用户，当年也经历过轮着安装不同的linux发行版，轮着试不同的桌面环境的阶段。
最后停在了使用awesome作为桌面环境，很喜欢tiling window manager的简洁高效，可以用键盘
替代大部分鼠标操作的特性。但是之前一直没有找到一个非常和心意的display manager，也就是
平常看到的登陆界面程序。gdm3和lightdm之类的都太重，一段时间之前翻archlinux的wiki找到
了ly这个极简display manager。它用pam作认证，设置一些必须的环境变量，然后用execl运行
xinit命令启动相应的x-window-manager，比如awesome。没有其他多余的东西，依赖也很少。
<!--more-->
## 桌面环境启动过程
1. 通常的linux桌面环境启动，通过init系统，不论是原来的sysv的还是现在（2018）流行的systemd，
启动一个display manager。一些比较漂亮的dm，比如gdm3, lightdm，运行在Xserver之上的xclient，
而也有一些简单的dm不需要启动Xserver就可以运行，比如cdm，ly等。
2. 通过display manager作认证，通过认证之后，display manager会启动Xserver（如果display manager
不是作为xclient运行于XServer上的话），然后启动相应的Window-manager，或者是大型桌面环境
的启动器，比如KDE的startkde，启动器会额外做一些工作，当然window-manager是必须启动的。

### 额外小知识：直接运行xclient于XServer之上
按照上述的桌面环境启动流程，wm启动之后，其他的图形软件就这么运行与window-manager管理之下，
通过XServer显示在屏幕之上，并与用户交互。

实际上就是xclient运行于XServer之上的流程。那么我们可不可以不经过display manager，window-manager，
直接启动图形程序运行与XServer之上？答案是确定的，完全可以。但是一些严重依赖于桌面环境提供
的一些服务、后台功能的程序会出现问题。如果独立性较强的话，是可以完全脱离dm和wm运行的。启动
一个程序运行与XServer之上的命令为

    xinit /fule/path/to/executable args -- /usr/bin/X DISPLAY vtN

在`--`之前的为需要运行的xclient的可运行文件的完整路经，后面可以更上需要的参数，`--`之后
为XServer的运行文件完整路径以及参数。DISPLAY用`:`+`数字`替代，比如`:0`、`:1`等，这是
指定XServer运行的地质，前面省略了IP或者域名，因为XServer设计时是可以跨网络运行的，不一定
需要client和server在一台电脑上，这也是ssh -X可以运行的原因。在同一台电脑上，可以运行
多个不同的XServer，只需要指定不同的DISPLAY即可，也就是说同一台电脑是可以同时运行多个桌面环境的。
当然只能是一些简单环境，比如awesome，复杂的桌面环境后台会有很多独一无二的程序运行，会起冲突。
xclient运行时，需要指定DISPLAY参数，当然设置环境变量也可以，这样可以在终端环境中运行xclient
显示于不同的XServer上，当然这中间可以设
认证措施。`vtN`比如`vt0`，这个参数指定XServer运行于哪个virtual terminal当中，现在的linux
启动之后一般会初始化好几个virtual terminal，可以用`ctrl+alt+Fn`进行切换。如果以非管理员
身份运行，想要在非当前virtual terminal上运行XServer会提示权限不足出错。以一般用户权限运行
XServer只能指定vt为当前的vritual teriminal。

举个运行firefox于XServer上的例子，当前在vt1也就是第二个vritual terminal上运行了一个
正常的桌面环境，比如gnome，这时我们`ctrl+alt+F3`切换到vt2，登陆然后运行

  xinit /usr/bin/firefox -- /usr/bin/X :1 vt2

这样就会在vt2上运行一个XServer，Xserver中只有一个firefox浏览器运行，基本的鼠标键盘
之类的操作响应由XServer本身提供。但是这个时候很难受，连个输入法都没有怎么办？可以切
换回vt1的gnome环境，打开一个terminal emulator窗口，也可以`ctrl+alt+Fn`切换到某个
vritual terminal。然后

    export DISPLAY=:1
    fcitx
通过指定环境变量中的DISPLAY为`:1`也就是响应firefox的XServer的地址，让fcitx运行于那个
XServer之中，然后就可以用fcitx输入了，虽然没有显示输入法的状态栏，但是确实可用了。在
某些嵌入式环境中，这种方式可以用于开机启动某个或某几个图形程序，然后后台还可以加一个检测进程，
崩溃之后重新启动图形程序。

## ly关于Xsession的修改
ly是一个极简的display manager。使用pam作用户验证，使用ncurses提供界面，只做极少量的环境
设置。本人fork了一波，做了一些超简单的修改，让它能够在debian中编译运行。

但是使用中遇到了一点问题，比如evince或者nautilus之流的程序启动时间很慢，结果看log发现
是dbus的原因，然后google之发现来源在于dbus-update-activation-environment没运行，从而一些
dbus的环境变量没有设置好。但是实际上作为优良的发行版，debian做了相关的工作，除了这个dbus
问题的fix之外，还有其他一些修改，都放在`/etc/X11/Xsession.d`文件夹当中，由`/etc/X11/Xsession`
运行。问题在于ly没有运行这个脚本，所以就没有运行那些修复动作。所以，在ly的代码当中，运行
相关的x-window-manager之前，用`system("/bin/sh /etc/X11/Xsession")`运行一下相关的修
复脚本。fork修改后的[地址](https://github.com/linuxhenhao/ly)
