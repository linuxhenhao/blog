+++
title = "Python软件打包分发"
date = 2018-01-22T20:11:12+08:00
tags = ["python", "打包"]
categories = ['programming']
keywords = ['python','打包']
markup = "goldmark"
+++
使用PySide写的软件需要打包分发，需要满足在其他机器没有安装python，自然
也就没有安装代码中所用到的一系列第三方包的情况下，能够将依赖完整的打包，
并制作成可安装文件，采用的方案为PyInstaller+InstallForge
<!--more-->
## 1. PyInstaller
[PyInstaller][1]在windows平台上会用到PyWin32，这是一款非常简单好用的
打包软件，支持PyQt和PySide。运行`pyinstaller  xxx.py`会分析import以及
import的python文件里的import，从而将运行`xxx.py`所需的依赖打包。用到的
项目类的纯py文件都能够自动导入，而第三方的包含额外二进制库的包的导入需要一些
额外的工作，所幸pyinstaller支持了不少第三方包了。打包时所有的py文件都会被
生成相应的pyc文件然后打包，不会包含原始的py明文文件。

### 1.1 打包模式
pyinstaller打包有单文件模式和文件夹模式，其中单文件模式运行时会将所有的文件
解压到一个临时文件夹运行。文件夹模式会将所有打包文件放置在一个文件夹中，还会
包含一个exe文件用于启动软件。项目或者说package中的py文件都会被编译成python
byetcode，然后和bootloader嵌在一起，组成单个exe文件，而其他的库二进制文件
会被放在同这个exe文件同一文件夹下（单文件模式时会在解压的临时文件夹下）。
由于变成单个文件，原来在package类具备的文件夹结构都消失了，代码中的import
都被PyInstaller用PEP302的import hooks处理过，使其能够正常载入模块。如果
原文件中使用了`__file__`来判断路径，也会由于这个处理出问题，需要注意。

### 1.2 隐藏启动cmd窗口
打包时可以用`--windowed`参数，指定打包为窗口模式，运行软件时不显示黑色的cmd
窗口，这在一切调试完毕发布软件时是应该使用的参数。如打包`main.py`

    pyinstaller --windowed main.py

如果需要显示console信息用于调试和监控软件运行状态，则不能使用此参数：

    pyinstaller --windowed main.py

### 1.3 包含其他非py文件
在软件项目中往往会用到一些非py的文件，如图片，此时就需要手工将所需的文件包含到打包
当中；使用`--add-data "src;dst"`其中的`;`是windows平台下的分隔符，`:`是*nix
平台下的分隔符。如

    pyinstaller --windowed --add-data "soft\resources;resources" main.py

该命令会将运行pyinstall所在文件夹下的`soft\resources`文件夹复制到打包目的文件夹下，
名字为`resources`。

### 1.4 XP系统下PyInstaller的安装和代码加密
由于软件需要支持XP系统，所以只能使用pyton3.4，但是直接使用pip安装pyinstaller时发现
pypi里已经没有支持该版本的pywin32包了。解决方案也很简单，到pywin32的github项目主页
的release标签下找到最新的还包含32位python3.4的安装包，安装之后重新用pip安装pyinstaller。

尽管pyinstaller打包的是字节码文件，也就是python interpreter能够识别的`汇编代码`，但是
还是很容易被逆向的，所以pyinstaller还提供了加密选项`--key xxx`指定密码，会使用该密码加密
代码，在运行exe时实时解密，在牺牲一部分启动时间的情况下能增强代码的安全性。但是由于运行时
用户无感，因此exe文件本身必然包含了解密所需的东西，完全靠它来保密代码是不可靠的。

### 1.5 使用spec文件
spec文件是对打包过程的描述文件，可以将参数全部保存在spec文件当中，修改spec文件内容以修改
打包过程，在需要打包时只需要运行

    pyinstaller xxx.spec

spec文件可以用上述的打包命令添加`--specpath`指定spec文件保存的文件夹。

    pyinstaller --specpath . --windowed --add-data "a;a" main.py
上述命令会在当前文件夹下生成打包main.py过程的参数信息的main.spec，如果下次需要重复该打包
过程，只需直接运行

    pyinstaller main.spec
可以修改main.spec文件中的内容改变打包的参数。比如:`datas=[('a','a')]`中包含的是所有的额
外文件的打包信息，需要添加额外打包的文件，只需要在该list中添加新的`(src,dst)`即可。修改
`exe=`后面的`console=False`的值来更改打包后的exe运行时是否显示console窗口。

### 1.6 非`import`载入的包(importlib.import)
在使用非标准的`import`或者`import from`载入模块时，PyInstaller无法自动找到这些需要的模块
文件。PyInstaller通过入口py文件，递归的检查所有的上述两种导入的包，并打包相关的非系统
路径（如linux下的/lib，/usr/lib）下的二进制依赖。如果通过`__import__`或者`importlib.import`
方式导入文件，则需要手动添加。有两种方式，一种是上述的修改spec文件的datas参数，在其中添加
需要的tuple。另一种是通过`--hidden-import`命令参数方式添加。其中第一种需要注意，由于py文件
被嵌入到一个exe文件当中，所有源文件中的import都被hook过，import的查找顺序为1. exe文件中
2\. 当前文件夹 3\. 运行文件夹；比如，需要import abc，首先看exe文件中是否包含该文件，如无，
则看exe一块的同一文件夹下的abc.py或者是`abc/__init__.py`，最后是运行该exe命令所在文件夹
下的abc.py或者abc包。

--hidden-import=参数在只有一两个模块的时候还可以，如果模块多的话用起来不方便，使用hooks
会舒服。PyInstaller的hook文件就是一个python文件，它的hook分两种，一种修改运行时的行为，
一种修改分析打包所需文件的行为。这里用到的当然是修改打包行为的hook。

PyInstaller在分析所需文件时会在所有的import或者from import分析进行时，在指定的hooks文件
夹里查找是否有对应的hook文件，如果有，载入执行。文件的对应方式为`hook-<name>.py`，如在
`import abc.d`或者`from abc import d`时，查找hooks文件夹，发现了`hook-abc.d.py`，那么就
会执行这个hook。为了添加额外的模块，需要使用`hiddenimports = []`，只需要在这个list中添加
所需要额外载入的模块即可。如`hiddenimports = ['socket']`会被PyInstaller当成python文件中
包含`import socket`指令来打包所需的文件。hooks文件夹的位置一个是PyInstaller自带的位置，
一般额外的针对项目的hooks通过`--additional-hooks-dir=`来指定。

__官方文档错误__，这里由一个官方文档错误，文档中用了`['dns.rdtypes.*']`这种方式，但是
python3的语法已经不支持`import dns.rdtypes.*`这种方式了，所以不能用这种通配符指定了。

具体例子：现在有一个包`A`，通过一个`run.py`运行。
run.py中通过`from A import main`来导入main函数运行。
如果A中的`B`文件夹有一系列的`py`文件，未被显式import，这时，添加一个hook-A.main.py文件，内
容为`hiddenimports = ['A.B.*']`，那么PyInstaller会当作分析的包对象中有`import A.B.*`这么
一句的方式来打包依赖。

PS: 然而，实际在使用hook-<name>.py模式添加hiddenimports的时候，提示`WARNING: Hidden import
 not found!`，特地去看了github上PyInstaller的源码，发现对于python3，载入方式就是绝对路径
 载入，而由于需要载入的模块的包在当前运行命令文件夹下，如上述的`A`，`A.B.*`的例子，添加
 `A.B.xxx.py`系列到hiddenimports的list内应该可以正确导入才对，可实际上就是出错。__最终通过
 修改spec文件中的hiddenimports内容正常找到了需要的包__，这应该是个bug吧，绝对路径导入，
 给的绝对路径，就是找不着。


## 2. InstallForge
在打包成文件夹后，如果只使用压缩文件分发，会显得很low，这时[InstallForge][2]就派上用场了，
这是一个简单易用的将文件文件夹打包成一个exe安装文件，在运行时可以指定安装位置，添加快捷方式，
并且会留下一个卸载的可运行文件放于安装目标文件夹类。

由于使用很简单，具体就不细说了，两点1. 在配置完打包的文件后可以将打包的配置保存，可以避免下次
打包需要重新添加文件。2. 添加文件和文件夹要分别进行，文件夹只能一次选一个，而文件可以一次多选。

在用PyInstaller和InstallForge打包完成后，一个python项目对应的像模像样的exe的安装包就生成
了：）。

## 3. InnoSetUP
[InnoSetup][3]是InstallForge同类型的打包软件，之前使用InstallForge在XP上打包的exe安装
包无法在Win7系统上安装。InnoSetup更新比较多，实时的跟进系统支持，对于Win7以及之上的win10
都有很好的支持，并且在本应用场景简单打包整个文件夹比InstallForge更好用，整体来说功能更强
一些。比较著名的本地文件夹同步软件FreeFileSync就是用InnoSetup打包的。使用时打开一个简单
的example.iss脚本，修改适应需要打包的内容，比如源文件夹，默认安装位置，exe输出位置，运行
的exe文件名称、路径等，然后build即可，还支持多语言安装包，可以对安装界面进行本地化。

修改记录：
    2018.02.02：添加非import载入包的处理和PyInstaller打包软件import查找的顺序
    2018.03.30：添加关于innosetup打包的内容

[1]:http://www.pyinstaller.org
[2]:http://www.installforge.net
[3]:http://www.jrsoftware.org/isinfo.php
