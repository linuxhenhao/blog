+++
categories = ["multimedia"]
date = "2017-06-07T22:23:11+08:00"
keywords = ["视频压缩", "ffmpeg"]
draft = false
title = "ffmpeg压缩视频文件"

+++

手机拍摄的视频默认格式是mov，用ffmpeg查看可以发现使用的是
libx264压缩格式，为了进一步的节约空间，将视频复制到电脑上
之后，将其使用libx265的方式压缩，可以节约很大一部分存储空间。
<!--more-->

一个普通的ios 10拍摄的4：35s的视频，默认情况下大小达到了600M，
使用libx265压缩之后，变成了60M，并且肉眼看起来视频质量没有区别。
这应该不是简单的libx264到libx265的压缩方式导致的区别，还有一些
其他原因，比如默认的`csr`选项值为23,而不是最高的`0`，但是对于
实际使用来说，只要没有明显的质量区别就可以了。


经测试，一个300M的libx264的视频，用下面的命令转码，
使用crf值为51时，输出只有4.6M，但是视频质量很模糊。<br/>
使用crf值为0 时，输出为2.3G，比原来的视频还大。
使用crf值为26时，输出为，视频质量与源视频相当。

压缩使用的命令：
```
ffmpeg -i i.mov  -vcodec libx265 -acodec copy  output.mp4
```
命令很简单，-i指定输入，-vcodec指定输出的encoder，-acodec指定
音频的格式，这里不需要改变，大小主要由视频影响，最后是输出文件
名。

在转码的时候发现速度很慢，想使用显卡加速，使用以下命令
```
ffmpeg -i i.mov -hwaccel vdpau -vcodec libx265 -acodec copy  output.mp4
```
`-hwaccel`硬件加速，但是实际运行该命令发现行不通，没有使用显卡
加速，cpu占用很高。于是apt-get source ffmpeg，查看ffmpeg的configure文件，
发现显卡加速encode是有条件的，必须使用相应的-vcodec，如h264_vaapi, mjpeg_vaapi, hevc_vaapi，这是debian testing默认的编译参数编译的ffmpeg支持硬件encoder的几个格式，
libx265并不支持硬件encode，所以老老实使用cpu压制。

这是一段关于libx264,libx265的crf参数的解释
```
The range of the quantizer scale is 0-51: where 0 is lossless, 23 is default, and 51 is worst possible. A lower value is a higher quality and a subjectively sane range is 18-28. Consider 18 to be visually lossless or nearly so: it should look the same or nearly the same as the input but it isn't technically lossless.
```
<!--more-->
