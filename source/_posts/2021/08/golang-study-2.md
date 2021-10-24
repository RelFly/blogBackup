---
title: Go学习笔记-数据类型
date: 2021-08-13 10:11:36
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言
  在简单了解了下Go语言后，开始进入正式的学习。
  本章就先来学习Go的数据类型及使用方法。
<!-- more -->

### 基本类型

#### 整数类型

  Go的整数类型除了有符号的int类型外，还提供了无符号的uint类型。
  同样类似Java的byte，short，int，long按照最大位数不同提供了int8，int16，int32，int64四种
  类型（uint也是一样分为uint8，uint16，uint32，uint64）
  若使用int，uint定义整数，则在32位系统下表示32位，64位系统下表示64位大小
  所以建议使用int8，int16等进行精确的定义。

#### 浮点数类型

  浮点整数有两种：float32，float64，分别代表32位和64位，也就是单精度和双精度浮点类型
  float32能精确到小数点后7位，而float64则能精确到小数点后15位

#### 复数类型

  复数类型同样按位数大小分为complex64和complex128两种类型

> 我们把形如a+bi（a,b均为实数）的数称为复数，其中a称为实部，b称为虚部，i称为虚数单位。
> 当虚部等于零时，这个复数可以视为实数；当z的虚部不等于零时，实部等于零时，常称z为纯虚数
  
  定义方式如下：
{% codeblock lang:go %}
var complexNum complex64 = complex(1, 2)
// complex的两个参数分别表示复数的实部与虚部，其类型都是用的float32类型
{% endcodeblock %}

#### 布尔类型

  使用关键字bool定义，与Java基本一样，只有true和false两个值

#### 字符类型

  首先看看builtin.go文件中定义string类型的注释：

{% codeblock lang:go %}
package builtin
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
{% endcodeblock %}
  可以看到，string是一个8字节byte类型的集合，且与Java一样，string的值也是不可变的

  然后看看string.go文件中的定义：
{% codeblock lang:go %}
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte
{% endcodeblock %}

  佐证了builtin.go文件中的注释，其是一个byte类型的数组。

  下面看看string的用法示例：
{% codeblock lang:go %}
  // 定义方式
  str1 := "123"
  var str2 = "one two three"
  // 以数组的形式遍历
  for _ , e := range str1 {
    fmt.Printf("%d,%q\n", e, e)
  }
  for i , _ := range str2 {
    fmt.Printf("%d,%q\n", str2[i], str2[i])
  }
  // 输出结果：49,'1'
  //          50,'2'
  //          51,'3'
  //          111,'o'
  //          110,'n'
  //          101,'e'
  //          32,' '
  //          116,'t'
  //          119,'w'
  //          111,'o'
  //          32,' '
  //          116,'t'
  //          104,'h'
  //          114,'r'
  //          101,'e'
  //          101,'e'
  // 不可变 所以下述代码会报错
  //str1[0] = "1"
{% endcodeblock %}

##### strings包
   
   在刚开始用string会疑惑，类似Java中split()，subString()这些方法到哪去了呢？
   go其实也提供了类似的方法，不过并不能像Java中一样直接用String类型的对象调用对应方法，而是定义了一个strings包
{% codeblock lang:go %}
  str := "hello world"
  // 分割
  str1 := strings.Split(str, " ")
  for i, e := range str1 {
    fmt.Printf("第%v个是:%v\n", i, e)
  }
  // 包含判断
  fmt.Printf("是否包含：%v\n", strings.Contains(str, "h"))
  // 在被统计字符串中出现的次数
  fmt.Printf("出现次数：%v\n", strings.Count("aabbdddccc", "a"))
  fmt.Printf("出现次数：%v\n", strings.Count("aabbdddccc", "c"))
  // 比较字符串
  fmt.Printf("比较结果:%v\n", strings.Compare("aaa", "aaa"))
  fmt.Printf("比较结果:%v\n", strings.Compare("aaa", "Aaa"))
{% endcodeblock %}

### 指针类型

   golang不同于Java，有指针类型，但又不同于C语言，其指针是不参与运算的，下面看看基本用法的示例
{% codeblock lang:go %}
  str := "aaa"
  point1 := &str
  fmt.Printf("输出地址值：%v\n", point1)
  fmt.Printf("输出值：%v\n", *point1)
  var pointArray []*int
  for i := 0; i < 5; i++ {
    // 此时i的地址不变
    fmt.Printf("i的地址：%v\n", &i)
    num := i
    pointArray = append(pointArray, &num)
  }
  fmt.Printf("指针数组：%v\n", pointArray)
  for i, e := range pointArray {
    fmt.Printf("i的地址：%v，e的地址：%v;", &i, &e)
    fmt.Printf("第%v个指针元素：%v;", i, e)
    fmt.Printf("第%v个元素：%v\n", i, *e)
  }
// 输出结果：
// 输出地址值：0xc000042240
// 输出值：aaa
// i的地址：0xc00000a0a8
// i的地址：0xc00000a0a8
// i的地址：0xc00000a0a8
// i的地址：0xc00000a0a8
// i的地址：0xc00000a0a8
// 指针数组：[0xc00000a0d0 0xc00000a0d8 0xc00000a0e0 0xc00000a0e8 0xc00000a0f0]
// i的地址：0xc00000a0f8，e的地址：0xc000006038;第0个指针元素：0xc00000a0d0;第0个元素：0
// i的地址：0xc00000a0f8，e的地址：0xc000006038;第1个指针元素：0xc00000a0d8;第1个元素：1
// i的地址：0xc00000a0f8，e的地址：0xc000006038;第2个指针元素：0xc00000a0e0;第2个元素：2
// i的地址：0xc00000a0f8，e的地址：0xc000006038;第3个指针元素：0xc00000a0e8;第3个元素：3
// i的地址：0xc00000a0f8，e的地址：0xc000006038;第4个指针元素：0xc00000a0f0;第4个元素：4
{% endcodeblock%}
  基本用法还是和C语言类似，通过&取地址，通过*取值。
  这里需要注意的是在for循环中定义的i或者使用range返回的i，e的地址都是不变的，如果有取地址赋值的操作需要注意。
  就像Java中for循环插入对象类型数据，因为没有重新new，所以集合里都是同一个实例的情况。

### 数组与切片

### 结构体