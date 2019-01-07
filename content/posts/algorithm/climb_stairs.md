---
title: "爬楼梯"
date: 2019-01-07T10:56:51+08:00
showDate: true
draft: false
tags: ["blog","story"]
---

>假设你正在爬楼梯。需要 n 阶你才能到达楼顶。
> </br>
>每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？
> </br>
>注意：给定 n 是一个正整数。
> </br>
>示例 1：
> </br>
>输入： 2
> </br>
>输出： 2
> </br>
>解释： 有两种方法可以爬到楼顶。
> </br>
>1.  1 阶 + 1 阶
> </br>
>2.  2 阶
> </br>
>示例 2：
> </br>
>输入： 3
> </br>
>输出： 3
> </br>
>解释： 有三种方法可以爬到楼顶。
> </br>
>1.  1 阶 + 1 阶 + 1 阶
> </br>
>2.  1 阶 + 2 阶
> </br>
>3.  2 阶 + 1 阶
> </br>

列出前面几个就找到规律了

```bash
1阶 ==> 1
2阶 ==> 2
3阶 ==> 3
4阶 ==> 5
5阶 ==> 8
```

直接斐波那契数列

```golang
func climbStairs(n int) int {
	if n == 1 || n == 2{
		return n
	}
	return climbStairs(n - 1) + climbStairs(n - 2)
}
```

但跑起来就知道了，一旦n大了直接就超时了，想了好久，想起来以前做过递归缓存，于是通过增加一个map来存已经计算过的阶数。

```golang
func climbStairs(n int) int {
	var m map[int]int
	m = make(map[int]int)
	return fib(n, m)
}

func fib(n int, m map[int]int)int{
	fmt.Println(m)
	if n == 1 || n == 2{
		m[n] = n
		return n
	}
	var ok bool

	if _, ok = m[n]; !ok{
		m[n] = fib(n-1, m) + fib(n-2, m)
	}
	return m[n]
}
```

## 后续

中途和丁公子想用数学的方法来做，通过排列组合

1阶 只有零个2一种情况

$$ C_0^0 = 1 $$

2阶 有一个2和零个2两种情况

$$ C_1^0 + C_2^2 = 2 $$

3阶 有一个2和零个二两种情况

$$ C_2^1 + C_3^3 = 3 $$

9阶

$$ C_5^1 + C_6^3 + C_7^5 + C_8^7 + C_9^9 = 55 $$

10阶

$$ C_5^0 + C_6^2 + C_7^4 + C_8^6 + C_9^8 + C_10^10 = 89 $$  （好像加载不了C1010~）

n阶

n为奇数

$$ C_{n/2+1}^{1} + ... + C_n^n $$

n为偶数

$$ C_{n/2}^{1} + ... + C_n^n $$

这种方法如果要换成代码的话可能要考虑到求阶乘，好像更麻烦了~