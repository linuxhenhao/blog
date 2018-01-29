+++
title = "Python软件打包分发"
date = 2018-01-22T20:11:12+08:00
tags = ["python", "打包"]
categories = ['program']
keywords = ['python','打包']
markup = "mmark"
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
包含一个exe文件用于启动软件。

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

## 2. InstallForge
在打包成文件夹后，如果只使用压缩文件分发，会显得很low，这时[InstallForge][2]就派上用场了，
这是一个简单易用的将文件文件夹打包成一个exe安装文件，在运行时可以指定安装位置，添加快捷方式，
并且会留下一个卸载的可运行文件放于安装目标文件夹类。

由于使用很简单，具体就不细说了，两点1. 在配置完打包的文件后可以将打包的配置保存，可以避免下次
打包需要重新添加文件。2. 添加文件和文件夹要分别进行，文件夹只能一次选一个，而文件可以一次多选。

在用PyInstaller和InstallForge打包完成后，一个python项目对应的像模像样的exe的安装包就生成
了：）。

[1]:http://www.pyinstaller.org
[2]:http://www.installforge.net