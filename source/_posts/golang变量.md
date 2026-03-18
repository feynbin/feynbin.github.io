---
title: Go语言变量
top_img: false
tags: golang
abbrlink: a1b2c3d
date: 2026-03-11 10:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言变量

> 本文是Go语言学习系列的第二篇。第一篇[《golang简介》](/posts/d0ea244/)涵盖了语言背景、发展历程与开发环境搭建，建议先行阅读。

## 变量声明

Go是**静态类型语言**，每个变量都有明确的类型。Go提供了多种声明变量的方式，适用于不同场景。

### 1. `var` 关键字声明

最基础的声明方式，显式指定类型：

```go
var name string        // 声明一个string类型变量，零值为 ""
var age int            // 声明一个int类型变量，零值为 0
var isActive bool      // 声明一个bool类型变量，零值为 false
```

声明的同时赋值：

```go
var name string = "Gopher"
var age int = 18
```

Go可以根据右侧的值**自动推断类型**，此时可省略类型声明：

```go
var name = "Gopher"    // 推断为 string
var age = 18           // 推断为 int
var pi = 3.14          // 推断为 float64
```

### 2. 批量声明

当需要同时声明多个变量时，使用 `var()` 块可以让代码更整洁：

```go
var (
    name   string = "Gopher"
    age    int    = 18
    height float64
)
```

### 3. 短变量声明 `:=`

**函数内部**最常用的声明方式，简洁高效：

```go
func main() {
    name := "Gopher"   // 自动推断为 string
    age := 18           // 自动推断为 int
    x, y := 10, 20      // 同时声明多个变量
}
```

> **注意**：`:=` 只能在函数内部使用，不能用于全局变量声明。

### 4. 各方式对比

| 声明方式 | 适用范围 | 示例 | 特点 |
|---------|---------|------|------|
| `var name type` | 全局 + 局部 | `var x int` | 显式类型，零值初始化 |
| `var name = value` | 全局 + 局部 | `var x = 10` | 类型推断 |
| `name := value` | 仅局部 | `x := 10` | 最简洁，函数内首选 |

## 零值（Zero Value）

Go中变量声明后如果不赋值，会被自动初始化为该类型的**零值**，不会出现"未初始化"导致的未定义行为：

| 类型 | 零值 |
|------|------|
| `int`, `float64` 等数值类型 | `0` |
| `bool` | `false` |
| `string` | `""` (空字符串) |
| 指针、切片、map、channel、函数、接口 | `nil` |

```go
var count int      // 0
var label string   // ""
var ok bool        // false
fmt.Println(count, label, ok) // 输出: 0  false
```

## 局部变量与全局变量

### 全局变量

在函数外部、包级别声明的变量。整个包内的所有函数都可以访问：

```go
package main

import "fmt"

// 全局变量 —— 在函数外部声明
var globalCount int = 100
var appName = "MyApp"

func main() {
    fmt.Println(appName, globalCount) // 可以直接使用
    increment()
    fmt.Println(globalCount) // 输出: 101
}

func increment() {
    globalCount++ // 其他函数也可以访问和修改
}
```

**全局变量的特点**：

- 必须使用 `var` 关键字声明（不能用 `:=`）
- 生命周期与程序一致，程序运行期间始终存在
- 首字母大写的全局变量可被其他包访问（导出），小写则为包内私有

### 局部变量

在函数内部声明的变量。只在该函数（或更小的作用域）内有效：

```go
func main() {
    // 局部变量 —— 只在main函数内有效
    localMsg := "hello"
    fmt.Println(localMsg)

    // 更小的作用域：if块内的变量
    if x := 10; x > 5 {
        fmt.Println(x) // x 只在这个if块内有效
    }
    // fmt.Println(x) // 编译错误：x 在此处未定义
}
```

**局部变量的特点**：

- 可使用 `var` 或 `:=` 声明
- 生命周期仅限于所在的作用域（函数、if/for块等）
- 函数执行结束后，局部变量会被垃圾回收

### 同名变量的遮蔽（Shadowing）

当局部变量与全局变量同名时，局部变量会**遮蔽**全局变量：

```go
package main

import "fmt"

var name = "Global"

func main() {
    fmt.Println(name) // 输出: Global

    name := "Local"   // 声明了一个同名的局部变量，遮蔽全局变量
    fmt.Println(name) // 输出: Local
}
```

> **建议**：避免局部变量与全局变量同名，防止产生难以排查的bug。可以使用 `go vet -vettool` 配合 `shadow` 检查器来发现遮蔽问题。

## 完整示例

```go
package main

import "fmt"

// 全局变量
var version = "1.0.0"
var counter int

func main() {
    // 局部变量
    greeting := "Hello, Go!"
    fmt.Println(greeting)

    // 使用全局变量
    fmt.Printf("Version: %s\n", version)

    // 批量声明
    var (
        width  = 100
        height = 200
    )
    fmt.Printf("Area: %d\n", width*height)

    // 调用函数修改全局变量
    addCount()
    addCount()
    fmt.Printf("Counter: %d\n", counter) // 输出: Counter: 2
}

func addCount() {
    counter++
}
```

运行：

```bash
go run main.go
```

输出：

```
Hello, Go!
Version: 1.0.0
Area: 20000
Counter: 2
```

---

## 小结

| 概念 | 要点 |
|------|------|
| 声明方式 | `var`（全局+局部）、`:=`（仅局部） |
| 零值 | 未赋值的变量自动初始化为类型零值 |
| 全局变量 | 包级作用域，`var` 声明，首字母大小写控制可见性 |
| 局部变量 | 函数/块级作用域，推荐使用 `:=` |
| 遮蔽 | 同名局部变量会遮蔽全局变量，应尽量避免 |

下一篇将介绍Go的**常量与iota**以及**基本数据类型**。
