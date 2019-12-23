---
title: "hack101 level 10 && level 11"
date: 2019-12-04T16:40:15+08:00
draft: false
tags: ["security","hacker", "ctf"]
categories: ["security"]
---

# 第十关 Petshop Pro

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

![10-1](./10-1.png)

应该是常规的url encode，解码看一下。

![10-2](./10-2.png)

传这么多数据干嘛？试试少几个字段会怎样，发现只有name和price是可用的，而且返回的结果和这个有关。
尝试把price都改成0。第一个flag就出来了。

![10-3](./10-3.png)

### flag2

剩下几个api也没找到什么问题，后来看了下hint。`There must be a way to administer the app`，这是在让我尝试找登录点嘛。

于是我试了下`/login`，果然有登录界面

![10-4](./10-4.png)

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

![10-5](./10-5.png)

登录后第二个flag也有了。

![10-6](./10-6.png)

### flag3

登陆之后多了一个`/edit?id=0`页面，分别试了一下xss和注入，都没出flag。
各种尝试无果，
看了下hint，`Bugs don't always appear in a place where the data is entered`。有点像csrf哦。

构造payload`<img src="http://xxx/add/2">`，然后访问`http://xxx/cart`，get it~


![10-7](./10-7.png)

 
# 第十一关 TempImage


老套路先看有哪些可以访问的路径，

```

/
/upload.php
POST /doUpload.php
```

这题应该是考绕过上传。

### flag1

先随便上传个空文件，结果返回`ERROR: Only PNG format supported in trial.`

改后缀名，返回
```
<br />
<b>Notice</b>:  getimagesize(): Read error! in <b>/app/doUpload.php</b> on line <b>2</b><br />
ERROR: Only PNG format supported in trial.
```

换个正常的png，把filename改成`test.php%00png`尝试截断文件名，但是好像没有卵用。

试试加上路径符号，`test.png../php`，flag就这样出来了？我也不懂。

![10-8](10-8.png)

### flag2（伪

第二个flag我一开始想的是上传一个png后缀的php文件上去，查了下上传文件绕过，发现有一种可行的办法。
因为从上面可以知道他这里是通过php的getimagesize来实现文件类型判断的。所以只要把文件头部加上png的二进制内容就可以了。

用十六进制编辑器打开文件在文件头部加入

`8950 4E47 0D0A 1A0A`

然后在上传就会被识别成png图片

![10-9](10-9.png)
![10-10](10-10.png)


然后在上传的时候我试着把filename改成php后缀，但是在访问的时候还是并没有执行phpinfo()，对php不是很熟悉，但是我猜想应该是在这个目录里的php不会执行。
再次尝试

![10-11](10-11.png)

这个请求会把文件放到他原本的php所在位置，这样果然可以执行（注：cmd参数的值最后一定要加`';'`,可能是php像c那样严格控制冒号吧，在这里卡了好久，还以为我的一句话写错了）

![10-12](10-12.png)

虽然能够执行命令了，但是我还是不知道flag在哪，一顿乱找，最后在/app目录里面找到了一个flag。

![10-13](10-13.png)

不过这个flag一提上去发现跟之前那个路径的是同一个，甘凌娘！
不过也算是学到了一个新姿势。

### flag3 （真

虽然用上面的命令查看了app目录里面所有的文件内容，没有找到flag，但是因为`xargs`用的不多，所以我始终有点怀疑。
抱着怀疑的心态我尝试了挨个使用cat命令。果然在index.php里面发现了一个带着注释的flag。

![10-14](10-14.png)

然后我又在burp里跑了一下前面那个xargs命令，原来并不是没有，只是浏览器不显示注释。
以后还是直接看burp吧。
