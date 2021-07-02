---
title: 算法笔记-深度优先搜索
date: 2021-04-27 20:54:21
tags:
- 算法笔记
categories:
- 算法
- 算法笔记
- 深度优先搜索
---

### 思路
  
    深度优先搜索是一种遍历思路，因为总是优先遍历新节点，所以其遍历路径看起来是向着深处进行。
    例如：在遍历一个树的所有节点时，先遍历完根节点某个子节点下所有节点，以这种方式依次遍历各个
    子节点。
<!-- more -->

### Max Area of Island

#### 问题描述

	  给定一个二维的 0-1 矩阵，其中 0 表示海洋，1 表示陆地。单独的或相邻的陆地可以形成岛
	屿，每个格子只与其上下左右四个格子相邻。求最大的岛屿面积。

#### 解题思路

	1.从任意节点开始，使用深度优先搜索进行遍历，直到没有陆地节点
	2.因为是个矩阵结构，要判断是否相邻，则需要遍历该节点上下左右四个节点
	3.会存在节点被重复遍历到的情况，只需将值置为0，表示该节点被统计过

{% codeblock lang:go %}
import "container/list"

func maxAreaOfIsland(grid [][]int) int {
	size := len(grid)
	if size == 0 {
		return 0
	}
	eSize := len(grid[0])
	// 定义栈
	stack := list.New()
	result := 0
	change := []int{-1, 0, 1, 0, -1}
	// 遍历矩阵的每个节点
	for x := 0; x < size; x++ {
		for y := 0; y < eSize; y++ {
			// 若节点为陆地，则从该节点开始使用深度优先搜索
			if grid[x][y] == 1 {
				// 入栈
				stack.PushBack([2]int{x, y})
				grid[x][y] = 0
				currentCount := 1
				// 遍历栈
				for stack.Len() > 0 {
					// 出栈
					current := stack.Front()
					stack.Remove(current)
					// 需要使用断言转换类型
					value, _ := current.Value.([2]int)
					// 弹出元素的数组下标
					v1 := value[0]
					v2 := value[1]
					// 遍历上下左右四个节点，并将最新节点压入栈中
					for e := 0; e < 4; e++ {
						ev1 := v1 + change[e]
						ev2 := v2 + change[e+1]
						if ev1 >= 0 && ev1 < size && ev2 >= 0 && ev2 < eSize {
							if grid[ev1][ev2] == 1 {
								stack.PushBack([2]int{ev1, ev2})
								grid[ev1][ev2] = 0
								currentCount++
							}
						}
					}
				}
				if currentCount > result {
					result = currentCount
				}
			}
		}
	}
	return result
}
{% endcodeblock %}

###  Friend Circles

#### 问题描述

      给定一个二维的 0-1 矩阵，如果第 (i, j) 位置是 1，则表示第 i 个人和第 j 个人是朋友。已知
    朋友关系是可以传递的，即如果 a 是 b 的朋友，b 是 c 的朋友，那么 a 和 c 也是朋友，换言之这
    三个人处于同一个朋友圈之内。求一共有多少个朋友圈。

#### 解题思路

    1.每个节点代表两个人，所以不用遍历到每个节点，而是遍历每个人。
    2.为了防止重复计算，当[a,b]==1，即a，b为朋友时，需要将[a,b]，[b,a]，[b,b]都置为0

{% codeblock lang:go %}
func findCircleNum(isConnected [][]int) int {
	size := len(isConnected)
	if size == 0 {
		return 0
	}
	// 定义栈
	stack := list.New()
	result := 0
	for x := 0; x < size; x++ {
		if isConnected[x][x] == 1 {
			result++
			isConnected[x][x] = 0
			stack.PushBack(x)
			for stack.Len() > 0 {
				element := stack.Front()
				stack.Remove(element)
				value, _ := element.Value.(int)
				for i := 0; i < size; i++ {
					if isConnected[value][i] == 1 {
						stack.PushBack(i)
						isConnected[value][i] = 0
						isConnected[i][value] = 0
						isConnected[i][i] = 0
					}
				}
			}
		}
	}
	return result
}
{% endcodeblock %}

### 总结

    深度优先遍历适用于树，图这种数据结构。他可以满足特定条件的目标搜索。
    例如在最大陆地问题中，以某个节点开始用深度优先搜索与其相连的陆地，统计个数。
    通常，深度优先搜索都是结合递归或者栈配合进行，为了防止重复遍历，会需要
    记录或者修改节点的值。