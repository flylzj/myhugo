---
title: "Hacker101"
date: 2019-10-11T16:31:29+08:00
showDate: true
draft: false
tags: ["security","hacker", "ctf"]
categories: ["security"]
---

# hacker101学习笔记

## 前言

记录自己在hacker101 ctf的学习笔记


### 第一关 	A little something to get you started (1/1 flag)

第一次接触ctf没什么经验， 各种看网页源代码，看http请求有点像无头苍蝇，后来看了下hint `Take a look at the source for the page`,然后发现了一个没有出现在页面里的style标签

```html
<style>
	body {background-image: url("background.png");}
</style>
```

试了下访问http://domain.com/background.png，报了404，然后试了试在当前目录加上/background.png，解决


### 第二关 	Micro-CMS v1 （2/4 flag) 

有了第一关的经验，我大概懂了一些套路，第二关给了一个input框一个textarea标签

![2-1](./2-1.png)

大概猜出来应该是考xss，现在input框里试了一下`<script>alert('xss')</script>`，爆出了第一个flag

![2-2](./2-2.png)

接着在textarea试了试`<script>alert('xss')</script>`, 虽然提交的是script， 看起来是后端把script替换成scrubbed。 

然后试了一下`<BODY onload="alert('XSS')">`，成功爆出flag

![2-3](./2-3.png)

既然是script被过滤的，那就再试试img标签吧。textarea里写了`<IMG SRC="" onerror="alert('XSS')">`，但是好像和上一个算同一个flag

![2-4](./2-4.png)


未完待续~



