---
title: "hacker101 第11关"
date: 2019-12-23T11:59:40+08:00
showDate: true
draft: false
tags: ["security","hacker101", "ctf"]
categories: ["security"]
---

## 第11关 TempImage


老套路先看有哪些可以访问的路径，

```

/
/upload.php
POST /doUpload.php
```

这题应该是考绕过上传。

### flag1

先随便上传个空文件，结果返回`ERROR: Only PNG format supported in trial.`

改后缀名，返回
```
<br />
<b>Notice</b>:  getimagesize(): Read error! in <b>/app/doUpload.php</b> on line <b>2</b><br />
ERROR: Only PNG format supported in trial.
```

换个正常的png，把filename改成`test.php%00png`尝试截断文件名，但是好像没有卵用。

试试加上路径符号，`test.png../php`，flag就这样出来了？我也不懂。

![11-1](11-1.png)

### flag2（伪

第二个flag我一开始想的是上传一个png后缀的php文件上去，查了下上传文件绕过，发现有一种可行的办法。
因为从上面可以知道他这里是通过php的getimagesize来实现文件类型判断的。所以只要把文件头部加上png的二进制内容就可以了。

用十六进制编辑器打开文件在文件头部加入

`8950 4E47 0D0A 1A0A`

然后在上传就会被识别成png图片

![11-2](11-2.png)
![11-3](11-3.png)


然后在上传的时候我试着把filename改成php后缀，但是在访问的时候还是并没有执行phpinfo()，对php不是很熟悉，但是我猜想应该是在这个目录里的php不会执行。
再次尝试

![11-4](11-4.png)

这个请求会把文件放到他原本的php所在位置，这样果然可以执行（注：cmd参数的值最后一定要加`';'`,可能是php像c那样严格控制冒号吧，在这里卡了好久，还以为我的一句话写错了）

![11-5](11-5.png)

虽然能够执行命令了，但是我还是不知道flag在哪，一顿乱找，最后在/app目录里面找到了一个flag。

![11-6](11-6.png)

不过这个flag一提上去发现跟之前那个路径的是同一个，甘凌娘！
不过也算是学到了一个新姿势。

### flag3 （真

虽然用上面的命令查看了app目录里面所有的文件内容，没有找到flag，但是因为`xargs`用的不多，所以我始终有点怀疑。
抱着怀疑的心态我尝试了挨个使用cat命令。果然在index.php里面发现了一个带着注释的flag。

![11-7](11-7.png)

然后我又在burp里跑了一下前面那个xargs命令，原来并不是没有，只是浏览器不显示注释。
以后还是直接看burp吧。

![11-8](11-8.png)

