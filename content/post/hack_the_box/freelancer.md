---
title: "Hack the Box——FreeLancer"
date: 2020-09-10T21:27:42+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

# FreeLancer

好久没玩ctf了，最近下班之后一直都在打游戏。今天刚好有了兴趣就在htb上随便找个题目玩玩吧。

## 0x00

首先启动实例，访问之后是一个类似于博客网站的界面

![1.png](1.png)

随便翻翻，最下面有个可以留联系方式的`form`，随便输入点什么然后提交

然后服务器就500了

![2.png](2.png)

又试了几种输入，发现不管输入什么都500了。感觉像是一个陷阱，先放这里。

## 0x01

仔细看网页源码，发现了有js、vendor、img三个目录，随便抓了一个直接访问目录，发现开了目录浏览

![3.png](3.png)

我本以为这样就可以实现目录穿越了呢。试了试`../../../../etc/passwd`，发现好像并没有什么卵用。然后搜了一下apache相关的目录穿越漏洞，也没搜到什么结果，放弃，暂且认为他可能又是一个陷阱。

## 0x02

继续看源码，发现了这么一行注释

`<!-- <a href="portfolio.php?id=3">Portfolio 3</a> -->`

可以，这很htb。

直接访问，看起来像是个debug页面。

![4.png](4.png)


试试注入。

`portfolio.php?id=3+AND+1=1`和`portfolio.php?id=3+AND+1=2`回显不一样，看来是由注入漏洞的。

直接上sqlmap。解出来两张表。如下。

![5.png](5.png)

## 0x03

数据库都拿下了，有账号密码，但是也没有登陆点啊，继续从源码找，真的一滴都没有了。尝试了一些经典的路径

`admin.php`
`login.php`
`administer.php`

都不行，开始用工具扫，也没扫出结果。绝望了，我要，这账密有何用~

没有思路，去逛下论坛，发现有个哥们直接用`dirp`就扫出后台了。原来是我字典没选好。

我也尝试用`dirp`扫了一下，果然出后台了

![6.png](6.png)

## 0x04

现在才发现还有一个问题，数据库里拿到的密码是加密的，登录的话应该是拿原来的密码加密后和数据的密码比对的，那我不知道原密码好像还是没法登录？

还是逛论坛，看到了这句话，`you can read some files with that tool(s***p)`这个s姓的tool显然说的是我sqlmap，搜了下sqlmap读取文件，果然有。之前看注入总结有见过load_file，但是没实际用过，所以不是很熟悉。

搜了下基本用法

读取文件

`sqlmap -u url --file-read "/remote/file.php"`

而且还有写文件

`sqlmap -u url --file-write "/local/file.php" --file-dest "/remote/file.php"`

如果有写权限的的话可以直接用--os-shell

`sqlmap -u url --os-shell`


还有就是得猜一下服务器上的目录，不过基本都是默认目录吧，搜一下默认目录，apache的在`/var/www/html`，先试试


`sqlmap -u http://docker.hackthebox.eu:30152/portfolio.php?id=1 --file-read "/var/www/html/index.php"`


果然是可以读取的

![7.png](7.png)


这样的话那可以直接读`/administrat/index.php`，看看登录的逻辑有没有什么漏洞


`sqlmap -u http://docker.hackthebox.eu:30152/portfolio.php?id=1 --file-read "/var/www/html/administrat/index.php"`

![8.png](8.png)


我看到这个判断session逻辑，以为只要加上`loggedin`字段就好了，事实好像并不可以。但是他这里又多了个没见过的php文件，直接读出来吧。

`sqlmap -u http://docker.hackthebox.eu:30152/portfolio.php?id=1 --file-read "/var/www/html/administrat/panel.php"`

这就找到flag啦。

![9,png](9.png)

## 0x05

这题难度还行，算比较简单的吧，但是还是在两个地方卡住了。

一个是扫后台，这个感觉还是需要工具好用一点比较方面。

另一个就是文件读取，这个算是解锁新姿势了，之前只是听过没用过，这次算用上了。


## 0x06

参考资料

[Bcrypt加密之新认识](https://www.jianshu.com/p/2b131bfc2f10)

[SQL注入实战— SQLMap之服务器端文件读写](https://blog.csdn.net/God_XiangYu/article/details/96994970)