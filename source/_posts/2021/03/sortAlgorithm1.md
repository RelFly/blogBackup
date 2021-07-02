---
title: 排序算法总结
date: 2021-03-07 12:13:24
tags:
- 排序算法
categories:
- 算法
- 排序算法
---

### 前言
  本章对常用的排序算法做个系统的回顾与总结。
<!-- more -->

### 冒泡排序

  基本思路：每次遍历，两两比较，确定一个位置的值
  平均时间复杂度：O(n^2)
  是否稳定：是

{% codeblock lang:java %}
public void algorithmTest() {
    List<Integer> list = 
    	    Lists.newArrayList(61, 2, 13, 5, 11, 4, 55, 12, 45, 21, 56, 76, 1, 22);
    // 两两比较，外层循环表示循环次数
    for (int i = 0; i < list.size() - 1; i++) {
        for (int j = list.size() - 1; j > i; j--) {
            // 把小的交换到前面
            if (list.get(j) < list.get(j - 1)) {
                swap(list, j, j - 1);
            }
        }
    }
}
{% endcodeblock %}

### 选择排序

  基本思路：每一次遍历，确定一个位置的数据
  平均时间复杂度：O(n^2)
  是否稳定：否

{% codeblock lang:java %}
private void chooseSort(List<Integer> chooseList) {
    for (int i = 0; i < chooseList.size(); i++) {
        // 每次循环最小值的下标
        int choose = i;
        // 向后遍历，找出最小值元素所在下标
        for (int j = i + 1; j < chooseList.size(); j++) {
            if (chooseList.get(j) < chooseList.get(i)) {
                choose = j;
            }
        }
        // 将最小值与当前首位交换
        swap(chooseList, i, choose);
    }
}
{% endcodeblock%}

### 插入排序
  
  基本思路：将第一个元素看作一个已经排好序的序列，从第二个元素开始，依次按顺序插入序列中。
  平均时间复杂度：O(n^2)\
  是否稳定：是

{% codeblock lang:java %}
private void insertSort(List<Integer> insertList) {
    int temp;
    int j;
    // 因为第一个元素可以看作有序的，所以从第二个元素开始排序
    for (int i = 1; i < insertList.size(); i++) {
        // 待插入元素
        temp = insertList.get(i);
        j = i - 1;
        // 依次向前比较，当前元素大于temp时，将该元素向后移动，然后继续向前遍历
        // 因为是有序序列，如果当前元素小于等于temp，那么更前面的元素一定小于temp，不用继续遍历
        for (; j >= 0 && insertList.get(j) > temp; j--) {
            insertList.set(j + 1, insertList.get(j));
        }
        // 最后j的前一位便是应当插入的位置
        insertList.set(j + 1, temp);
    }
}
{% endcodeblock %}

### 快速排序

  基本思路：
  1. 先设定一个基准值(可以是首位，也可以是随机一个)，然后从末尾向首位遍历，遇到比基准小的
     便移往左边，再然后从首位向末尾遍历，遇到比基准大的便移往右边。最后得到基准值的下标
  2. 以基准值下标为界，分为左右两部分，继续按照1中的逻辑分别继续递归处理


  平均时间复杂度：O(NlogN)
  是否稳定：否

{% codeblock lang:java %}
public void algorithmTest() {
    // 快速排序
    List<Integer> quickList =
            Lists.newArrayList(61, 2, 13, 5, 11, 4, 55, 12, 45, 21, 56, 76, 1, 22);
    quickSort(quickList, 0, quickList.size() - 1);
}
// 递归处理
private void quickSort(List<Integer> list, int x, int y) {
    if (x < y) {
        int middle = getMiddle(list, x, y);
        quickSort(list, x, middle - 1);
        quickSort(list, middle + 1, y);
    }
}
// 找出基准值的下标
private int getMiddle(List<Integer> sortList, int start, int end) {
    int compareInt = sortList.get(start);
    while (start < end) {
        while (start < end && sortList.get(end) >= compareInt) {
            end--;
        }
        swap(sortList, start, end);
        while (start < end && sortList.get(start) <= compareInt) {
            start++;
        }
        swap(sortList, start, end);
    }
    sortList.set(start, compareInt);
    return start;
}
{% endcodeblock%}

### 归并排序

  基本思路：
  1. 将每一个元素看做一个有序序列
  2. 将这些有序序列两两合并形成新的有序序列
  3. 依次执行直到最终合并成一个有序序列


  平均时间复杂度：O(NlogN)
  是否稳定：是
{% codeblock lang:java %}
private void mergeSort(List<Integer> mergeList) {
    List<Integer> tempList;
    int size = mergeList.size();
    for (int space = 1; space < size; space * = 2) {
        tempList = Lists.newArrayListWithCapacity(mergeList.size());
        for (int start = 0; start < size; start += space * 2) {
            // 分为两个有序序列 start-mid   mid-high
            int mid = start + space < size ? start + space : size;
            int high = mid + space < size ? mid + space : size;
            // 定义两个序列起始下标
            int start1 = start;
            int start2 = mid;
            while (start1 < mid && start2 < high) {
                // 从两个序列队首开始取其中小的塞入临时队列
                int min = mergeList.get(start1) < mergeList.get(start2) ? 
                          mergeList.get(start1++) : mergeList.get(start2++);
                tempList.add(min);
            }
            // 处理可能剩下的数据
            while (start1 < mid) {
                tempList.add(mergeList.get(start1++));
            }
            while (start2 < high) {
                tempList.add(mergeList.get(start2++));
            }
        }
        // 结束一次归并 将临时队列复制到原队列中 继续排序
        mergeList.clear();
        mergeList.addAll(tempList);
        tempList.clear();
    }
}
{% endcodeblock %}


### 堆排序

    堆的定义：用数组实现的二叉树，分为最大堆，最小堆两类。
             在最大堆中，父节点的值比每一个子节点的值都要大；而在最小堆中，父节点的值比每一个
             子节点的值都要小。
             因此，最大堆的根节点既是数组中的最大值，同样，最小堆的根节点就是其最小值。

  基本思路：
  1. 利用堆的特性，首先将无序序列构建成一个堆，得到其最大|最小值
  2. 将根节点与序列队尾置换
  3. 抛开队尾元素，调整序列使其重新成为一个堆，然后重复2，3步骤


  平均时间复杂度：O(NlogN)
  是否稳定：否


{% codeblock lang:java %}
private void heapSort(List<Integer> heapList) {
    int size = heapList.size();
    maxHeap(heapList, size);
    // 开始排序
    for (int j = size - 1; j > 0; j--) {
        // 首尾互换
        swap(heapList, 0, j);
        // 构造最大堆
        maxHeap(heapList, j);
    }
}

// 构造最大堆
private void maxHeap(List<Integer> heapList, int size) {
    // 第一个非叶子节点 2n+1 = size-1
    int start = size / 2 - 1;
    // 构造最大堆
    for (int root = start; root > -1; root--) {
        // 左右子节点
        int left = 2 * root + 1;
        int right = left + 1;
        // 取子节点最大值
        int swap = left;
        if (right < size && heapList.get(left) < heapList.get(right)) {
            swap = right;
        }
        // 取最大值
        if (heapList.get(swap) > heapList.get(root)) {
            swap(heapList, root, swap);
        }
    }
}
{% endcodeblock%}

### 希尔排序
  
  基本思路：希尔排序是基于插入排序做的优化，插入排序在序列基本有序的前提下性能会大大提升。
  所以希尔排序的思路就是一步步使得序列基本有序。
  1. 定义一个增量，如size/2，根据增量将序列分为n个序列
  2. 对每个序列做插入排序
  3. 按照计算方式，缩小增量，然后重复步骤2，直到增量为1，形成一个序列
  4 .对最后一个序列做插入排序，因其基本有序，性能会大大提高


  平均时间复杂度：受增量影响
  是否稳定：否

{% codeblock lang:java %}
private void shellSort(List<Integer> shellList) {
    int size = shellList.size();
    // 步长
    for (int stepSize = size / 2; stepSize > 0; stepSize = stepSize / 2) {
        // 遍历序列 根据步长插入排序
        for (int i = stepSize; i < size; i++) {
            // 待插入元素
            int temp = shellList.get(i);
            int j = i - stepSize;
            // 依次向前比较
            for (; j >= 0 && shellList.get(j) > temp; j = j - stepSize) {
                shellList.set(j + stepSize, shellList.get(j));
            }
            shellList.set(j + stepSize, temp);
        }
    }
}
{% endcodeblock %}

### 总结
   
    排序算法最关注的应该是时间复杂度了，在对排序算法的选择上，当然是优先选择更快的算法。
    在没有要求稳定的情况下，快排是第一选择。
    冒泡，选择，插入三种排序应该避免使用，因为有更好选择。
    归并排序由于其空间要求，所以需要考虑实际应用中内存空间的占用。可以作为在要求稳定情况
    下的快排的替代。
    希尔排序基于插入排序的优化，可以满足一般情况下的需求，而且相对快排实现更简单。
