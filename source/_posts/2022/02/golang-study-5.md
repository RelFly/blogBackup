---
title: Go学习笔记-select和channel
date: 2022-02-10 16:17:30
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

本章来了解下 golang 中 select 和 channel 的使用。Channel 是 Go 并发模型的核心——goroutine 之间通过 channel 进行通信和同步；而 select 则是多路 channel 操作的调度器，类似于网络编程中的 `select`/`epoll`。

<!-- more -->

### 一、Channel 基础

#### 1.1 创建与基本操作

```go
// 声明方式
var ch1 chan int          // nil channel，不能读写
ch2 := make(chan int)     // 无缓冲 channel（同步）
ch3 := make(chan int, 10) // 有缓冲 channel（异步，容量 10）

// 发送与接收
ch2 <- 42       // 发送数据到 channel
v := <-ch2      // 从 channel 接收数据
v, ok := <-ch2  // 接收 + 判断是否关闭（ok=false 表示已关闭）
```

#### 1.2 无缓冲 vs 有缓冲

**无缓冲 channel**（`make(chan T)`）：发送方会阻塞直到接收方准备好，反之亦然。用于 goroutine 之间的**同步握手**：

```go
func main() {
    ch := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch <- "hello"  // 发送：等待接收者
    }()

    msg := <-ch        // 接收：等待发送者
    fmt.Println(msg)   // hello
}
```

```
发送方 ──[阻塞]──> │无缓冲channel│ <──[阻塞]── 接收方
                      (容量=0，必须双方就绪)
```

**有缓冲 channel**（`make(chan T, N)`）：缓冲区未满时发送不阻塞，未空时接收不阻塞。用于**异步解耦**：

```go
ch := make(chan int, 3)

ch <- 1  // 不阻塞
ch <- 2  // 不阻塞
ch <- 3  // 不阻塞
// ch <- 4  // 阻塞！缓冲区满了

fmt.Println(<-ch)  // 1
fmt.Println(<-ch)  // 2
fmt.Println(<-ch)  // 3
// fmt.Println(<-ch)  // 阻塞！缓冲区空了
```

```
发送方 ──[写入]──> │[1][2][3]│ <──[读取]── 接收方
                     容量=3，有空间就不阻塞
```

#### 1.3 关闭 channel

```go
close(ch)  // 只能由发送方关闭

// 判断是否关闭
v, ok := <-ch
if !ok {
    fmt.Println("channel 已关闭")
}

// 用 range 遍历直到关闭
for v := range ch {
    fmt.Println(v)
}
```

> **注意**：
> - 向已关闭的 channel 发送会 **panic**
> - 从已关闭的 channel 接收会立即返回零值（不会阻塞）
> - 多次 close 会 panic

#### 1.4 单向 channel（类型约束）

```go
// 函数参数中限制方向
func producer(ch chan<- int) { ... }   // 只能发送
func consumer(ch <-chan int) { ... }   // 只能接收

// 转换（双向 → 单向，隐式）
var ch = make(chan int)
producer(ch)   // chan(int) → chan<(int)
consumer(ch)   // chan(int) → <-(int)
```

单向 channel 主要用于**接口约束**，明确表达函数对 channel 的使用意图。

---

### 二、Select 机制

#### 2.1 基本语法

`select` 用于同时等待多个 channel 操作，**哪个先就绪执行哪个**。如果多个同时就绪，随机选一个：

```go
select {
case v := <-ch1:
    fmt.Println("从 ch1 收到:", v)
case v := <-ch2:
    fmt.Println("从 ch2 收到:", v)
case ch3 <- 42:
    fmt.Println("发送成功")
default:
    fmt.Println("没有 channel 就绪")
}
```

#### 2.2 典型模式

**模式一：超时控制**

```go
select {
case result := <-doWork():
    handle(result)
case <-time.After(3 * time.Second):
    fmt.Println("超时了！")
}
```

**模式二：退出信号**

```go
func worker(done chan struct{}) {
    ticker := time.NewTicker(500 * time.Millisecond)

    for {
        select {
        case <-ticker.C:
            fmt.Println("工作中...")
        case <-done:
            fmt.Println("收到退出信号")
            return
        }
    }
}

done := make(chan struct{})
go worker(done)
time.Sleep(2 * time.Second)
close(done)  // 通知退出
```

**模式三：多路合并**

```go
// 合并多个 channel 的输出到同一个 channel
func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    for _, c := range cs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                out <- v
            }
        }(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

#### 2.3 空选择 — 永久阻塞

```go
select {}  // 永远阻塞，常用于阻止 main goroutine 退出
```

这等价于 `for {}` 或 `<-make(chan struct{})`，但语义更清晰——明确表示"我在等待某个事件"。

---

### 三、常见陷阱与注意事项

#### 陷阱 1：向 nil channel 发送/接收永远阻塞

```go
var ch chan int  // nil channel

select {
case ch <- 1:
    // 永远不会执行
default:
    fmt.Println("进入 default")  // 走这里
}

// 利用 nil channel 实现"动态开关"
func process(input <-chan int, stop <-chan struct{}) {
    var in <-chan int

    for {
        select {
        case <-stop:
            return
        case v, ok := <-in:
            if !ok {
                in = nil  // 关闭后置为 nil，后续 case 不再匹配
                continue
            }
            fmt.Println(v)
        }

        if in == nil && input != nil {
            in = input  // 启用输入通道
        }
    }
}
```

> 这是利用 nil channel 做**动态启停**的常用技巧。

#### 陷阱 2：select 的 case 分支是随机选择

当多个 channel 同时就绪时，Go 运行时**随机**选择一个 case 执行（不是按代码顺序）：

```go
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)
ch1 <- 1
ch2 <- 2

// 多次运行，ch1 和 ch2 的顺序不固定
select {
case <-ch1:
    fmt.Println("ch1")
case <-ch2:
    fmt.Println("ch2")
}
```

> 这保证了公平性——避免某个 channel 总是被优先选中而饿死其他 channel。

#### 陷阱 3：for-select 中忘记 break

```go
// ❌ break 只跳出 select，没有跳出 for
for {
    select {
    case <-quit:
        break  // 只是退出 select，下一次循环继续
    }
}

// ✅ 使用 label 或 return
loop:
for {
    select {
    case <-quit:
        break loop  // 跳出 for 循环
    }
}
```

#### 陷阱 4：buffer 大小设置不当导致死锁

```go
// ⚠️ 缓冲太小可能死锁
ch := make(chan int, 1)
ch <- 1
ch <- 2  // 阻塞！没人接收

// ✅ 合理预估缓冲大小或用 goroutine 解耦
go func() { ch <- 2 }()
```

---

### 四、实战示例：带超时的任务分发器

```go
type Task struct{ ID int }

func runTasks(tasks []Task, timeout time.Duration) []Task {
    results := make([]Task, 0, len(tasks))
    taskCh := make(chan Task, len(tasks))
    doneCh := make(chan struct{})

    // 工作池
    const workers = 4
    for i := 0; i < workers; i++ {
        go func(id int) {
            for t := range taskCh {
                time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond) // 模拟工作
                results = append(results, t)
                fmt.Printf("[worker-%d] 完成 task-%d\n", id, t.ID)
            }
        }(i)
    }

    // 投递任务
    go func() {
        for _, t := range tasks {
            taskCh <- t
        }
        close(taskCh)
    }

    // 超时控制
    timer := time.NewTimer(timeout)
    defer timer.Stop()

    select {
    case <-doneCh:
        fmt.Println("全部完成")
    case <-timer.C:
        fmt.Printf("超时 %s，已完成 %d/%d\n", timeout, len(results), len(tasks))
    }

    return results
}
```

### 五、总结

| 特性 | Channel | Select |
|------|---------|--------|
| 核心作用 | goroutine 间通信 | 多路 channel 调度 |
| 同步机制 | 无缓冲 = 同步握手 | 阻塞等待任一就绪 |
| 关键点 | 方向性、关闭、nil channel | 随机公平、default 分支 |
| 典型用途 | 数据传递、信号通知 | 超时控制、退出监听 |

> Go 并发哲学：**Don't communicate by sharing memory; instead, share memory by communicating.**
