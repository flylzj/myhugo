---
title: "CTFHub——Web题总结"
date: 2021-03-19T21:22:55+08:00
showDate: true
draft: false
tags: ["security","ctfhub", "ctf"]
categories: ["security"]
---

# 0x00 前言

之前接触到一个国内的ctf平台ctfhub，于是趁热做了一波题目，这里把记录一下自己的解题过程


# 0x01 题目列表


## 第一题 afr-2 

### 解题过程

这道题一进来就是文字加gif

![1-1.png](1-1.png)

直接先看源代码

![1-2.png](1-2.png)

有一个img/img.gif路径，去掉文件名，开了目录访问。

![1-3.png](1-3.png)

根据题目的考点——文件读取，试试目录穿越，把访问的路径改为`/img../`。直接跳到根目录了，flag也刚好就在根目录

![1-4.png](1-4.png)

点击下载即可拿到flag

### 参考资料

[三个案例看Nginx配置安全](https://www.leavesongs.com/PENETRATION/nginx-insecure-configuration.html)


## 第二题 afr-1

### 解题过程

这道题进来是一个helloword界面，但是url里面有一个`?p=hello`的参数

![2-1.png](2-1.png)


先fuzz一下参数p，发现，当`p=index`的时候，服务器会返回502，这个时候猜测，这个服务的逻辑应该是 `include($_GET['p'] . ".php")`，然后由于包含自身，导致了502。

这个时候可以使用伪协议来读取文件

令`?p=php://filter/read=convert.base64-encode/resource=index`

这个时候服务端返回了一串base64串，解码一下得到源码，跟我们猜的基本一样。

![2-2.png](2-2.png)

这个时候，既然是ctf，就猜一下flag是不是当前目录的flag.php（虽然这个猜很没有道理，但是毕竟都是套路）

访问`?p=php://filter/read=convert.base64-encode/resource=flag`

通过base64解出flag

![2-3.png](2-3.png)


### 参考资料

[文件包含漏洞与PHP伪协议](https://www.smi1e.top/%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E%E4%B8%8Ephp%E4%BC%AA%E5%8D%8F%E8%AE%AE/)