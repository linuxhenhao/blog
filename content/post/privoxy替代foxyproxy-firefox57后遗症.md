+++
title = "Privoxy替代foxyproxy Firefox57后遗症"
date = 2018-01-14T16:28:08+08:00
tags = ["网络访问"]
categories = ['network']
keywords = ['privoxy','firefox57', 'foxyproxy']
markup = "mmark"
+++
正常的网络访问之前使用的是foxyproxy+gfwlist-rules的方式实现的。在firefox更新到57之后，扩展
大变革，原有的foxyproxy不能用了，而新的foxyproxy功能明显很弱。虽然采用import之前配置文件的
方式可以将之前包含有gfwlist-rules的配置导入，但是能够明显感觉到速度很慢。为了能够无感觉的对block
和非block的网络进行访问，寻找privoxy+gfwlist作为替代，将整个浏览器的代理设为privoxy，从而将
选择性代理的任务交给privoxy。
<!--more-->
## 1.浏览器方面-直接设置或者使用foxyproxy
如前所述，firefox57之后的foxyproxy需要重写，目前`2018-1-14`功能太弱，所以浏览器方面可以将
所有的网络访问都经过privoxy代理，或者仍然可以使用foxyproxy，在完全经由代理时选择ss，而不需要
完全代理时使用privoxy。这里选择使用foxyproxy，因为有时候即使不使用代理就能够访问的网站也需要
使用代理访问加快速度，总之能够更灵活。

关于privoxy代理使用，通常默认的监听位置是127.0.0.1:8118，为http代理，在foxyproxy或者浏览器
自带代理设置处设置即可。

## 2. privoxy
### 2.1 生成gfwlist.action
privoxy会根据配置文件选择相应的动作，为了实现自动选择性代理，需要生成gfwlist.action。这里有个
[gfwlist2privoxy](1)的工具可以使用。

    gfwlist2privoxy -f gfwlist.action -p 127.0.0.1:1080 -t socks5

上述命令自动从github上的gfwlist项目获取最新的gfwlist并生成相应的action配置文件，通过`-p`配置
上级代理的ip和端口，通过`-t`配置代理的类型。如此，所有gfwlist中的网址都会使用设置的代理访问。

需要将生成的gfwlist.action副指导privoxy的配置文件夹中。

### 2.2 修改privoxy的config
将privoxy的配置文件中的config进行修改，将生成的gfwlist.action添加到actionfiles当中去。
找到默认配置文件中的`actionsfile`段，将gfwlist.action添加。

    actionsfile match-all.action
    actionsfile default.action
    actionsfile user.action
    # 添加 gfwlist.action
    actionsfile gfwlist.action

重启privoxy，大功告成！效率很高，感觉不到规则匹配的耗时。

修改：
  添加gfwlist2privoxy的repo地址
[1](https://github.com/snachx/gfwlist2privoxy)
