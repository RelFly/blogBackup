---
title: 算法笔记-双指针
date: 2021-04-18 20:00:25
tags:
- 算法笔记
categories:
- 算法
- 算法笔记
- 双指针
---

### 思路

双指针，顾名思义，是指在遍历某个数组时用两个指针指向不同元素，协同达成某些目的。
也可扩展为多数组多指针。

<!-- more -->

### Two Sum II

#### 问题描述

	在一个增序的整数数组里找到两个数，使它们的和为给定值。
	已知有且只有一对解

#### 解题思路

{% codeblock lang:go %}
func twoSum(numbers []int, target int) []int {
	left := 0
	right := len(numbers) - 1
	for left < right {
		leftNum := numbers[left]
		rightNum := numbers[right]
		total := leftNum + rightNum
		if total == target {
			break
		}
		if total > target {
			right--
			continue
		}
		if total < target {
			left++
			continue
		}
	}
	result := [2]int{left + 1, right + 1}
	return result[0:]
}
{% endcodeblock %}


### Merge Sorted Array

#### 问题描述

    给定两个排序整数数组nums1和nums2，合并nums2成nums1为一个排序后的数组。
    元件的数量初始化中nums1和nums2是m和n分别。您可以假定nums1大小等于m + n，使其具有足够的空间来容纳
    中的其他元素nums2。

#### 解题思路

##### 思路1

    因为nums1是有序的，
    所以可以使用插入排序将nums2中的数据依次插入到nums1中

{% codeblock lang:go %}
func merge(nums1 []int, m int, nums2 []int, n int) {
	if m == 0 {
		copy(nums1, nums2)
		return
	}
	pointer1 := m - 1
	pointer2 := 0
	for ; pointer1 < m+n && pointer2 < n; pointer1++ {
		if nums2[pointer2] >= nums1[pointer1] {
			nums1[pointer1+1] = nums2[pointer2]
		} else if nums2[pointer2] < nums1[pointer1] {
			count := pointer1
			for ; count >= 0 && nums2[pointer2] < nums1[count]; count-- {
				nums1[count+1] = nums1[count]
			}
			nums1[count+1] = nums2[pointer2]
		}
		pointer2++
	}
}
{% endcodeblock %}

##### 思路2

	但是使用插入排序就浪费了nums2有序的条件，所以这里更好的思路是从两个数组最大的数开始比较，将其中更
	大的插入nums1的尾部。

{% codeblock lang:go %}
func merge(nums1 []int, m int, nums2 []int, n int) {
	// 指向nums1尾部的指针
	last := m + n - 1
	// 遍历nums1的指针
	p1 := m - 1
	// 遍历nums2的指针
	p2 := n - 1
	for p1 >= 0 && p2 >= 0 {
	    // 将二者更大的一个放入nums1的尾部
		if nums1[p1] >= nums2[p2] {
			nums1[last] = nums1[p1]
			p1--
		} else {
			nums1[last] = nums2[p2]
			p2--
		}
		// 尾部指针前移
		last--
	}
	// 若此时nums2还有剩余则依次插入nums1即可
	for ; p2 >= 0; p2-- {
		nums1[last] = nums2[p2]
		last--
	}
}
{% endcodeblock %}

### Linked List Cycle II

#### 问题描述
  
    给定一个链表，返回循环开始的节点。如果没有循环，则返回null。
    如果列表中有某些节点可以通过连续跟随next 指针再次到达，则链接列表中会有一个循环 。
    在内部，pos 用于表示尾部next 指针连接到的节点的索引 。

#### 解题思路

    使用快慢指针(Floyd判圈法)就很容易的解决这个问题。
    首先定义fast,slow两个指针，一个每次遍历两个节点，一个每次遍历一个节点。
    如果，fast遍历到链表尽头，则表示没有循环；
    反之，当fast，slow节点第一次相遇时，将fast节点复位到头节点，然后调整
    fast节点每次也只遍历一个节点，这时，当fast与slow第二次相遇时的节点就是
    循环的起始节点。

{% codeblock lang:go %}
type ListNode struct {
	Val  int
	Next *ListNode
}

func detectCycle(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return nil
	}
	fast := head.Next.Next
	slow := head.Next
	for fast != slow {
		if fast == nil || fast.Next == nil {
			return nil
		}
		fast = fast.Next.Next
		slow = slow.Next
	}
	fast = head
	for fast != slow {
		fast = fast.Next
		slow = slow.Next
	}
	return fast
}
{% endcodeblock %}

### Minimum Window Substring

#### 问题描述

    给定两个字符串 S 和T，求 S 中包含T 所有字符的最短连续子字符串的长度，同时要求时间
    复杂度不得超过 O(n)。

#### 解题思路

    使用滑动窗口，定义两个指针start，end。先移动end找到包含目标字符串的子字符串，然后移动将start
    向end靠近，尝试找到最短的子字符串。

{% codeblock lang:go %}
func minWindow(s string, t string) string {
	tSize := len(t)
	sSize := len(s)
	// 用于判断字符是否存在的map集合
	// key为目标字符串的每个字符
	// value是字符存在的个数，可能会有重复字符
	checkArray := make(map[uint8]int, tSize)
	// 用于统计原字符串字符匹配的情况
	countArray := make(map[uint8]int, tSize)
	// start end 指针用于构造窗口
	// minS用于记录子字符串的头
	// totalCount 目标字符串完全匹配的标识
	// countNum 当前匹配的标识
	start, end, minS, totalCount, countNum := 0, 0, 0, 0, 0
	// 记录子字符串的最小长度
	minLen := sSize + 1
	// 初始化判断集合和匹配标识
	for e := range t {
		if checkArray[t[e]] == 0 {
			totalCount++
		}
		checkArray[t[e]]++
	}
	// 找出最短匹配子字符串
	for ; end < sSize; end++ {
	    // 找出匹配的子字符串
		if checkArray[s[end]] > 0 {
			countArray[s[end]]++
			if countArray[s[end]] == checkArray[s[end]] {
				countNum++
			}
		}
		// 滑动窗口，尝试找出更短的子字符串
		for ; totalCount == countNum; start++ {
			currentLen := end - start + 1
			if minLen > currentLen {
				minS = start
				minLen = currentLen
			}
			if checkArray[s[start]] > 0 {
				countArray[s[start]]--
				if countArray[s[start]] < checkArray[s[start]] {
					countNum--
				}
			}
		}
	}
	if minLen == sSize+1 {
		return ""
	}
	return s[minS : minS+minLen]
}
{% endcodeblock %}


### 总结

    双指针算法很显然，适用于对数组这类数据的操作问题。
    但他的变化比较多，首先会有多指针多数组的情况，不同的问题，解决思路也大同小异，
    例如上述判圈法和滑动窗口，虽然都是两个指针协作的思路，但具体实现却很不一样。
    一方面需要把握其通过多个指针协同解决的主题思路，另一方面还是需要多多练习不同的
    题目。