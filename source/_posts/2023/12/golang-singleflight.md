---
title: golang-singleflight
date: 2023-12-03 11:51:14
tags:
- Go学习笔记
categories:
- Go
- 学习笔记
---

### 前言

`golang.org/x/sync/singleflight` 是 Go 扩展库中提供的一个并发原语，用于**抑制对相同 key 的重复请求**。当多个 goroutine 同时请求同一个资源时，singleflight 只让其中一个执行，其余等待并共享结果。这在缓存击穿、防止 DB 重复查询等场景非常实用。

<!-- more -->

### 一、基本用法

#### 1.1 核心类型与方法

```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

// Do — 对相同 key 只执行一次 fn，其他调用者等待结果
result, shared, err := g.Do(key string, func() (interface{}, error) {
    // 这里只会被一个 goroutine 执行
    return fetchDataFromDB(key)
})

// result: fn 的返回值
// shared: true 表示结果被多个调用者共享（即你是等待者而非执行者）
// err:   错误

// DoChan — 返回 channel 版本（适合 select/超时场景）
ch := g.DoChan(key, func() (interface{}, error) { ... })
select {
case res := <-ch:
    fmt.Println(res.Val, res.Err, res.Shared)
case <-time.After(timeout):
    // 超时处理
}
```

#### 1.2 最简示例

```go
package main

import (
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/singleflight"
)

var sf singleflight.Group

func fetchURL(url string) (string, error) {
    fmt.Printf("  [实际请求] %s\n", url)
    time.Sleep(100 * time.Millisecond) // 模拟网络延迟
    return "response from: " + url, nil
}

func main() {
    var wg sync.WaitGroup
    url := "https://example.com/api/data"

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            val, shared, err := sf.Do(url, func() (interface{}, error) {
                return fetchURL(url)
            })
            if err != nil {
                fmt.Printf("[goroutine-%d] 错误: %v\n", id, err)
                return
            }
            fmt.Printf("[goroutine-%d] 结果: %s (shared=%v)\n", id, val.(string), shared)
        }(i)
    }

    wg.Wait()
}
```

输出（多次运行结果可能略有不同）：

```
[goroutine-2] [实际请求] https://example.com/api/data
[goroutine-2] 结果: response from: https://example.com/api/data (shared=false)
[goroutine-0] 结果: response from: https://example.com/api/data (shared=true)
[goroutine-1] 结果: response from: example.com/api/data (shared=true)
[goroutine-3] 结果: response from: https://example.com/api/data (shared=true)
[goroutine-4] 结果: response from: https://example.com/api/data (shared=true)
```

> **关键观察**：5 个 goroutine 同时发起请求，但 `fetchURL` **只执行了一次**，其余 4 个共享了同一个结果。

### 二、典型应用场景

#### 场景一：缓存击穿防护

```go
type Cache struct {
    data map[string]string
    mu   sync.RWMutex
    sf   singleflight.Group
}

func (c *Cache) Get(key string) (string, error) {
    c.mu.RLock()
    if v, ok := c.data[key]; ok {
        c.mu.RUnlock()
        return v, nil
    }
    c.mu.RUnlock()

    // 缓存未命中，用 singleflight 防止穿透
    v, err, _ := c.sf.Do(key, func() (interface{}, error) {
        // 只有 1 个 goroutine 会走到这里
        value, err := queryDB(key)
        if err != nil {
            return nil, err
        }

        c.mu.Lock()
        c.data[key] = value
        c.mu.Unlock()

        return value, nil
    })

    if err != nil {
        return "", err
    }
    return v.(string), nil
}
```

> 没有单飞保护：缓存失效瞬间，1000 个请求同时打到数据库 → **缓存击穿**
> 有单飞保护：1000 个请求合并为 1 次 DB 查询 → 安全

#### 场景二：API 聚合去重

```go
// 多个前端请求同时需要同一份聚合数据
func GetAggregatedData(ctx context.Context, userID string) (*AggData, error) {
    ch := sf.DoChan(userID, func() (interface{}, error) {
        return aggregateFromMultipleServices(ctx, userID)
    })

    select {
    case result := <-ch:
        if result.Err != nil {
            return nil, result.Err
        }
        return result.Val.(*AggData), nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

#### 场景三：配置热更新

```go
// 多个服务实例同时发现配置过期，只拉取一次
func GetConfig(version string) (*Config, error) {
    cfg, _, err := sf.Do("config:"+version, func() (interface{}, error) {
        return fetchConfigFromCenter(version)
    })
    return cfg.(*Config), err
}
```

### 三、进阶用法与注意事项

#### 3.1 Forget — 取消正在执行的请求

```go
// 如果某个请求耗时过长，可以用 Forget 让后续请求重新触发
go func() {
    time.Sleep(500 * time.Millisecond)
    sf.Forget(key)  // 取消当前正在进行的请求
}()

val, _, err := sf.Do(key, slowFunc)
```

#### 3.2 错误传播行为

```go
// fn 返回错误时，所有等待的 goroutine 都会收到同一个错误
val, shared, err := sf.Do("key", func() (interface{}, error) {
    return nil, fmt.Errorf("查询失败")
})
// 所有调用者都会得到 err != nil
```

> 如果希望错误时不让其他调用者也失败，需要在 fn 内部做重试或返回默认值。

#### 3.3 与 context 配合超时控制

```go
func safeGet(key string) (interface{}, error) {
    type Result struct {
        Val interface{}
        Err error
    }

    ch := make(chan Result, 1)

    go func() {
        val, _, err := sf.Do(key, expensiveQuery)
        ch <- Result{Val: val, Err: err}
    }()

    select {
    case res := <-ch:
        return res.Val, res.Err
    case <-time.After(3 * time.Second):
        sf.Forget(key) // 放弃本次请求，允许下次重新触发
        return nil, errors.New("超时")
    }
}
```

#### 3.4 内存泄漏风险

如果 `Do` 调用的 `fn` 长时间不返回（如死锁、网络无限阻塞），所有等待的 goroutine 都会一直阻塞：

```go
// ⚠️ 危险：fn 可能永远不返回
sf.Do("key", func() (interface{}, error) {
    return http.Get("http://slow-server")  // 无超时设置
})
```

**防御措施**：
1. fn 内部使用带超时的 HTTP client / context
2. 外层配合 `DoChan` + `select` + 超时 + `Forget`
3. 设置合理的超时时间

### 四、与其他方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| **singleflight** | 开箱即用、零侵入 | 依赖扩展库 |
| **互斥锁 + 双检锁** | 标准库即可 | 实现复杂、容易出错 |
| **本地缓存 + TTL** | 简单直接 | TTL 到期仍有击穿风险 |
| **Redis 分布式锁** | 跨进程安全 | 引入外部依赖、性能开销大 |

对于**单进程内**的去重需求，singleflight 是最简洁的选择。

### 五、总结

`singleflight` 是 Go 并发工具箱中的利器：

- **核心价值**：相同 key 的重复调用 → 合并为一次执行 → 共享结果
- **最佳场景**：缓存击穿防护、API 去重、配置加载
- **注意事项**：配合超时和 Forget 使用，避免长时间阻塞；注意错误传播语义
- **一句话**：当你发现"同一时刻有大量重复请求"时，就是 singleflight 登场的时机。
