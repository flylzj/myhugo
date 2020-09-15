---
title: "Hack the Box——Breaking Grad"
date: 2020-09-13T00:03:48+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00 Breaking Grad

首先访问主页，星战的bgm加太空背景+死星，有星战那味儿了。

![1.png](1.png)

翻翻源码，在js里面找到了一个api

```
POST /api/caculate

body
{
    "name": "xxx"
}

response
{
    "pass":"n000o0000o0pe"
}
```

其它的好像都没有了，为了防止有什么其它的api，先上kali用dirb扫一扫看看，再来仔细研究这个api

## 0x01 测试api

首先传个不完整的json试试。

![2.png](2.png)

这里可以看到报错并给了报错信息，看起来后台是node，啊，又是不会的，要学的太多了吧。

再试试该下值的类型。

![3.png](3.png)

这次是报了不一样的错误。

那在试试不传这个键会怎么样吧。

![4.png](4.png)

还是和之前不一样的错误。

## 0x02 梳理逻辑

根据上面的三个报错，可以理一下服务端的逻辑。

`客户端传json`=>`服务端解析`=>`获取到json中键为name的值value`=>`调用value的includes方法`

没传name报错和传错类型报错应该都是因为错误的调用了includes方法。

这两个报错可以直接在浏览器测试出来。

![5.png](5.png)

写到这dirb刚好跑完了common字典，既然没扫到其它api那就算了吧。先再研究研究现在这个api。

## 0x03 发现漏洞



