---
title: "hacker101 第0关&&第1关"
date: 2019-10-11T10:38:28+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

# hacker101学习笔记

## 前言

记录自己在hacker101 ctf的学习笔记


### 第0关 	A little something to get you started (1/1 flag)

第一次接触ctf没什么经验， 各种看网页源代码，看http请求有点像无头苍蝇，后来看了下hint `Take a look at the source for the page`,然后发现了一个没有出现在页面里的style标签

```html

<style>
	body {background-image: url("background.png");}
</style>
```

试了下访问http://domain.com/background.png，报了404，然后试了试在当前目录加上/background.png，解决


### 第1关 	Micro-CMS v1 （4/4 flag) 

#### flag 1

有了第0关的经验，我大概懂了一些套路，第1关给了一个input框一个textarea标签

![1-1](./1-1.png)

大概猜出来应该是考xss，现在input框里试了一下`<script>alert('xss')</script>`，爆出了第一个flag

![1-2](./1-2.png)

#### flag 2

接着在textarea试了试`<script>alert('xss')</script>`, 虽然提交的是script， 看起来是后端把script替换成scrubbed。 

然后试了一下`<BODY onload="alert('XSS')">`，成功爆出flag

![1-3](./1-3.png)

既然是script被过滤的，那就再试试img标签吧。textarea里写了`<IMG SRC="" onerror="alert('XSS')">`，但是好像和上一个算同一个flag

![1-4](./1-4.png)


#### flag 3

在textarea标签里试了很多种xss，但是flag都是一样的，那看起来接下来的问题就不是考xss了。
里面有一个创建page的页面，试着创建了一个page发现创建出来的page的id好像不连续（初始的两个是1、2，创建的是9），试了下访问`/page/3`，报not found，再试了试`/page/4`，报forbidden，看来这里应该有问题，因为每个page都有一个对应的eidt页面`/page/edit/page_id`，试着访问`/page/edit/4`，出了flag

![1-5](1-5.png)

#### flag 4

第四个flag就有点顺了，访问完`/page/edit/4`，想看看有没有注入点，然后试了试`/page/edit/4'`，最后一个flag就这样出了。

![1-6](1-6.png)

#### 参考资料

[常见xss过滤](https://damit5.com/2017/10/15/XSS-%E7%BB%95%E8%BF%87%E5%B8%B8%E7%94%A8%E8%AF%AD%E5%8F%A5/)