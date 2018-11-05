---
title: "Nginx学习"
date: 2018-11-04T10:23:42+08:00
showDate: true
draft: false
tags: ["nginx","linux"]
---

# Nginx配置学习总结

最近因为买了域名然后备完案了，就想着把服务器上的服务都用子域名来访问。然后就打算用nginx来做反代，但是nginx不是很熟悉，所以去图书馆借了本nginx的书，结合官方文档，学习了下nginx。

## 一丶nignx基础配置结构

`nginx.conf`的配置结构大概可以归纳为

```shell
···                                              # 全局块
events
{
    ···                                          # events块
}

http                                             # http块
{
    ···                                          # http全局块

    server                                       # server块
    {
        ···                                      # server全局块
        location                                 # location块
        {
            ···
        }

        location                                 # location块
        {
            ···
        }
    }

    server
    {
        ···
    }
}
```

### 各个块的作用

#### 全局块

设置nginx服务器整体运行的等配置指令:

1. 配置运行nginx服务器的用户(组)
2. 运行生成的worker_process数量
3. nginx进程PID存放路径
4. 日志存放路径的类型以及配置文件的引入
5. etc.

#### event块

影响nginx服务器与用户的网络连接的等设置:

1. 是否开启对多work_process下的网络连接进行序列化
2. 是否允许同时接受多个网络连接
3. 选区哪种事件驱动模型处理请求连接
4. 每个work_process可以同时支持的最大连接数
5. etc

#### http块

设置代理丶缓存和日志定义等

1. 文件引入
2. MIME-Type定义
3. 日志定义
4. 是否使用sendfile
5. etc

#### server块

server相当于是虚拟主机，每个server的配置相当于为虚拟主机配置

#### location块

每个server块可以包含多个location块，严格意义上讲，location其实是server块的一个指令

## 二丶nginx常用配置指令

| 作用 | 指令 |
|:------|:------|
|配置运行nginx服务器的用户/用户组|`user user [group]`|
|配置允许生成的worder process数量|worker_processes number \| auto|
|配置错误日志存放路径|error_log file \| stderr [debug \| info \| notice \| warn \| error \| crit \| alert \| emerg]|
|引入配置文件|include file|
|设置网络链接的序列化|accept_mutex on \| off|
|配置最大连接数|worker_connection number(默认为12)|

## 三丶虚拟主机的配置

在nginx中，http块里的每个server块都可以被看作一个虚拟主机

```js
server
{
    listen [address:][port] [default_server] [rcvbuf=size] [sendbuf=size] [deferred]
    server_name www.domain.com *.domain.com domain.* ~^www\.domain\.com;
    ...
    index file;
    location [= | ~ | ~* | ^~] uri {
        root path;
        ...
    }
}
```

**listen指令用于配置ip和监听端口**

- address， IP地址
- port，端口号，默认80
- default_server，标识符，将此虚拟主机设置为默认主机

**server_name用于配置域名**

server_name的匹配顺序是

确切域名→通配符开始的的最长字符串→通配符结尾的最长字符串→正则表达式

**location**

uri为待匹配的字符串，可以为正则表达式也可以为正则表达式，如果规定不含正则表的是的为”标准uri“， 含正则表达式的为”正则uri“， 中括号里的是可选项可以理解为：

- ”=“，用于标准url前，严格匹配
- ”~“，用于表示uri包含正则表达式，区分大小写
- ”~*“，用于表示uri包含正则表达式，不区分大小写

**root**

配置请求资源的根目录

**index**

指定默认的请求资源

## 实例

通过强大的nginx，我把自己的服务都跑在docker里，通过只能127.0.0.1:ip访问，然后用nginx给这些服务做反代，配上子域名，这样所有的服务在我服务器就只有一个入口了。配置文件和效果图如下

![nginx主配置文件](../nginx_conf.png)

![nginx博客配置文件](../nginx_blog_conf.png)

![nginx效果图](../nginx.png)