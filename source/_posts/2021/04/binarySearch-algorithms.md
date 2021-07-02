---
title: 算法笔记-二分查找
date: 2021-04-22 21:26:49
tags:
- 算法笔记
categories:
- 算法
- 算法笔记
- 二分查找
---

### 思路

     二分查找，也可以称作折半查找。
     即在一个有序序列中，通过每次对中间位置元素的判断来逐步缩小查找范围。
<!-- more -->

### Sqrt(x)

#### 问题描述

    给定一个非负整数，求它的开方，向下取整。

#### 解题思路

    这个问题比较简单，一个非负整数的开发必然是小于等于他本身的。
    所以我们可以将[0,x]看做一个有序序列，用二分查找的思想确定结果

{% codeblock lang:go %}
func mySqrt(x int) int {
	if x <= 1 {
		return x
	}
	start := 0
	end := x
	i := start + (end-start)/2
	for ; i != start; i = start + (end-start)/2 {
		temp := i * i
		if temp == x {
			return i
		}
		if temp < x {
			start = i
		}
		if temp > x {
			end = i
		}
	}
	return i
}
{% endcodeblock %}

### Find First and Last Position of Element in Sorted Array

#### 问题描述

	给定一个增序的整数数组和一个值，查找该值第一次和最后一次出现的位置。

#### 解题思路

    因为数组是增序的，那么目标值肯定是连续的，只需要先找到任意一个目标值的位置，
    然后向两边遍历就能找到其第一次和最后一次出现的位置。

{% codeblock lang:go %}
func searchRange(nums []int, target int) []int {
	result := []int{-1, -1}
	size := len(nums)
	if size == 0 {
		return result
	}
	start := 0
	end := size - 1
	mid := start + (end-start)/2
	// 使用二分查找查找目标值的任意一个位置
	for start < end && nums[mid] != target {
		if nums[mid] > target {
			end = mid - 1
		} else {
			start = mid + 1
		}
		mid = start + (end-start)/2
	}
	if nums[mid] != target {
		return result
	}
	// 向两侧遍历找出第一次和最后一次出现的位置
	min, max := mid-1, mid+1
	for min >= start && nums[min] == target {
		min--
	}
	for max <= end && nums[max] == target {
		max++
	}
	result = []int{min + 1, max - 1}
	return result
}

{% endcodeblock %}

### Search in Rotated Sorted Array II

#### 问题描述

      一个原本增序的数组被首尾相连后按某个位置断开（如 [1,2,2,3,4,5] → [2,3,4,5,1,2]，在第
    一位和第二位断开），我们称其为旋转数组。给定一个值，判断这个值是否存在于这个为旋转数组中。

#### 解题思路

    虽然数组不是整体有序，但是以断开点为准分为左右两个子集，都是各自增序的。只要在二分查找中判断
    当前所在是左子集还是右子集就行。

{% codeblock lang:go %}
func search(nums []int, target int) bool {
	start := 0
	end := len(nums) - 1
	mid := start + (end - start)/2
	for ;start < end; mid = start + (end - start)/2{
		if target == nums[mid]{
			return true
		}
		if nums[mid] == nums[start]{
			// 无法判断哪个子集，头指针向右移
			start++
		}else{
			if nums[mid] > nums[start]{
			    // 中间节点在左子集
				if target == nums[start]{
					return true
				}
				if target < nums[mid] && target > nums[start]{
				    // 判断目标在左子集范围
					end = mid-1
				}else {
				    // 无法判断 头指针右移
					start = mid+1
				}
			}else {
			    // 中间节点右子集
				if target == nums[end]{
					return true
				}
				if target > nums[mid] && target < nums[end]{
					// 判断目标在右子集范围
					start = mid+1
				}else {
				    // 无法判断 头指针右移
					end = mid-1
				}
			}
		}
	}
	if target == nums[mid]{
		return true
	}
	return false
}
{% endcodeblock %}


### 总结
   
    二分查找很好理解，十分依赖集合有序这个点。但并不一定是整体有序，就如旋转数组一样。
    所以在使用二分查找时，需要找出目标集合的有序性，依据其有序性做为二分查找的依据。