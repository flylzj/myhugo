---
title: "合并两个有序链表"
date: 2019-01-07T12:43:38+08:00
showDate: true
draft: false
tags: ["算法"]
categories: ["算法"]
---

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

![归并排序](./Merge-sort.gif)

先申请一个新的空节点，当两个链表指针指向的当前节点都不为空时，比较当前节点的Val，大的就放进空节点，然后自己指针后移，循环比较，当某个节点变成空指针的时候，把另一个节点剩下的部分拷贝到新节点。

```golang
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
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