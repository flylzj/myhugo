---
title: "Leetcode"
date: 2018-12-28T09:10:17+08:00
showDate: true
draft: false
tags: ["algorithm","story"]
---

# 这篇博客用来记录刷leetcode学到的有趣的东西~

## 2018-12-28

今天刷到的算法题


>给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。
>
>示例 1:
>
>输入: "abcabcbb"
>
>输出: 3
> 
>解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。


我自己想到的方法就只有暴力破解，找到字符串所有的字串，从最长的开始遍历，出现不含有重复字符的字串就返回他的长度，但是这种方法当给定字符串长度很长时我的代码在leetcode上就超时了。

看了解题思路提到了`滑动窗口`,没有很理解它上面的说法，于是查了下关于滑动窗口，这篇博客[http://blog.51cto.com/13642075/2088761](http://blog.51cto.com/13642075/2088761)让我懂了。

>假设序列元素从0开始编号，所求连续子序列的左端点为L,又短点为R。首先，先考虑L=0的情况。可以从R=0不断增加R，相当于，把所求序列的右端点往右延伸。当R不能往右继续延伸时（即出现相同的元素时），只需将L不断增加至没有相同的元素时为止。然后再不断增加R，重复此步骤。

index相当于是一个字符索引表，它存的是某个字符的索引位置，这样只需要遍历一遍字符串就能找到符合条件的最大子串

时间复杂度为 `O(n)`

```golang
func lengthOfLongestSubstring2(s string) int{
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

## 2018-12-31

今天刷到的算法题

>将两个有序链表合并为一个新的有序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。
>
>示例：
>
>输入：1->2->4, 1->3->4
>
>输出：1->1->2->3->4->4

一开始我的思路是先合并两个链表，然后排序。

看了下评论区某位的思路，利用类似归并排序

我也是第一次了解归并排序,我觉得这种图就比较让人理解了

![归并排序](Merge-sort.gif)

先申请一个新的空节点，当两个链表指针指向的当前节点都不为空时，比较当前节点的Val，大的就放进空节点，然后自己指针后移，循环比较，当某个节点变成空指针的时候，把另一个节点剩下的部分拷贝到新节点。

```golang
	if l1 == nil{
		return l2
	}else if l2 == nil{
		return l1
	}
	var res, p *ListNode
	res = &ListNode{}
	p = res
	for{

		if l1.Val > l2.Val{
			res.Val = l2.Val
			l2 = l2.Next
		}else{
			res.Val = l1.Val
			l1 = l1.Next
		}
		if l1 == nil{
			res.Next = l2
			break
		}else if l2 == nil{
			res.Next = l1
			break
		}
		res.Next = &ListNode{}
		res = res.Next
	}
	return p
	//var p, q *ListNode
	//p = l1
	//q = l1
	//if l1 == nil && l2 == nil{
	//	return nil
	//}else if l1 == nil && l2 != nil{
	//	l1 = l2
	//	p = l1
	//	q = l1
	//}else{
	//	for l1.Next != nil{
	//		l1 = l1.Next
	//	}
	//	l1.Next = l2
	//}
	//for{
	//	flag := 0
	//	for p.Next != nil{
	//		if p.Val > p.Next.Val{
	//			tmp := p.Val
	//			p.Val = p.Next.Val
	//			p.Next.Val = tmp
	//			flag += 1
	//		}
	//		p = p.Next
	//	}
	//	if flag == 0{
	//		break
	//	}
	//	p = q
	//}
	//return q
}
```

注释是我一开始的代码，也过了测试