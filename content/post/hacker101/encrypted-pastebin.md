---
title: "hacker101-Encrypted Pastebin"
date: 2020-08-13T22:38:28+08:00
showDate: true
draft: true
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---


# 前言

自从开始搞论文到毕业再到工作，已经很久没有玩ctf了，现在工作还算清闲，打算争取每周玩一道题吧。先把之前没做完的hacker101补了

## 开始

###  Encrypted Pastebin

进入页面首先是一段话，还有两个输入框，一个submit按钮，那句话大概是说他用了军事级别的128位AES加密，加密数据的密码不存在服务端，任何黑客都不可能获得授权。

#### flag1

![1.png](./1.png)

先随便输入点东西，然后点post试试，有一个302，跳到一个param是`post`的链接

![2.png](./2.png)

这个`post`的值应该就是提交表单被加密的结果。试试随便把post的值改一下。

![3.png](./3.png)

这就出flag了，开心~

#### flag2

不仅爆了flag，还有一些服务端的代码报错。

```
Traceback (most recent call last):
  File "./main.py", line 69, in index
    post = json.loads(decryptLink(postCt).decode('utf8'))
  File "./common.py", line 46, in decryptLink
    data = b64d(data)
  File "./common.py", line 11, in <lambda>
    b64d = lambda x: base64.decodestring(x.replace('~', '=').replace('!', '/').replace('-', '+'))
  File "/usr/local/lib/python2.7/base64.py", line 328, in decodestring
    return binascii.a2b_base64(s)
Error: Incorrect padding
```

稍微分析一次，出错的地方是base64解密错误，显然是因为`b64d = lambda x: base64.decodestring(x.replace('~', '=').replace('!', '/').replace('-', '+'))`这行代码里面的x不是base64串导致解码失败然后报错。


既然是base64解码错误，那我传一个自己构造的base64串会怎样呢，这次报了另一种错误。

```
Traceback (most recent call last):
  File "./main.py", line 69, in index
    post = json.loads(decryptLink(postCt).decode('utf8'))
  File "./common.py", line 48, in decryptLink
    cipher = AES.new(staticKey, AES.MODE_CBC, iv)
  File "/usr/local/lib/python2.7/site-packages/Crypto/Cipher/AES.py", line 95, in new
    return AESCipher(key, *args, **kwargs)
  File "/usr/local/lib/python2.7/site-packages/Crypto/Cipher/AES.py", line 59, in __init__
    blockalgo.BlockAlgo.__init__(self, _AES, key, *args, **kwargs)
  File "/usr/local/lib/python2.7/site-packages/Crypto/Cipher/blockalgo.py", line 141, in __init__
    self._cipher = factory.new(key, *args, **kwargs)
ValueError: IV must be 16 bytes long
```

这里多了一个信息，可以看到`cipher = AES.new(staticKey, AES.MODE_CBC, iv)`说明服务端加密用的是CBC，查一下可以知道`CBC加密需要一个十六位的key(密钥)和一个十六位iv(偏移量)`，既然报错是因为IV不是16比特长度，说明IV和post的值有关。

尝试传一个`{“iv”: "asdasd"}`过去，会有什么效果。这次又是不一样的报错了。这次是数组越界，越来越懵逼了。

```
Traceback (most recent call last):
  File "./main.py", line 69, in index
    post = json.loads(decryptLink(postCt).decode('utf8'))
  File "./common.py", line 49, in decryptLink
    return unpad(cipher.decrypt(data))
  File "./common.py", line 19, in unpad
    padding = data[-1]
IndexError: string index out of range
```

再试试`{"iv": "asdasd","title":"asd", "body":"asd"}`的base64串，报错又不一样了，有毒。。。

```
Traceback (most recent call last):
  File "./main.py", line 69, in index
    post = json.loads(decryptLink(postCt).decode('utf8'))
  File "./common.py", line 49, in decryptLink
    return unpad(cipher.decrypt(data))
  File "/usr/local/lib/python2.7/site-packages/Crypto/Cipher/blockalgo.py", line 295, in decrypt
    return self._cipher.decrypt(ciphertext)
ValueError: Input strings must be a multiple of 16 in length
```

虽然有这些报错，



https://zhuanlan.zhihu.com/p/131324301
https://www.cnblogs.com/wh4am1/p/6557184.html
https://paper.seebug.org/1123/
https://www.freebuf.com/vuls/98156.html
http://www.361way.com/aes/5830.html








