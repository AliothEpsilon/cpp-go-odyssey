# Go 语法基础

本文档旨在快速回顾 Go 语言的核心基础语法，为深入学习 Go 的高级特性和内部机制提供必要的知识准备。

## 1. 程序结构

Go 程序由包（`package`）组成。每个 Go 源文件都必须以 `package` 声明开头，定义该文件所属的包。

*   **`package`**: 声明包名。`main` 包是程序的入口点。
*   **`import`**: 导入其他包以使用其功能。可以导入标准库包（如 `"fmt"`, `"os"`）或第三方包。

```go
package main // 声明这是 main 包

import (
    "fmt"  // 导入 fmt 包用于格式化 I/O
    "math" // 导入 math 包
)

func main() {
    fmt.Println("Hello, Go!")
}
```

## 2. 变量与常量

### 变量 (Variables)
使用 `var` 关键字声明变量，或使用短变量声明 `:=`（仅在函数内部使用）。

```go
var name string        // 声明一个 string 类型的变量 name
var age int = 30       // 声明并初始化
var height = 1.75      // 类型推断
city := "Beijing"      // 短声明，类型推断。等价于 var city string = "Beijing"

// 多变量声明
var x, y int = 1, 2
var a, b, c = true, 2.5, "hello"
d, e := 3, 4           // 短声明多变量
```
### 常量 (Constants)
使用 `const` 关键字声明常量，值在编译时确定且不可修改。

```go
const Pi = 3.14159
const (
    StatusOK       = 200
    StatusNotFound = 404
)
```

### 作用域 (Scope)
*   **包级作用域**: 在函数外部声明的变量/常量/函数，可在整个包内访问（导出时首字母大写可在包外访问）。
*   **块级作用域**: 在 `{}` 内部（如函数、if、for 语句块）声明的变量，仅在该块内及其嵌套块内可见。

### 关键字 (Keywords)
Go 语言有 25 个预定义的、不能用作标识符（如变量名、函数名、类型名等）的关键字。它们构成了语言的基本语法结构。

以下是 Go 语言的完整关键字列表：

*   **声明与定义**: `const`, `var`, `type`, `func`, `package`, `import`
*   **控制流**:
    *   `if`, `else`, `switch`, `select`, `case`, `default`
    *   `for`, `range`, `break`, `continue`, `goto`, `fallthrough`
*   **并发**: `go`, `chan`
*   **延迟与返回**: `defer`, `return`

**重要提示**:
*   `const` 和 `var` 用于声明常量和变量。
*   `type` 用于定义新类型。
*   `func` 用于声明函数。
*   `package` 和 `import` 用于组织和引入代码包。
*   `go` 启动一个新 goroutine。
*   `chan` 用于声明通道类型。
*   `defer` 用于延迟函数调用的执行。

## 3. 基础数据类型

Go 是静态类型语言，变量类型在声明时确定。

*   **布尔类型**: `bool` (`true`, `false`)
*   **字符串类型**: `string` (UTF-8 编码的字节序列)
*   **整数类型**:
    *   有符号: `int`, `int8`, `int16`, `int32`, `int64`
    *   无符号: `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`
    *   `byte` 是 `uint8` 的别名，常用于处理单个字节。
    *   `rune` 是 `int32` 的别名，常用于表示 Unicode 码点。
*   **浮点类型**: `float32`, `float64`
*   **复数类型**: `complex64`, `complex128`

## 4. 复合基础类型

### 数组 (Array)
固定长度的同类型元素序列。长度是类型的一部分。

```go
var numbers [5]int           // 声明一个长度为5的整数数组
numbers[0] = 10              // 赋值
fmt.Println(numbers[2])      // 访问，输出 0 (零值)
primes := [3]int{2, 3, 5}    // 初始化
```

### 切片 (Slice)
动态长度的序列，是对底层数组的**引用**。更常用。

```go
var slice []int              // 声明 nil 切片
slice = []int{1, 2, 3}       // 字面量初始化
slice = append(slice, 4)     // 添加元素
slice = slice[1:3]           // 切片操作 [start:end)，左闭右开
len(slice)                   // 长度
cap(slice)                   // 容量
```

### 映射 (Map)
键值对（Key-Value）的无序集合。

```go
var ages map[string]int      // 声明 nil map
ages = make(map[string]int)  // 必须先 make 才能使用
ages["Alice"] = 25
ages["Bob"] = 30
age, exists := ages["Alice"] // 查询，exists 为 bool 表示键是否存在
delete(ages, "Bob")          // 删除键值对
```

## 5. 控制流

### `if` / `else`
```go
if x > 10 {
    fmt.Println("x is greater than 10")
} else if x < 0 {
    fmt.Println("x is negative")
} else {
    fmt.Println("x is between 0 and 10")
}

// if 可以带初始化语句
if val := getValue(); val > 0 {
    fmt.Println("Value is positive:", val)
}
```

### `for` 循环
Go 中 `for` 是唯一的循环关键字，可实现 `while` 和 `do-while` 的功能。

```go
// 经典 for 循环
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// while 风格
j := 0
for j < 3 {
    fmt.Println(j)
    j++
}

// 无限循环
for {
    // ... break 来退出
}

// for range (遍历)
for index, value := range slice {
    fmt.Printf("Index: %d, Value: %d\n", index, value)
}
// 遍历 map
for key, value := range ages {
    fmt.Printf("Name: %s, Age: %d\n", key, value)
}
// 只需要索引/键
for index := range slice { ... }
// 只需要值
for _, value := range slice { ... } // _ 是空标识符，忽略索引
```

### `switch` 语句
比 C 风格更强大，`case` 自动 break。

```go
switch day {
case "Monday", "Tuesday":
    fmt.Println("Weekday")
case "Saturday", "Sunday":
    fmt.Println("Weekend")
default:
    fmt.Println("Invalid day")
}

// switch 表达式可以省略 (相当于 switch true)
switch {
case score >= 90:
    grade = "A"
case score >= 80:
    grade = "B"
    // ... fallthrough 可用于穿透，但通常不推荐
}
```

## 6. 函数 (Functions)

使用 `func` 关键字定义函数。

```go
// 函数签名: func 函数名(参数列表) 返回值列表
func add(a int, b int) int {
    return a + b
}

// 多返回值 (非常常见)
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// 调用函数
sum := add(3, 4)
result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println("Result:", result)
```

## 7. 结构体基础 (Struct)

`struct` 是一组任意类型的命名字段的集合，用于创建自定义数据类型。

```go
// 定义结构体类型
type Person struct {
    Name string
    Age  int
}

// 创建结构体实例 (字面量初始化)
p1 := Person{Name: "Alice", Age: 30}
p2 := Person{"Bob", 25} // 按字段顺序初始化
var p3 Person            // 零值初始化，Name="", Age=0

// 访问字段
fmt.Println(p1.Name, p1.Age)
p1.Age = 31 // 修改字段
```
