---
title: Go语言函数
top_img: false
tags: golang
abbrlink: f6a8b3c
date: 2026-03-22 16:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言函数

> 本文是Go语言学习系列的第六篇。前五篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)，建议先行阅读。

函数是Go程序的基本构建单元。Go的函数设计简洁而强大——支持多返回值、命名返回值、可变参数、匿名函数和闭包，同时函数本身也是一等公民（first-class），可以作为参数传递和返回。

---

## 函数定义

### 基本语法

使用 `func` 关键字定义函数：

```go
func 函数名(参数列表) 返回值 {
    函数体
}
```

```go
func greet(name string) string {
    return "Hello, " + name
}

fmt.Println(greet("Gopher")) // Hello, Gopher
```

### 同类型参数简写

连续多个参数类型相同时，前面的类型可以省略，只保留最后一个：

```go
// 完整写法
func add(x int, y int) int {
    return x + y
}

// 简写：x 和 y 都是 int
func add(x, y int) int {
    return x + y
}

// 多个参数组合简写
func compute(a, b int, op string, verbose bool) int { ... }
```

### 可变参数（...）

使用 `...` 定义可变参数，在函数内部以**切片**形式接收：

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

fmt.Println(sum(1, 2, 3))       // 6
fmt.Println(sum(1, 2, 3, 4, 5)) // 15
```

**规则**：可变参数必须是参数列表的**最后一个**：

```go
// 正确：固定参数在前，可变参数在后
func printf(format string, args ...interface{}) { ... }

// 编译错误：可变参数不在最后
func bad(args ...int, name string) { ... }
```

传递切片给可变参数函数，使用 `...` 展开：

```go
nums := []int{1, 2, 3, 4}
fmt.Println(sum(nums...)) // 10
```

---

## 返回值

### 无返回值

```go
func printMsg(msg string) {
    fmt.Println(msg)
}
```

### 单个返回值

```go
func double(x int) int {
    return x * 2
}
```

### 多返回值

Go支持返回多个值，最常见的用法是返回结果和错误：

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("除数不能为零")
    }
    return a / b, nil
}

result, err := divide(10, 3)
if err != nil {
    fmt.Println("错误:", err)
    return
}
fmt.Println(result) // 3.3333...
```

不需要某个返回值时，用 `_` 忽略：

```go
result, _ := divide(10, 3) // 忽略错误（不推荐）
```

### 命名返回值（Named Return）

返回值可以命名，在函数体内作为局部变量使用，`return` 时自动返回这些变量的当前值：

```go
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = fmt.Errorf("除数不能为零")
        return // 等价于 return result, err → return 0, error
    }
    result = a / b
    return // 等价于 return result, err → return 计算结果, nil
}
```

**命名返回值的好处**：

**1. 零值初始化**——命名返回值自动初始化为类型零值，错误路径中不需要手动构造零值：

```go
// 无命名返回值：错误时必须写出零值
func parse(s string) (int, bool, string, error) {
    if s == "" {
        return 0, false, "", fmt.Errorf("空字符串") // 每个零值都要写
    }
    // ...
}

// 命名返回值：只需设置 err，其余自动为零值
func parse(s string) (num int, ok bool, msg string, err error) {
    if s == "" {
        err = fmt.Errorf("空字符串")
        return // num=0, ok=false, msg="", err=error
    }
    // ...
}
```

**2. 文档作用**——返回值有名字，调用者一看签名就知道每个返回值的含义：

```go
// 不清晰：两个 int 分别是什么？
func getSize() (int, int)

// 清晰：宽度和高度
func getSize() (width, height int)
```

**3. 在 defer 中修改返回值**——这是命名返回值最强大的特性，常用于统一的错误处理和资源清理：

```go
func readFile(path string) (content string, err error) {
    f, err := os.Open(path)
    if err != nil {
        return
    }
    defer func() {
        // defer 中可以修改命名返回值
        if closeErr := f.Close(); closeErr != nil && err == nil {
            err = closeErr // 确保 Close 的错误不被忽略
        }
    }()

    data, err := io.ReadAll(f)
    if err != nil {
        return
    }
    content = string(data)
    return
}
```

> **注意**：命名返回值虽然方便，但在函数体较长时，裸 `return`（不带参数的return）会降低可读性——读者需要回溯寻找各返回值的最新赋值。**建议**：短函数可以用裸return，长函数或逻辑复杂时显式写出返回值。

---

## 匿名函数

没有名字的函数，可以在定义时立即调用，或赋值给变量：

```go
// 赋值给变量
add := func(a, b int) int {
    return a + b
}
fmt.Println(add(1, 2)) // 3

// 立即调用（IIFE）
result := func(x int) int {
    return x * x
}(5)
fmt.Println(result) // 25
```

匿名函数常用于：
- 作为参数传递给高阶函数
- 在 goroutine 中执行
- 实现闭包

```go
// goroutine 中使用匿名函数
go func(msg string) {
    fmt.Println(msg)
}("异步执行")
```

---

## 高阶函数

函数在Go中是**一等公民**，可以作为参数和返回值。接收函数作为参数或返回函数的函数，称为高阶函数。

### 函数作为参数

```go
func apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, v := range nums {
        result[i] = fn(v)
    }
    return result
}

nums := []int{1, 2, 3, 4}

doubled := apply(nums, func(n int) int { return n * 2 })
fmt.Println(doubled) // [2 4 6 8]

squared := apply(nums, func(n int) int { return n * n })
fmt.Println(squared) // [1 4 9 16]
```

### 函数作为返回值

```go
func multiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

double := multiplier(2)
triple := multiplier(3)

fmt.Println(double(5)) // 10
fmt.Println(triple(5)) // 15
```

### 函数类型

可以用 `type` 为函数签名定义别名，提高可读性：

```go
type MathFunc func(int, int) int

func calculate(a, b int, fn MathFunc) int {
    return fn(a, b)
}

result := calculate(10, 3, func(a, b int) int { return a + b })
```

---

## 闭包（Closure）

闭包是引用了外部变量的函数。闭包"捕获"外部变量的**引用**，而非值的副本——外部变量的修改对闭包可见，闭包的修改对外部也可见：

```go
func counter() func() int {
    count := 0
    return func() int {
        count++ // 捕获并修改外部变量 count
        return count
    }
}

c := counter()
fmt.Println(c()) // 1
fmt.Println(c()) // 2
fmt.Println(c()) // 3

// 每次调用 counter() 创建独立的闭包环境
c2 := counter()
fmt.Println(c2()) // 1（与 c 互不影响）
```

### 闭包的经典陷阱

在循环中创建闭包时，要注意捕获的变量：

```go
// Go 1.21及之前的陷阱
funcs := make([]func(), 3)
for i := 0; i < 3; i++ {
    funcs[i] = func() {
        fmt.Println(i) // 捕获的是变量 i 的引用，不是值
    }
}
for _, f := range funcs {
    f()
}
// Go 1.21: 输出 3 3 3（所有闭包共享同一个 i，循环结束时 i=3）
// Go 1.22+: 输出 0 1 2（每次迭代 i 是独立变量）
```

Go 1.22 之前的解决方式：

```go
for i := 0; i < 3; i++ {
    i := i // 创建局部变量，遮蔽循环变量
    funcs[i] = func() {
        fmt.Println(i)
    }
}
```

---

## 值传递与引用传递

### Go只有值传递

Go中所有函数参数都是**值传递**——传入的是参数的副本。没有引用传递。

```go
func modify(x int) {
    x = 100 // 修改的是副本
}

a := 1
modify(a)
fmt.Println(a) // 1，未被修改
```

### 指针：间接实现"引用效果"

通过传递指针，可以在函数内修改外部变量：

```go
func modify(x *int) {
    *x = 100 // 通过指针修改原变量
}

a := 1
modify(&a)
fmt.Println(a) // 100
```

虽然指针本身也是值传递（复制了一份指针），但由于指针指向同一个地址，效果等同于修改原变量。

### 指针基础

```go
x := 42
p := &x          // & 取地址，p 是指向 x 的指针，类型为 *int
fmt.Println(p)   // 0xc0000b4008（内存地址）
fmt.Println(*p)  // 42（* 解引用，获取指针指向的值）

*p = 100         // 通过指针修改 x 的值
fmt.Println(x)   // 100
```

| 操作 | 语法 | 含义 |
|------|------|------|
| 取地址 | `&x` | 获取变量 x 的内存地址 |
| 解引用 | `*p` | 获取指针 p 指向的值 |
| 指针类型 | `*int` | 指向 int 的指针 |
| 零值 | `nil` | 指针的零值，未指向任何地址 |

### 不同类型的传参行为

虽然Go只有值传递，但不同类型"被复制的东西"不同，导致表现差异很大：

| 类型 | 复制的是什么 | 函数内修改是否影响原数据 |
|------|-------------|----------------------|
| int, float, bool, string | 值本身 | 不影响 |
| 数组 | 整个数组 | 不影响 |
| 切片 | SliceHeader（指针+len+cap） | 修改已有元素影响原数据，append 可能不影响 |
| map | 指针 | 影响 |
| 指针 | 指针值（地址） | 通过解引用影响 |
| struct | 整个结构体 | 不影响（除非字段含引用类型） |

```go
// 切片：修改已有元素影响原数据
func modifySlice(s []int) {
    s[0] = 99 // 影响原切片（共享底层数组）
}

// map：直接影响原数据
func modifyMap(m map[string]int) {
    m["new"] = 1 // 影响原 map
}

// struct：不影响原数据
func modifyStruct(u User) {
    u.Name = "X" // 不影响原 struct
}
```

### 什么时候用指针？

| 场景 | 建议 |
|------|------|
| 需要在函数内修改外部变量 | 用指针 |
| 结构体较大，避免复制开销 | 用指针 |
| 结构体较小且只读 | 用值，更安全 |
| 切片、map 本身 | 不需要指针（内部已是引用） |
| 需要表达"可能为空" | 用指针（nil 表示无值） |

---

## 面试高频题

### Q1：Go 是值传递还是引用传递？

**答**：Go 只有值传递，没有引用传递。所有函数参数都是传入值的副本。但由于切片、map、channel 等类型内部包含指针，复制的是头部结构（指针+元信息），所以函数内修改它们的内容会影响原数据。这不是引用传递——传入的仍然是副本，只是副本中的指针指向了同一块底层数据。

### Q2：下面代码输出什么？

```go
func modify(s []int) {
    s[0] = 99
    s = append(s, 100)
    s[1] = 88
}

func main() {
    s := []int{1, 2, 3}
    modify(s)
    fmt.Println(s)
}
```

**答**：输出 `[99 2 3]`。`s[0] = 99` 修改了共享的底层数组，外部可见。`append` 触发扩容（len=3, cap=3），`s` 指向新的底层数组。之后 `s[1] = 88` 修改的是新数组，对外部不可见。

### Q3：命名返回值有什么好处？有什么注意点？

**答**：三个好处。第一，零值初始化——错误路径不需要手动构造零值返回。第二，文档作用——调用者看函数签名就能理解每个返回值的含义。第三，可以在 defer 中修改返回值，用于统一错误处理和资源清理。注意点是：在长函数中使用裸 return（不带参数的return）会降低可读性，建议长函数或复杂逻辑中显式写出返回值。

### Q4：什么是闭包？闭包捕获的是值还是引用？

**答**：闭包是引用了其外部作用域变量的函数。闭包捕获的是变量的**引用**（准确说是变量本身），不是值的副本。外部变量的修改对闭包可见，闭包对变量的修改对外部也可见。每次创建闭包都会形成独立的环境，不同闭包实例的捕获变量互不影响。

### Q5：下面代码输出什么？

```go
func counter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

func main() {
    a := counter()
    b := counter()
    fmt.Println(a(), a(), a()) // ?
    fmt.Println(b(), b())     // ?
}
```

**答**：输出 `1 2 3` 和 `1 2`。`a` 和 `b` 是两次调用 `counter()` 返回的闭包，各自捕获了独立的 `n` 变量。`a` 的三次调用使 `a` 的 `n` 递增到3，`b` 的两次调用使 `b` 的 `n` 递增到2，互不影响。

### Q6：可变参数和切片参数有什么区别？

```go
func f1(nums ...int)  {}
func f2(nums []int)   {}
```

**答**：在函数内部，两者的 `nums` 都是 `[]int` 切片，使用方式完全一样。区别在调用方式：`f1` 可以传任意数量的 int 参数 `f1(1, 2, 3)`，也可以用 `f1(s...)` 展开切片；`f2` 只能传切片 `f2([]int{1, 2, 3})`。可变参数本质是语法糖，编译器将参数打包成切片。另外，传零个参数时，`f1()` 合法（nums 是 nil 切片），`f2(nil)` 也合法但语义不同。

### Q7：下面的 defer 输出什么？为什么？

```go
func foo() (result int) {
    defer func() {
        result++
    }()
    return 0
}

func main() {
    fmt.Println(foo())
}
```

**答**：输出 `1`。执行过程：`return 0` 先将命名返回值 `result` 赋值为 0，然后执行 defer 函数，defer 中 `result++` 将 `result` 修改为 1，最后函数返回 `result` 的当前值 1。这就是命名返回值配合 defer 的特性——defer 可以在 return 之后、函数真正返回之前修改返回值。

### Q8：函数作为一等公民意味着什么？

**答**：意味着函数和其他类型（int、string）地位相同——可以赋值给变量、作为参数传递、作为返回值、存储在数据结构中。这使得Go支持高阶函数和函数式编程范式。实际应用包括：回调函数、策略模式、中间件链、`sort.Slice` 自定义排序等。Go还支持用 `type` 定义函数类型，使函数签名更清晰。

---

## 小结

| 概念 | 要点 |
|------|------|
| 函数定义 | `func name(params) returns {}`，同类型参数可简写 |
| 可变参数 | `...T` 必须在最后，函数内以切片形式使用 |
| 多返回值 | 用逗号分隔，惯用 `(result, error)` 模式 |
| 命名返回值 | 自动零值初始化、文档作用、defer 中可修改 |
| 匿名函数 | 无名函数，可立即调用或赋值给变量 |
| 高阶函数 | 函数作为参数或返回值 |
| 闭包 | 捕获外部变量的引用，每次创建独立环境 |
| 值传递 | Go 只有值传递，切片/map 因内部含指针表现为"引用效果" |
| 指针 | `&` 取地址、`*` 解引用，用于函数内修改外部变量 |

下一篇将介绍Go的**结构体（Struct）与方法**。
