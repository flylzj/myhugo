---
title: "废弃笔记本利用计划（二）：智能家居平台home assistant搭建及使用"
showDate: true
date: 2022-01-06T15:20:27+08:00
draft: false
---

## 0x00 前言

打通了内外网之后，玩法就很多了，我首先想到的就是远程控制电脑，但是要远程控制电脑得保证电脑时刻开着，这样并不是很优雅，应该有可以控制电脑开关机的方案。

简单搜索了一下远程开机方案，一个是买开机卡，开机卡联网，然后在app上控制，另一个是用wake on lan。

开机卡需要额外花钱购买，而且我有两台pc就得买两个。

wol的话是只能实现开机，关机得话还得rdp连上电脑关机也不是很优雅。

又经过一番搜索，我看到了一个优雅的解决方案，`home assistant`集成了wake on lan功能，并且也给出了关机的办法——配置ssh命令关机。

## 0x01 hass安装

和之前一样，直接创建docker-compose.yaml文件进行一键安装

```
version: '2'
service:
  hass:
    image: homeassistant/home-assistant
    restart: always
    network_mode: host
    volumes:
      - ./hass/config:/config
      - /etc/localtime:/etc/localtime:ro
```

## 0x02 windows ssh服务端安装

在windows上安装ssh服务端也很简单，直接根据[官方教程](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)在控制面板添加就行了。

## 0x03 hass配置及ssh配置

都安装完成之后进入配置，首先根据hass官网的配置，在configure.yaml文件加入

```
# wake on lan
switch:
  - platform: wake_on_lan
    host: "你电脑1的内网ip1"
    mac: "你的电脑1的mac"
    name: "pc1"
    turn_off:
      service: shell_command.turn_off_pc1

  - platform: wake_on_lan
    host: "你电脑2的内网ip2"
    mac: "你电脑2的mac"
    name: "pc2"
    turn_off:
      service: shell_command.turn_off_pc2

shell_command:
  turn_off_pc1: "ssh 你的windows用户名@你电脑1的内网ip1 shutdown /s /t 0"
  turn_off_pc2: "ssh 你的windows用户名@@你电脑2的内网ip2 shutdown /s /t 0"
```

这里配置用的是免密登录，所以得把hass所在docker里的公钥加到windows ssh服务的authorization_keys里面，这一步可以看[这里](https://www.google.com/search?q=ssh%E5%85%8D%E5%AF%86%E7%99%BB%E5%BD%95&oq=ssh%E5%85%8D%E5%AF%86%E7%99%BB%E5%BD%95&aqs=chrome..69i57j0i512l4j69i61l3.11092j0j1&sourceid=chrome&ie=UTF-8)

都配置完成之后重启hass，就可以在hass的概览里看到配置的两台电脑的开关了

![1.png](1.png)

这里有一个坑就是上面配置文件的host的目的是用来检测你电脑的开机状态，不知道hass使用什么方法检测的，我是在关闭防火墙之后显示的状态才是正常的。

## 0x04 接入homekit

到这里我们就拥有了可以远程开电脑的开关了，但是还不够优雅，我还得打开浏览器，登录hass，点击开关才能开机。借助hass的集成，我们可以把开关接入homekit，这样就通过siri来控制开关了。在hass中点击`配置`->`设备与服务`->`添加集成`，搜索homekit点击提交再选择switch就可以添加了，然后用ios设备在家庭里点添加配件，扫hass提供的二维码就可以打通hass和homekit了。效果如图

![2.png](2.png)

这个时候就解放双手了，直接siri开机关机

{{<raw_video>}}
<video id="video" controls="" preload="none" poster="封面">
      <source id="mp4" src="1.mp4" type="video/mp4">
</video>
{{</raw_video>}}

## 0x05 用hass api控制

hass还提供了[REST API](https://developers.home-assistant.io/docs/api/rest/)的调用方式，刚好我把之前部在公网的nonebot机器人迁到了内网，那就可以通过编写命令，让bot去调用api，编写命令和调用api的[代码](https://github.com/flylzj/mycoolq/tree/master/mynonebot/coolq/plugins/smart_home)，就可以在qq里控制电脑了。

![3.png](3.png)

我就拥有了两台可以随时远程开机关机的云电脑了，实测用流量可以通过zerotier打通的内网流畅rdp连接，云游戏的话估计有点难。

## 0x06 米家设备接入hass

在hass自带的集成里的xiaomi iot好像有点问题，我的wifi智能插座v2并不支持，通过安装al-one的[hass-xiaomi-miot](https://github.com/al-one/hass-xiaomi-miot)，可以把家里的米家设备也集成到hass，然后再桥接到homekit，这样siri也能控制米家设备了。

## 0x07 总结

感觉智能家居这块国内玩的人并不多，搜索出来的东西都在论坛里，文档也是支离破碎，只能看官方的英文文档，不过好在大部分工作前人都已经做好了，我们只需要按照文档一步一步配置，很容易就能做出来。

