# 第三篇：高级控制——生命周期、取消与最佳实践

## 1. 新挑战：如何取消一个正在运行的 Goroutine？

Goroutine 一旦启动，**无法从外部强制停止**。  
但我们可以通过**通信机制**优雅地通知它退出。

### 解法：`context.Context`

`context` 是 Go 中用于**传递请求范围的截止时间、取消信号、和元数据**的标准方式。

---

## 2. Context 的核心方法

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // 确保释放资源

go worker(ctx)

// 一段时间后取消
time.Sleep(2 * time.Second)
cancel() // 发送取消信号
```

---

## 3. 示例：可取消的长时间任务

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func longTask(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("🛑 任务被取消:", ctx.Err())
            return
        default:
            fmt.Println("🔄 工作中...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    go longTask(ctx)

    // 模拟主程序运行
    time.Sleep(5 * time.Second)
    fmt.Println("🔚 主程序结束")
}
```

> ✅ 输出中你会看到任务在 3 秒后被自动取消。

---

## 4. Context 的四种派生方式

| 函数 | 用途 |
|------|------|
| `WithCancel` | 手动取消 |
| `WithTimeout` | 超时自动取消 |
| `WithDeadline` | 指定截止时间取消 |
| `WithValue` | 传递请求数据（如 trace ID） |

---

## 5. 常见陷阱与最佳实践

### ❌ 错误 1：Goroutine 泄漏
```go
ch := make(chan int)
go func() {
    // 永远不会发送数据
    // 这个 Goroutine 永远阻塞，无法回收！
}()

// 没有接收者 → 发送者永远阻塞
```

✅ **解决**：使用 `context` 控制生命周期，设置超时。

---

### ❌ 错误 2：竞态条件（Race Condition）
```go
var counter int
for i := 0; i < 100; i++ {
    go func() {
        counter++ // 多个 Goroutine 同时修改！
    }()
}
```

✅ **解决**：
- 使用 `sync.Mutex`
- 或使用 `channel` 传递修改请求
- 或使用 `sync/atomic`

---

## 6. 总结：Goroutine 的完整心智模型

| 层级 | 工具 | 目的 |
|------|------|------|
| **启动** | `go func()` | 创建并发任务 |
| **等待** | `WaitGroup`, `channel` | 同步完成 |
| **通信** | `channel` | 安全传递数据 |
| **控制** | `context.Context` | 取消、超时、传递元数据 |
| **保护** | `mutex`, `atomic` | 避免竞态 |

> ✅ **终极原则**：
> - 优先使用 **channel** 进行通信与同步
> - 用 **context** 管理生命周期
> - 避免共享可变状态
