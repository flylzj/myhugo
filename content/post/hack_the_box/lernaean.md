---
title: "Hack the Box——Lernaean"
date: 2019-12-26T17:31:37+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---


本来打算按顺序做web题的，但是一上来就被第一题虐了一整天。还是先整点简单的题目吧~

## Lernaean

这题的看起来就只有一个输入的密码界面

![1](1.png)

试了下注入，没用。

再试了试用中文字符，也没有爆出啥错误。

然后我想着刚好到了吃饭时间，就丢到burp的intruder里，然后跑去吃饭。

吃完饭回来发现，这果然是道爆破题~

![2](2.png)
