---
title: "记一次小项目经历"
date: 2018-12-04T16:02:26+08:00
showDate: true
draft: flase
tags: ["golang","spider"]
categories: ["golang"]
---

# 开始

之前用go把python的一个爬虫+tk的一个项目的功能用go实现了，goroutine爬取速度感人，之前python五万数据要20分钟（之前的设计也有问题），现在只要两分钟。发给要用的那位朋友之后他觉得很好用，然后希望我把整个项目都用go重构了，我想这既能帮到他也能练练go就答应了他。

## 技术选择

开始的时候我就想用go的gui库来实现，网上搜了一圈，go好像不像python那样有自带的tk，go没有官方的gui，比较热门的就是[`gotk`](https://github.com/gotk3/gotk3), [`gogtk`](https://github.com/mattn/go-gtk/), `go-qt`这几个，他们的都是需要安装各种依赖的，尝试了下`gotk`和`gogtk`，光是装依赖和解决安装时候的问题都花了好久，装好本想看文档好好学学，但是发现他喵的好像没有文档，只有github上的example和demo。在找文档和时候还在reddit上发现了`go-gtk`和`go-sqlite3`的作者，我还尬问了一下。

![reddit](./reddit.png)

没有文档就只能照着他的example还有自己之前python tk的经验来慢慢写了，渐渐的也熟悉了一点。等到界面和逻辑都写好了，但是发现奇丑无比，而且还有很多bug。

![go-gtk](./go-gtk.png)

看起来比我之前写得tk界面丑多了，本来想着代码写好了再来优化界面，但是发现这样进度又很慢。因为要期末考试我还得复习，所以可能时间会不够，然后就在我考第一场的时候，那个朋友给我发了个微信说那个老版的软件总是运行一段时间就自己挂了（开卷考试然后老师允许我们用手机，老师可能低估了我们对手机的掌握程度~）。然后我就想到如果把服务直接跑在我服务器上，然后让康康写个前端页面，这样不是又稳定有持久嘛。考完试我就跟他说了我的想法，但是他又有点犹豫，她可能觉得不想出服务器的开支。回寝室的路上我又想到，如果我直接跑服务端在本地，页面也直接在本地就完事儿了吧。
于是我就决定了，我用go写爬虫和后端，让康康用vue帮我前端。

## 产品设计

为了能让康康快速的理解我要做什么，我一个小开发专门去研究了下墨刀然后画了一个原型图，虽然有点丑，但是至少把我自己的想法表达出来了~

![demo](./demo.png)

## 产品开发

画完原型图因为没有设计，康康就直接开始写，我也就开始设计数据库了。我比较喜欢用Xmind先设计好表的结构，这里一共四张表。

![table](./table.png)

建完表我开始考虑用原生go还是使用框架，我去看了看小黑在工作室gitlab的代码，发现了`gin`，感觉这个框架很不错，于是我就打算用`gin`了。数据库这块因为要跑在本地，那还是用`sqlite`比较合适了,因为写`python`习惯了用orm，这次也就直接用了`gorm`（gorm的文档很全，也挺详细的）。

之所以我会想到做web还有一个原因就是之前写过一个类似的项目，我发现可以借鉴过来。大概的框架就是主线程运行后端api，一个goroutine运行定时爬虫，一个goroutine运行前端web容器。从main函数也能看出来了😁。

```go
func main(){
	go spider.MainSpider()
	go StaticRouter()
	fmt.Println("静态资源加载完成")
	fmt.Println("若系统没有自动打开，请手动在浏览器输入http://127.0.0.1:8081")
	StartRearEndRouter()
}
```

爬虫块是我早就写好了的，稍微改改就好了。后端api直接就rest风格，配置路由和Handle函数就行了,看起来和flask的app.route()差不多，挺好理解的。而且gin的Context指针可以直接返回Json，这一点也很不错。

```go
func StartRearEndRouter(){
	router := gin.Default()
	router.Use(Cors()) // 跨域
	router.GET("/api/time_interval", getTimeConf)
	router.POST("/api/time_interval", updateTimeConf)
	router.GET("/api/email", getEmailConf)
	router.POST("/api/email", updateEmailConf)
	router.GET("/api/history/:abiid", getGoodHistory)
	router.GET("/api/good", getBeMonitoredGoods)
	router.POST("/api/good", addGood)
	router.DELETE("/api/good", deleteGood)
	router.POST("/api/good/upload", getExcelFile)
	router.Run(":8080")
}
```

写完后端api，就直接和康康一起测试了，因为不在一个局域网，所以就是我编译一个exe给他，他打包一个静态文件给我，遇到问题我改完再重新给他编译一个。测完之后，晚上上课我又想到了一般rest项目，前端是静态文件，需要一个web容器，但是我总不能在朋友那里装一个nginx吧。我就想试试直接用go写一个简单http服务器不知道可行不可行，没想到直接就可以用了，太强了8。就开一个http服务就可以了。

```go
func StaticRouter(){
	http.Handle("/", http.FileServer(http.Dir("vue-static")))
	http.ListenAndServe(":8081",nil)
}
```

## 测试

好像现在也不太会也单元测试，只会用postman测测api，这个以后得学学。（暑假的时候还因为测试用例不足导致工作室出了一次事故，那时候刚好开学季，流量贼大）所以这次测试只是我们前后端简单的点点鼠标，流程走一遍，没有问题就算测试完成了。

## 上线

把编译好的exe和静态文件发给他也就算上线了，界面还是很丑，但感觉比我写gui好多了

![project](./product.png)

## 总结

这算是我和康康两个人一起完成的第二个项目了，耗时两天，比我写gui效率高多了，以后类似的项目感觉也可以借鉴一下这种模式。