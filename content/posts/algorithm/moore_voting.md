---
title: "Moore_voting"
date: 2019-02-02T09:31:48+08:00
showDate: true
draft: true
tags: ["algorithm"]
---

## 2019-2-1

### 求众数

给定一个大小为 n 的数组，找到其中的众数。众数是指在数组中出现次数大于 ⌊ n/2 ⌋ 的元素。

你可以假设数组是非空的，并且给定的数组总是存在众数。

示例 1:

输入: [3,2,3]
输出: 3
示例 2:

输入: [2,2,1,1,1,2,2]
输出: 2

这道题第一反应就是在遍历数组的时候用map把数字出现的次数存起来

```go
func majorityElement(nums []int) int {
	var numMap map[int]int
	numMap = make(map[int]int)
	for _, num := range nums{
		if _, ok := numMap[num]; !ok{
			numMap[num] = 1
		}else{
			numMap[num]++
		}
		count := numMap[num]
		if count > len(nums) / 2{
			return num
		}
	}
	return 0
}
```

但是如果不用map,还有没有第二种实现方法呢.看到评论区有人说了摩尔投票法,查了一下,大概实现了一下

```go
func majorityElement(nums []int) int {
	var most, count int
	count = 0
	for _, num := range nums{
		if count == 0{
			most = num
			count++
		}else{
			if most == num{
				count++
			}else{
				count--
			}
		}
	}
	return most
}
```

