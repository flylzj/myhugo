---
title: "Phonebook"
date: 2020-12-17T15:55:31+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

0x00 访问首页

进去就是一个访问页面

![1.png](1.png)

看看网页源码，还出了一个好像是登录之后的页面。

![2.png](2.png)

有个搜索的输入框，试了下，但是403了。暂时好像就这么多东西。

登录框试了下sql注入也没成。

0x01 爆破

开始尝试爆破，用gobuster爆破目录，然后用burpsuite爆破账号密码，运气还挺好，居然爆出来密码

![3.png](3.png)

0x02 寻找漏洞

现在搜索可以成功使用了

![4.png](4.png)

感觉漏洞应该就在这个搜索里了，搜索传参的时候传个无效json看看，服务端报错`{"error":"unexpected EOF"}`，

原来的搜索需要的body是`{"term":""}`，看看如果term的值不是字符串会怎样。服务端报错`{"error":"json: cannot unmarshal object into Go struct field Query.term of type string"}`

从这里可以知道服务端是golang实现的。
