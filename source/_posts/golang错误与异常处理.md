---
title: Go语言错误与异常处理
top_img: false
tags: golang
abbrlink: f5b9c3d
date: 2026-03-23 18:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言错误与异常处理

> 本文是Go语言学习系列的第十二篇。前十一篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)、[《Go语言函数》](/posts/f6a8b3c/)、[《Go语言init与defer》](/posts/a7c9d4e/)、[《Go语言结构体》](/posts/b8d1e5f/)、[《Go语言自定义类型与接口》](/posts/c9e2f6a/)、[《Go语言协程与信道》](/posts/d3f4a7b/)、[《Go语言线程安全与sync.Map》](/posts/e4a8b2c/)，建议先行阅读。

Go语言没有 `try/catch` 异常捕获机制——这是**刻意的设计选择**。Go认为错误是程序的正常组成部分，应该被显式处理而非隐藏在异常流中。函数作为Go的一等公民，通过多返回值将error作为结果的一部分返回，调用方必须决定如何处理它。

---

## Go的错误处理哲学

在Java/Python中，异常通过 `try/catch` 捕获，错误沿调用栈自动传播。Go选择了不同的路：

```
Java:   异常自动传播，调用方可以选择"不处理"
Go:     错误显式返回，调用方必须"做出决定"
```

这意味着在Go中，你会频繁看到这样的代码：

```go
result, err := doSomething()
if err != nil {
    // 必须处理
}
```

这看似啰嗦，但带来了两个好处：
1. **错误处理路径清晰可见**——代码审查时一眼能看到每个错误是否被处理
2. **没有隐藏的控制流**——不会有异常突然跳过几层函数调用

---

## error接口

Go的错误就是一个接口：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error() string` 方法的类型都是error。最常用的创建方式：

```go
import "errors"

// 方式一：errors.New
err := errors.New("文件不存在")

// 方式二：fmt.Errorf（支持格式化）
err := fmt.Errorf("用户 %s 不存在", username)
```

### 自定义错误类型

当需要携带更多信息时，可以定义自己的错误类型：

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("字段 %s 校验失败: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 || age > 150 {
        return &ValidationError{
            Field:   "age",
            Message: "年龄必须在0-150之间",
        }
    }
    return nil
}
```

---

## 三种异常处理策略

Go中处理错误有三种基本策略，适用于不同严重程度的场景：

### 策略一：向上抛（返回error）

最常见的方式——函数不处理错误，包装后返回给调用方：

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("读取配置文件失败: %w", err) // %w 包装原始错误
    }
    return data, nil
}

func initApp() error {
    config, err := readConfig("config.yaml")
    if err != nil {
        return fmt.Errorf("初始化失败: %w", err) // 继续向上抛
    }
    // 使用config...
    return nil
}

func main() {
    if err := initApp(); err != nil {
        fmt.Println("启动失败:", err)
        // 输出: 启动失败: 初始化失败: 读取配置文件失败: open config.yaml: no such file or directory
    }
}
```

`%w` 是Go 1.13引入的错误包装动词，保留了错误链，可以用 `errors.Is` 和 `errors.As` 解包。

**适用场景**：大多数情况。函数不知道如何处理错误时，应该返回给调用方决策。

### 策略二：中断程序（panic / log.Fatalln）

当遇到不可恢复的严重错误时，直接终止程序：

#### panic

`panic` 会立即停止当前函数执行，逐层执行defer后终止程序：

```go
func mustConnect(dsn string) *sql.DB {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        panic(fmt.Sprintf("数据库连接失败: %v", err))
    }
    if err = db.Ping(); err != nil {
        panic(fmt.Sprintf("数据库不可达: %v", err))
    }
    return db
}
```

panic执行流程：
```
panic触发
  → 当前函数停止
  → 执行当前函数的defer（LIFO顺序）
  → 返回调用方，执行调用方的defer
  → 逐层向上，直到main退出
  → 打印panic信息和堆栈
```

#### log.Fatalln

`log.Fatalln` 打印日志后直接调用 `os.Exit(1)`，**不执行defer**：

```go
func main() {
    f, err := os.Open("important.dat")
    if err != nil {
        log.Fatalln("无法打开关键文件:", err)
        // 打印日志后直接退出，defer不会执行
    }
    defer f.Close() // 如果上面Fatal了，这行不会执行
}
```

#### panic vs log.Fatalln

| 特性 | panic | log.Fatalln |
|------|-------|-------------|
| defer执行 | **会执行** | **不会执行** |
| 可被recover | 是 | 否（直接os.Exit） |
| 输出 | 错误信息 + 完整堆栈 | 仅日志信息 |
| 适用场景 | 程序bug、不变量被破坏 | 启动阶段的致命错误 |

**适用场景**：
- `panic`：程序逻辑错误、不可能出现的情况（类似assert）、初始化必须成功的资源
- `log.Fatalln`：main函数启动阶段加载配置、连接数据库等失败，无法继续运行

> **原则**：库代码不应该panic（应返回error），只有应用层的main或init中才适合panic/Fatal。

### 策略三：恢复程序（recover）

`recover` 是Go唯一能"捕获"panic的机制，**只能在defer中使用**：

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("运行时异常: %v", r)
        }
    }()

    return a / b, nil // b=0时panic: runtime error: integer divide by zero
}

func main() {
    result, err := safeDivide(10, 0)
    if err != nil {
        fmt.Println("错误:", err) // 错误: 运行时异常: runtime error: integer divide by zero
    } else {
        fmt.Println("结果:", result)
    }
}
```

#### recover的规则

1. **只能在defer函数中调用**，直接调用无效：

```go
// ❌ 无效：不在defer中
func bad() {
    recover() // 永远返回nil
}

// ✅ 正确：在defer中
func good() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获:", r)
        }
    }()
    panic("boom")
}
```

2. **只能捕获当前goroutine的panic**，无法跨goroutine：

```go
func main() {
    defer func() {
        recover() // 无法捕获子goroutine的panic
    }()

    go func() {
        panic("子goroutine崩溃") // 程序直接崩溃
    }()

    time.Sleep(time.Second)
}
```

3. recover之后，panic后面的代码不会继续执行，从defer返回后函数正常返回：

```go
func example() {
    defer func() {
        recover()
    }()

    fmt.Println("before") // 执行
    panic("crash")
    fmt.Println("after")  // 不执行
}
// 函数正常返回（不会再panic）
```

#### 实际应用：HTTP服务器防崩溃

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recovered: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

> 这是recover最经典的使用场景——单个请求的panic不应该崩掉整个服务。

---

## 错误包装与解包（Go 1.13+）

### 包装错误：fmt.Errorf + %w

```go
func readUser(id int) (*User, error) {
    row := db.QueryRow("SELECT * FROM users WHERE id = ?", id)
    var u User
    if err := row.Scan(&u.Name, &u.Email); err != nil {
        return nil, fmt.Errorf("查询用户 %d 失败: %w", id, err)
    }
    return &u, nil
}
```

### 解包错误：errors.Is 和 errors.As

```go
// errors.Is：判断错误链中是否包含特定错误值
if errors.Is(err, sql.ErrNoRows) {
    fmt.Println("用户不存在")
}

// errors.As：从错误链中提取特定类型的错误
var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Printf("字段 %s 校验失败: %s\n", ve.Field, ve.Message)
}
```

`errors.Is` 和 `errors.As` 会沿着错误链逐层查找，所以即使错误被多层包装，仍然能匹配到原始错误。

### %w vs %v

```go
// %w：包装错误，保留错误链（errors.Is/As可以解包）
fmt.Errorf("操作失败: %w", err)

// %v：格式化为字符串，断开错误链（无法解包）
fmt.Errorf("操作失败: %v", err)
```

**选择原则**：如果调用方需要判断根因（如 `sql.ErrNoRows`），用 `%w`；如果只是记录日志不需要程序化判断，用 `%v`。

---

## 哨兵错误（Sentinel Errors）

预定义的错误值，用于和 `errors.Is` 配合判断：

```go
// 标准库中的哨兵错误
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrTimeout      = errors.New("timeout")
)

func GetUser(id int) (*User, error) {
    // ...
    if notFound {
        return nil, fmt.Errorf("获取用户: %w", ErrNotFound)
    }
    return user, nil
}

// 调用方
user, err := GetUser(42)
if errors.Is(err, ErrNotFound) {
    // 用户不存在的处理逻辑
}
```

> **命名规范**：哨兵错误以 `Err` 开头（如 `ErrNotFound`），错误类型以 `Error` 结尾（如 `ValidationError`）。

---

## 易错点总结

| 问题 | 后果 | 解决方案 |
|------|------|----------|
| 忽略返回的error（`_ = err`） | 静默失败，难以排查 | 必须处理或显式注释说明原因 |
| recover不在defer中 | 无法捕获panic | 始终在defer func中调用 |
| 想跨goroutine recover | 程序崩溃 | 在每个goroutine内部defer recover |
| 用 `%v` 替代 `%w` 包装 | 错误链断裂 | 需要解包时用 `%w` |
| 库代码中使用panic | 调用方难以处理 | 库应返回error，不应panic |
| log.Fatalln后期望defer执行 | defer不执行，资源泄漏 | 改用panic或返回error |
| 比较error用 `==` | 包装后的error无法匹配 | 用 `errors.Is()` |

---

## 面试题精选

### 基础题

**Q1：Go为什么没有try/catch？这样设计有什么优缺点？**

> Go认为错误是正常的返回值，不是异常流。优点：（1）错误处理路径在代码中显式可见，不会被隐藏；（2）没有隐式的控制流跳转，代码更容易理解；（3）强制开发者思考每个错误怎么处理。缺点：（1）`if err != nil` 大量重复，代码冗长；（2）错误容易被 `_` 忽略（虽然显式，但仍可能偷懒）。Go团队认为显式处理的好处远大于代码冗长的代价。

**Q2：panic和error有什么区别？分别在什么时候使用？**

> `error` 是函数返回值，表示**可预见的错误**（文件不存在、网络超时等），调用方应该处理。`panic` 表示**不可恢复的程序错误**（数组越界、nil指针等），类似Java的RuntimeException。使用原则：（1）可以预见并处理的情况用error；（2）程序逻辑错误、不可能出现的分支用panic；（3）库代码几乎不应该panic，应返回error。

**Q3：以下代码能正确recover吗？为什么？**

```go
func main() {
    defer recover()
    panic("crash")
}
```

> **不能**。`recover` 必须在defer函数中**直接调用**才有效。这里 `defer recover()` 虽然在defer中，但recover是被直接注册为defer函数，而不是在defer函数内部调用。正确写法是 `defer func() { recover() }()`。实际上Go规范要求recover必须由deferred function直接调用。

### 进阶题

**Q4：errors.Is和errors.As有什么区别？分别用在什么场景？**

> `errors.Is(err, target)` 沿错误链比较**错误值**，用于判断是否是某个特定的哨兵错误（如 `sql.ErrNoRows`）。`errors.As(err, &target)` 沿错误链查找**错误类型**，用于提取自定义错误类型中的附加信息。类比：`errors.Is` 像 `==` 比较，`errors.As` 像类型断言。

**Q5：%w和%v包装错误有什么区别？什么时候用哪个？**

> `%w` 保留错误链，被包装的错误可以通过 `errors.Is` / `errors.As` 解包匹配；`%v` 只是把错误信息格式化为字符串，丢失了原始错误的类型信息。选择标准：如果上层需要根据根因做不同处理（如判断是不是 `ErrNotFound`），用 `%w`；如果只是拼接日志信息、不需要程序化匹配，用 `%v`（此时也避免了暴露内部实现细节）。

**Q6：如何在goroutine中安全地处理panic？**

```go
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}
```

> 每个goroutine必须自己recover，父goroutine无法捕获子goroutine的panic。生产中通常封装一个 `safeGo` 工具函数，在最外层defer recover，防止单个goroutine的panic导致整个程序崩溃。HTTP框架（如Gin、Echo）内置了recovery中间件，原理相同。

### 高级题

**Q7：panic → recover的完整执行流程是怎样的？**

> （1）panic触发，当前函数立即停止执行，panic之后的代码不再运行；（2）按LIFO顺序执行当前函数已注册的defer；（3）如果某个defer中调用了recover，panic被捕获，函数正常返回（返回零值或命名返回值）；（4）如果没有recover，panic继续向上层函数传播，重复步骤2-3；（5）传播到goroutine顶层仍无recover，打印panic信息和完整堆栈，程序以非零状态码退出。

**Q8：设计一个错误处理方案，支持错误码、错误信息、错误链。**

```go
type AppError struct {
    Code    int    // 业务错误码
    Message string // 用户友好信息
    Err     error  // 原始错误（支持错误链）
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// 实现Unwrap支持errors.Is/As
func (e *AppError) Unwrap() error {
    return e.Err
}

// 构造函数
func NewAppError(code int, msg string, err error) *AppError {
    return &AppError{Code: code, Message: msg, Err: err}
}

// 使用
var ErrUserNotFound = errors.New("user not found")

func GetUser(id int) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, NewAppError(404, "用户不存在", ErrUserNotFound)
        }
        return nil, NewAppError(500, "数据库查询失败", err)
    }
    return user, nil
}

// 调用方
user, err := GetUser(42)
if err != nil {
    var appErr *AppError
    if errors.As(err, &appErr) {
        // 可以拿到错误码返回给前端
        respondJSON(w, appErr.Code, appErr.Message)
    }
    if errors.Is(err, ErrUserNotFound) {
        // 也可以判断根因
    }
}
```

> 关键点：实现 `Unwrap() error` 方法让自定义错误类型融入Go的错误链体系。

**Q9：log.Fatalln、os.Exit、panic三者的区别是什么？**

| 特性 | `panic` | `log.Fatalln` | `os.Exit` |
|------|---------|---------------|-----------|
| 执行defer | 是 | 否 | 否 |
| 可recover | 是 | 否 | 否 |
| 输出内容 | 错误 + 堆栈 | 日志信息 | 无 |
| 退出码 | 2 | 1 | 自定义 |
| 适用场景 | 程序bug | 启动致命错误 | 正常退出/脚本 |

> `log.Fatalln` 底层就是 `log.Println + os.Exit(1)`。因为不执行defer，使用时要注意不要在已获取需要清理的资源之后调用。`panic` 是唯一会执行defer的"异常退出"方式，因此也是唯一可以被recover的。

---

## 小结

| 策略 | 函数/关键字 | 适用场景 |
|------|------------|----------|
| 向上抛 | `return error` / `fmt.Errorf("%w")` | 大多数情况，让调用方决策 |
| 中断程序 | `panic` / `log.Fatalln` | 不可恢复的严重错误 |
| 恢复程序 | `defer` + `recover` | HTTP服务防崩溃、库的边界保护 |

Go错误处理的核心思想：**错误是值，应该被显式处理**。不要忽略error，不要滥用panic，在合适的层级做出合适的决策。
