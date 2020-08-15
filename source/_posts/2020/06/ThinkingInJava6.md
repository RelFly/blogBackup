---
title: ThinkingInJava-6
date: 2020-06-14 11:20:50
tags:
- ThinkingInJava
categories:
- 读书笔记
- ThinkingInJava
---

### 前言

  集合是Java中较为重要的一个模块。也是我们日常使用较多的功能。容器的种类繁多，各有特点，所以只有了解
  掌握好各个容器的特点才能在适合的场景使用正确的容器。
  本章涉及的都是常用的同步容器，如ArrayList,LinkedList,HashMap等。
<!-- more -->

### 持有对象
  
  本书第十一章，从对象的保存引入集合概念。介绍了Java中常用的几大集合类型即迭代器等工具的使用。

#### Collection
  
    独立元素的序列，其下又分为List，Queue，Set三大类。

##### List
  
    List是一个按照插入顺序排序的集合，常用的有ArrayList和LinkedList。

1. ArrayList：基于数组结构的集合，优势在于随机访问，但是新增和删除操作较慢。可以参考[\<ArrayList(jdk1.8)>](https://rel-fly.com/2020/05/26/arrayList1/)

2. LinkedList：实现List接口的同时还实现了Deque(Queue的子接口)，所以可以用作栈，队列等结构。其基于双向循环链表，优势在于顺序访问及代价较低的新增和删除操作，但是随机访问较慢。

##### Queue
  
      Queue的大多实现遵循先进先出的规则，事物的放入顺序与取出顺序一样。因为其特点，在并发中
    起到了非常重要的作用。

1. PriorityQueue：优先队列，其队列规则是下一个弹出的元素是优先级最高的而不是等待时间最长(先进先出即弹出等待时间最长的)。

##### Set

    非重复元素集合，常用作元素在集合中存在性的判断。

1. HashSet：底层基于HashMap存储数据，实现Set接口，所以具有Set集合的特点，不接受重复元素。其查询效率较高。

2. TreeSet：与HashSet不同的是实现了NavigableSet(SortedSet的子接口)，元素处于排序状态。

3. LinkedHashSet：以插入顺序保存元素。

#### Map

    Map是一种对象与对象关联的设计，键值对类型能够让基于key的查询保持很高的效率。

1. HashMap：基于数组+链表/红黑树的结构。因为键值对的特点，根据key值的查询较快，新增删除如果涉及到扩容和链表/树结构的变化代价会较大。具体可参考[\<HashMap(jdk1.8)>](https://rel-fly.com/2020/05/30/hashMap1/)

2. TreeMap：基于红黑树的结构。保持key始终处于排序状态。

3. LinkedHashMap：HashMap的子类，在HashMap的结构基础上增加了一个双向链表记录元素插入顺序。所以他的key保持插入顺序排序，同时查询速度也很快。

### 小结

      哪怕只是同步集合，其实内容也比较多。相对来说，自己对集合的使用还是较为死板。像List可能就无
    脑用ArrayList了，这点需要在实际开发中注意。而且还有对并发集合的选择，正确的选择才能有效提高
    代码效率和安全性。
      书中还提到了用其他的Collection对象来完成初始化，Arrays.asList()的使用，以及集合与迭代器
    的使用等。这些留待之后单独的一一研究记录。本章只是单纯的对常用同步集合做了一个罗列，也准备之后
    挨个以源码解读的方式记录。

> 下一章 [\<ThinkingInJava-7>](https://rel-fly.com/2020/06/15/ThinkingInJava7/)