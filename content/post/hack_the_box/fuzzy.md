---
title: "Hack the Box——Fuzzy"
date: 2019-12-29T10:45:48+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---


## Fuzzy（逃课

这道题的描述是`We have gained access to some infrastructure which we believe is connected to the internal network of our target. We need you to help obtain the administrator password for the website they are currently developing.`

是让我们找密码，然而访问这个网站，是一个纯静态的网页。

![1](1.png)

### 直接爆破

页面上没有任何可点击或者可输入的地方。那看起来是要我们直接用工具扫吧。但是之前也没接触过目录扫描，想起之前在freebuf上mark了一篇关于目录扫描工具dirmap的文章。

顺手搜了下dirmap看了下安装和用法，直接开始扫。

```zsh
$python3 dirmap.py -i http://docker.hackthebox.eu:32411/ -lcf
```

只扫出来了一个`/api/`目录，而且body里面还是空的，什么信息都没有。

### 换工具

后来觉得可能是工具的问题，再查了查扫描工具。查到了一款叫御剑另一款叫WebPathBrute。
分别试了试效果，结果还是跟开始的dirmap一样，只有`/api/`是200，其它都是404。

### 换字典

这些工具里的字典都只有几百上千条，可能是因为太少了跑不出来吧。之前在github上有fork一份[SecLists](https://github.com/danielmiessler/SecLists)
于是我觉得去里面找找可行的字典。但是基本都试了一遍，都没有结果。
我开始怀疑可能自己的姿势不是，这不一定是一道爆破题吧？题目叫Fuzzy，我记得我在kali里面好像看到过类似的名字，
跑过去一看发现还真有一个叫wfuzz的软件。查了下，还是爆破软件，那题目的作者是不是在暗示我们用这个来爆破呢？
看了下用法，又跑了遍他自带的字典，和之前结果一样。

### 换方向

跑了这么多字典都没结果，肯定不是爆破题！我决定从其它地方入手。
随便访问了几个404，


`http://docker.hackthebox.eu:32411/login.php`
`http://docker.hackthebox.eu:32411/login.html`

好像他们的结果并不一样

![2](2.png)
![3](3.png)

php后缀的返回有点奇怪，看看burp里信息

![4](4.png)
php后缀多了一个`X-Powered-By: PHP/7.2.13`

我猜想应该是.php后缀的文件被转发到了php，但是非php后缀的在nginx就被抛出了404。

### 回到爆破                                                                                                                                                                   
那是要找一个存在的php文件吗?那岂不是又回到了爆破。又尝试了很多字典，但是都还是没有结果。

### 选择逃课

对于这种题真的是束手无策啊，只能看看答案。

果然是扫目录。
一共四次

第一次发现`/api/`

第二次发现`/api/action.php`

第三次发现`/api/action.php?reset=`

第四次发现`/api/action.php?reset=20`

flag藏在第四次的api里面

![5](5.png)

### 小结一下

虽然没有自己做出答案，但是也学到了一些关于目录扫描的工具和一些字典

#### 工具

* [gobuster](https://github.com/OJ/gobuster)

* [wfuzz](https://github.com/xmendez/wfuzz)

* [dirmap](https://github.com/xmendez/wfuzz)

* [dirbuster](https://github.com/KajanM/DirBuster)

#### 字典

* [SecList](https://github.com/danielmiessler/SecLists)

* kali里dirbuster的字典(`/usr/share/dirbuster/wordlists/`)


