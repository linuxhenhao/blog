+++
title = "XMind打包脚本"
date = 2018-05-21T21:52:09+08:00
tags = ["deb", "xmind"]
categories = ['operating_system']
keywords = ['XMind','deb package']
markup = "goldmark"
+++
[xmind][1]是非常好用的基于eclipse的mindmap软件，由于java天然的跨平台特性，xmind也是支持
linux系统的。但是由于官方人员的懒，或者是官方的人力不足的原因，又或者是linux用户太少（最有可能）,
官方之提供了tgz的压缩包，解压后使用，不太方便。我更喜欢用统一的系统包管理的方式来管理软件，因此
写了一个简单的打包脚本。
<!--more-->
## java运行时选择的坑|oracle-java
遇到一个问题，无论是openjdk的8还是10的版本，运行XMind都会出错，下载了oracle的jre8之后成功运行
起来。debian下有个叫java-package的包，安装之后有一个`make-jpkg`命令，可以将下载来的oracle的
java打包成deb，然后安装。

## 脚本
这个脚本也是受这个包的刺激，xmind几乎是每次装系统之后的必备，之前手动做过几次deb包，这次看到
`make-jpkg`这个东西，干脆也做一个好了：）
```~bash
#!/bin/bash
if [ ! $# -eq 2 ];then
    echo "Usage: $0 path/to/xmind.zip VERSION_NUM"
    exit 0
fi


function arch(){
    ar=$(uname -m)
    if [ x$ar == "xx86_64" ];then
        echo amd64
    else
        echo i386
    fi
}

curDir=$(pwd)
tmpDir=/tmp/xmind-$(date +%M%H%S)
mkdir -p $tmpDir

# change workdir to tmpDir
cd $tmpDir

mkdir -p opt/xmind/
mkdir -p usr/share/applications/
mkdir DEBIAN

# gen control file for deb package
# oracle jre > 8 is requred, openjdk cannot run this
# software correctly
cat > DEBIAN/control <<EOF
Package: xmind
Version: $2
Architecture: amd64
Maintainer: Yu Huang <diwang90@gmail.com>
Depends: libgtk2.0-0, libwebkitgtk-1.0-0, lame, libc6, libglib2.0-0
description: great mind map software
Homepage: https://www.xmindchina.net/
EOF

# gen .desktop file for xmind
cat > usr/share/applications/xmind.desktop <<EOF
[Desktop Entry]
Name=xmind
GenericName=Mindmap creator
Exec=/opt/xmind/XMind_$(arch)/XMind %F
Terminal=false
Type=Application
Icon=/opt/xmind/xmind.png
StartupNotify=false
EOF
cd opt/xmind
# extract all files in xmind.zip into this dir
7z x $1

# download logo
wget http://www.xmindchina.net/uploads/images/xmind/logo.png -O xmind.png

# from opt/xmind to opt/xmind/XMind_$arch
cd XMind_$(arch)
# update XMind.ini
sed -i "s/\.\.\/workspace/@user\.home\/\.xmind\/workspace/" XMind.ini
sed -i "s/\.\/configuration/\/opt\/xmind\/XMind_$(arch)\/configuration/"  XMind.ini

cd ../../../
dpkg -b $tmpDir
mv ${tmpDir}.deb ${curDir}/xmind-$2.deb


```

[1]: http://www.xmindchina.net
