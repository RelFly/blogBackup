---
title: golang-append
date: 2023-03-20 23:02:07
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

`append` 是 Go 中操作切片（slice）最常用的内置函数，看似简单，但背后涉及切片的底层结构、扩容机制，在并发场景下更是容易踩坑。本文系统梳理 `append` 的使用方式、常见陷阱、相关方法扩展以及并发安全问题的分析与解决。

<!-- more -->

### 一、append 基础用法

#### 1.1 基本语法

```go
func append(slice []T, elements ...T) []T
```

向切片尾部追加元素，返回新的切片引用：

```go
s := []int{1, 2,3}
s = append(s, 4)        // [1 2 3 4]
s = append(s, 5, 6)     // [1 2 3 4 5 6]
```

#### 1.2 追加另一个切片（`...` 展开运算符）

```go
s1 := []int{1, 2, 3}
s2 := []int{4, 5, 6}
s1 = append(s1, s2...)  // [1 2 3 4 5 6]
```

**注意**：必须使用 `...` 将 `s2` 展开，否则编译报错。

#### 1.3 从零开始构建切片

```go
// 方式一：先声明再 append
var s []int
s = append(s, 1)  // []

// 方式二：make 预分配容量（推荐）
s := make([]int, 0, 10)
```

> **最佳实践**：当大致知道元素数量时，用 `make([]T, 0, cap)` 预分配容量，避免多次扩容带来的性能损耗。

---

### 二、底层原理与扩容机制

#### 2.1 切片的内存布局

Go 切片是一个包含三个字段的结构体：

```
┌─────────────┬─────────────┬─────────────┐
│   pointer   │    len      │    cap      │
│ (指向数组)   │  (元素个数)  │ (底层数组容量) │
└─────────────┴─────────────┴─────────────┘
```

#### 2.2 扩容策略

`append` 的核心逻辑：

```go
// 简化版伪代码
func append(slice []T, elems ...T) []T {
    oldLen := len(slice)
    newLen := oldLen + len(elems)

    if newLen <= cap(slice) {
        // 容量足够，原地写入
        slice = slice[:newLen]
        copy(slice[oldLen:], elems)
        return slice
    }

    // 容量不足，分配新数组
    newCap := growCap(oldLen, cap(slice), newLen)
    newSlice := make([]T, newLen, newCap)
    copy(newSlice, slice)
    copy(newSlice[oldLen:], elems)
    return newSlice
}
```

**Go 1.18+ 的扩容规则**（以 `int` 为例）：

| 条件 | 新容量计算 |
|------|-----------|
| `newLen > 1024` | `oldCap + (oldCap * 3) / 4`（增长 25%） |
| `newLen <= 1024` | **翻倍** `oldCap * 2` |
| 计算结果 < `newLen` | 直接取 `newLen` |

> 具体实现参考 [`runtime/growslice.go`](https://go.dev/src/runtime/slices.go)，不同类型因内存对齐会有细微差异。

#### 2.3 扩容示例

```go
s := make([]int, 0, 1) // cap=1
fmt.Println(cap(s))     // 1

s = append(s, 1)        // len=1, cap=1（够用）
s = append(s, 2)        // 触发扩容 → cap=2
s = append(s, 3)        // 触发扩容 → cap=4
s = append(s, 4)        // 够用
s = append(s, 5)        // 触发扩容 → cap=8
```

---

### 三、常见陷阱

#### 陷阱 1：忽略返回值（最常见错误）

```go
s := []int{1, 2, 3}
append(s, 4)  // ⚠️ 错误！没有用返回值赋值
fmt.Println(s) // 还是 [1 2 3]，修改丢失了
```

**原因**：当发生扩容时，`append` 会返回指向新数组的切片。即使未扩容，返回值的 `len` 字段也已更新。

**正确写法**：

```go
s = append(s, 4)
```

#### 陷阱 2：共享底层数组的多个切片互相影响

```go
a := []int{1, 2, 3, 4, 5}       // len=5, cap=5
b := a[:3]                        // b=[1 2 3], 共享同一底层数组
c := append(b, 100)               // c=[1 2 3 100] ✅
fmt.Println(a)                    // [1 2 3 100 5] ⚠️ a 被意外修改！
```

**原因**：`a` 的 `cap=5`，`b` 的 `len=3` 但 `cap=5`（因为从 `a` 切出来），所以 `append(b, 100)` 写入的是 `a` 的第 4 个位置。

下面用图说明共享底层数组的过程：

**步骤 1 — 初始状态**

```
a (len=5, cap=5)
┌───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   ← 底层数组
└───┴───┴───┴───┴───┘

b = a[:3]  →  b (len=3, cap=5) 指向同一底层数组的 [0:3]
```

**步骤 2 — 执行 `append(b, 100)`**

```
a 的视角（看到全部）          b 的视角（只看到前3个）
┌───┬───┬───┬────┬───┐      ┌───┬───┬───┐
│ 1 │ 2 │ 3 │ 100│ 5 │      │ 1 │ 2 │ 3 │  ← len=3，看不到后面
└───┴───┴───┴────┴───┘      └───┴───┴───┘
                  ↑                    ↑
            原本是 4              cap=5 > len=3
           被 100 覆盖❌         无需扩容，原地写入！
```

**步骤 3 — 结果：`a` 被污染**

```
fmt.Println(a)  →  [1 2 3 100 5]  ⚠️ 不是预期的 [1 2 3 4 5]
fmt.Println(b)  →  [1 2 3]        （b 只看前3个，看起来正常）
```

> 关键点：`b` 虽然只有 `len=3`，但 `cap=5` 继承自 `a`。`append` 发现容量够用就直接往底层数组第 4 个位置写，而那个位置恰好也是 `a` 看得到的。

**解决方式**：截断时强制拷贝：

```go
b := make([]int, 3)
copy(b, a[:3])  // 深拷贝，不再共享底层数组
```

或使用 Go 1.21+ 的 `slices.Clone()`：

```go
import "slices"
b := slices.Clone(a[:3])
```

#### 陷阱 3：循环中 append 到自身导致无限循环

```go
// ⚠️ 危险代码
for i := 0; i < len(s); i++ {
    s = append(s, s[i]*2)  // 每次追加都改变 len，条件永远为真
}
```

**正确做法**：先缓存长度：

```go
n := len(s)
for i := 0; i < n; i++ {
    s = append(s, s[i]*2)
}
```

#### 陷阱 4：nil 切片 vs 空切片

```go
var s1 []int           // nil 切片，len=0, cap=0
s2 := []int{}          // 空切片，len=0, cap=0
s3 := make([]int, 0)   // 同上

s1 = append(s1, 1)  // 正常工作 ✅
s2 = append(s2, 1)  // 正常工作 ✅

fmt.Println(s1 == nil)  // true
fmt.Println(s2 == nil)  // false
```

> 功能上等价，但序列化（JSON 等）时行为不同：
> - `nil` → JSON 输出 `null`
> - `[]` → JSON 输出 `[]`

---

### 四、相关方法扩展

#### 4.1 copy — 切片拷贝

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)
n := copy(dst, src)  // dst=[1 2 3], n=3（实际拷贝数量）

// 完整拷贝
dstFull := make([]int, len(src))
copy(dstFull, src)
```

**vs append 拷贝对比**：

```go
// append 拷贝（更简洁）
dst := append([]int(nil), src...)

// copy 拷贝（语义更明确）
dst := make([]int, len(src))
copy(dst, src)
```

两者功能等价，`append` 更简洁，`copy` 可精确控制拷贝数量和目标位置。

#### 4.2 删除元素

Go 没有内置删除函数，常用模式：

**删除中间元素（保持顺序）：**

```go
func removeOrdered[T any](s []T, idx int) []T {
    return append(s[:idx], s[idx+1:]...)
}

s := []int{1, 2, 3, 4, 5}
s = removeOrdered(s, 2) // [1 2 4 5]
```

**删除中间元素（不保持顺序，更快）：**

```go
func removeUnordered[T any](s []T, idx int) []T {
    s[idx] = s[len(s)-1]
    return s[:len(s)-1]
}

s := []int{1, 2, 3, 4, 5}
s = removeUnordered(s, 2) // [1 2 5 4]（末尾元素移到被删位置）
```

> 不保序版本避免了 `append` 导致的数据搬移，O(1) 时间复杂度。

#### 4.3 过滤元素（Go 1.21+ 泛型写法）

```go
package main

import "cmp"

// Filter 保留满足条件的元素
func Filter[S ~[]E, E any](s S, fn func(E) bool) S {
    result := make(S, 0, len(s))
    for _, v := range s {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// 使用示例
nums := []int{1, 2, 3, 4, 5, 6}
evens := Filter(nums, func(n int) bool { return n%2 == 0 })
// evens = [2 4 6]
```

#### 4.4 截断与预留空间

```go
s := []int{1, 2, 3, 4, 5}

// 截断（保留前 N 个元素）
s = s[:3]          // [1 2 3]

// 清空但保留底层数组容量（避免下次 append 重新分配）
s = s[:0]          // len=0, cap=5
s = append(s, 99)  // 直接复用原数组，无分配
```

这个技巧在**对象池复用**场景非常有用：

```go
var buf []byte

func process(data []byte) {
    buf = buf[:0]            // 重置长度，保留容量
    buf = append(buf, data...)  // 复用已分配的内存
    // ... 处理 buf ...
}
```

#### 4.5 Go 1.21+ 标准库 `slices` 包

Go 1.21 引入了泛型 `slices` 包，很多场景可以替代手写 `append` 操作：

```go
import "slices"

s := []int{3, 1, 4, 1, 5}

slices.Insert(s, 1, 9, 8)       // 在索引1插入 [3, 9, 8, 1, 4, 1, 5]
slices.Replace(s, 1, 3, 7, 7)    // 替换索引1~3 [3, 7, 7, 1, 5]
slices.Delete(s, 2, 4)           // 删除索引2~4 [3, 7, 5]
slices.Clone(s)                  // 深拷贝
slices.Reverse(s)                // 反转 [5, 7, 3]
slices.Contains(s, 7)            // true
```

> 这些方法底层都是基于 `append` / `copy` 实现，但语义更清晰、不易出错。

---

### 五、并发安全问题

#### 5.1 问题分析：append 不是原子操作

`append` 本质上做了三件事：

```
1. 读取 len/cap     ← 读
2. 判断是否扩容     ← 判断
3. 写入数据         ← 写
```

这不是原子操作，多 goroutine 并发执行会导致**数据竞争（data race）**：

```go
// ⚠️ 危险！存在数据竞争
var s []int

func worker(id int) {
    for i := 0; i < 1000; i++ {
        s = append(s, id*1000+i)  // 多个 goroutine 同时读写
    }
}

func main() {
    for i := 0; i < 10; i++ {
        go worker(i)
    }
    time.Sleep(time.Second)
    fmt.Printf("len=%d, expect=10000\n", len(s))
    // 结果不确定：可能丢失数据、panic、或输出错误值
}
```

运行 `go run -race main.go` 会直接报 **WARNING: DATA RACE**。

**具体风险**：

| 风险类型 | 说明 |
|---------|------|
| 数据丢失 | 多个 goroutine 基于旧的 len 写入，互相覆盖 |
| 索引越界 | 一个 goroutine 已扩容但另一个还在用旧指针写 |
| 内存损坏 | 最严重的情况，可能导致程序 panic 或静默错误 |

#### 5.2 解决方案一：互斥锁（Mutex）

最通用的方案：

```go
var (
    mu sync.Mutex
    s  []int
)

func workerSafe(id int) {
    for i := 0; i < 1000; i++ {
        mu.Lock()
        s = append(s, id*1000+i)
        mu.Unlock()
    }
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                mu.Lock()
                s = append(s, id*1000+j)
                mu.Unlock()
            }
        }(i)
    }
    wg.Wait()
    fmt.Printf("len=%d\n", len(s)) // 稳定输出 10000
}
```

**优化：批量加锁减少竞争**

如果每个元素都加锁，锁竞争会很激烈。可以批量收集后一次性 append：

```go
func workerBatch(id int) {
    local := make([]int, 0, 1000)  // 每个 goroutine 用自己的局部切片
    for i := 0; i < 1000; i++ {
        local = append(local, id*1000+i)
    }
    mu.Lock()
    s = append(s, local...)  // 只锁一次
    mu.Unlock()
}
```

性能可提升 **数十倍**，是生产环境推荐的模式。

#### 5.3 解决方案二：通道（Channel）

利用 channel 串行化写入：

```go
func producer(ch chan<- int, id int) {
    for i := 0; i < 1000; i++ {
        ch <- id * 1000 + i
    }
}

func consumer(ch <-chan int, done chan struct{}) {
    for v := range ch {
        s = append(s, v)  // 单个 goroutine 安全 append
    }
    close(done)
}

func main() {
    ch := make(chan int, 128) // 带缓冲减少阻塞
    done := make(chan struct{})

    go consumer(ch, done)

    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            producer(ch, id)
        }(i)
    }
    wg.Wait()
    close(ch)
    <-done
    fmt.Printf("len=%d\n", len(s)) // 10000
}
```

**适用场景**：数据流式处理，天然适合生产者-消费者模型。

#### 5.4 解决方案三：sync.Pool 减少分配压力

高并发场景下，频繁创建临时切片会造成 GC 压力。用 `sync.Pool` 复用切片：

```go
var pool = sync.Pool{
    New: func() interface{} {
        return make([]int, 0, 256) // 预分配合理容量
    },
}

func processWithPool(data int) []int {
    buf := pool.Get().([]int)
    defer func() {
        buf = buf[:0]  // 归还前清空
        pool.Put(buf)
    }()

    buf = append(buf, data)
    // ... 处理逻辑 ...
    result := make([]int, len(buf))
    copy(result, buf)  // 返回独立副本
    return result
}
```

> 注意：从 pool 取出的切片**必须视为脏数据**，使用前重置 `buf[:0]`；放回池中也应清空，避免内存泄漏。

#### 5.5 各方案对比

| 方案 | 适用场景 | 性能 | 复杂度 |
|------|---------|------|--------|
| **Mutex 批量** | 通用，大多数场景 | ⭐⭐⭐⭐ | 低 |
| Mutex 逐条 | 简单场景，低频写入 | ⭐⭐ | 最低 |
| **Channel** | 流式数据处理 | ⭐⭐⭐ | 中 |
| **sync.Pool** | 高频分配/回收 | ⭐⭐⭐⭐⭐ | 中高 |

---

### 六、性能优化建议

#### 6.1 预分配容量

```go
// ❌ 差：不知道最终大小，反复扩容
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)  // 约 13 次扩容
}

// ✅ 好：预分配已知大小
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)  // 零次扩容
}
```

#### 6.2 三要素法则（Three-Index Trick）

截取切片时同时指定三个索引，**只暴露需要的部分**，隐藏多余容量：

```go
s := []int{1, 2, 3, 4, 5}

// ❌ 两索引：保留了尾部容量，有误改风险
sub := s[:3]  // len=3, cap=5

// ✅ 三索引：截断容量，安全
safe := s[0:3:3]  // len=3, cap=3
_ = append(safe, 999)  // 触发新分配，不影响原始数组
```

这是 Go 官方推荐的防御性编程手法。

#### 6.3 避免不必要的 append

某些场景可以直接通过索引赋值：

```go
// ❌
result := make([]int, n)
for i := 0; i < n; i++ {
    result = append(result, calculate(i))  // 可能触发扩容
}

// ✅
result := make([]int, n)
for i := 0; i < n; i++ {
    result[i] = calculate(i)  // 直接赋值，不会扩容
}
```

---

### 七、总结

`append` 虽然只是一个内置函数，但用好它需要理解：

1. **始终接收返回值** — 忘记赋值是最常见的 bug
2. **理解扩容机制** — 合理预分配容量，避免性能问题
3. **注意底层数组共享** — 使用三索引截断或 `slices.Clone` 隔离
4. **并发场景加保护** — Mutex 批量 / Channel / Pool，根据场景选择
5. **善用标准库** — Go 1.21+ 的 `slices` 包提供了更安全的封装

> 一句话总结：**append 很简单，但要写出正确的 append 并不容易。**

### 参考资料

- [Go Slices: usage and internals](https://go.dev/blog/slices)
- [Effective Go - Slices](https://go.dev/doc/effective_go#slices)
- [The Go Programming Language Specification - Appending to and copying slices](https://go.dev/ref/spec#Appending_and_copying_slices)
- [runtime/slices.go (Go 源码)](https://github.com/golang/go/blob/master/src/runtime/slices.go)
