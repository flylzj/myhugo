---
title: "废弃笔记本利用计划（一）：使用zerotier one打通内外网 && 搭建开源私有云盘"
showDate: true
date: 2023-01-04T15:20:27+08:00
draft: false
---

## 0x00 起因

之前买了台Oculus quest2玩，但是因为是fb家的，所以需要连外网才能玩，但是就想着用闲置的小米路由器刷个`openwrt`，结果操作失误路由器变砖了。

后来朋友推荐刷软路由，刚好我有台吃灰的笔记本，于是就开始了折腾之路。刷上了`koolshare`改的`lede`之后，简直打开了我对路由器的大门，原来路由器还有这么多花里胡哨的功能。酷软里的`frps`客户端和`docker`配合，再连一台公网服务器，岂不是就相当于拥有一台内网服务器。

恰好双十一入了三年的腾讯云8M带宽的服务器，正好可以拿来当作内网服务器的出口。

## 0x01 简单调研

稍微看了一下主流的几种内外网连通的方案`frp`，`DDNS`, `zerotier one`

`frp`需要配置端口映射，而且只能通过公网来访问

`DDNS`需要公网ip，搜了下流程，还是算了

最终选定了`zerotier one`，跨网创建虚拟装用网络，pc，移动端都可以加入，这样既可以通过虚拟网络访问，也可以把内网服务器服务转发到公网服务器让没有加入`zerotier`的设备访问

## 0x02 docker-compose一键搭建zerotier planet

已经有人做好了zeroiter planet的[docker-compose.yaml](https://gitee.com/Jonnyan404/zerotier-planet)，在公网服务器建好之后，创建一个network，然后在各种设备（pc，Android，ipad）安装zerotier的官方应用，效果如图，所有的设备现在都在一个内网了。

![1](./1.png)

下一步就是在内网服务器搭建一些服务，然后让它们或通过内网被访问，或通过转发到公网服务器被访问。

## 0x03 seafile和nextcloud

因为看到nextcloud是php写的，但是因为php是世界上最好的语言，所以我选择了python实现的seafile，用官方提供了[docker-compose.yaml](https://docs.seafile.com/d/cb1d3f97106847abbf31/files/?p=/docker/docker-compose.yml)搭建。

搭建完成后界面大概长这样，看起来也挺清爽了。

![2](2.png)

但是有两个问题让我放弃了它，

一个是文件是分块加密上传的，所以服务端也拿不到原始数据，虽然提供了webdav，但是我在ipad上装了各种支持webdav的视频播放软件都不能正常播放视频或者加载数据，浏览器是可以播放的，这样就不能用ipad快乐看剧了呀。

另一个就是文件下载url`FILE_SERVER_ROOT`是自己配置的，而且只能一个值，这样就不能满足我即通过公网访问又通过内网访问了。

于是我转向了`nextcloud`，官方提供了各种版本的[docker-compose.yaml](https://github.com/nextcloud/docker/tree/master/.examples/docker-compose/)文件, 一键安装就完事儿了。

## 0x04 nextcloud

在内网服务器装好nextcloud，内网ip就可以直接访问了，但是这个时候还需要用公网的nginx做反向代理公网才能访问。给公网nginx加上反向代理配置，然后绑定域名并加上ssl，这样公网也可以访问了。通过内网访问的时候速度有大概50M/s，通过公网的话是800k左右。