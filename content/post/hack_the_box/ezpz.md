---
title: "Hack the Box——Ezpz"
date: 2019-12-30T12:26:33+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## ezpz

访问HTB给的地址，进来就看想两个类似报错的东西。

![1](1.png)


### 尝试

这报错一看应该是少传了什么参数吧。试试加上个get参数

`?obj=1`

第一行报错没了，剩下第二行

![2](2.png)

试试再加一个参数

`?obj=1&ID=1`

错误并没有消失。因为第一个报错和第二个报错的内容不是很一样，我猜ID这个参数会不会是从POST传过去的。

![3](3.png)

报错以然存在，看来思路错了。

仔细读一下第二行报错`尝试从一个空对象获取ID属性`，是不是ID是附加在obj参数里的呢。

又尝试了几种payload

```

?obj=ID%3d1
?obj={"ID":"1"}
?obj=<ID>1</ID>
```

依然无效。

### 转机

因为对于burp的repeater不是很有把握，我决定用python写脚本再试试。

我发现打印的response.text里的最后一行居然有个hint的注释

![4](4.png)

出题人用了好多空行把它藏在了最后导致burp一眼看到。

![5](5.png)

既然他给了hint，那就试试吧。

把上面几个payload分别用base64编码。最后`?obj={"ID":"1"}`编码后得到payload为

`?obj=eyJJRCI6IjEifQ==`

这个参数请求后两个错误都消失了

### 真正的挑战

虽然错误没了，但是并没有出现flag，那说明关卡还要继续。因为每次都要base64编码一下参数用burp有点麻烦。顺手写了个脚本

```python   

def injection(payload):
    params = {
        "obj": base64.b64encode(payload.encode())
    }
    url = "http://docker.hackthebox.eu:30548/index.php"
    r = requests.get(url, params=params)
    soup = BeautifulSoup(r.text, 'html.parser')
    print(soup.body.text.strip())

if __name__ == '__main__':
    payload1 = '{"ID": "1"}'
    injection(payload1)
```

先试试sql注入`payload1 = '{"ID": "1"}'`，返回了新的错误信息，但是好像没太多有用的。

![6](6.png)

不过也说明了他这里是存在注入的吧。

### 开始测注入

#### 整型还是字符串型？

尝试`payload1 = '{"ID": "1 and 1=2 #"}'`返回正常结果

再尝试`payload = '{"ID": "1\' and 1=2 #"}'`,返回空。

这么看起来应该是第二个的and有效果，至于第一个为什么能返回正常结果我也不太清楚，下次用php试试。

这里应该就能猜出他这个`ID`是字符型。

#### 继续猜信息



