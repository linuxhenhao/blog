+++
title = "Debian系简易内核打包脚本"
date = 2017-11-27T20:29:02+08:00
tags = ["debian", "linux", "kernel package"]
categories = ['operating_system']
keywords = ['kernel deb package','debian']
+++

作为一个尝鲜党，经常会关注内核的新特性，喜欢在自己的电脑上装上最新的内核。维护了一个
较为精简的适用于自用笔记本和台式机的内核配置文件[github](https://github.com/linuxhenhao/config-files)。

既然经常追新，那么就免不了经常更新内核，使用系统本身的包管理系统是一个非常方便的选择。
但是又不喜欢安装太多的打包的`dpkg-package`工具，也没有时间去了解debian打包官方模式。
简单的使用`dpkg -b`打包文件为deb成了我的常用打包方式。配套的，为`arm|x86_64`两个架构
创建了打包脚本。 *误：发现内核自带了打包脚本，详情见具体文章内容*
<!--more-->
搞半天，发现我写的打包脚本生成的linux-header包很大，而且不够完善，在x86编译内核外模块时
会出现缺少文件的情况，然后发现内核自带了打包脚本，包括deb和rpm系。如果只需要二进制包不打包
源码包的话`make bindeb-pkg`即可。还有以下几个target：

    make deb-pkg  # 包含源码的deb包
    make rpm-pkg  # 生成包含源码包在内的rpm包
    make binrpm-pkg  # 生成不包含源码包在内的rpm包
虽然如此，但是之前的脚本工作倒也不是无用工，其实内核带的打包脚本差不太多，主要是对于linux-headers
包里面到底需要哪些文件不需要哪些文件的认识不够清晰全面。
---
其中的`linux-headers`包，为了让`arm`上`make script`能够成功，从而能够
在arm上编译对以内核的第三方模块，比通常的官方包多了很多文件。

注:script里面有很多二进制文件，没有cross_compile选项，所以只能安装到arm后编译，所以需要
能够完整的构建编译过程的文件。当然有些精简可以作，懒得扣太细了。

    #!/bin/bash

    # export cross compile envs if arm specified as a parameter
    if [ x"$1" == "x" ]; then
    echo "Usage: $0 <arm|default>"
    echo "      default means host platform"
    exit 0
    else
    case $1 in
        "arm")
            export ARCH=arm
            export CROSS_COMPILE=arm-linux-gnueabihf-
            ;;
        "default")
            echo $(uname -m)
            ARCH=$(uname -m)
            ;;
    esac
    fi

    function GetPKGArch()
    {
    if [ $(uname -m) == "x86_64" ];then
    echo "amd64"
    else
    echo "i386"
    fi
    }

    function DirCheck()
    {
    if [ ! -d $1 ];then
    mkdir -p $1
    fi
    }

    function fileCheck()
    {
    if [ -e $1 ]; then
        rm -rf $1
    fi
    }

    # copy with parents path, only can do this in kernel source dir
    function copy_fullpath()
    {
    find $1 -name "*.h" -exec cp --parents {} $2 \;
    find $1 -name "*.conf" -exec cp --parents {} $2 \;
    find $1 -name "Makefile*" -exec cp --parents {} $2 \;
    find $1 -name "Kbuild*" -exec cp --parents {} $2 \;
    find $1 -name "Kconfig*" -exec cp --parents {} $2 \;
    # find $1 -type f -executable -exec cp --parents {} $2 \;
    }
    # copy essensial header files to dest dir
    # eg: headers_prepare /usr/src/linux-headers-$VERSION
    function headers_prepare()
    {
    # copy to build headers deb
    # cp -r ./include ${HDR_DIR}/usr/src/linux-headers-$VERSION
    # cp ./.kernelvariables $1
    make prepare
    make scripts
    # make scripts will produce include/config/auto.conf
    # which is needed by the build process of any external module
    cp ./.config $1
    cp ./Module.symvers $1
    cp --parents ./arch/Kconfig  $1
    cp --parents ./include/config/kernel.release $1

    cp ./Makefile $1
    #cp ./Kbuild $1
    cp ./Kconfig $1

    # copy dir in include to $1
    # copy_fullpath src dst
    dirs=(block certs crypto drivers firmware fs include \
        init ipc kernel lib mm net samples security sound \
        tools usr virt)
    # dirs=(include)
    for dir in ${dirs[@]};
    do
        # copy executables/makefile/kbuild/kconfig/.h
        echo "copy_full path $dir $1"
        copy_fullpath $dir $1/
    done

    # for arch dir, only copy targed arch dir for cross compile
    #if [ "x$ARCH" == "xx86_64" ]; then
    #    HDR_ARCH=.  # copy other architectures' headers for
    #                # cross compile
    #else
    #    HDR_ARCH=$ARCH
    #fi
    #copy_fullpath ./arch/${HDR_ARCH} $1/
    copy_fullpath ./arch/ $1/

    cp -r scripts $1/
    }


    RELEASE_FILE=include/config/kernel.release
    if [ -e $RELEASE_FILE ]; then
    VERSION=$(cat include/config/kernel.release)
    else
    echo "make prepare and then make, this will build the release file needed "
    exit 0
    fi
    INSTALL_PATH=./linux-image-$VERSION/boot/
    INSTALL_MOD_PATH=./linux-modules-$VERSION/lib/
    HDR_PATH=./linux-headers-$VERSION/usr/src/linux-headers-$VERSION
    IMG_DIR=./linux-image-$VERSION
    MOD_DIR=./linux-modules-$VERSION
    HDR_DIR=./linux-headers-$VERSION



    if [ "x$ARCH" != "xarm" ];then
    PKG_ARCH=$( GetPKGArch )
    ARCH_ALIA=${PKG_ARCH}
    UPDATE_GRUB="update-grub"
    KERNEL_DEPS=""
    MAKE_UINITRD=""
    else
    ARCH_ALIA=${ARCH}hf
    UPDATE_GRUB=""
    KERNEL_DEPS="u-boot-tools"
    MAKE_UINITRD="genuinitrd"
    fi


    echo $ARCH


    DirCheck ${IMG_DIR}/boot
    DirCheck ${IMG_DIR}/DEBIAN
    DirCheck ${HDR_DIR}/usr/src/linux-headers-$VERSION/
    DirCheck ${HDR_DIR}/DEBIAN

    DirCheck ${MOD_DIR}/lib
    DirCheck ${MOD_DIR}/DEBIAN
    rm -rf ${INSTALL_PATH}/*
    rm -rf ${INSTALL_MOD_PATH}/*

    fileCheck ./${IMG_DIR}/DEBIAN/control
    cat >./${IMG_DIR}/DEBIAN/control<<EOF
    Package: linux-image-$VERSION
    Version: $VERSION
    Architecture: $ARCH_ALIA
    Depends: $KERNEL_DEPS
    Maintainer: huangyu<diwang90@gmail.com>
    Description: linux kernel
    EOF


    fileCheck ./${IMG_DIR}/DEBIAN/postinst
    cat >./${IMG_DIR}/DEBIAN/postinst<<EOF
    #!/bin/bash
    function genuinitrd()
    {
    cp /boot/initrd.img-$VERSION /tmp
    cd /tmp
    mkimage -A arm -T ramdisk -C none -n uInitrd -d initrd.img-$VERSION uinitrd
    cp uinitrd /boot/initrd.img-$VERSION
    }
    if [ -e /boot/initrd.img-$VERSION ];then
    rm /boot/initrd.img-$VERSION
    update-initramfs -k $VERSION -c
    fi
    EOF

    chmod 0555 ./${IMG_DIR}/DEBIAN/postinst

    fileCheck ./${IMG_DIR}/DEBIAN/postrm
    cat >./${IMG_DIR}/DEBIAN/postrm<<EOF
    #!/bin/bash
    ${UPDATE_GRUB}
    EOF

    chmod 0555 ./${IMG_DIR}/DEBIAN/postrm


    fileCheck ./${MOD_DIR}/DEBIAN/control
    cat >./${MOD_DIR}/DEBIAN/control<<EOF
    Package: linux-modules-$VERSION
    Version: $VERSION
    Architecture: $ARCH_ALIA
    Depends: linux-image-$VERSION
    Maintainer: huangyu<diwang90@gmail.com>
    Description: linux kernel modules
    EOF


    fileCheck ./${MOD_DIR}/DEBIAN/postinst
    cat >./${MOD_DIR}/DEBIAN/postinst<<EOF
    #!/bin/bash
    function genuinitrd()
    {
    cp /boot/initrd.img-$VERSION /tmp
    cd /tmp
    mkimage -A arm -T ramdisk -C none -n uInitrd -d initrd.img-$VERSION uinitrd
    cp uinitrd /boot/initrd.img-$VERSION
    }
    if [ -e /boot/initrd.img-$VERSION ];then
    rm /boot/initrd.img-$VERSION
    fi
    update-initramfs -k $VERSION -c
    ${MAKE_UINITRD}
    ${UPDATE_GRUB}
    EOF

    fileCheck ./${HDR_DIR}/DEBIAN/control
    cat >./${HDR_DIR}/DEBIAN/control<<EOF
    Package: linux-headers-$VERSION
    Version: $VERSION
    Architecture: $ARCH_ALIA
    Depends: linux-image-$VERSION, linux-modules-$VERSION
    Maintainer: huangyu<diwang90@gmail.com>
    Description: linux kernel headers
    EOF

    fileCheck ./${HDR_DIR}/DEBIAN/postinst
    cat >./${HDR_DIR}/DEBIAN/postinst<<EOF
    #!/bin/bash
    if [ -e /lib/modules/$VERSION ];then
    ln -sf /usr/src/linux-headers-$VERSION /lib/modules/$VERSION/build
    ln -sf /usr/src/linux-headers-$VERSION /lib/modules/$VERSION/source
    fi
    EOF

    chmod 0555 ./${MOD_DIR}/DEBIAN/postinst
    chmod 0555 ./${HDR_DIR}/DEBIAN/postinst

    INSTALL_PATH=./${IMG_DIR}/boot/ make install
    INSTALL_MOD_PATH=./${MOD_DIR}/ make modules_install

    # headers file prepare
    headers_prepare ${HDR_PATH}

    #find ./scripts -type f -executable -exec cp --parents {} ${KBUILD_PATH} \;  
    # copy to build kbuild

    # delete firmware, get them from official repo
    if [ -e ${MOD_DIR}/lib/firmware ]; then
    rm -r ${MOD_DIR}/lib/firmware
    fi
    # delete symlink to this compile dir
    rm  ${MOD_DIR}/lib/modules/$VERSION/build
    rm  ${MOD_DIR}/lib/modules/$VERSION/source

    # for arm kernel, using zImage and uInitrd
    if [ "x$ARCH" == "xarm" ];then
    cp arch/arm/boot/zImage ${INSTALL_PATH}/vmlinuz-$VERSION
    fi


    sudo chown -R root:root ${IMG_DIR} ${MOD_DIR} ${HDR_DIR}
    dpkg -b ${IMG_DIR}
    dpkg -b ${MOD_DIR}
    dpkg -b ${HDR_DIR}


更新：2018-06-18：添加关于内核自带打包脚本的内容
