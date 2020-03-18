---
title: "hacker101 第6关"
date: 2019-11-11T10:49:28+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

### 第6关

进入首页，看起来是个写文章的平台，后端是php

![6-1.png](6-1.png)

先尝试注册账号。

登录之后有几个可以访问的路由分别为

```

index.php
index.php?page=home.php&message=
index.php?page=create.php
index.php?page=profile.php&id=d
index.php?page=account.php
index.php?page=sign_out.php
```

#### flag1


这关一共7个flag，应该每个页面都有，先试试这个最特别的

`index.php?page=profile.php&id=d`

访问页面，是一个展示个人信息的页面

![6-2.png](6-2.png)

这个参数`id=d`看着很怪，尝试把它改成`id=1`，页面上的username没了

![6-3.png](6-3.png)

然后试了一下ab，`id=b`的时候，页面变成了别人的用户名和别人的posts

![6-4.png](6-4.png)

访问那个secure blog，第一个flag get。

![6-5.png](6-5.png)

#### flag2

接着从主页面找，有一个create的表单

![6-6.png](6-6.png)

尝试create一个，burp里面看看post请求

![6-7.png](6-7.png)

用id字段来识别post的创建者？repeater里改一下user_id试试。

果然是个flag~

![6-8.png](6-8.png)

#### flag3

这个页面创建之后，出现了可以删除自己的post

![6-9.png](6-9.png)

发一个delete，参数为

`page=delete.php&id=e4da3b7fbbce2345d7772b0674a318d5`

这个id长32位，盲猜md5，而且肯定和post_id有关，这个post_id是5，直接试着加密一下，这不就是5的md5值嘛

![6-10.png](6-10.png)

那我是不是可以删除不属于我的post呢？

试了一下第二个id的md5，成了，越权漏洞!

![6-11.png](6-11.png)

#### flag4

这个页面还有一个edit子页面

`index.php?page=edit.php&id=4`

改一下id=1然后save

![6-12.png](6-12.png)

第四个flag ok~

![6-13.png](6-13.png)


#### flag5

页面里的内容都找的差不多了，接下来看看http头部，看了下他的cookie，我发现我每次登录退出之后再登录cookie都一样，看起来是和用户有关，还是32位，估计还是md5，再比对一下发现就是user_id的md5值，那这样我可以构造别人的id然后登录了呀。

构造一下id=1的cookie进去发现是admin的账号

![6-14.png](6-14.png)

把他的密码改成admin然后登录，成功

![6-15.png](6-15.png)


#### flag6

因为还有个user账号，用同样的方法登上去也有个flag

![6-16.png](6-16.png)

#### flag7

访问id=945的post，拿到flag。（hint太蠢了）
