---
title: "Hack the Box——邀请码"
date: 2019-12-25T09:41:35+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

hacker101的web的题目基本做的差不多了（除了web加密和php~

安卓的和php的等寒假学一波再说吧。

来玩一玩新的平台————hack the box

### level 0 ———— hack the box获取邀请码

登录hackthebox.eu的官网，找了半天只找到了登录没有找到注册。然后发现了一个join的入口

![1.png](0-1.png)

进去之后只有一个填邀请码的输入框，其它啥也没有。看来是个challenge~

![2.png](0-2.png)

第一关应该没什么难度吧，直接用chrome玩，打开开发者工具，刷新一下页面，先看看network,找到了一个奇怪的js。

![3.png](0-3.png)

这段代码好像是定义了那么几个函数，去console执行试试。好像makeInviteCode()这个函数有点奇怪。 这个enctype应该是加密方式吧。
google一下`ROT13`发现果然是一种加密方式。[ROT13（回转13位，rotate by 13 places，有时中间加了个连字符称作ROT-13）是一种简易的替换式密码。](https://zh.wikipedia.org/wiki/ROT13)

![4.png](0-4.png)

google的时候刚好还发现了ROT13在线加解密。把data丢进去试试，解密出了明文`in order to generate the invite code, make a post request to /api/invite/generate`

![5.png](0-5.png)

看来还要去请求一下这个api才能拿到邀请码，用curl发个post试试。返回了一个code。

![6.png](0-6.png)

不知道这个是不是最终邀请码，拿去试试，还是不对。再看看这个format字段的值是encoded，可能说明这个还是被编码的吧。最后用`=`补齐的我盲猜base64（其实我就知道那么几种编码方式

去试试base64解码吧。解码之后的结果为`EEJTO-BUFSB-TMIBT-NVAKS-WIDPR`,这个看起来就有点像邀请码了，再去试试，成功跳转到注册页面，Cool~

![7.png](0-7.png)

