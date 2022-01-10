---
title: Go学习笔记-for和range
date: 2021-12-03 11:02:38
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

  本章来介绍下golang中for和range的用法。
<!-- more -->

### for的基本用法

{% codeblock lang:go %}
func main() {
    sliceA := make([]int, 5)
    // 普通for循环
    for i := 0; i < 5; i++ {
        sliceA[i] = i
    }
}
{% endcodeblock %}

  golang中for的基本用法与Java中并无二致，这里不多赘述。

### for&range

  除了基本用法，golang中还提供了range关键字，进一步简化了遍历中取值的写法。

#### slice/数组的遍历
    
  这里看看range在对slice/数组遍历中的几种用法：

{% codeblock lang:go %}
func main() {
    sliceA := make([]int, 5)
    // for-range
    // i表示遍历的下标，v表示对应元素
    for i := range sliceA {
        fmt.Printf("num1:%v,v:%v\n", i, sliceA[i])
    }
    for i, v := range sliceA {
        fmt.Printf("num:%v,v:%v\n", i, v)
    }
}
{% endcodeblock %}

  golang中的普通for循环与Java没啥差别。
  不过golang提供了range关键字，能够更方便的取得被遍历集合的元素。

##### 永动机

  看到range的用法时，是否会想到，如果在遍历切片时不停的向其中塞数据，会变成一个永动机吗？

{% codeblock lang:go %}
for i, v := range sliceA {
    sliceA = append(sliceA, i)
    fmt.Printf("yy:%v,v:%v\n", i, v)
}
fmt.Printf("sliceA:%+v\n", sliceA)
{% endcodeblock %}

  运行后会发现，上述代码并不会一直执行下去，循环次数取决于sliceA的初始长度。

  这里可以看看源码range.go文件中的walkrange函数中的处理逻辑：

{% codeblock lang:go %}
// 这里是针对遍历切片的处理
case TARRAY, TSLICE:
    if arrayClear(n, v1, v2, a) {
        lineno = lno
        return n
    }
    // order.stmt arranged for a copy of the array/slice variable if needed.
    ha := a
    hv1 := temp(types.Types[TINT])
    // 这里hn表示的是切片长度，这里初始化了，所以不会形成永动机
    hn := temp(types.Types[TINT])
    init = append(init, nod(OAS, hv1, nil))
    init = append(init, nod(OAS, hn, nod(OLEN, ha, nil)))
    n.Left = nod(OLT, hv1, hn)
    n.Right = nod(OAS, hv1, nod(OADD, hv1, nodintconst(1)))
{% endcodeblock %}

##### 指针问题

{% codeblock lang:go %}
func main() {
    sliceA := make([]int, 0)
    for i := 0; i < 9; i++ {
        sliceA = append(sliceA, i)
    }
    for _, v := range sliceA {
        fmt.Printf("value:%v\n", &v)
    }
}
{% endcodeblock %}

  执行上述代码会发现，输出的都是同一个地址，这也是for range循环中的一个需要注意的地方。
  每次循环中，v都是使用的同一个地址，所以这里赋值时不要使用&v的方式，
  这里也可以看看源码中的处理，源码中是每次都会将遍历到的值赋值给v，v的内存地址其实是没有变化的。

#### hash遍历

  对于hash遍历需要注意的是他的不确定性：

{% codeblock lang:go %}
// 遍历的写法与slice遍历一样，不过这里的i表示的是map的key
func main() {
    mapA := make(map[int]string, 0)
    mapA[0] = "s"
    mapA[1] = "h"
    mapA[2] = "e"
    for i, v := range mapA {
        fmt.Printf("key:%v,value:%v\n", i, v)
    }
}
{% endcodeblock %}

  将上述代码多执行几次，你会发现输出的顺序并不一致。
  如果想知道为什么，可以在源码map.go文件的mapiterinit函数中找到答案。
  下面是截取的部分代码：
{% codeblock lang:go %}
    // fastrand()会返回一个随机值
    r := uintptr(fastrand())
    if h.B > 31-bucketCntBits {
        r += uintptr(fastrand()) << 31
    }
    // 通过随机值设置桶的起始遍历位置
    it.startBucket = r & bucketMask(h.B)
    it.offset = uint8(r >> h.B & (bucketCnt - 1))
{% endcodeblock %}

### 总结

    golang中保留了与Java中一样的for用法基础上，还提供了range关键字进一步简化了遍历中
    取值的写法，让遍历写起来更加高效简单。
    当然，因此也引出了类似指针地址不变的问题，需要我们在开发中多注意。

### 参考
> [Go语言for和range的实现](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/)