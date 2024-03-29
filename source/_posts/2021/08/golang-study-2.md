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
  同样类似Java的byte，short，int，long按照最大位数不同提供了int8，int16，int32，int64四种。
  类型（uint也是一样分为uint8，uint16，uint32，uint64）。
  若使用int，uint定义整数，则在32位系统下表示32位，64位系统下表示64位大小。
  所以建议使用int8，int16等进行精确的定义。

#### 浮点数类型

  浮点整数有两种：float32，float64，分别代表32位和64位，也就是单精度和双精度浮点类型。
  float32能精确到小数点后7位，而float64则能精确到小数点后15位。

#### 复数类型

  复数类型同样按位数大小分为complex64和complex128两种类型

> 我们把形如a+bi（a,b均为实数）的数称为复数，其中a称为实部，b称为虚部，i称为虚数单位。
> 当虚部等于零时，这个复数可以视为实数；当z的虚部不等于零时，实部等于零时，常称z为纯虚数。
  
  定义方式如下：

{% codeblock lang:go %}
var complexNum complex64 = complex(1, 2)
// complex的两个参数分别表示复数的实部与虚部，其类型都是用的float32类型
{% endcodeblock %}

#### 布尔类型

  使用关键字bool定义，与Java基本一样，只有true和false两个值。

#### 字符类型

  首先看看builtin.go文件中定义string类型的注释：

{% codeblock lang:go %}
package builtin
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
{% endcodeblock %}

  可以看到，string是一个8字节byte类型的集合，且与Java一样，string的值也是不可变的。

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
  go其实也提供了类似的方法，不过并不能像Java中一样直接用String类型的对象调用对应方法，而是定义了一个strings包。

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

  golang不同于Java，有指针类型，但又不同于C语言，其指针是不参与运算的，下面看看基本用法的示例：

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

#### 数组

  go中的数组与Java差不多，都是长度固定的集合，使用示例如下：

{% codeblock lang:go %}
// 数组声明方式
array := [5]int{1, 2, 3, 4, 5}
// array1 := [...]int{1,2,3} 以元素数量为数组长度，不用指定
// 声明空数组
arrayEmpty := [5]int{}

fmt.Println(array)
for i := 0; i < 3; i++ {
	arrayEmpty[i] = array[i]
}
// 未设值的元素用默认值填充
fmt.Println(arrayEmpty)

// 结果输出：
// [1 2 3 4 5]
// [1 2 3 0 0]
{% endcodeblock %}

#### 切片

  go中的切片是个可以动态扩容的集合结构，其本质可以看作一个指向某段数组的指针。
  有点像是Java中ArrayList的定位。
  首先看看切片的定义：

{% codeblock lang:go %}
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
{% endcodeblock %}

  从切片的定义上看，他是一个包含指针和当前长度和最大容量信息的结构体。
  其中指针指向某个数组，len和cap分别维护切片当前长度和最大容量。
  切片的声明有两种，一种是指向已有的数组，一种是使用make方法声明。

{% codeblock lang:go %}
array := [5]int{1, 2, 3, 4, 5}
// 指向某个数组的切片
sliceFromArray := array[2:4]
fmt.Printf("修改前:%v\n", sliceFromArray)
// 对数组的修改会影响切片的值
array[3] = 99
fmt.Printf("修改后：%v\n", sliceFromArray)
fmt.Printf("切片的len：%v，cap：%v\n", len(sliceFromArray), cap(sliceFromArray))
// 对切片的操作，如果是指向数组的部分则会修改愿数组的值
sliceFromArray = append(sliceFromArray, 98, 97, 96)
fmt.Printf("append后的切片：%v\n", sliceFromArray)
fmt.Printf("append后的数组:%v\n", array)
fmt.Printf("切片的len：%v，cap：%v\n", len(sliceFromArray), cap(sliceFromArray))
// 结果输出：
// 修改前:[3 4]
// 修改后：[3 99]
// 切片的len：2，cap：3
// append后的切片：[3 99 98 97 96]
// append后的数组:[1 2 3 99 98]
// 切片的len：5，cap：6
{% endcodeblock %}
  可以看到，如果是基于数组构造的切片，因为切片是指向数组某段的指针，所以无论是修改数组还是切片都会影响到另一方。
  特别需要注意的是，像上述示例中一样，对切片使用append方法时，如果元素下标仍在数组长度范围内，也会改变数组的值。
  使用make方法初始化切片则没有这种烦恼，因为是重新开辟一个空间：
{% codeblock lang:go %}
// make方法初始化切片，参数分别是切片类型和切片的len，cap
sliceByMake := make([]int, 2, 6)
fmt.Printf("make初始化切片的len：%v，cap：%v\n", len(sliceByMake), cap(sliceByMake))
sliceByMake = append(sliceByMake, 3)
// make初始化长度对应的元素会用默认值填充，append会接着塞入，而不是从下标0开始
fmt.Printf("sliceByMake:%v\n", sliceByMake)
fmt.Printf("make初始化切片的len：%v，cap：%v\n", len(sliceByMake), cap(sliceByMake))
// 也可以不指定cap
sliceByMakeNoCap := make([]int, 2)
fmt.Printf("不指定cap的make初始化切片的len：%v，cap：%v\n", len(sliceByMakeNoCap), cap(sliceByMakeNoCap))
sliceByMakeNoCap = append(sliceByMakeNoCap, 3)
// make初始化长度对应的元素会用默认值填充，append会接着塞入，而不是从下标0开始
fmt.Printf("sliceByMakeNoCap:%v\n", sliceByMakeNoCap)
fmt.Printf("不指定cap的make初始化切片的len：%v，cap：%v\n", len(sliceByMakeNoCap), cap(sliceByMakeNoCap))
// 结果输出:
// make初始化切片的len：2，cap：6
// sliceByMake:[0 0 3]
// make初始化切片的len：3，cap：6
// 不指定cap的make初始化切片的len：2，cap：2
// sliceByMakeNoCap:[0 0 3]
// 不指定cap的make初始化切片的len：3，cap：4
{% endcodeblock %}
   使用make初始化切片可以指定len和cap，当然在不确定cap的情况下可以不指定，不过指定cap可以减少扩容次数，效率更高。

### map

  go中的map概念上没有什么不同，先看看源码中的定义：

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
  
  与slice类似，定义了一个结构体封装了map的相关信息。可以看到结构设计上与Java的类似，采用的也是拉链法的设计思想。
  这里不详细解析，直接看看其具体用法：

{% codeblock lang:go %}
mapInt := make(map[string]string, 0)
mapInt["0"] = "hello"
for i, v := range mapInt {
	fmt.Printf("map_key:%v,map_value:%v\n", i, v)
}
 // 取值方法
value := mapData["0"]
fmt.Printf("map_value:%v\n", value)
value, ok := mapData["0"]
fmt.Printf("是否取到值：%v,值为:%v\n", ok, value)
value, ok = mapData["1"]
fmt.Printf("是否取到值：%v,值为:%v\n", ok, value)
// 结果输出：
// map_key:0,map_value:hello
// 是否取到值：true,值为:hello
// 是否取到值：false,值为:
{% endcodeblock %}

  同样是使用make方法初始化，只是类型参数指定的是map类型，然后没有切片的cap参数。
  另外，取值方法可以返回一个bool类型表示是否取到值的标识。
  这是因为int，string等数据类型的默认值不是nil，当map中不存在对应key的值时，返回的是对应数据类型的默认
  值，就没法判断到底是没取到值还是map中就是存的默认值。
  而map其他的取值赋值甚至for循环range等操作上不能说和数组切片类似，只能说一摸一样，这也体现了go语法简化的特点。
   
### 结构体

   结构体也是golang中的基本类型，定位上可以对标Java的class，但却有很多不同点。

#### 定义

{% codeblock lang:go %}
type demo struct {
	Hello       func(int) int
	textPrivate string
	TextPublic  string
	Number      int32
	Pointer     *int32
	MyDemo      *demo
	emptyAttr   empty
	emptyPoint  *empty
}

type empty struct {
}
{% endcodeblock %}
  结构体的属性类型可以是int，string这种基本类型，也可以是一个函数，当然也包括其他的结构体或其指针类型。
  但是要注意的是，结构体可以有自身指针类型的属性，但不能包含自身，如上述例子中的MyDemo属性，就不能是demo类型。
  与Java中的class相比，结构体有很多相似的地方，但没有class那么大而重。
  在Java中通常一个class就是一个文件，而结构体显得更加小巧灵活，一个文件中可以定义多个结构体。
  同时结构体中虽然有函数类型的属性，但与class中的定义的方法不同，其定义时是没有具体的方法实现，也因此
  让结构体显得更加精简。

#### 初始化

{% codeblock lang:go %}
func main() {
	// 初始化空结构体
	var l empty
	fmt.Printf("空结构体：%v\n", l)
	// 初始化结构体
	var num int32 = 2
	ss := demo{
		Hello: func(r int) int {
			fmt.Printf("结构体函数属性输出--%v\n", r)
			return 1
		},
		textPrivate: "private text；",
		TextPublic:  "public text；",
		Number:      1111,
		Pointer:     &num,
		MyDemo:      &demo{TextPublic: "child~~"},
		emptyAttr:   empty{},
		emptyPoint:  &empty{},
	}
	fmt.Printf("结构体：%v\n", ss)
	hello := ss.Hello
	hello(1)
	// 访问公有属性
	fmt.Printf("公有属性:%v\n", ss.TextPublic)
	// 访问私有属性
	fmt.Printf("私有属性:%v\n", ss.textPrivate)
	// 访问结构体属性
	fmt.Printf("结构体属性：%v,树状访问：%v\n", ss.MyDemo, ss.MyDemo.TextPublic)
}

type demo struct {
	Hello       func(int) int
	textPrivate string
	TextPublic  string
	Number      int32
	Pointer     *int32
	MyDemo      *demo
	emptyAttr   empty
	emptyPoint  *empty
}

type empty struct {
}
// 结果输出：
// 空结构体：{}
// 结构体：{0x1219ec0 private text； public text； 1111 0xc000019954 0xc000079cc0 {} 0x147d740}
// 结构体函数属性输出--1
// 公有属性:public text；
// 私有属性:private text；
// 结构体属性：&{<nil>  child~~ 0 <nil> <nil> {} <nil>},树状访问：child~~
{% endcodeblock %}

#### 访问控制

  Java中class为了安全有访问权限关键字来控制对类属性及方法的访问，golang同样简化了这一部分，直接通过属
  性名的首字母大小来区分包内访问还是公有制访问(golang中只有这两种访问权限)。
  当首字母为小写时，只能在包内访问；相反为大写时则任何地方都能访问。
  (这种规则不仅适用于结构体的属性，也适用于结构体本身，包括定义的函数，常量等)

#### Tag

{% codeblock lang:go %}
type demo struct {
	Hello       func(int) int `json:"hello"`
	TextPublic  string        `json:"text_public"`
	Number      int32         `json:"number"`
	Pointer     *int32        `json:"pointer"`
	MyDemo      *demo         `json:"my_demo"`
}
{% endcodeblock %}
  在定义结构体时可以在行尾使用``符号定义一个tag，其作用是标识一些信息，如上例中标识出每个属性转json的key值。
  (对私有属性定义json的tag会报警告：Struct field 'XXXX' has 'json' tag but is not exported ）

#### 匿名属性

  结构体中不设置属性名，直接指定类型的属性叫做匿名属性。
  这种属性的属性名由他的类型决定，比如：一个string类型的匿名属性，他的属性名就是string。
  另外，如果匿名属性是一个结构体类型，可以跳过中间的结构体名称直接访问其属性。示例如下：

{% codeblock lang:go %}
type Demo struct {
	other
	string
	int
}

type other struct {
	Name string
	age int
}

func test() {
	demo := Demo{}
	demo.other.Name = "jjj"
	demo.Name = "xxx"
	demo.string = "ddd"
	demo.int = 1
}
{% endcodeblock %}

  这里比较特殊的一点是：当匿名属性为结构体类型且其属性为公有属性(如例子中的other的Name属性)，虽然匿
  名属性首字母小写，无法包外访问，但是仍然可以访问到其公有属性。

{% codeblock lang:go %}
func main() {
	ss := model.Demo{}
	ss.Name = "jjj"
	fmt.Printf("匿名属性的公有属性：%v\n", ss.Name)
}
{% endcodeblock %}

### 总结
  
    golang的数据类型大致介绍到这里，可以看到基本类型部分包括string与Java的区别不太大。
    有较大不同的是与class定位类似的结构体部分，放弃了继承实现这种关系结构，使其更加精简灵活，也符合golang
    的特点。
    另外虽然没有继承实现，但可以使用组合的方式变相实现这种依赖关系。

### 下一章
> [Go学习笔记-函数](https://rel-fly.com/2021/11/09/golang-study-3/)