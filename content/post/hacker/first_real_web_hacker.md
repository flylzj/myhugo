---
title: "记我的第一次成功（伪）的web渗透实站"
date: 2020-12-06T20:20:48+08:00
showDate: true
draft: false
tags: ["security","web hakcer"]
categories: ["security"]
---

# 0x00 前言

之前一直做web ctf题，这次终于可以拿一个真的网站练手了。这次只拿到了webshell，试了很多种方法都没有成功提权，但是毕竟是第一次实战，我已经很满意了，这也相当于是我的一个复盘回忆。

# 0x01 开始

## adminer客户端任意文件读取

直接访问ip，就是目录访问，从目录时间来看已经是个很老的站了，有一个adminer.php。其它的目录点进去之后就是一个网站，大都是wordpress建的。

![1.png](1.png)

搜了下adminer.php，发现有个类似经历的文章，作者介绍了一个[Rogue-MySql-Server](https://github.com/allyshka/Rogue-MySql-Server)，可以读取mysql客户端文件，
把代码clone到我的vps，原来的代码随机从filelist元组随机取文件来读，得改造成读我们想要的文件。

把
```
filelist = (
    '/etc/passwd',
)
...
filename = random.choice(filelist)
```

改成
```
file_index = 0
filelist = []
with open("file.txt") as f:
    for line in f:
        filelist.append(line.strip("\r\n"))
print filelist
...
global file_index
filename = filelist[file_index]
log.info('Query:' + filename)
log.info("file_index:" + str(file_index))
file_index += 1
```

接下来把想要读取客户端的文件名丢到`file.txt`就可以读到文件内容了。

点击登录就vps就可以收到客户端发过来的文件内容了

![2.png](2.png)

最后得到以下信息

- 网站根目录为`/var/www`
- 在某个网站的`wp-config.php`读到了数据库信息
    * `DB_HOST` - `localhost`
    * `DB_USER` - `root`
    * `DB_PASSWORD` - `********`

## 登录本地数据库

拿到数据库密码，就可以用adminer直接登录了。登上之后发现有很多数据库，名字基本都和前面的目录对上了，应该就对应那些网站的数据库了

![3.png](3.png)

这里我的第一反应是在把adminer当入口，通过某种方式拿到服务器shell，搜了下相关的资料，了解到了mysql udf提权。

这种方式需要mysql文件导出功能开启并且有对插件目录的写入权限，试了下这个方法，从结果看应该是没有写入权限

![6.png](6.png)
![4.png](4.png)
![5.png](5.png)


## 从wordpress下手

既然数据库走不通，就反过来从网站入手，随便进一个网站，发现可以登录wordpress

![7.png](7.png)

不知道账号密码？库都有了，直接数据库里插一条，插入完直接用新增的账号登录。成功登录wordpress后台。

![8.png](8.png)

开始搜索wordpress后台getshell姿势，好像可以直接编辑主题的php代码，但是为啥我的编辑完没有保存文件按钮，搜了下好像是没权限，哦豁。

![9.png](9.png)

试了下上传插件、编辑插件，上传媒体，都行不通，好像这条路也走不通了。

## 找到别的网站

翻了翻其它的目录，发现了个不是wordpress的网站，用和wordpress一样的方法进入网站后，

![10](10.png)

瞎逛之中，找到一处上传点，先随便传个图片，成功上传了。直接传php文件没有回显，但是访问不了，应该是失败了。再传个带PNG头的phpinfo，上传成功并且可以访问了~

![11.png](11.png)

接下来就用剑蚁直接连webshell了。

## webshell提权（失败）

虽然失败了，但是还是记录一下我搜到（真面向谷歌渗透）的尝试

### suid提权

suid提权我看大都是用的Nmap, Vim, find, bash, more, less, nano, cp等等，但是我这个搜到的都没有。放弃~

![12.png](12.png)

### 内核漏洞

用searchsploit搜到的几个，应该目标没有gcc，我直接本地编译然后传上去运行，都失败了。此外还试了传说中的脏牛，也没成功。

![13.png](13.png)


# 0x02 总结

虽然是一次失败的经历，但是也学到了很多新东西，这个周末没白过了~

# 0x03 参考资料

* [记一次webshell的获取](https://xz.aliyun.com/t/6587)
* [MYSQL提权--UDF提权](http://www.oniont.cn/index.php/archives/310.html)
* [Wordpress 后台GetShell总结](https://dylan903.github.io/2019/11/06/wordpress-hou-tai-getshell-zong-jie/#toc-heading-1)
* [谈一谈Linux与suid提权](https://www.leavesongs.com/PENETRATION/linux-suid-privilege-escalation.html)
* [linux系统提权——基于已经拿到www-data权限](https://blog.csdn.net/qq_45836474/article/details/107873338)
* [对Linux 提权的简单总结](https://xz.aliyun.com/t/7924)
