+++
title = "记wireguard的ip Rule配置问题"
date = 2017-10-28T21:24:28+08:00
tags = ["route", "linux"]
categories = ['network', 'linux']
keywords = ['ip-rule','wireguard']
+++
虽然学过计算机网络, 但是实践却并不多, 这里以wireguard建立的连接为例, 实践了一下路由配置,
更详细的了解了linux系统下路由的体系.
<!--more-->
## 几个概念
### 多路由表
linux系统在2.1、 2.2 版本内核就支持了多路由表。简单来说，就是系统中同时存在多个路由表
（这个解释贼6）。不同的路由表同时存在，通过ip rule来进行配置，决定某个包由哪个路由表来
进行路由。

### ip-rule
这是linux下iproute2工具中的一个，用以配置决定多路由表情况下的包路由。   
格式
  ip rule add SELECTOR ACTION
SELECTOR是用来匹配的条件， ACTION是匹配之后的动作。一个常见的场景是，多网卡情况下，
比如两个网卡`eth0, 192.168.0.3`， `eth1, 10.0.0.8`， 路由情况为   

    default via 192.168.0.1 dev eth0
    192.168.0.1/24 dev eth0
    10.0.0.8/24 dev eth1
在这种情况下，如果`eth1`上收到来自某处的包，比如`111.11.1.1`发来一个请求，通过`eth1`到达
系统，在进行回复时，由于默认路由的配置，会自动走`eth0`。但是这往往不是我们想要的结果。Yeah！
这个时后我们的ip rule要发挥作用了！！！

    ip rule add iif eth1 table 111
    ip route add default via 10.0.0.1 dev eth1 table 111
第一条命令，所有从`eth1`进来的连接都采用`table 111`这个路由表进行路由。   
第二条命令，添加路由规则到`table 111`中，这个表某人所有的包都从`eth1`发出。   
在这样之后，所有从`eth1`来的访问，回复的包也会从`eth1`发出。

## 问题来了
对的，那么问题来了，这些和wireguard由哪几毛钱的关系？由于众所周知的原因，我们这里称呼
wireguard为‘虚拟连接工具’，是众多‘虚拟连接’实现方式的一种。最近发现的wireguard，发现
这个家伙配置起来十分的简单方便，相比其他的各种虚拟连接工具，比如l2tp、IPSec容易太多。
所以实践了一下，很容易的达到了连接的目的。并且，这个工具最诱人的一点是，建立虚拟连接的
两点只需要一方具备WAN IP地址，这对于目前国内几大ISP很难获取到公网IP的情况十分适用。

### 简单配置
wireguard用类似RSA的一组key来定义一个peer， 一个`private key`以及其一个`public key`，
pubkey给别人用来建立和自身的连接，而prikey则可以判定只有确实发送目标是本peer的包才接受
和处理。根据作者的介绍，在安全方面做了很多的考量，而且由于代码量很少，引入不可预计的bug
的可能性也相对较小，因此这是一个很不错的虚拟连接工具。简单举个例子：

    # server端
    [Interface]
    Address = 10.0.0.1  # 可以有多个，指定不同的地址
    PrivateKey = aaaaaaaaaaaa  # 唯一identify本peer的key
    ListenPort = 1032  # 监听端口

    [peer]
    PublicKey = dddddddddddddd  # 所要连接的对方peer的公共key
    AllowedIPs = 10.0.0.1/24  # 从该peer发来的包所被允许的IP地址

    # 对应的client端的配置
    [Interface]
    Address = 10.0.0.2  # 可以有多个，指定不同的地址
    PrivateKey = ccccccccccc  # 唯一identify本peer的key
    ListenPort = 1111  # 监听端口

    [peer]
    PublicKey = bbbbbbbbbbbbbb  
    # 所要连接的对方peer(这里指server)的公共key
    AllowedIPs = 0.0.0.0/0  
    # 这里不做任何限定，这样可以在server端作NAT，可以
    # 有来自任一IP地址的包从server peer发送过来
    Endpoint = server_ip:1032  
    # 这里server peer必须具备公网IP，这样才可以让连接
    # 建立，而对于server端，在收到client的包时会自动配置
    # 直接实现了UDP puchhole

在server端做好iptables nat配置，很容易就可以实现客户端数据以server作为出口。

### 遇到的问题
到这里遇到了一个很缺乏常识造成的问题。请注意，这里的虚拟连接建立之后在两边虽然都多出来
一个网卡，但是他们只是虚拟网卡，所有的数据仍然都是从实际的网卡发出去的，如果按照把虚拟网卡
当真的情况，就会出现包发布出去的现象。如果是采用wg-quick作为工具，它在发现client端的
`AllowedIPs = 0.0.0.0/0`的情况下，会判断本虚拟连接的用途为以server作为数据出入口。因此
会自动进行一下配置，避免我后面将要遇到的情况

    wg set wg0 fwmark 5324
    ip rule add not fwmark 5324 table 5324
    ip route add default dev wg0 table 5324
第一，将所有由虚拟连接万口`wg0`发出的数据添加一个fwmark标签。   
第二，所有没有该标签的数据走路由表5324， 很关键的一步，因为`wg0`的数据实际还是要用本机的
实际网口将数据发出，如果不添加`wg0`网口发出的数据为例外直接将`wg0`设为默认出口，会导致数据
无法发出。

剩下的部分将在下一篇“linux下让某程序的所有数据包通过指定网口发出”中介绍。
