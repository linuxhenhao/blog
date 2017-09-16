---
date: "2017-09-16T09:17:19+08:00"
categories:
- programming
tags:
- twisted
- 分层
keywords:
- 分层
- 抽象
- twisted
#thumbnailImage: //example.com/image.jpg
---

## 缘由
最近在项目中有一个需求，将其进行前后端分离。其中一个重点是后端，也就是server端，需要不停
的向前端client发送数据。考虑到将来跨平台的简便性，基于浏览器广泛支持的websocket是非常好的
选择。  
但是一个额外的强制要求限制了该需求的实现，那就是对XP系统的支持，没错2017年对XP系统的支持。
软件界面使用的是PySide+python3.4实现的。而PySide中并没有对Websocket的支持，在PyQt5或者
PySide2中已经包含了webosocket的实现，但是无法在XP系统中运行。
<!--more-->
## 寻找相关库
用python的一个最大的好处是很多你想要的功能都已经有人用库实现了，只需要根据需求寻找合适的
库就行。所以，开始了寻库之旅。ws4py、websocket、websockets等等，找到了许多python关于
websocket的库，这些库本身也很优秀，用得人也不少。但是一圈下来，都不满足我的需求。因为有
一个基本点，需要和Qt的eventloop结合，运行在一个线程当中，而这些库都严重直接依赖于要么
是asyncio的loop，要么严重依赖python标准库中的socket模块。无法和qt的loop有机结合在一起
运行。

## 春天
但是功夫不负有心人，最终还是找到了合适的实现方式，那就是twisted。twsited的设计十分优秀，
用reactor（提供eventloop，控制运行）、protocol（对通信的具体协议实现，响应、生成发送数据）、
factory（对每个连接生成protocol实例，保存超过protocol生成周期的数据）、endpoints（具体
的transport实现，处理连接细节，发送、获取数据）。对每一个角色做了很好的抽象和分层，它们各自
只做各自的事情，实现事先制定好的接口规范，对于不该知道的细节绝不处理。这种良好的设计，使得
其各个部分的实现可重用性极好，可以适应各种不同的需求。  
在本项目的需求中，只需要将twisted的reactor和qt的eventloop结合起来，那么其他部分，当然
包括twisted实现的websocket协议，都可以随意使用，不用作任何更改。而reactor在设计接口时
本身就做好了和各种eventloop相结合的准备，并且早有人做好了相关的工作[qtreactor](https://github.com/ghtdak/qtreactor)。
这个reactor支持PyQt4和PySide，使用Qt的Eventloop作为reactor。reactor的工作就是根据监听
的socket的状态调用不同的接口作出相应的处理，通过QSocketNotifier实现了qteventloop和twisted
的联动，用qteventloop来驱动twisted体系的运行。

## 总结
在计算机体系当中，分层抽象随处可见，而从twisted的设计当中，可以发现良好的分层抽象对于代码
的可重用性、对复杂环境的适应性都有极大的作用，其他那些不能在本场景中使用的代码都是因为抽象
不够，和底层细节结合太紧密，导致可迁移性差（当然在其各自适用的环境中很棒！）。在以后写代码
和设计的时候应当借鉴，分好角色，将细节底层的东西作为一类，作出抽象，其他的组件只调用抽象的
接口，从而和细节解耦。
