---
title: Go学习笔记-Hello Golang
date: 2021-08-01 20:01:36
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言
  终于要开始Go的学习记录，本章先简单了解下Go。
<!-- more -->

### Hello Golang
   按照惯例，学习一个语言要从Hello World开始：
{% codeblock lang:go %}
package main

import (
	"fmt"
)

func main(){
	var str = "Hello World，Hello Golang"
	fmt.Println(str)
}
{% endcodeblock %}
  
   接下来，以这个Hello World为例，来初步了解下Go。

#### package和import
   package用来设置包名，import用来导入依赖的其他包。这一块与Java还是比较像的。

#### func
   func是定义函数的关键字，可执行的Go程序都必须定义一个名称为main的函数，作为程序启动的入口。
   可以看到main的定义与Java就有些不同了，即没有定义返回类型，也没有入参等等。当然，只是简单
   的main函数看不出太多差异点。

#### var
   var是创建对象的关键字，这里的语法更偏向C语言。
   与java不同的是，并不需要指定类型。而且还能更简化的直接用*str := "hello world"* 替代。

#### fmt.Println()
   最后一行的*fmt.Println()* 是调用了导入的依赖包fmt的方法，来在控制台输出定义的字符串。

### Go的诞生

> Go 语言起源 2007 年，并于 2009 年正式对外发布。它从 2009 年 9 月 21 日开始作为 Google 公司 20% 兼职项目，即
> 相关员工利用 20% 的空余时间来参与 Go 语言的研发工作。
> 2009 年 11 月 10 日，开发团队将 Go 语言项目以 BSD-style 授权（完全开源）正式公布了 Linux 和 Mac OS X 平台上
> 的版本。同年 11 月 22 日公布了 Windows 版本。

### Go的特点   

> Go 语言是一种类型安全和内存安全的编程语言

> Go 语言的一个目标是对于网络通信、并发和并行编程的极佳支持，从而更好地利用大量的分布式和多核的计算机。
> 设计者通过 goroutine 这种轻量级线程的概念来实现这个目标，然后通过 channel 来实现各个 goroutine 之间的通信。他们实现
> 了分段栈增长和 goroutine 在线程基础上多路复用技术的自动化。
> 这个特性显然是 Go 语言最强有力的部分，不仅支持了日益重要的多核与多处理器计算机，也弥补了现存编程语言在这方面所存在的不足。

> Go 语言从本质上（程序和结构方面）来实现并发编程。

> 因为 Go 语言没有类和继承的概念，所以它和 Java 或 C++ 看起来并不相同。但是它通过接口（interface）的概念来实现多态性。
> Go 语言有一个清晰易懂的轻量级类型系统，在类型之间也没有层级之说。因此可以说这是一门混合型的语言。

### 总结
    初步了解了Go的诞生背景，特点后，免不了会与Java比较下。
    Java基于多年发展的成熟生态，已经形成了一整套层次分明，行之有效的技术方案。不过与此同时，
  Java的“沉重”也一直饱受诟病。
    作为新生的编程语言的Go，不仅语法上做了很多减法，使其更简洁明了，同时在设计上没有Java中
  类，继承，实现这些面向对象的概念。虽然没有了Java那种层次分明的感觉，但结构会变得更为简单。
    而且Go对并发，网络通信这些场景的原生支持，也是让Go现在如此火热的重要的原因之一。

### 参考
> [Go语言设计与实现](https://draveness.me/golang/)
> [Go简易教程](https://learnku.com/docs/the-little-go-book)
> [Go入门指南](https://learnku.com/docs/the-way-to-go)

### 下一章
> [Go学习笔记-数据类型](https://rel-fly.com/2021/08/01/golang-study-2/)