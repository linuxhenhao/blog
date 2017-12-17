+++
title = "Iptables中owener信息的来源以及wireguard发包带有owner信息的原因探索"
date = 2017-10-30T19:41:47+08:00
tags = ["iptables", "owner"]
categories = ['linux', 'network']
keywords = ['iptables','owner match']
+++
在上一篇关于iptables让特定程序走特定网卡的文章中用到了iptables的owner match，有些好奇
到底是如何知道`uid、gid`信息的。简单看了眼kernel的代码，应该是找到了答案。
<!--more-->
google的时候找到了netfilter_queue的代码，其中关于uid、gid获取的部分用的是

    struct ntf_data->sk->uid、gid
那么socket结构当中是不是本身就包含了`uid、gid`信息？在内核代码中找到了答案。
用

    find . -exec grep -iE '(uid|gid)' {} \;
在4.13.10内核代码的net文件夹下的socket.c文件中找到了

    struct socket *sock_alloc(void)
    {
    	struct inode *inode;
    	struct socket *sock;

    	inode = new_inode_pseudo(sock_mnt->mnt_sb);
    	if (!inode)
    		return NULL;

    	sock = SOCKET_I(inode);

    	kmemcheck_annotate_bitfield(sock, type);
    	inode->i_ino = get_next_ino();
    	inode->i_mode = S_IFSOCK | S_IRWXUGO;
    	inode->i_uid = current_fsuid();
    	inode->i_gid = current_fsgid();
    	inode->i_op = &sockfs_inode_ops;

    	this_cpu_add(sockets_in_use, 1);
    	return sock;
    }

该代码用来初始化一个用来通讯的socket，不管是tcp还是udp等，都会调用到这个接口。创建一个虚拟
的inode，*nix下一切皆文件的思想很明显，这个inode通过current_fsuid()和current_fsgid()获取
到uid和gid. 然后通过SOCKET_I生成socket用来进行通讯。所有的包都在sk_buff（简称ＳＫＢ）中传输
，其中从本系统发出的包的SKB中会有一个`skb->sock`指向这个包的来源。netfilter作为内核组件，
自然能够得到该sock，从而简单的得到包的`uid、gid`。

## wireguard相关问题
### 一般虚拟连接的
虚拟连接软件都是创建一个虚拟网卡，然后系统将其当成实际的网卡来使用，从而达到利用虚拟连接软件
建立的虚拟连接传输包的目的。

虚拟连接软件的虚拟网卡应该模拟实际网卡的行为。那么我们看wireguard的问题。从系统来说，一个
包从应用层一直传输到device，从device发出之后的包当然不会再带有本系统发包来源sock信息了，
这应该只是内核空间能看到的信息。但是wireguard却不是如此，它从头到尾用的都是初始的skb，而
这个skb带有sock信息，这造成了包发到它创建的虚拟网卡之后，从内核再次发出时仍带有包发到虚拟网卡
之前的信息，特别是所关注的owner信息。这种非常规的行为会在进行`iptables`规则设置时产生我所遇到
的问题。

### 源码分析
在wireguard的device.c当中，`xmit`为发包函数

    static netdev_tx_t xmit(struct sk_buff *skb, struct net_device *dev)
其skb对象就包含了用户软件的`sock`信息。它通过`__skb_queue_tail(&packets, skb)`直接将
该skb添加到了对应route的peer（wireguard的其他结点都是peer，维护了一个peer结构，用来保存
peer相关信息，它包含一个代发送包列表）待发送队列当中。然后通过调用send.c中的

    void packet_send_staged_packets(struct wireguard_peer *peer)
将各个`peer`对象中队列的包发往相应的网络上的主机。该函数通过调用同一文件内的

    static void packet_create_data(struct sk_buff *first)
    queue_enqueue_per_device_and_peer(&wg->encrypt_queue, &peer->tx_queue, first,
       wg->packet_crypt_wq, &wg->encrypt_queue.last_cpu);
    # queueing.h中上述enqueue函数的关键点
    queue_work_on(cpu, wq, &per_cpu_ptr(device_queue->worker, cpu)->work)    
将要发送的包添加加密队列中。然后首先放到加密worker中进行包的加密，此时skb中的
sock信息还是那个从软件发出的信息，没有改变。加密通过send.c中的skb_encrypt加密：

    static inline bool skb_encrypt(struct sk_buff *skb,
                         struct noise_keypair *keypair, bool have_simd)
查看该函数源码，使用了加密函数加密，变更了skb里面的数据内容，更新checksum，但是，skb->
sock仍然没有更改，在此之后，包含skb的list就被放入发送队列进行发送了。    

    queue_enqueue_per_peer(&PACKET_PEER(first)->tx_queue, first, state);
send.c中有tx_queue的work具体实现，通过使用
    static void packet_create_data_done(struct sk_buff *first, struct wireguard_peer *peer)
    {
      ...
      if (likely(!socket_send_skb_to_peer(peer, skb, PACKET_CB(skb)->ds) && !is_keepalive))
      ...
    }
进行数据包的发送，使用的时socket_send_skb_to_peer，该函数用`send4`和`send6`，在`sendx`
函数中，对skb的dev、mark进行了设置，但是同样skb->sock没有更改，然后调用

    udp_tunnel_xmit_skb(rt, sock, skb, fl.saddr, fl.daddr, ds,
      ip4_dst_hoplimit(&rt->dst), 0, fl.fl4_sport, fl.fl4_dport, false, false);
    {
      ...
      if (!skb->sk)
        skb->sk = sk;
      ...
    }
__这就是关键，这个判断，skb->sk由于出自本机，是带有sk的，这里作判断，如果有就不做更改，没有，
将该skb的sk设置成wireguard用来发送的sock__，这个机制应该是不合理的，虚拟连接软件应当做到
尽可能的与实际网卡相同的行为，从而在配置诸如iptables、ip-rule方面简单方便。网络包到网卡之后
其发送的sk信息应当被抛弃，所以这个判断应该删除。
