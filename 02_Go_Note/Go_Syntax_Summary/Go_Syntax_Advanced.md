# Go 语法进阶

## 1. 指针 (Pointers)

指针保存的是另一个变量的内存地址。

*   **`&` (取地址)**: 获取变量的内存地址。
*   **`*` (解引用)**: 获取指针所指向的变量的值。

```go
x := 10
p := &x        // p 是一个 *int 类型的指针，指向 x
fmt.Println(*p) // 输出 10 (解引用 p)
*p = 20        // 通过指针修改 x 的值
fmt.Println(x)  // 输出 20
```

### 指针接收者 vs 值接收者
定义方法时，接收者可以是值也可以是指针。
*   **值接收者**: 方法接收的是接收者类型的一个副本。适用于小型、不可变或不需要修改接收者的场景。
*   **指针接收者**: 方法接收的是接收者类型的指针。适用于需要修改接收者状态、或接收者是大型结构体（避免复制开销）的场景。

```go
type Person struct {
    Name string
    Age  int
}

// 值接收者方法 (接收 Person 的副本)
func (p Person) Greet() string {
    return "Hello, I'm " + p.Name
}

// 指针接收者方法 (接收 *Person)
func (p *Person) HaveBirthday() {
    p.Age++ // 修改 p 指向的 Person 的 Age
}

func main() {
    alice := Person{Name: "Alice", Age: 25}
    fmt.Println(alice.Greet()) // 调用值接收者方法
    alice.HaveBirthday()       // 调用指针接收者方法，Age 变为 26
    fmt.Println(alice.Age)     // 输出 26
}
```

## 2. 方法 (Methods)

方法是与特定类型关联的函数。使用 `func (receiver ReceiverType) MethodName(...)` 语法定义。

*   **方法集 (Method Set)**:
    *   对于类型 `T`，其方法集包含所有接收者为 `T` 的方法。
    *   对于类型 `*T`，其方法集包含所有接收者为 `T` 或 `*T` 的方法。
    *   这意味着 `*T` 的变量可以调用 `T` 和 `*T` 的方法，而 `T` 的变量只能调用 `T` 的方法（除非 Go 自动取地址，见下文）。

> **注意**: 当你有一个 `T` 类型的变量并调用一个指针接收者方法时，Go 会自动获取该变量的地址（如果变量是可寻址的）。反之则不行。

## 3. 接口 (Interfaces)

接口定义了一组方法的集合。任何类型只要实现了接口中的所有方法，就**隐式地**实现了该接口。

*   **定义接口**:
    ```go
    type Speaker interface {
        Speak() string
    }
    ```

*   **隐式实现**:
    ```go
    type Dog struct{}
    func (d Dog) Speak() string { return "Woof!" }

    type Cat struct{}
    func (c *Cat) Speak() string { return "Meow!" } // 注意：接收者是指针

    func MakeItSpeak(s Speaker) {
        fmt.Println(s.Speak())
    }

    func main() {
        dog := Dog{}
        cat := Cat{}
        MakeItSpeak(dog) // OK: Dog 实现了 Speaker
        MakeItSpeak(&cat) // OK: *Cat 实现了 Speaker, 传 cat 的地址
        // MakeItSpeak(cat) // 错误！cat (Cat) 本身没有实现 Speak() 方法集
    }
    ```

*   **空接口 `interface{}`**: 不包含任何方法的接口。**任何类型都实现了空接口**。常用于需要处理任意类型的场景（类似其他语言的 `Object`）。
    ```go
    var anything interface{}
    anything = 42
    anything = "hello"
    anything = Person{Name: "Bob"}
    ```

*   **类型断言 (Type Assertion)**: 用于从接口值中提取具体类型的值。
    ```go
    // value, ok := interfaceValue.(ConcreteType)
    str, ok := anything.(string)
    if ok {
        fmt.Println("It's a string:", str)
    } else {
        fmt.Println("It's not a string")
    }
    // 或者直接断言 (不安全，可能 panic)
    // str := anything.(string)
    ```

*   **类型选择 (Type Switch)**: 一种特殊的 `switch` 语句，用于比较接口值的具体类型。
    ```go
    func describe(i interface{}) {
        switch v := i.(type) {
        case int:
            fmt.Printf("Integer: %d\n", v)
        case string:
            fmt.Printf("String: %s\n", v)
        case Person:
            fmt.Printf("Person: %+v\n", v)
        default:
            fmt.Printf("Unknown type: %T\n", v)
        }
    }
    ```

## 4. 包 (Packages) 与 导出规则

*   **包 (Package)**: Go 代码的组织单元。一个目录下的所有 `.go` 文件属于同一个包。
*   **导出规则**: 标识符（变量、函数、类型、方法名等）如果以**大写字母**开头，则是**导出的**，可以在包外部访问。以小写字母开头的标识符是包内私有的。
    ```go
    // 在 mypkg 包中
    var ExportedVar int    // 包外可访问
    var unexportedVar int  // 仅包内可访问

    func ExportedFunc() {} // 包外可访问
    func unexportedFunc(){} // 仅包内可访问
    ```

## 5. 模块 (Modules)

Go Modules 是 Go 的依赖管理工具。

*   **`go mod init <module-name>`**: 在项目根目录初始化一个新模块，创建 `go.mod` 文件。
*   **`go mod tidy`**: 清理 `go.mod` 和 `go.sum`，移除未使用的依赖，添加缺失的依赖。
*   **`go get <package>`**: 下载并添加依赖到 `go.mod`。
*   **`go.mod`**: 定义模块路径、Go 版本和直接依赖。
*   **`go.sum`**: 记录所有依赖模块的特定版本的校验和，确保构建的可重现性。

## 6. 核心编程惯用法 (Idioms)

### `defer`
`defer` 语句用于延迟函数调用，直到包含它的函数即将返回时才执行。常用于资源清理。

*   **执行时机**: 在 `return` 语句执行*之后*，函数实际退出*之前*执行。
*   **常见模式**:
    ```go
    func processFile(filename string) error {
        file, err := os.Open(filename)
        if err != nil {
            return err
        }
        defer file.Close() // 确保函数退出前文件被关闭

        // ... 处理文件 ...
        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
            // ...
        }
        return scanner.Err()
    }
    ```
    *   多个 `defer` 语句按**后进先出 (LIFO)** 顺序执行。

### 错误处理
Go 通过返回 `error` 类型的值来处理错误。

*   **`error` 接口**: `type error interface { Error() string }`
*   **`nil` 错误检查**: 函数调用后，通常立即检查返回的 `error` 是否为 `nil`。
    ```go
    result, err := someFunction()
    if err != nil {
        log.Printf("Error occurred: %v", err)
        return err
    }
    // 使用 result
    ```
*   **自定义错误**: 使用 `errors.New()` 或 `fmt.Errorf()` 创建错误。
    ```go
    if x < 0 {
        return 0, fmt.Errorf("cannot calculate square root of negative number: %f", x)
    }
    ```

### 并发基础: `go` 关键字
使用 `go` 关键字可以启动一个新的 Goroutine，它是 Go 并发的基石。

```go
func sayHello() {
    fmt.Println("Hello from a goroutine!")
}

func main() {
    go sayHello() // 启动一个新的 goroutine 执行 sayHello
    time.Sleep(100 * time.Millisecond) // 主 goroutine 等待，否则程序可能在 sayHello 执行前结束
    fmt.Println("Main function")
}
```

### 命名返回值
函数可以为返回值命名，这些名字在函数体内作为变量使用，并在 `return` 语句中可以省略。

```go
func split(sum int) (x, y int) { // 命名返回值 x, y
    x = sum * 4 / 9
    y = sum - x
    return // 隐式返回 x 和 y 的当前值
}
```

### 空标识符 `_`
下划线 `_` 是一个特殊的标识符，称为“空标识符”或“blank identifier”。用于忽略不需要的值。

```go
_, err := fmt.Println("Hello") // 忽略返回的字节数
for _, value := range slice {  // 忽略索引
    fmt.Println(value)
}
```
