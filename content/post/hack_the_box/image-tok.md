---
title: "Image Tok"
date: 2020-09-17T21:22:55+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00

Image Tok,又是一道将近一半人投了brainfuck难度的题目。

## 0x01 初探

访问主页，又是熟悉的画风，这道题的作者还是出`interdimensional internet`那题的那个。画风都差不多。

![1.png](1.png)

养成习惯，先看html源码。

![2.png](2.png)

只有一个上传表单。

先抓包看看逻辑，上传完成之后会跳转到一个`/image/xxx.png`路径,里面只有一张刚上传的图片。看了下源码，是base64编码的图片。

![3.png](3.png)

## 0x02 迷茫

暂时只能找到这么多有用的东西，先看上传点有没有什么漏洞。

### 测试上传

大概试了一些文件发现，服务端会校验是否为PNG文件。校验原理其实就是看文件头部的一些值，具体可以看[PNG文件结构分析 ---Png解析](https://www.cnblogs.com/lidabo/p/3701197.html)。

搜了下关于文件上传的漏洞，校验文件头的话可以通过伪造头部的方式来绕过。具体就是在比如php木马前面加上PNG头部信息，伪造成PNG图片。

试了下这招，但是因为他这个返回图片内容返回的是base64编码过后的图片。php木马都被编码进去了，而且因为返回的文件被强制改名了，这招算是没用了。

![4.png](4.png)
![5.png](5.png)

上传还有关于条件竞争的，我感觉可能可以利用，但是暂时不知道怎么利用，上传这个就暂时搁置了。

### 解密session

还有一个看起来比较像是有漏洞的地方就是`PHPSESSID`，他返回的session id是一个长得有点像jwt但又不是jwt的东西`eyJmaWxlcyI6W3siZmlsZV9uYW1lIjoiMWZmNjMucG5nIn0seyJmaWxlX25hbWUiOiI1ZmUzNy5wbmcifV0sInVzZXJuYW1lIjoiNWY2MzYzZjJlNTQ4NyJ9.JDJ5JDEwJFg3VTV6WTY1N0dtMjNBVEhBNG9ZYnVrNUJ5LmR5Q2NRTTRoLkc4Z1FWSUJoZTdJN3Z6TS5l`，

前半段解码出来是一个json串

```
{"files":[{"file_name":"1ff63.png"},{"file_name":"5fe37.png"}],"username":"5f6363f2e5487"}
```

后半段解码出来是一个bcrypt hash

```
$2y$10$X7U5zY657Gm23ATHA4oYbuk5By.dyCcQM4h.G8gQVIBhe7I7vzM.e
```

因为没写过php的web，而且以前遇到的sessionid也不长这样，所以看起来这里是有问题的。

猜测一下他这个应该是前半段通过bcrypt加密得到后半段，后半段就类似一个签名，用来验证前半段的完整性。

试了下用前半段的base64串、json串、username值做bcrypt的验证，都失败了。这个地方也卡住了。

### 发现定时任务

在测试上传的时候，我还发现了一个问题，刚上传的图片，第二次访问就404了，但是有时候又能访问很多次，我猜可能是有一个定时任务一直每隔一段时间就在清上传的图片，所以如果刚上传完然后就碰到定时任务，那很可能就直接被删除了。带着这个想法，写了个代码看他到底每隔多久删除一次文件

```
def test_alive_time():
    img_url = upload_image('1.png')
    start_time = time.time()
    r = requests.get(img_url)
    global proxies
    while r.status_code != 404:
        r = requests.get(img_url, proxies=proxies)
    end_time = time.time()
    return img_url, end_time - start_time

if __name__ == '__main__':
    test_time = 10
    while test_time > 0:
        print(test_alive_time())
        test_time -= 1
```

就上传一个文件然后一直访问这个文件直到他404，第二次上传图片的时候并知道它404的时间应该就是时间间隔了

![6.png](6.png)

从输出结果可以看到，时间间隔应该是两分钟，那么这个有什么用呢？我也不知道。

## 0x03 转机

卡了很久，我现在的思路有那么几个

1. 上传那块，是不是可以通过上传某些字符来base64产生我们想要的字符串
2. session那块是不是可以通过爆破来拿到他的密钥从而伪造session

但是带着这两个思路搜了很久都没有进展。

在无聊刷新界面的时候，我发现，在访问带有php后缀的路径的时候和其它路径的时候响应虽然都是404，但是内容确实不一样的。

php后缀响应的是这样的

![7.png](7.png)

不带php后缀的是这样的

![8.png](8.png)


我猜测是php后缀的在nginx那一层就被拦下来并返回404了，而其它后缀的是到了php这层，由php来抛了这个404。

另外一个思考就是，我在访问`/image/xxx.png`的时候，他居然不是直接返回一张图片，而是返回一个html文档，然后里面加上了base64编码后的图片。那一定是代码层面控制了。猜测逻辑如下

`访问xxx.png`=>`匹配到某个路由函数`=>`读取xxx.png的内容（不存在返回404）`=>`返回加上了base64图片的html文档`

那么，如果这个`xxx.png`换成`/etc/passwd`,他还会去读取吗。

这个思路是在下班回家的地铁上突然想到的，但是地铁上信号不好，访问老是超时，到家了我立马试了试。

芜湖，起飞~

![9.png](9.png)

可以看到，响应内容不是404，只是base64那里可能是因为不是图片然后报错了，但起码证明，思路对了。


## 0x0

[PNG文件结构分析 ---Png解析](https://www.cnblogs.com/lidabo/p/3701197.html)