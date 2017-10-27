+++
title = "google scholar 备忘"
date = 2017-10-27T21:50:33+08:00
tags = ["scholar", "ipv4", "ipv6", "blocked"]
categories = ['备忘']
keywords = ['scholar','blocked']
+++
1\. 由于各大数据库提供商本身提供的搜索功能只能搜索他们自己的数据库.
2\. 由于各vps供应商所在的ip网段有太多的对google的搜索请求   

==>   

1\. google scholar可以在文献查找上提供极大的便利   
2\. 而各vps供应商所在的ip网段请求太多,基本都已经被google个block掉了.   

<!--more-->
也就是通过vps访问scholar时:
We're Sorry, ... may be sending automated queries. To protect our users ...   

==>   

解决办法:   
幸好vps基本都带有一段ipv6地址, 而且这个ipv6地址一般是没有被google ban掉的. 因此, 我们
可以通过提供google scholar的ipv6地址, 让vps上的请求scholar内容的软件通过ipv6来获取
google scholar的内容. 但是, 直接ping6 scholar.google.com可以发现, 没有该网址的v6地址.

但是别慌, google的服务器是分布式的, 找个google.com的ipv6地址, 然后写入到hosts里.   
比如在hosts文件里添加这么一段(ipv6地址也可能发生变化, 如果不工作, 那就另找一个)
`2607:f8b0:4008:80a::200e scholar.google.com`   
那么就下来让请求内容的软件解析ipv6地址就ok!

之前忘了这件事情, 换vps的时候受到了'We're sorry'一万点伤害, 记下来免得下次又忘了.
