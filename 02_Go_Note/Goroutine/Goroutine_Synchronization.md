# 第二篇：同步与通信——如何协调多个 Goroutine？

## 1. 新问题：主函数提前退出

```go
func main() {
    go fmt.Println("Hello")
    // main 函数结束 → 整个程序退出
    // 上面的 Goroutine 可能根本没执行！
}
```

> ❌ **问题**：Goroutine 启动后，main 不等待它。

---

## 2. 方案一：`sync.WaitGroup`——等待任务完成

`WaitGroup` 像一个计数器：
- `Add(n)`：增加待完成任务数
- `Done()`：完成一个任务（减 1）
- `Wait()`：阻塞，直到计数为 0

### 示例：等所有快递员完成任务
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func deliver(name string, wg *sync.WaitGroup) {
    defer wg.Done() // 任务完成
    time.Sleep(1 * time.Second)
    fmt.Printf("🚚 %s 完成配送\n", name)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go deliver(fmt.Sprintf("快递员%d", i), &wg)
    }

    wg.Wait() // 阻塞等待
    fmt.Println("✅ 所有配送完成！")
}
```

> ⚠️ 注意：`wg` 必须传指针，否则每个 Goroutine 拿到的是副本。

---

## 3. 方案二：Channel——通过通信共享内存

Go 的哲学：
> **“不要通过共享内存来通信，而应该通过通信来共享内存。”**

### 什么是 Channel？
- 一个**类型化管道**，用于在 Goroutine 之间传递数据
- 提供同步机制（发送/接收阻塞）

### 创建与使用
```go
ch := make(chan string) // 字符串通道

go func() {
    ch <- "任务完成" // 发送
}()

msg := <-ch // 接收
fmt.Println(msg)
```

### 示例：用 Channel 等待
```go
func main() {
    ch := make(chan bool)

    go func() {
        time.Sleep(1 * time.Second)
        fmt.Println("任务执行中...")
        ch <- true // 通知完成
    }()

    <-ch // 等待信号
    fmt.Println("✅ 收到完成信号")
}
```

---

## 4. Channel 的类型

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **无缓冲 channel** | `make(chan T)` | 同步点（发送/接收同时准备好） |
| **有缓冲 channel** | `make(chan T, 3)` | 解耦生产/消费速度 |

```go
ch := make(chan int, 2) // 缓冲区大小为 2
ch <- 1 // 不阻塞
ch <- 2 // 不阻塞
ch <- 3 // 阻塞！缓冲区满
```

---

## 5. 小结

| 工具 | 用途 | 原则 |
|------|------|------|
| `WaitGroup` | 等待一组任务完成 | 适合“通知完成” |
| `Channel` | 传递数据 + 同步 | 更强大，推荐作为首选 |

