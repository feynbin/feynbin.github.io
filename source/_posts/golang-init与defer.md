---
title: Go语言init与defer
top_img: false
tags: golang
abbrlink: a7c9d4e
date: 2026-03-22 18:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言init与defer

> 本文是Go语言学习系列的第七篇。前六篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)、[《Go语言函数》](/posts/f6a8b3c/)，建议先行阅读。

`init` 和 `defer` 是Go中两个特殊的函数机制。`init` 负责包的初始化，在程序启动时自动执行；`defer` 负责延迟调用，在函数返回前执行。两者都不需要手动调用，由运行时自动管理。

---

## init 函数

`init` 是Go中专门用于**包初始化**的特殊函数。它不能被手动调用，不能有参数和返回值，由运行时在 `main` 函数执行前自动调用。

### 基本语法

```go
package main

import "fmt"

func init() {
    fmt.Println("init 执行")
}

func main() {
    fmt.Println("main 执行")
}
// 输出:
// init 执行
// main 执行
```

### init 的四个特性

**1. 无参数、无返回值、不能手动调用**

```go
func init() {
    // 正确：无参数无返回值
}

func init() int {     // 编译错误：不能有返回值
    return 0
}

func main() {
    init() // 编译错误：不能手动调用
}
```

**2. 一个文件可以有多个 init，按声明顺序执行**

```go
package main

import "fmt"

func init() {
    fmt.Println("第一个 init")
}

func init() {
    fmt.Println("第二个 init")
}

func init() {
    fmt.Println("第三个 init")
}

func main() {
    fmt.Println("main")
}
// 输出:
// 第一个 init
// 第二个 init
// 第三个 init
// main
```

这是Go中唯一允许同名函数在同一个包内重复定义的特例。

**3. 包级变量 → init → main 的执行顺序**

```go
package main

import "fmt"

var x = initVar()

func initVar() int {
    fmt.Println("包级变量初始化")
    return 42
}

func init() {
    fmt.Println("init 执行, x =", x)
}

func main() {
    fmt.Println("main 执行")
}
// 输出:
// 包级变量初始化
// init 执行, x = 42
// main 执行
```

执行顺序始终是：**包级变量初始化 → init() → main()**。

**4. 依赖包的 init 先执行**

如果 main 包导入了包 A，A 又导入了包 B，那么执行顺序是：

```
B 的包级变量 → B 的 init()
  → A 的包级变量 → A 的 init()
    → main 的包级变量 → main 的 init()
      → main()
```

```
main 导入 A，A 导入 B：

B.var → B.init() → A.var → A.init() → main.var → main.init() → main()
```

依赖链越深的包越先初始化，保证被依赖的包在使用前已完成初始化。

### init 的典型用途

**1. 注册驱动/插件**——标准库中最常见的用法：

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql" // 只执行 init，注册 MySQL 驱动
)
```

`_ "包路径"` 是空导入（blank import），不使用包的任何导出内容，仅触发其 `init` 函数。MySQL 驱动的 `init` 会调用 `sql.Register("mysql", &MySQLDriver{})` 完成驱动注册。

**2. 配置检查和环境校验**：

```go
func init() {
    if os.Getenv("DB_HOST") == "" {
        log.Fatal("DB_HOST 环境变量未设置")
    }
}
```

**3. 初始化全局资源**：

```go
var config *Config

func init() {
    data, err := os.ReadFile("config.json")
    if err != nil {
        log.Fatal("读取配置失败:", err)
    }
    config = &Config{}
    if err := json.Unmarshal(data, config); err != nil {
        log.Fatal("解析配置失败:", err)
    }
}
```

> **注意**：过度使用 `init` 会导致包的初始化逻辑不透明、难以测试。如果初始化逻辑复杂，建议提供显式的初始化函数（如 `Setup()`），让调用者主动调用。

---

## defer 函数

`defer` 用于注册一个延迟调用，在**当前函数返回前**执行。无论函数是正常 return 还是 panic，defer 都会执行。

### 基本语法

```go
func main() {
    fmt.Println("开始")
    defer fmt.Println("延迟执行")
    fmt.Println("结束")
}
// 输出:
// 开始
// 结束
// 延迟执行
```

### defer 的三个特性

**1. 后进先出（LIFO）——多个 defer 按栈顺序执行**

```go
func main() {
    defer fmt.Println("第一个 defer")
    defer fmt.Println("第二个 defer")
    defer fmt.Println("第三个 defer")
}
// 输出:
// 第三个 defer
// 第二个 defer
// 第一个 defer
```

先注册的后执行，像栈一样后进先出。这保证了"先打开的资源后关闭"的自然顺序。

**2. 参数在 defer 注册时求值，不是在执行时**

```go
func main() {
    x := 10
    defer fmt.Println("defer 中 x =", x) // x 在此时求值为 10
    x = 20
    fmt.Println("main 中 x =", x)
}
// 输出:
// main 中 x = 20
// defer 中 x = 10（不是 20）
```

defer 注册时会立即对参数求值并保存副本。如果需要延迟求值，使用闭包：

```go
func main() {
    x := 10
    defer func() {
        fmt.Println("defer 中 x =", x) // 闭包捕获 x 的引用
    }()
    x = 20
}
// 输出: defer 中 x = 20
```

**3. 可以修改命名返回值**

这在上一篇函数文章中提到过，defer 可以在 return 之后修改命名返回值：

```go
func foo() (result int) {
    defer func() {
        result += 10
    }()
    return 5
}
// return 5 先将 result 赋为 5
// defer 将 result 修改为 15
// 最终返回 15
```

理解 `return` 的三步过程很关键：

```
① 给返回值赋值（命名返回值 = xxx，或创建临时变量）
② 执行 defer 函数
③ 真正返回
```

---

## defer 的实际应用

### 1. 资源释放——文件操作

defer 最经典的用途——确保文件在函数结束时关闭，无论中间是否出错：

```go
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close() // 无论后续是否出错，文件一定会被关闭

    data, err := io.ReadAll(f)
    if err != nil {
        return nil, err // 即使这里返回，f.Close() 也会执行
    }
    return data, nil
}
```

**关键原则**：在成功获取资源**之后**立即写 defer 释放。不要在错误检查之前 defer，否则可能对 nil 资源调用 Close：

```go
// 错误：Open 失败时 f 为 nil，defer f.Close() 会 panic
defer f.Close()
f, err := os.Open(path)

// 正确：先检查错误，再 defer
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()
```

### 2. 资源释放——锁

```go
var mu sync.Mutex

func safeUpdate(m map[string]int, key string, value int) {
    mu.Lock()
    defer mu.Unlock() // 确保函数返回时释放锁

    m[key] = value
    // 即使这里 panic，锁也会被释放
}
```

### 3. 资源释放——网络连接

```go
func fetchURL(url string) (string, error) {
    resp, err := http.Get(url)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close() // 确保 HTTP 响应体被关闭

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }
    return string(body), nil
}
```

### 4. 数据库连接

```go
func queryUser(db *sql.DB, id int) (*User, error) {
    rows, err := db.Query("SELECT name, age FROM users WHERE id = ?", id)
    if err != nil {
        return nil, err
    }
    defer rows.Close() // 确保结果集被关闭，释放数据库连接

    // 处理查询结果...
}
```

### 5. 异常恢复——recover

`defer` 配合 `recover` 可以捕获 panic，防止程序崩溃：

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("捕获到 panic: %v", r)
        }
    }()

    return a / b, nil // b=0 时会 panic
}

result, err := safeDivide(10, 0)
if err != nil {
    fmt.Println(err) // 捕获到 panic: runtime error: integer divide by zero
}
```

`recover` 只能在 defer 函数中调用才有效，在普通函数中调用始终返回 nil。

### 6. 计时——性能度量

```go
func trackTime(name string) func() {
    start := time.Now()
    return func() {
        fmt.Printf("%s 耗时: %v\n", name, time.Since(start))
    }
}

func doWork() {
    defer trackTime("doWork")() // 注意这里的 ()：立即调用 trackTime，defer 的是返回的函数

    time.Sleep(2 * time.Second)
}
// 输出: doWork 耗时: 2.001234s
```

---

## defer 的性能

Go 1.14 之后对 defer 做了显著优化，在大多数场景下 defer 的开销接近于直接调用，几乎可以忽略。不需要为了性能而避免使用 defer。

但在**超高频调用的热路径**中（如每秒百万次调用的编解码函数），如果性能分析证实 defer 是瓶颈，可以考虑手动管理资源释放。

---

## init 与 defer 对比

| 特性 | init | defer |
|------|------|-------|
| 调用时机 | 程序启动时，main 之前 | 函数返回前 |
| 触发方式 | 自动执行，不能手动调用 | `defer` 语句注册，自动执行 |
| 执行次数 | 每个 init 只执行一次 | 每次函数调用都会执行 |
| 执行顺序 | 按声明顺序 | 后进先出（LIFO） |
| 参数/返回值 | 不能有 | defer 的函数可以有参数，参数在注册时求值 |
| 同一文件多个 | 允许（唯一同名函数例外） | 允许，按栈顺序执行 |
| 核心用途 | 包初始化、驱动注册 | 资源释放、异常恢复 |

---

## 面试高频题

### Q1：init 函数的执行顺序是什么？

**答**：分三个层次。第一层，同一个文件内多个 init 按声明顺序执行。第二层，同一个包内多个文件的 init 按文件名字母序执行（依赖编译器实现，不应依赖此顺序）。第三层，不同包之间按依赖关系执行——被依赖的包先初始化。整体顺序是：包级变量初始化 → init() → main()。

### Q2：下面代码输出什么？

```go
func main() {
    fmt.Println("A")
    defer fmt.Println("B")
    fmt.Println("C")
    defer fmt.Println("D")
    fmt.Println("E")
}
```

**答**：输出 `A C E D B`。正常语句按顺序执行：A、C、E。defer 按后进先出执行：D（后注册先执行）、B（先注册后执行）。

### Q3：下面代码输出什么？为什么？

```go
func main() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
}
```

**答**：输出 `2 1 0`。两个原因：第一，defer 参数在注册时求值，所以三次 defer 分别保存了 i=0、i=1、i=2。第二，defer 按 LIFO 顺序执行，所以先输出2，再1，最后0。

### Q4：下面两段代码的输出有什么不同？为什么？

```go
// 代码A
func f1() int {
    x := 0
    defer func() {
        x++
    }()
    return x
}

// 代码B
func f2() (x int) {
    defer func() {
        x++
    }()
    return 0
}
```

**答**：`f1()` 返回 `0`，`f2()` 返回 `1`。`f1` 使用匿名返回值，`return x` 将 x 的值复制给一个临时返回变量，defer 修改的是局部变量 x，不影响已复制的返回值。`f2` 使用命名返回值 `x`，`return 0` 先将 x 赋为 0，defer 中 `x++` 修改的就是返回值本身，所以返回 1。

### Q5：recover 为什么必须在 defer 中调用？

**答**：panic 发生后，当前函数立即停止执行，开始逐层执行已注册的 defer 函数。只有在 defer 函数中，程序还处于 panic 的展开过程中，`recover` 才能捕获 panic 值并恢复正常流程。在普通代码中调用 `recover` 时没有 panic 正在发生，所以始终返回 nil。这是语言设计上的约束，确保 recover 只在明确的错误恢复路径中使用。

### Q6：空导入 `_ "pkg"` 的作用是什么？

**答**：空导入只执行目标包的 init 函数，不使用包中任何导出标识符。最典型的用途是注册驱动，如 `_ "github.com/go-sql-driver/mysql"` 会触发 MySQL 驱动的 init 函数，将驱动注册到 `database/sql` 中。如果不使用空导入，Go编译器会报"imported and not used"错误。

### Q7：defer 会影响性能吗？什么时候需要注意？

**答**：Go 1.14 之后，大多数 defer 被编译器优化为内联调用（open-coded defer），开销接近于直接函数调用，日常开发不需要担心性能。只有在每秒百万级调用的热路径中，且性能分析确认 defer 是瓶颈时，才需要考虑手动释放资源。绝大多数情况下，defer 带来的代码安全性和可读性远大于微小的性能开销。

### Q8：在循环中使用 defer 有什么问题？

```go
func processFiles(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close() // 有问题吗？
    }
    // ...处理
}
```

**答**：有问题。defer 在**函数返回时**才执行，不是在循环迭代结束时。如果 paths 有1000个文件，所有文件都会保持打开状态直到函数返回，可能耗尽文件描述符。解决方式是将循环体提取为独立函数，让 defer 在每次迭代结束时执行：

```go
func processFiles(paths []string) error {
    for _, path := range paths {
        if err := processFile(path); err != nil {
            return err
        }
    }
    return nil
}

func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // 每次调用 processFile 返回时关闭

    // ...处理
    return nil
}
```

---

## 小结

| 概念 | 要点 |
|------|------|
| init 定义 | 无参数无返回值，不能手动调用 |
| init 执行顺序 | 包级变量 → init() → main()，被依赖的包先执行 |
| init 可重复 | 同一文件可以有多个 init，按声明顺序执行 |
| init 用途 | 驱动注册（空导入）、环境校验、全局资源初始化 |
| defer 执行时机 | 函数返回前，无论正常 return 还是 panic |
| defer 顺序 | 后进先出（LIFO），栈结构 |
| defer 参数求值 | 注册时立即求值，不是执行时；用闭包可延迟求值 |
| defer + 命名返回值 | defer 可以修改命名返回值 |
| defer 典型用途 | 文件关闭、锁释放、HTTP Body 关闭、recover 异常恢复 |
| defer 注意点 | 避免在循环中 defer，注意参数求值时机 |

下一篇将介绍Go的**结构体（Struct）与方法**。
