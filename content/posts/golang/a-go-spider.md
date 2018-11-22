---
title: "A Go Spider"
date: 2018-11-21T21:06:42+08:00
showdate: true
---

最近两周学了下golang，看了下基础语法（暑假留校的时候就已经看过了，但是因为有其它项目要做所以就没有了后续），对go有了一定的了解，听说go的goroutine对并发很友好。为了巩固对自己的go语言学习，花了一周的时间写了个go的爬虫。

## 关于这个爬虫

这个爬虫是帮一个做代购的朋友写的，爬的是一个app里面的数据，目的是拿到这个app所有上架的代购商品，之前已经用python实现了。但是之前那个写得并不是很好，这次重新设计了下思路。

## 爬虫设计

这个app是通过分类来展示商品的，就是说每种商品都有自己的categoryId。那么只要拿到所有的categoryId就能通过分页拿到所有商品了。但是还有一个问题，就是我们不知道每个分类有多少页数（有的分类有几百页，有的分类只有几十页甚至没有），如果用多个goroutine来爬取，就不知道什么时候停止。Google了很多都没有找到适合我的解决方案，于是我自己设计了几种。

1. 第一种方法，因为不知道商品最大页数，干脆就每个分类都构造一千页，分类有四十多种，那一共就四万多url。但是后来觉得这种做法很浪费资源，就否决掉了

2. 第二种方法，负责构造url的函数只把第一页的url丢进urlChan,获取url信息的函数在拿到body之后如果里面的data少于20（每页最大数量）就把当前page+1然后丢回urlChan。这种方法如果我开100个goroutine来获取页面，但是因为urlChan一开始就四十几个url(因为就四十几种商品),那就剩下很多goroutine是空闲状态，而且速度很慢。

3. 第三种方法，这种方法是从第二种方法稍微改进而来的。因为每次丢进urlChan的url太少了，然后我就改成了如果`page % 10 == 1`,就一下丢十个url进去。从page=1开始，一下就有400多个url进了urlChan，就能保证每个goroutine都没有空闲，而且请求空页的次数也变少了（运气不好的好每个分类也就请求了十个空页），资源也怎么浪费。90秒不到六万多的数据都存进了sqlite，goroutine真是牛逼呀。

```js
start at: 2018-11-22 08:41:54.7265882 +0800 CST m=+0.007978701
end: 2018-11-22 08:43:22.8076787 +0800 CST m=+88.089069201
```

```go
func GetOnePageGoods(urlChan chan [2]string, goodChan chan Good, token string, wg *sync.WaitGroup) {
    ···
		if goodsCount == 20{
			page, _ := strconv.Atoi(urlArr[0])
			if page % 10 == 1 {
				fmt.Println("page", page)
			    for i := 0; i < 10; i ++ {
			    	page = page + 1
					urlChan <- [2]string{strconv.Itoa(page + 1), urlArr[1]}
				}
			}
        }
    ···
	}
}
```

## 遇到的坑

因为是第一次写go的小项目，所以还是遇到了比较多的坑的

* 关于go的channel，如果不加缓存的话，它是一种阻塞操作，就必须发送者和接收者都准备好才能进行通信。如果channel里没有元素了，那接收者也会处于阻塞状态。这就导致我的`GetOnePageGoods`函数如果url再发过来就会进入阻塞状态，因为这个函数我写的是死循环，我一开始甚至不知道怎么结束整个程序。网上查了后知道了可以通过select关键字来设置队列超时。

```go
		var urlArr [2]string
		select {
		    case urlArr = <- urlChan:
		    	//
		    case <- time.After(time.Second):
		    	fmt.Println("no url")
		        urlArr[0] = ""
		}
		if urlArr[0] == "" {
			break
		}
```

* 关于go的json解析，go自带的`encoding/json`包有解析json的方法，但是需要定义`struct`来匹配json，对于需要处理复杂json的爬虫不是很适用，于是我改用了`github.com/bitly/go-simplejson`。比较适合复杂json

```go
	jsonDate, err := simplejson.NewJson(responseBody)
	if err != nil {
		fmt.Println("json error")
		fmt.Println(err)
		return nil, err
	}
	return jsonDate, nil
```

* 关于goroutine，就是前面的问题，我不知道怎么，什么时候去关闭goroutine，网上看到一篇《如何优雅的退出goroutine的博客》，里面有句话让我印象比较深刻，`当你开始一个goruntine的时候，一定要知道goruntine会在何时停止。`我这里关闭goroutine的方式就是它没有url需要处理的时候就会自己退出，这样100个goroutine慢慢的减少直到爬虫结束，goroutine全部结束。虽然效果是达到了但感觉还不是很优雅，我可能还需要去看看别人的代码，多写写。

* 关于数据库，一开始我是从goodChan一个一个拿出good然后拼接sql，存入数据库，但是发现速度极慢，插入一条数据得花几百毫秒，爬虫都结束了还有也就插入了一千多条数据。然后我想到了多几个goroutine来插入数据，配上锁，还是很慢。找了各种sqlite的并发方法，都没有效果。后来想到如果我把所有的insert语句拼成一条insert语句，只执行一次sql，会怎样。然后woc，几百毫秒？

```go
func DomToDB(goodChan chan Good, wg *sync.WaitGroup) {
	sqlStr := fmt.Sprintf("INSERT OR REPLACE INTO good(abiid, mainname, price, stock) values")
	for {
		fmt.Println("还剩", len(goodChan), "个物品")
		var good Good
		sign := false
		select {
		case good = <-goodChan:
			//
		case <-time.After(time.Second * 5):  //因为网络延迟需要等待一定的时间才能确定是没有商品了
			fmt.Println("no goods")
			sign = true
		}
		if sign {
			break
		}

		values := fmt.Sprintf("(%d, '%s', %d, '%s'),", good.Abiid, strings.Replace(good.Mainname, "'", "\"", -1), good.Price, good.Stock)
		sqlStr += values
	}
	sqlStr = strings.Trim(sqlStr, ",") // 删掉sql末尾的逗号
	db, _ := sql.Open("sqlite3", "good.db")
	res, err := db.Exec(sqlStr)
	if err != nil{
		fmt.Println(err)
	}
	res.RowsAffected()
	defer db.Close()
	defer wg.Done()
}
```
