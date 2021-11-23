---
title: Go学习笔记-函数
date: 2021-11-09 10:11:36
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

  本章来了解下golang中函数的使用。

### 基本使用

  golang中定义函数使用func关键字，实例如下：

{% codeblock lang:go %}
func helloWorld(param int) (result,err){
  // TODO anything
  return nil
}
{% endcodeblock %}
  与Java中的方法的结构大同小异，都是由函数名，入参，返回类型，方法体构成。
  不过golang中支持多个不同类型的返回结果，这一点值得注意。

### 参数传递

  参数传递是函数不可跳过的一个话题，虽然参数传递分为值传递和引用传递，不过我们都知道引用传递的本质还是值传递，
  只不过因为指针的特殊性导致其表现效果与值传递不一样，也就是方法体内对参数的修改会影响原值，所以才单独归为一类。

  golang与Java一样，参数的传递也是值传递，下面通过实例来验证下。

#### 基本类型/string

{% codeblock lang:go %}
func BaseData(num1 int, num2 float32, str string) {
	num1 = num1 + 1
	num2 = 3.2
	str = "hello"
	fmt.Printf("after change---num1:%v,num2:%v,str:%v\n", num1, num2, str)
}
func main() {
	num1 := 1
	var num2 float32 = 2.3
	str := "kkll"
	functest.BaseData(num1, num2, str)
	fmt.Printf("result------num1:%v,num2:%v,str:%v\n", num1, num2, str)
}
// 结果输出：
// after change---num1:2,num2:3.2,str:hello
// result------num1:1,num2:2.3,str:kkll
{% endcodeblock %}
  
  基本类型和string类型不用多说，值传递无疑，方法体内的修改不会影响到原值。

#### 数组/slice

{% codeblock lang:go %}
func ArrayData(array [3]int, sliceParam []int, arrayPoint [3]*int) {
	array[1] = 1
	fmt.Printf("array change:%v\n", array)
	sliceParam[0] = 0
	sliceParam = append(sliceParam, 10)
	fmt.Printf("slice change:%v\n", sliceParam)
	num := 11
	arrayPoint[0] = &num
	fmt.Printf("arrayPoint change:%v\n", *arrayPoint[0])
}
func main() {
	array := [3]int{1, 2, 3}
	sliceParam := make([]int, 0)
	sliceParam = append(sliceParam, 1, 2, 3)
	num := 1
	arrayPoint := [3]*int{&num}
	functest.ArrayData(array, sliceParam, arrayPoint)
	fmt.Printf("array:%v,slice:%v,arrayPoint:%v\n", array, sliceParam, *arrayPoint[0])
}
// 结果输出
// array change:[1 1 3]
// slice change:[0 2 3 10]
// arrayPoint change:11
// array:[1 2 3],slice:[0 2 3],arrayPoint:1
{% endcodeblock %}

  数组这里也是值传递没啥问题，数组元素为指针类型也一样，传递的是整个数组的副本。
  切片就有点不同了，首先，切片是对某一段数组的引用，他的定义如下：

{% codeblock lang:go %}
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
{% endcodeblock %}

   slice本质上是一个结构体，按照值传递理解，这里会传一个这个结构体的副本，也就是这里的array指针，len，cap都会
   复制一份。
   那么对数组片段的引用是不变，所以这也解释了示例中，修改切片元素值会导致原切片对应元素的修改。
   但append方法不会，因为原切片引用的片段不变。

#### map

{% codeblock lang:go %}
func MapData(map1 map[int]string, map2 map[int]*string) {
	map1[0] = "first"
	map1[10] = "changed"
	str := "jjkjk"
	map2[0] = &str
	fmt.Printf("map1:%v,map2:%v\n", map1, map2)
}
func main() {
	map1 := make(map[int]string, 0)
	map2 := make(map[int]*string, 0)
	map1[10] = "change"
	functest.MapData(map1, map2)
	fmt.Printf("map1:%v,map2:%v\n", map1, map2)
}
// 结果输出：
// map1:map[0:first 10:changed],map2:map[0:0xc00016ed00]
// map1:map[0:first 10:changed],map2:map[0:0xc00016ed00]
{% endcodeblock %}
   
   可以看到，map与slice不同，不论是对原有元素的修改还是新增元素都会改变原map集合。
   这里先看看map的定义：

{% codeblock lang:go %}
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
{% endcodeblock %}

   先默念参数传递的本质是值传递这一点，再来看看map的结构。

{% codeblock lang:go %}
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	nextOverflow *bmap
}
// A bucket for a Go map.
type bmap struct {
	tophash [bucketCnt]uint8
}
{% endcodeblock %}

	可以看到，无论是代表桶的buckets还是存储键值对的mapextra结构，都是用的指针类型，所以传入map参数时，其副本保存的
	也是这些指针的值，自然对map元素的修改或新增元素多会影响到原map集合。
	
#### struct

    其实在看了slice和map的传参示例后，struct的传参表现已经很明了了，不过这里还是看一个示例，眼见为实：

{% codeblock lang:go %}
type Father struct {
	Name     string
	Age      int
	Wife     *string
	ChildOne Child
	ChildTwo *Child
}

type Child struct {
	Name string
}

func StructData(father Father, father2 Father) {
	father.Name = "qwq"
	father.Age = 11
	wife := "xvxvxv"
	father.Wife = &wife
	father.ChildOne = Child{
		Name: "newOne",
	}
	father.ChildTwo = &Child{
		Name: "newTwo",
	}
	father2 = Father{
		Name: "mmkmk",
		Age:  12,
	}
	fmt.Printf("father1:%+v,father2:%+v\n", father, father2)
}
func main() {
	wife := "jkjkll"
	father := functest.Father{
		Name: "jjjj",
		Wife: &wife,
		ChildOne: functest.Child{
			Name: "one",
		},
		ChildTwo: &functest.Child{
			Name: "two",
		},
	}
	father2 := functest.Father{
		Name: "eee",
		ChildOne: functest.Child{
			Name: "one",
		},
		ChildTwo: &functest.Child{
			Name: "two",
		},
	}
	fmt.Printf("father1:%+v,father2:%+v\n", father, father2)
	functest.StructData(father, father2)
	fmt.Printf("father1:%+v,father2:%+v\n", father, father2)
}
// 结果输出：
// father1:{Name:jjjj Age:0 Wife:0xc00016ed00 ChildOne:{Name:one} ChildTwo:0xc00016ed10},
// father2:{Name:eee Age:0 Wife:<nil> ChildOne:{Name:one} ChildTwo:0xc00016ed20}

// father1:{Name:qwq Age:11 Wife:0xc00016ed30 ChildOne:{Name:newOne} ChildTwo:0xc00016ed40},
// father2:{Name:mmkmk Age:12 Wife:<nil> ChildOne:{Name:} ChildTwo:<nil>}

// father1:{Name:jjjj Age:0 Wife:0xc00016ed00 ChildOne:{Name:one} ChildTwo:0xc00016ed10},
// father2:{Name:eee Age:0 Wife:<nil> ChildOne:{Name:one} ChildTwo:0xc00016ed20}
{% endcodeblock%}

	不出所料，struct类型的参数传递仍然是传递副本，所以方法体内的修改不影响原struct的值。

#### 小结
    
	这里就不再看指针类型的示例了，小结一下：golang中参数传递的本质仍然是值传递，只不过因为一些数据结构包含指针，导致这些
	数据类型的传参会是引用传递。所以这里只要把握住值传递并了解各个数据类型的结构，例如map的结构，就能准确判断每种数据类型
	在传参中的表现。

### 方法

    方法可以理解为有归属对象的函数，例如Java中的方法都属于某一个类对象。
	golang中也有方法的概念，不过定义上与Java大不相同，下面看一个例子：

{% codeblock lang:go %}
type Owner struct {
	Text string
}

func (o *Owner) Show(text string) {
	o.Text = text
	fmt.Printf("point-Owner:%v\n", *o)
}

func (o Owner) Show2(text string) {
	o.Text = text
	fmt.Printf("no-point-Owner:%v\n", o)
}
func main() {
	owner := functest.Owner{
		Text: "111",
	}
	owner.Show("jjjj")
	fmt.Printf("owner:%v\n", owner)
	owner.Show2("xxxx")
	fmt.Printf("owner2:%v\n", owner)
}
// 结果输出：
// point-Owner:{jjjj}
// owner:{jjjj}
// no-point-Owner:{xxxx}
// owner2:{jjjj}
{% endcodeblock %}

	通过将某个函数归属到某个结构体类型下，使之成为这个结构体的一个方法。

	golang的方法直接在定义时就声明了一个归属者的对象(如：func (o Owner) show() 中的o)，所以golang中没有
	this这种指针，而是直接使用定义好的对象实例。而根据实例定义时是否使用了指针类型，可以决定方法是否能影响到原
	实例的值。

	另外，方法的归属对象并不限于结构体类型：

{% codeblock lang:go %}
type NewInt int

func (i NewInt) Add(){
	i = i+1
}
{% endcodeblock %}

	使用type关键字定义的内置类型也可以作为方法的归属对象。

#### 组合与继承

	golang舍弃了Java中继承，实现等概念，不过可以通过组合的方式变相实现这种关系。

{% codeblock lang:go %}
type Owner struct {
	Text string
}

func (o *Owner) Show(text string) {
	o.Text = text
	fmt.Printf("point-Owner:%v\n", *o)
}

func (o Owner) Show2(text string) {
	o.Text = text
	fmt.Printf("no-point-Owner:%v\n", o)
}

type Boss struct {
	Owner
	Text string
}

func (b Boss) Show(text string) {
	b.Text = text
	fmt.Printf("boss:%v\n", b)
}
func main() {
	owner := functest.Owner{
		Text: "1212",
	}
	boss := functest.Boss{
		Owner: owner,
		Text:  "llkk",
	}
	boss.Show("xixix")
	fmt.Printf("boss.owner.text:%v\n", boss.Owner.Text)
	boss.Owner.Show("jljl")
	boss.Show2("jjj")
}
// 结果输出：
// boss:{{1212} xixix}
// boss.owner.text:1212
// point-Owner:{jljl}
// no-point-Owner:{jjj}
{% endcodeblock %}

	当两个结构体形成组合的关系时，外部的结构体，就可以访问内部结构体的属性与方法。
	特别是使用匿名属性时，可以省略内部结构体名称直接调用，达到类似继承中子类调用父类方法的效果。
	而如果有同名的方法，属性时，就会访问到外部结构体自己的属性与方法。

	看起来像是继承中子类同名方法覆盖父类方法，但本质是不同的，外部结构仍然可以通过他的结构体属性访问到该属性拥有的方法。
	所以需要注意这里仍然是两个结构体的组合关系，不能与继承混为一谈。

### 接口

	golang中的接口是一种内置类型，代表一组抽象方法。而且与Java不一样的是，不需要显示声明实现，只要有
	结构体拥有同名的方法就算是实现了该接口的抽象方法。
	下面来看一下具体例子：

{% codeblock lang:go %}
type Method interface {
	PpMm(s string)
}

type Pm struct {
	Text string
}

func (p Pm) PpMm(tt string) {
	p.Text = tt
	fmt.Printf("p的tt:%v\n", p)
}

type NoPm struct {
	Str string
}

func (n NoPm) NoPpMm(ss string) {
	n.Str = ss
	fmt.Printf("noPm:%v\n", n)
}
func main() {
	pm := functest.Pm{
		Text: "sss",
	}
	noPm := functest.NoPm{
		Str: "ttt",
	}
	noPm.NoPpMm("jl")
	var infe functest.Method
	infe = pm
	infe.PpMm("kk")
	fmt.Printf("interface:%v\n", infe)
	//infe = noPm 无法通过编译
}
// 结果输出：
// noPm:{jl}
// p的tt:{kk}
// interface:{sss}
{% endcodeblock %}

	这里可以看到，拥有与接口同名方法的结构体Pm自动与接口Method形成了实现的关系，Pm可以自动转成interface类型。
	但没有同名方法的NoPm则不行。

### 总结

	golang的函数与Java的方法在声明定义，参数传递上都比较相似，很容易理解。
	不过因为没有Java中class，继承，实现这些概念，所以golang中的方法和接口就与Java的大不一样。
	不过从使用上来来看，这种设计确实更加简单灵活，没有Java那么紧密耦合的关系。