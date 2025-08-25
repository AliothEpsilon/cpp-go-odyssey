# 第一篇：Goroutine 入门——并发的第一性原理

## 1. 问题的起点：程序为什么“慢”？

想象你是一家快递站的员工，要完成两项任务：
1. 给客户打电话确认地址（耗时 2 秒）
2. 打包包裹（耗时 1 秒）

如果**顺序执行**：
```go
callCustomer() // 等 2 秒
packPackage()  // 再等 1 秒
// 总耗时：3 秒
```

但现实中，你不会傻等电话接通。你会：
- 拨出电话 → 等待对方接听（这期间你空闲）
- 趁等待时 → 先去打包包裹
- 电话接通后 → 继续沟通

> 💡 **第一性原理**：  
> **等待是低效的根源**。CPU 在等待 I/O（网络、磁盘、用户输入）时，应该去做别的事。

---

## 2. 什么是 Goroutine？—— Go 的并发原语

Goroutine 是 Go 运行为你提供的**轻量级执行单元**。

- 它不是操作系统线程（OS Thread）
- 它由 Go 运行时（runtime）调度
- 启动成本极低（约 2KB 栈空间）
- 一个程序可轻松运行数百万个

### 启动方式：`go` 关键字
```go
go function() // 一句话，开启并发
```

### 示例：模拟快递任务
```go
package main

import (
    "fmt"
    "time"
)

func callCustomer() {
    time.Sleep(2 * time.Second)
    fmt.Println("✅ 电话确认完成")
}

func packPackage() {
    time.Sleep(1 * time.Second)
    fmt.Println("📦 包裹打包完成")
}

func main() {
    go callCustomer() // 并发执行
    go packPackage()  // 并发执行

    fmt.Println("📦 开始处理...")
    
    // 等待足够时间让 goroutine 完成（仅演示）
    time.Sleep(3 * time.Second)
}
```

**输出**：
```
📦 开始处理...
📦 包裹打包完成
✅ 电话确认完成
```

> ✅ **总耗时 ≈ 2 秒**（取最长任务），效率提升 33%！

---

## 3. Goroutine 的本质：协作式多任务

- Go 使用 **M:N 调度模型**：M 个 Goroutine 映射到 N 个系统线程
- 调度由 Go runtime 控制，非操作系统
- Goroutine 在 I/O 阻塞、channel 操作、系统调用时自动让出（yield）

> 🧠 **类比**：  
> 操作系统线程 = 全职员工（资源多，切换成本高）  
> Goroutine = 兼职学生（轻量，可大量使用）

---

## 4. 小结

| 概念 | 说明 |
|------|------|
| `go func()` | 启动一个 Goroutine |
| 轻量 | 初始栈小，动态增长 |
| 并发 | 多个任务交替执行（不一定并行） |
| runtime 调度 | Go 自己管理，不依赖 OS |
