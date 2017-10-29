+++
title = "Linux下让某程序的所有数据包通过指定网口发出"
date = 2017-10-29T13:27:35+08:00
tags = ["route", "linux"]
categories = ['network', 'linux']
keywords = ['ip-rule','iptables']
+++
让某程序的所有数据通过指定网口发出。这是结合上一篇搭建的虚拟连接实现的特定功能，达到一些
目的——比如让没有WAN地址的服务通过vps具备的WAN地址提供交互（可以了解一下[frp][1]）。
<!--more-->
## iptables机制
iptables是linux常用的通讯管理工具，是内核netfilter组件的用户空间工具。使用iptables可以
实现各种各样的发包控制。而最关键的是知道IP包是如何在系统中穿梭的。如图所示
<img src='/images/post-images/tables_traverse.jpg' alt='packet traverse path'
style="width:60%">
iptables就是在包的流通过程中流过的链中添加一定的规则，从而对包作出一些处理以实现目标。
iptables有许多的hook点，通过netfilter提供的接口运行，比如OUTPUT、INPUT、FORWARDING、
POSTROUTIING，就在图中所描述的各个位置。而iptables根据应用的特点，分成几个类型，nat包转发、
mangle包修改、filter包过滤，在各个hook点包流过几类处理的顺序为mangle->nat->filter。
称各个OUTPUT、INPUT之类的为`chain`，可以在`chain`中添加`rule`，被一条`chain`处理完之后
才会被下一条`chain`处理。在`chain`中流过`rule`的顺序是，先流过先添加的`rule`。当然，如果
某`rule`匹配了包，并作出了`DROP`处理，这个包就被丢弃，后面所有的过程都不进行了，如果进行
`RETURN`，则分两种情况，如果是在OUTPUT、INPUT等顶级的`chain`中`RETURN`，则直接跳过该链
中后续的所有`rule`，直接设为该链的默认状态，比如`PREROUTING`一般为`ACCEPT`。如果是从
上述的顶级链中跳转到的次级甚至更低级的链中`RETURN`，则返回上级链进行跳转的位置，并继续
流过上级链的`rule`。

## wireguard的fwmark
iptables的mangle->nat->filter流通过程中，可以对包添加mark，用以区分包，从而对不同特征
的包作不同的处理，mark只在内核里面有，在从要网卡发出时会将其去除。wireguard也具备添加mark
的能力，但是它的添加方式是直接在源码里面实现，未利用iptables，直接在生成的包中添加了mark。
也就是在图中的`local process`部分生成的包就包含了mark，可以在后续的`OUTPUT`以及`POSTROUTING`
链中做一些其他的处理。通过以下命令可以验证：

    iptables -t mangle -A OUTPUT -m mark --mark $wg_mark -j DROP
由于`mangle`是从内核空间出来后遇到的第一个chain，在此DROP掉所有包含wireguard的mark的包，
果然就无法ping通server端了。这就说明，wireguard的fwmark是直接在构建包的时候就添加的，而
不是利用iptables在后续过程添加的。

## 包转发场景
client，与server之间通过虚拟连接连接，IP为`10.0.0.3`，网口名`vlan3`, 本地具备路由器分配的
地址`192.168.1.3`用以普通通信，网口名`eth3`。   
server， 虚拟连接IP为`10.0.0.1`，网口名`vlan1`并且规定，从client发来的包只能为`10.0.0.0/24`网段，具备
公网IP: `111.100.13.92`， 网口名`eth1`   
需求： 将client中的程序A的所有包的接收和发送通过虚拟连接进行。

### client端
1\. 将所有从`vlan3`接收的包从`vlan3`返回

    ip rule add iif vlan3 table 3
    ip route add table 3 via 10.0.0.1
2\. 让所有A发出的包通过`vlan3`发出   
2.1 由于`iptalbes`已经不支持以发出包的程序`PID`来匹配包，采用发出包的`UID`来进行匹配，
新建一个用户专门用来运行程序A，我们称之为`UserA`

    useradd UserA
2.2 通过`iptables`对`UserA`发出的包添加`mark`，使用`ip-rule`根据mark指定路由表，在路由
表中（也就是上面的`table 3`），所有的包默认通过`vlan3`发出。

    iptables -t mangle -A OUTPUT -m owner --uid-owner UserA -j MARK --set-mark 1
    # 可以直接使用用户名匹配，给所有该用户发出的包打上`1`标签
    ip rule add fwmark 1 table 3
    # 让所有的`1`标签的包通过路由表3路由

PS：其实正常来说，上面几步做完应该就OK了，但是实际中发现怎么都无法连接，切换到`UserA`下
进行网络连接都连不上，所以需要进一步的debug。用到了`iptables`中的`LOG`目标。

    iptables -t mangle -A OUTPUT -m owner --uid-owner UserA -j LOG --log-prefix "owner-matched: "
    # 在上述的mangle型的OUTPUT链中再添加一条LOG记录， log-prefix是自定义log信息的开头
进行该项操作后，再用`UserA`进行网络请求，通过`dmesg`查看iptables记录的LOG，发现确实是
匹配到了用户请求，但是`Src`项竟然是`192.168.1.3`出口也是网卡`eth3`。接下来看看，
在OUTPUT之后的`route decision`之后发生了什么

    iptables -t mangle -A  POSTROUTING -m mark --mark 1 -j LOG  --log-prefix "postrouting: "
发现由于OUTPUT中`mark`的设置，路由确实是使用了`table3`， 出口变成了`vlan3`，但是`Src`并没有发生
改变，仍然是`192.168.1.3`，也就是说，虽然在`OUTPUT`之后再次`route decision`时根据路由表更改了
出口网卡，但是并不会对源地址进行更改，所以我们需要人为的处理一下。

    iptables -t nat -A POSTROUTING -m mark --mark 1 -j SNAT --to-source 10.0.0.3
再次以用户`UserA`进行网络请求，发现可以正常运作了。

## server端
server端相比就简单许多，由于他的路由表的默认路由就是我们的公网IP，首先为了让`10.0.0.1/24`
方面的包也会通过该IP转发，需要开启kernel的forwarding选项。编辑`/etc/sysctl.conf`，添加

    net.ipv4.conf.all.forwarding = 1
然后还需要做一个源地址变换，iptables有个MASQUERADE，可以不用人为选取替换的目标IP，而会
智能的选用系统用来连接外部网络的网址进行替换：

    iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE
到此，所有从client发出的包都能够从server发出到外网了，但是外网却不能主动发包到内网。
加入client的程序A在端口1000监听，等待外网的主动请求，那么在server端作以下转发即可：

    iptables -t nat -A PREROUTING -p tcp --dport 1000 -j DNAT --to 10.0.0.3:1000
    iptables -t nat -A PREROUTING -p udp --dport 1000 -j DNAT --to 10.0.0.3:1000
这样在紧跟PREROUTING后的`route decision`中，会进行`forwarding`，将其通过虚拟连接转发给
client。完美！


[1]:https://github.com/fatedier/frp
