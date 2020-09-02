---
title: KMP算法个人理解
tags: 
  - 算法 
  - LeetCode 
  - Golang
date: 2018-01-18 18:34:30
---

## 简介

> KMP算法是一种改进的字符串匹配算法，由D.E.Knuth，J.H.Morris和V.R.Pratt同时发现，因此人们称它为克努特—莫里斯—普拉特操作（简称KMP算法）。

KMP算法的关键是利用匹配失败后的信息，尽量减少模式串与主串的匹配次数以达到快速匹配的目的。具体实现就是实现一个next()函数，函数本身包含了模式串的局部匹配信息。时间复杂度O(m+n)。

## 理论介绍

设目标字符串target为：
`a b a c a a b a c a b a c a b a a b b`

模式字符串model为：`a b a c a b`

### 计算next数组

对此model串的next数组计算为`[0 0 1 0 1 2]`
计算规则为：截止到此index，首尾重复的子串最大长度。

* a 规定为0
* ab 首尾子串a和b，重复长度为0
* aba 首部子串a，ab，尾部子串a，ba，重复1
* abac 首部子串a，ab，aba，尾部子串c，ac，bac，长度为0
* abaca 首部子串a，ab，aba，abac，尾部子串a，ca，aca，baca，长度为1
* abacab 首部子串a，ab，aba，abac，abaca，尾部子串b，ab，cab，acab，bacab，长度为2

next数组就是这样子去算，不过从上述过程可以发现，next数组，上次的数值和下次的数值是有所关系的，有三种情况会发生：1.归零；2.归一；3.加一；

> 归零是加上一位字符后，没有重复首尾没有重复了，甚至首位和尾位单字符都不相同了，就归零。
归一是加上加上一位字符后，只剩下首位和尾位单字符相同。
加一是加上一位字符后，碰巧和首部那串字符相同顺序，多一个字符，就是多的那一个子串的长度。

举个例子对于字符串abcabcad
next[0]=0
next[1]=0
next[2]=0
next[3]=0+1
next[4]=1+1
next[5]=2+1
next[6]=1
next[7]=0

### 第一次匹配

![](http://static.nicesite.vip/2018-06-12-085014.jpg)

我们可以看出在model的index=5的字符（规定第一个为index=0）出现不匹配

那么我们需要把model后移，再次进行匹配
（移动是相对的，说把target前移或者model后移都可以的咯）

**此时就出现了问题，后移多少位？**

* 传统算法认为后移一位，继续匹配是最稳健安全的，但是这样太过浪费时间，设target长度为n，model的长度为m，那么复杂度为O(n*m)
(其实能优化成(n-m)*m)

* KMP算法则使model尽可能多的后移，我们看到，前面已经匹配了5个字符了，而next[4]=1，说明index=4的地方和index=0的地方是相等的。那么target与model在index=4的地方能匹配，就等同于在index=0的地方也能配对。我们此时移动了4位，这个4即5-1，已配对的长度-next[index-1]

### 第二次匹配

![](http://static.nicesite.vip/2018-06-12-085035.jpg)

index=1时不匹配，移动index-next[index-1]即1-0=1

### 第三次匹配

![](http://static.nicesite.vip/2018-06-12-085058.jpg)

很完美，匹配成功了

### 再换个例子解释下移动

#### 例2

![](http://static.nicesite.vip/2018-06-12-085117.jpg)

![](http://static.nicesite.vip/2018-06-12-085129.jpg)

前面匹配了6个字符ABCDAB，而最后的D未匹配，D前面的字符next[6-1]=2，6-2=4，所以model串向后移动了4个位置

### 换个角度来看待移动

我们看例2：
对于target串的指针位置，规定为i，model串的指针位置规定为j。
如果使得i从0到len(target)-1，每次i++，而且只增加不减少，只对j进行各种回退，即可实现复杂度为O(n)
此时i=10，j=6，不匹配。
之后i不变，j=next[j-1]=2，继续匹配，i++，j++
这样就相当于model串后移了6-2=4位。

## 算法代码

代码使用golang书写

```go
//时间复杂度为o(n)其中n为str长度
func getNext(model string) []int {
	len := len(model)
	// 声明动态长度为len的一个next数组
	next := make([]int, len)
	for i := 1; i < len; i++ {
		// 无非归零，归一，加一三种情况
		if next[i-1] > 0 && model[next[i-1]] == model[i] {
			next[i] = next[i-1] + 1
		} else if model[0] == model[i] {
			next[i] = 1
		}
		// 不修改表示next[i]=0
	}
	return next
}
```

```go
//时间复杂度为o(m)其中m为str长度
func KMP(model, target string) bool {
	next := getNext(model)
	same := 0
	// 保证i始终不退回，j尽量少退回
	for i, j := 0, 0; i < len(target); i++ {
		if model[j] == target[i] {
			same++
			j++
		} else if j > 0 {
			// 前面一位的next值表示各种回退后此时匹配的字符数
			same = next[j-1]
			// j就回退到same的值，前j位都匹配完成了，只需要往后匹配
			j = same
		}
		if same == len(model) {
			// 此时刚好匹配完成，开始位置为i-length+1，结束位置为i
			return true
		}
	}
	return false
}
```

