---
title: 算法笔记-贪心算法
date: 2021-04-12 21:48:57
tags:
- 算法笔记
categories:
- 算法
- 算法笔记
- 贪心算法
---

### 思路

贪心算法的思路即每一次操作都是最优的，那么这些最优操作的最终结果也就是最优解。

<!-- more -->

### 分配问题
   
#### 1.问题描述

	有一群孩子和一堆饼干，每个孩子有一个饥饿度，每个饼干都有一个大小。每个孩子只能吃最多
	一个饼干，且只有饼干的大小大于孩子的饥饿度时，这个孩子才能吃饱。求解最多有多少孩子可
	以吃饱。
  
#### 解题思路
    
     根据贪心算法的思路，每一次分配饼干都选择最容易满足的，即饥饿度最小的孩子。选择恰好
     能满足他的饼干，即饼干大小大于等于其饥饿度中最小的饼干。这样能保证每一次分配都不浪
     费，能够尽可能多的满足孩子。

{% codeblock lang:go %}
func assignCookies(childs, cookies MyInt) int {
	sort.Sort(childs)
	sort.Sort(cookies)
	i := 0
	j := 0
	for ; i < len(childs) && j < len(cookies); j++ {
		if childs[i] <= cookies[j] {
			// 喂饼干 并切换到下一个小孩
			i++
		}
	}
	return i
}
{% endcodeblock %}

#### 2.问题描述

	一群孩子站成一排，每一个孩子有自己的评分。现在需要给这些孩子发糖果，规则是如果一
	个孩子的评分比自己身旁的一个孩子要高，那么这个孩子就必须得到比身旁孩子更多的糖果。
	所有孩子至少要有一个糖果。求解最少需要多少个糖果。

#### 解题思路

	1. 因为要求每个孩子至少有一个糖果，所以最理想情况：所有孩子评分一样，那么结果是
	   每个孩子都有且只有一个糖果
	2. 以理想情况的结果为基础，先从左至右比较，因为要求最少糖果，所以只比评分低的孩
	   子多加一个糖果。这一轮比较完以后，可以确定最后一个孩子的糖果数就是最终结果
	3. 从最后一个孩子开始，从右至左开始比较，这样就能得到最后的结果

{% codeblock lang:go %}
func candy(childs []int) int {
	size := len(childs)
	if size < 2 {
		return size
	}
	result := make([]int, size, size)
	for i, element := range childs {
		if result[i] == 0 {
			result[i] = 1
		}
		if i == 0 {
			continue
		}
		if element > childs[i-1] && result[i] <= result[i-1] {
			result[i] = result[i-1] + 1
		}
	}
	count := 0
	for i := size - 1; i > 0; i-- {
		if childs[i] < childs[i-1] && result[i] >= result[i-1] {
			result[i-1] = result[i] + 1
		}
		count = count + result[i]
	}
	count = count + result[0]
	return count
}
{% endcodeblock %}

### 区间问题

#### 问题描述

    给定多个区间，计算让这些区间互不重叠所需要移除区间的最少个数。起止相连不算重叠

#### 解题思路

    需要统计重叠的最少区间，就需要按照一定顺序依次判断确定。
    1. 将区间集合按照尾端顺序排序
    2. 设置一个当前区间，初始化为第一个区间
    3. 从第二个区间开始遍历，与当前区间比较，重叠则统计为移除区间，反之取代当前区间
    4. 重复3，直至结束

{% codeblock lang:go %}
func eraseOverlapIntervals(param [][]int) int {
	size := len(param)
	if size < 2 {
		return 0
	}
	// 按照尾端升序排序
	for block := size / 2; block >= 1; block = block / 2 {
		for i := block; i < size; i = i + block {
			current := param[i]
			j := i - block
			for ; j >= 0; {
				loop := param[j]
				if loop[1] > current[1] {
					param[j+block] = loop
					j = j - block
				} else {
					break
				}
			}
			param[j+block] = current
		}
	}	// 筛选
	count := 0
	currentNum := param[0][1]
	for i := 1; i < size; i++ {
		if param[i][0] < currentNum {
			count++
			continue
		}
		currentNum = param[i][1]
	}
	return count
}
{% endcodeblock %}

### 总结
    
    贪心算法的策略是找到一个问题的每次操作的最优解，但对于每个问题的操作最优解的体现是不一样的。
    饼干分配的最优解是先满足饥饿度最小的孩子，这样能够余留足够多的饼干尽可能多的满足剩下的孩子。
    糖果分配的问题最优解是先确定一边的孩子糖果满足条件，再确定另一边的。
    区间问题则只要按照x轴从左到右确定区间，这样就能保证留下的区间是最多的，那么
 去除的区间自然就是最少的。
    总体而言，使用贪心算法的前提是能保证每次操作的最优解必然能得出最优的结果，反之则无法使用贪心算
    法解决问题。通常来看，其比较适合解决一些最大最小值的问题。