---
title: golang-defer
date: 2023-03-20 23:02:53
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

`defer` 是 Go 语言中一个独特的关键字，用于注册函数调用，使其在当前函数**返回之前**（无论正常返回还是 panic）按 **LIFO（后进先出）** 顺序执行。理解 defer 的执行机制、参数求值时机以及与 `recover` 的配合使用，是编写健壮的 Go 程序的基础。

<!-- more -->

### 一、基本用法

#### 1.1 语法

```go
func foo() {
    defer fmt.Println("defer 1")
    defer fmt.Println("defer 2")
    fmt.Println("body")
}

func main() {
    foo()
}
```

输出：

```
body
defer 2   // 后注册的先执行（LIFO）
defer 1
```

> **核心规则：多个 defer 按后进先出（LIFO）顺序执行。**

#### 1.2 常见用途

| 场景 | 示例 |
|------|------|
| 资源释放 | `defer file.Close()` / `defer db.Close()` / `defer resp.Body.Close()` |
| 解锁互斥锁 | `defer mu.Unlock()` |
| WaitGroup 计数 | `defer wg.Done()` |
| 错误恢复 | `defer recover()` |
| 记录耗时 | `defer trace(time.Now())` |

### 二、执行机制

#### 2.1 参数预求值（关键点）

**defer 注册时立即对参数求值**，而非执行时：

```go
func foo() {
    x := 10
    defer fmt.Println(x) // 此时 x=10 已被捕获

    x = 20
    fmt.Println("body:", x)
}
// 输出：
// body: 20
// 10      ← 打印的是注册时的值，不是执行时的值
```

如果需要获取执行时的值，用**闭包**或传指针：

```go
// 方式一：闭包
defer func() {
    fmt.Println(x) // 执行时读取，此时 x=20
}()

// 方式二：传递指针
defer func(p *int) {
    fmt.Println(*p)
}(&x)
```

#### 2.2 与 return 的交互

这是 defer 最容易让人困惑的地方。Go 的 `return` 不是原子操作，它分为两步：

```
1. 设置返回值
2. 执行 defer
3. 真正返回（RET 指令）
```

**命名返回值 + defer 修改返回值**：

```go
func foo() (result int) {  // 命名返回值
    result = 0

    defer func() {
        result++  // defer 可以修改命名返回值
    }()

    return  // 等价于 return result → 但 defer 在 return 和 RET 之间执行
}

func main() {
    fmt.Println(foo())  // 输出 1（不是 0！）
}
```

执行流程：

```
result = 0
return result        ← 第一步：设置返回值为 0
defer 执行 result++   ← 第二步：defer 将 result 改为 1
RET                   ← 第三步：实际返回 1
```

**匿名返回值不受 defer 影响**：

```go
func bar() int {       // 匿名返回值
    i := 0

    defer func() {
        i++           // 修改的是局部变量 i，不影响返回值
    }()

    return i           // 返回值在 return 时已确定为 0
}

fmt.Println(bar())    // 输出 0
```

> **总结**：defer 只能通过修改**命名返回值变量**来改变函数的返回结果。

### 三、defer 与 panic/recover

#### 3.1 panic 发生时的 defer 行为

当函数发生 panic 时，已注册的 defer **仍然会执行**——这正是 `recover` 生效的前提：

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("除零错误: %v", r)
        }
    }()

    result = a / b
    if result > 100 {
        panic("结果过大")  // 手动触发 panic
    }
    return
}

r, err := safeDivide(1000, 1)
fmt.Println(r, err)  // 0 除零错误: 结果过大
```

#### 3.2 recover 只能在 defer 中生效

```go
// ❌ 无效 — recover 必须在 defer 中调用
func bad() {
    panic("oops")
    _ = recover()  // 永远返回 nil
}

// ✅ 有效
func good() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println(" recovered:", r)
        }
    }()
    panic("oops")
}
```

> **原因**：`panic` 触发时，控制流开始向上 unwind 栈帧。只有在 defer 函数中调用 `recover`，才能在当前栈帧被销毁前"截获"这个 panic。

#### 3.3 defer 中再次 panic

```go
func tricky() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("第一次 recover:", r)
            panic("第二次 panic!")  // defer 中可以再触发 panic
        }
    }()
    defer func() {
        fmt.Println("第二个 defer（后注册，先执行）")
    }()

    panic("原始 panic")
}
```

输出：

```
第二个 defer（后注册，先执行）
第一次 recover: 原始 panic
panic: 第二次 panic!   ← 新的 panic 会继续向上传播
```

### 四、常见陷阱

#### 陷阱 1：循环中 defer 导致资源延迟释放

```go
// ❌ 所有文件句柄等到函数结束才释放，循环量大时会耗尽文件描述符
func processFiles(files []string) {
    for _, f := range files {
        file, _ := os.Open(f)
        defer file.Close()  // 危险！所有 Close 堆到函数末尾才执行
        // 处理文件...
    }
}

// ✅ 用闭包限制 defer 作用域
func processFiles(files []string) {
    for _, f := range files {
        func() {
            file, _ := os.Open(f)
            defer file.Close()  // 每次 iteration 结束就释放
            // 处理文件...
        }()
    }
}
```

> 这是生产环境最常见的 defer 误用之一。

#### 陷阱 2：defer 修改切片/Map 的副作用

```go
func modifySlice() []int {
    s := []int{1, 2, 3}
    defer func() {
        s[0] = 99          // ✅ 可以修改切片内容
        s = append(s, 4)   // ⚠️ 只是修改了局部变量 s，不影响外部
    }()

    s = append(s, 4)       // s 可能触发扩容变成新底层数组
    return s               // 返回的是新数组，defer 中改的是旧数组
}
```

#### 陷阱 3：方法表达式中的 receiver 提前求值

```go
type Counter struct{ n int }

func (c *Counter) Inc() int {
    c.n++
    return c.n
}

func demo() {
    c := &Counter{n: 0}
    defer c.Inc()  // receiver c 在注册时就确定，但 Inc() 是在返回前调用的
    fmt.Println(c.Inc()) // 1
}
// 输出：
// 1
// 2  ← defer 调用 Inc()
```

这里 `c.Inc()` 作为方法调用，receiver `c` 是引用类型，所以 defer 执行时操作的是同一个对象。但如果 receiver 是值拷贝则需注意。

### 五、性能考量

#### 5.1 defer 有开销吗？

有，但很小。defer 内部维护了一个链表结构来存储待执行的函数和参数：

- **每次 `defer` 调用开销**：约几十纳秒（分配 `_defer` 结构体、链接到链表）
- **大量 defer 在热路径中可能影响性能**

```go
// 高频场景下避免 defer
func fastLoop(n int) int {
    sum := 0
    for i := 0; i < n; i++ {
        mu.Lock()
        sum += i
        mu.Unlock()  // 比 defer 快几 ns
    }
    return sum
}
```

但在绝大多数业务场景下，defer 的开销**完全可忽略**，代码可读性和安全性更重要。

#### 5.2 性能数据参考

| 场景 | 相对耗时 |
|------|---------|
| 直接调用 | 1x |
| defer 调用 | ~30-50ns 额外开销 |
| defer + recover | ~200ns 额外开销 |

> Go 1.14+ 对 defer 做了优化（内联小 defer），性能比早期版本好很多。

### 六、最佳实践清单

| 规则 | 说明 |
|------|------|
| **及时释放资源** | Open 之后紧跟 `defer Close()` |
| **不要在循环中直接 defer** | 用匿名函数包裹或手动释放 |
| **理解参数预求值** | 需要运行时值就用闭包 |
| **recover 只在 defer 中用** | 其他位置调用无效 |
| **命名返回值慎用** | 配合 defer 时行为可能不符合直觉 |
| **优先保证正确性** | 除非是极端性能场景，否则放心用 defer |

### 七、总结

`defer` 的核心要点：

1. **LIFO 顺序** — 多个 defer 后进先出
2. **参数预求值** — 注册时计算参数，执行时才调用
3. **与 return 的交互** — 可通过命名返回值修改返回结果
4. **panic 安全网** — defer 保证即使 panic 也能执行清理逻辑
5. **recover 前提** — 必须在 defer 中调用才有效
6. **循环陷阱** — 循环中的 defer 要注意作用域

> 一句话：**defer 让你写出更安全、更优雅的代码——只要你理解它的行为。**
