+++
title = "编译安装python3.4.7 For XP"
date = "2017-09-20T21:47:34+08:00"
categories = ["python", "programming"]
markup = "mmark"
+++

## 前言
因为项目要求，必须支持XP系统。偶尔翻[python官网](https://www.python.org)支持Xp的最新的
python版本已经到了3.4.7，都是一些安全性的更新。但是官方如今不再提供直接可以安装的binary
文件下载，需要自己编译安装，因此做个记录。参考[官方wiki](https://wiki.python.org/moin/VS2010)
和[python-doc](https://docs.python.org/devguide/setup.html#windows-compiling)
[openssl wiki](https://wiki.openssl.org/index.php/Compilation_and_Installation)

<!--more-->
## 准备
### Visual Studio 2010 pro
可以在XP上运行的最高版本的vs，并且支持使用PCbuild中的方式，也就是新的python编译安装脚
本运行，因此首先需要下载安装此编译环境。安装的过程中，安装VC++即可，其他的诸如F#、C#、
office、SQL的相关东西可以自定义安装的时候取消勾选。从[MSDN itellyou][2]下载会比官网快。
安装完成后需要将vs stuido安装文件夹中的`VC\bin`添加到环境变量当中。使我们可以直接在cmd
中运行nmake之类的命令。

### [tortoiseSVN][1]
需要通过svn下载一些external source，脚本里面推荐使用的就是这个。注意，1.9以及之后的版
本不支持XP，因此需要下载[1.8版本][3]

### [perl][4]

编译openssl时需要perl，这里下载的是Strawberry perl。

## 编译
0. 所有的编译，都应该在上述软件安装完成后，重新打开cmd，以获得新的PATH。python的源码文件夹所在
的路径不能包含空格，否则会在编译tcl时失败。
1. 运行`PCbuild\build.bat -e`，加的参数会调用`get_externals.bat`并且编译外部依赖，包含tk、tcl、
openssl等库，完成之后在PCbuild文件夹下得到一个可以运行的python.exe
2. 打包，在编译完成之后希望能够打包成官方发布的msi形式的安装包，用于在其他电脑上安装。但是找了较长
时间，找到一些过时的文档，但是找到了`tools\msi\msi.py`这么个脚本，可是这里面的内容还是用python2
写的，所以为了打包编译出来的python3.4.7，又安装了一遍python2.7
  2.1 安装python2.7
  2.2 通过`get-pip.py`安装`pip`，将`python27/Scripts`文件夹添加到系统环境变量，使pip可以从cmd运行。
  2.3 从[微软官网下载][7] VC用于编译某些pip下载安装需要编译的模块。
  2.4 安装pywin32模块。运行`pip install pypiwin32`安装此模块。
  2.5 还需要安装[cabarc][6]。
  2.6 `msi.py`依赖了一些`hg manifest`来获取所有文件名列表，用于添加合适的文件到安装包。实际上，cpython
  的开发已经迁移到了git，下载过来的源码包里面并不包含一个`.hg`文件夹，`hg manifest`是无法运行成功的。这
  里写了一个脚本，实现了`msi.py`中返回文件列表dict的功能，修改一下msi.py中的`hgmanifest`函数，import此
  脚本，返回`myHgmanifest.hgmanifest(srcdir)`的结果，即可实现目的。点击[下载脚本](/uploads/myHgmanifest.zip)，
  用压缩包中的脚本替换掉`Tools\msi`中相应的脚本即可。

3. 编译文档。在第2步完成后，用python27运行msi.py，发现缺少文档，打包失败，因此需要生成打包所需的文档。
   3.1 将python27安装目录中的`python.exe`复制重命名为`py.exe`，因为编译文档的bat文件中用了此命令，实际就是运行`python.exe`
   3.2 pip install sphinx，安装生成文档所需的模块。
   3.3 到Python3.4.7源码目录中的`\Doc\`文件夹下运行`make.bat htmlhelp`命令，生成帮助文档。

4. 生成msi安装包。运行`python msi.py`，最终，在`Tools\msi`文件夹下会生成python3.4.7.xxxx.msi的安装包，可以拿到别的电脑上安装去啦！

## 总结
可以发现，给自己编译的困难主要源于未被维护的msi.py打包脚本了，后续发布的python安装包都是通过另外的
方式打包的，所以需要做一些修复工作，主要是python源码包里已经没有.hg文件夹了，需要用另外的方式生成制作
打包文件所需的文件dict。总而言之，并不复杂，主要是一个逐渐发现问题、解决问题、发现新问题、再解决的过程。
所幸，最终得到了所需的安装包。

更新记录：
    2018.1.22 修复了msdn itell you的链接

[1]:https://www.tortoisesvn.net/downloads.html
[2]:https://msdn.itellyou.cn
[3]:https://sourceforge.net/projects/tortoisesvn/files/1.8.12/Application/
[4]:http://strawberryperl.com/
[5]:https://stackoverflow.com/questions/1154660/fatal-error-u1087-cannot-have-and-dependents-for-same-target)
[6]:https://www.microsoft.com/en-us/download/details.aspx?id=16818
[7]:http://aka.ms/vcpython27
