---
title: "最长字串"
date: 2018-12-28T09:10:17+08:00
showDate: true
draft: false
tags: ["algorithm","story"]
---

## 2018-12-28

今天刷到的算法题

>给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。
></br>
>示例 1:
></br>
>输入: "abcabcbb"
></br>
>输出: 3
></br>
>解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。

我自己想到的方法就只有暴力破解，找到字符串所有的字串，从最长的开始遍历，出现不含有重复字符的字串就返回他的长度，但是这种方法当给定字符串长度很长时我的代码在leetcode上就超时了。

看了解题思路提到了`滑动窗口`,没有很理解它上面的说法，于是查了下关于滑动窗口，这篇博客[http://blog.51cto.com/13642075/2088761](http://blog.51cto.com/13642075/2088761)让我懂了。

>假设序列元素从0开始编号，所求连续子序列的左端点为L,又短点为R。首先，先考虑L=0的情况。可以从R=0不断增加R，相当于，把所求序列的右端点往右延伸。当R不能往右继续延伸时（即出现相同的元素时），只需将L不断增加至没有相同的元素时为止。然后再不断增加R，重复此步骤。

index相当于是一个字符索引表，它存的是某个字符的索引位置，这样只需要遍历一遍字符串就能找到符合条件的最大子串

时间复杂度为 `O(n)`

```golang
func lengthOfLongestSubstring(s string) int{
	var index [256]int
	index = [256]int{}
	n := len(s)
	var i, ans int
	i = 0
	ans = 0
	for j:= 0;j<n;j++{
		if index[s[j]] > i {
			i = index[s[j]]
		}
		if j - i + 1 > ans{
			ans = j - i + 1
		}
		index[s[j]] = j + 1
	}
	return ans
}
```