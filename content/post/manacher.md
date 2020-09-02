---
title: manacher算法
date: 2018-01-26 09:39:41
tags:
  - 算法
  - LeetCode
  - Golang
---

## 原题再现

LeetCode problem No.5

> Given a string s, find the longest palindromic substring in s. You may assume that the maximum length of s is 1000.  
> 从字符串s中找出最长的回文子串，s的长度不会超过1000

Example:
Input: "babad"
Output: "bab"
Note: "aba" is also a valid answer.

Example:
Input: "cbbd"
Output: "bb"

### 通俗解法

```go
func longestPalindrome(str string) string {
	start, end, len1, len2, lenPalindrome := 0, 0, 0, 0, 0
	for i := 0; i < len(str); i++ {
		// 奇偶要分开讨论
		len1 = palindromeLen(str, i, i)
		len2 = palindromeLen(str, i, i+1)
		lenPalindrome = maxInt(len1, len2)
		if lenPalindrome > end-start {
			start, end = i-(lenPalindrome-1)/2, i+lenPalindrome/2
		}
	}
	return str[start:end+1]
}

// golang中没有max，min这类函数，自己写一个
func maxInt(a, b int) int {
	if a > b {
		return a
	} else {
		return b
	}
}

func palindromeLen(str string, left, right int) int {
	i := 0
	for i = 0; left-i >= 0 && right+i < len(str); i++ {
		if str[left-i] != str[right+i] {
			break
		}
	}
	return 2*i - 1 + right - left
}
```

### 启发思考

受到之前kmp算法的提示，我们应该能感受到，这种字符串找规律的题目，应该会有更加操作简便的解法。

* 分开讨论的部分能不能一起讨论？
* 能不能整理出一个类似kmp的next数组的规律数组？
* 能不能以线性的复杂度搞定？

## 介绍

> 马拉车算法Manacher's Algorithm是用来查找一个字符串的最长回文子串的线性方法，由Manacher在1975年发明的，这个方法的最大贡献是在于将时间复杂度提升到了线性。

这个算法解决了奇偶分开讨论的问题，把冗余的循环节省掉，将时间复杂度降低到O(n)

## 原理

[LeetCode马拉车讲解（英文版）](https://articles.leetcode.com/longest-palindromic-substring-part-ii/)，这个推荐看一下，写的很棒。

### 改造字符串

```
source    =     a   b   c   a   c   b   a   d   f
strNew    = $ # a # b # c # a # c # b # a # d # f # &
prGroup[i]= 0 0 1 0 1 0 1 0 7 0 1 0 1 0 1 0 1 0 1 0 0
```

```
source    =     a   b   a   a   b   a
strNew    = $ # a # b # a # a # b # a # &
prGroup[i]= 0 0 1 0 3 0 1 6 1 0 3 0 1 0 0
```

每个字符中间都加上#，首尾也加上。为了防止字符串越界，首尾再增加一个不同的字符。
prGroup[i]是指以i为下标的字符，它的回文长度（去掉字符本身）palindrome-group

### 规律数组

![](http://static.nicesite.vip/2018-06-12-084839.jpg)

例如上图，我们已经知道了i=13以前所有的prprGroup，那么i=13时还需要再次一遍遍的往两侧循环检测吗？  
当然不需要，不然复杂度又成O(n^2)了

我们选取目前已知的，能够使右边界R的index最大的prGroup，此时R=20，L=2，C=11  
之后需要对不同的i进行判断
* 当 i <= R 时：
    - 若有 prGroup[i'] < R-i  我们已经知道了C=11处是对称点，如果C左侧的某个点的prGroup长度能够保证在[L,C]区间内，那么对称点必然相同。i=13时，有 prGroup[i']==prGroup[i]
    - prGroup[i'] >= R-i 那么此时的情况呢？当然不在保证 prGroup[i']==prGroup[i] 一定成立，因为从R之后的点都是未知的，但可以保证 prGroup[i] >= R-i。例如下图，i=15时，prGroup[i']已经超出[L,C]区间，有prGroup[i]>= R-i，R之后的字符是否符合需要一步一步往后验证，并更新R，L，C
    ![](http://static.nicesite.vip/2018-06-12-084925.jpg)
* 当 i > R 时，已经超过[L,R]区间，只能往后一个一个的验证，直至更新R，L，C

## 代码

```go
// 著名的马拉车算法，将复杂度回归线性O(n)
func manacher(source string) string {
	// 先对字符串增加#字符
	str := changeStr(source)
	lenStr := len(str)
	prGroup := make([]int, lenStr)
	tempCentre, tempRight, prMax, prMaxIndex := 0, 0, 0, 1
	for i := 1; i < len(str)-1; i++ {
		// golang没有三元运算符
		if i <= tempRight {
			prGroup[i] = minInt(tempRight-i, prGroup[2*tempCentre-i])
		} else {
			prGroup[i] = 0
		}
		for str[i-prGroup[i]-1] == str[i+prGroup[i]+1] {
			prGroup[i]++
		}
		if tempRight < prGroup[i]+i {
			tempCentre = i
			tempRight = i + prGroup[i]
		}
		if prMax < prGroup[i] {
			prMax = prGroup[i]
			prMaxIndex = i
		}
	}
	return source[(prMaxIndex-1-prMax)/2:(prMaxIndex-1+prMax)/2]
}

func minInt(a, b int) int {
	if a > b {
		return b
	} else {
		return a
	}
}

func changeStr(source string) string {
	b := bytes.Buffer{}
	b.WriteString("$#")
	for i := 0; i < len(source); i++ {
		b.WriteString(string(source[i]))
		b.WriteString("#")
	}
	b.WriteString("&")
	return b.String()
}
```

