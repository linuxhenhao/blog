+++
title = "Btrfs修复(伪)及损坏预防"
date = 2018-04-15T11:33:13+08:00
tags = ["orangepi", "btrfs"]
categories = ['operating_system']
keywords = ['orangepi','btrfs', 'repair']
markup = "mmark"
+++
可能由于经常使用智能插座直接断电关闭OrangePi，常在河边走哪有不湿鞋，用来存储数据的btrfs
分区挂掉了，当然由于有RAID1的备份，对于数据我是毫不担心，尝试修复以下btrfs。
<!--more-->
错误信息：

```bash
btrfsck
    parent transid verify failed on 64585728 wanted 65318 found 65317
    .....
    Ignoring transid failure
    leaf parent key incorrect 63799296
    bad block 63799296
    ERROR: errors found in extend allocation tree or chunk allocation
    Error: could not find extent items for root 617
    ERROR: failed to repair root items: no such file or director
```

## 1. 备份 metadata
首先要备份 metadata，也就是除了实际数据块之外的其他信息，占用空间小，在折腾后出了问题可以
方便的恢复折腾前的状态：

    btrfs-image -t 4 -c 5  # -t 指定线程数， -c 指定压缩级别

## 2. 恢复过程
尝试使用 recovery 模式挂载分区

    mount -orecovery /dev/sdb2 /mnt
如果失败，使用`dmesg`查看失败原因，如果是 log-tree 问题，可以使用 `btrfs-zero-log` 来
清空 log-tree ，之后就可以正常挂载。但是由于 log-tree 不匹配问题实际还是由于在写入过程发生
了问题产生的，所以虽然正常挂载了，但是可能存在文件丢失。

使用 'btrfsck -s 1/2/3 /dev/sdb2' 使用备份的 superblock 进行 fsck，可以指定为 1、2、3
等，如果没有问题，那么使用备份的 superblock 恢复即可， 使用命令`btrfs-select-super`。

在进行具体的修复之前，可以用 `btrfs restore` 命令将能够找到的文件备份到别的盘上，之后再
进行下面的尝试。

如果失败原因和 chunk-tree 相关，就可以使用 `btrfs rescue chunk-recover /dev/sdb2` 来
重建 chunk-tree ，这个过程将会根据硬盘大小使用大量的时间，所以除非确定是 chunk-tree 问题，
不着急使用该方法。

如果 extent-tree 相关问题存在，还需要使用 `btrfsck --repair --init-extent-tree` 重建
 extent-tree。 `btrfsck --repair` 具有破坏性，不要随意使用，不过备份好 metadata 的情况下，
 试试也无妨。

## 3. 实际恢复
实际恢复过程，首先 `btrfsck /dev/sdb2` 看到了那些错误，然后使用 btrfs-image 备份 metadata
，然后使用 recovery 选项挂载，发现不可挂载，查看`dmesg`输出， `parent transid verify failed
on xxxxx wanted xxxx`， 这是 log-tree 匹配错误，使用 `btrfs-zero-log /dev/sdb2` 修复之后，可以直接挂载，
但是挂载上去是 readonly 的，`dmesg`查看，发现还是有错误信息：`unable to find ref byte nr
63586304 parent 0 root 7 owner 2 offset 0`。

使用 `btrfsck -s 1` 如果整个过程没有出现 error，则可以使用备份的 superblock 来恢复
文件系统，`btrfs-select-super -s 1 /dev/sdb2`，使用位置 1 的 superblock 覆盖主 superblock
完成修复。

很不幸，对于我这个 btrfs， 问题不在 superblock, 在 extent-tree 上， dmesg 可以看到，
`__btrfs_free_extent:7059: errno=-2 No such entry` 的提示信息。那么尝试下重建 extent-tree，
`btrfs --repair --init-extent-tree`，操作失败，在修复过程中提示`errors found in extent allocation
 tree or chunk allocation`。

最后，使用耗时最长的 `btrfs rescue chunk-recover /dev/sdb2` 来尝试修复。这个操作非常耗时，
我们知道，硬盘的数据读取速率不仅受硬盘本身限制，也受计算机性能限制，因此这个操作最好直接放到
PC上作，不要在 OrangePi 上操作。

!!!!!最终受不了了！！！！几个小时还没完成，根据 2T 这个大小，加上 40MB/s 的读写速度，
chunk-recover 需要14个小时， 实际上我是有备份的，格式化这个数据分区后用btrfs send receive
重建就行了，只需要 170GB 数据的传输时间。然后修改对应的挂载 fstab 以及做好 .latest 软链接
用于下次增量备份就 OK。

## 4. 吃一堑长一智
回想这段时间的操作，这个自建 NAS 是在我上次由于 RAID1 硬盘盒的风扇噪音太大，有过移动整个硬盘
的操作，不记得是否有提前断电了，但是如果是由于运行时的振动，应该会导致硬盘坏道，而不是某些数据
不一致。所以，最有可能导致本次事故的原因是直接使用断电关机（智能插座远程断电）导致的。因此，拿
tornado 写个 Handler，用于关机。通过已经暴露到公网的 nginx 作代理，并且添加简单的 HTTP Auth，
由于整个通信过程是基于 https的，所以安全性可以保证。

以后不要再直接暴力断电关机了，同时，什么手段都不如有个备份，三个硬盘同时损坏的概率还是比较低的，
更可怕的灾难性事故就需要使用异地备份容灾了，可以考虑租一个 vultr/buyvm 的大存储型 vps。但是
这样就不可避免的存在数据泄漏的风险。如果单单承担 seafile 的备份，seafile 倒是支持本地加密，
然后只存储加密后的文件在远端。

    nginx配置
    location /shutdown/ {
      auth_basic "auth for action";
      auth_basic_user_file "/etc/nginx/htpassword";
      proxy_pass http://127.0.0.1:8888/shutdown/;
    }
    # tornado 监听在本地 8888 端口，只对 /shutdown/ 作出响应
    # tornado 通过 supervisord 控制，以 root 权限运行，只有通过验证的外部访问才能到达

参考资料：
Suse mail list
