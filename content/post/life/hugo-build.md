---
title: "Hugo学习"
showDate: true
date: 2018-10-29T17:10:07+08:00
tags: ["hugo"]
categories: ["hugo"]
draft: false
---

# 记录学习hugo的过程

## Hugo是由Go语言实现的静态网站生成器。简单、易用、高效、易扩展、快速部署。

### 一丶安装

### hugo的官网[点击链接](https://gohugo.io/)

ubuntu的话直接 `$ sudo apt install hugo` 安装，但是这样安装会的版本很低，导致甚至连官方教程的主题都用不了。所以还是直接去下github上下在deb包安装吧。[下载huo地址](https://github.com/gohugoio/hugo/releases)

### 二丶生成站点

生成站点的话直接 `$ hugo new site /path/to/site`

新建一篇文章 `$ hugo new first.md`，这条命令会在`/path/to/site/content`目录下生成first.md，也可以指定生成文章的目录。比如： `$ hugo new post/first.md` 会在`/path/to/site/content/post/`目录下生成first.md

### 三丶选择主题

没有主题hugo是无法生成静态文件的，所以需要从[链接](https://themes.gohugo.io/)挑选一款主题。以我这个主题为例，安装主题的方法为

```bash
# git clone https://github.com/vickylai/hugo-theme-sam.git themes/sam
# cp themes/sam/exampleSite/config.toml .
```

config.toml就是配置文件

### 四丶生成静态文件

用命令`hugo`生成静态文件，生成的文件在public目录

### 五丶访问

将puclic目录挂在到nginx，apache，iis等web容器就可以访问了，文章的路径就是在content目录的路径，比如content目录下的first.md可以用  `domain/first` 访问, `content/post/second.md` 可以用 `domain/post/second`访问

## 遇到的坑

1. 用hugo new命令生成的新文章，头部会有draft=true属性，表示为草稿，这样生成静态文件不会把该文章包括进去。要发布的话记得把这个值设置成false或者直接删除。