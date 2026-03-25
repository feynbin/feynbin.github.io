---
title: Go语言泛型
top_img: false
tags: golang
abbrlink: a6c0d4e
date: 2026-03-23 20:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言泛型

> 本文是Go语言学习系列的第十三篇。前十二篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)、[《Go语言函数》](/posts/f6a8b3c/)、[《Go语言init与defer》](/posts/a7c9d4e/)、[《Go语言结构体》](/posts/b8d1e5f/)、[《Go语言自定义类型与接口》](/posts/c9e2f6a/)、[《Go语言协程与信道》](/posts/d3f4a7b/)、[《Go语言线程安全与sync.Map》](/posts/e4a8b2c/)、[《Go语言错误与异常处理》](/posts/f5b9c3d/)，建议先行阅读。

泛型（Generics）是Go 1.18引入的重大特性。在此之前，处理多种类型要么用 `interface{}` 丢失类型安全，要么为每种类型复制一份代码。泛型让我们可以编写**一份代码处理多种类型，同时保留编译期类型检查**。

---

## 泛型函数

### 基本语法

在函数名后用方括号声明**类型参数（Type Parameter）**：

```go
// T 是类型参数，any 是类型约束（允许任意类型）
func Print[T any](value T) {
    fmt.Println(value)
}

func main() {
    Print[int](42)       // 显式指定类型
    Print("hello")       // 编译器自动推断类型为string
    Print(3.14)          // 推断为float64
}
```

### 多类型参数

```go
func Map[T any, R any](slice []T, fn func(T) R) []R {
    result := make([]R, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4}
    strs := Map(nums, func(n int) string {
        return fmt.Sprintf("No.%d", n)
    })
    fmt.Println(strs) // [No.1 No.2 No.3 No.4]
}
```

### 类型约束：限制类型范围

`any` 允许所有类型，但如果函数需要进行比较、运算等操作，就需要更具体的约束：

```go
// comparable：支持 == 和 != 的类型
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

func main() {
    fmt.Println(Contains([]int{1, 2, 3}, 2))         // true
    fmt.Println(Contains([]string{"a", "b"}, "c"))    // false
}
```

---

## 用接口定义类型约束

Go的泛型约束就是接口，但扩展了接口的语法，支持**类型集合（Type Set）**：

### 基本约束接口

```go
// 约束：必须支持加法和排序
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(Sum([]int{1, 2, 3}))         // 6
    fmt.Println(Sum([]float64{1.1, 2.2}))    // 3.3
}
```

### ~ 符号：包含底层类型

`~int` 表示底层类型是int的所有类型，包括自定义类型：

```go
type Age int   // 底层类型是int
type Score int

// 没有 ~：只匹配 int 本身
type StrictInt interface { int }

// 有 ~：匹配所有底层类型为 int 的类型
type FlexInt interface { ~int }

func Double[T FlexInt](v T) T {
    return v * 2
}

func main() {
    var age Age = 18
    fmt.Println(Double(age)) // 36，如果约束是 int 而不是 ~int，这里会编译失败
}
```

### 组合约束：方法 + 类型

接口约束可以同时要求方法和类型：

```go
type Stringer interface {
    ~int | ~string
    String() string
}
```

这要求类型的底层类型是int或string，**同时**实现了 `String()` 方法。

### 标准库约束：golang.org/x/exp/constraints

Go扩展库提供了常用约束（Go 1.21后部分已移入标准库 `cmp` 包）：

```go
import "golang.org/x/exp/constraints"

// constraints.Ordered：支持 < > <= >= 的类型
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// cmp.Ordered（Go 1.21+标准库）
import "cmp"

func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

---

## 泛型结构体

### 基本定义

```go
type Pair[T any, U any] struct {
    First  T
    Second U
}

func main() {
    p1 := Pair[string, int]{First: "age", Second: 25}
    p2 := Pair[string, float64]{First: "score", Second: 98.5}
    fmt.Println(p1, p2)
}
```

### 泛型结构体的方法

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Len() int {
    return len(s.items)
}

func main() {
    s := &Stack[int]{}
    s.Push(1)
    s.Push(2)
    val, _ := s.Pop()
    fmt.Println(val) // 2
}
```

> **注意**：方法的接收者声明中不能添加新的类型约束，只能使用结构体定义时的类型参数。即不能写 `func (s *Stack[T Number]) Sum()`。

### 实际案例：泛型JSON反序列化

```go
type ApiResponse[T any] struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    T      `json:"data"`
}

type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

type Product struct {
    ID    int     `json:"id"`
    Price float64 `json:"price"`
}

// 泛型反序列化函数
func ParseResponse[T any](jsonStr []byte) (*ApiResponse[T], error) {
    var resp ApiResponse[T]
    if err := json.Unmarshal(jsonStr, &resp); err != nil {
        return nil, fmt.Errorf("JSON解析失败: %w", err)
    }
    return &resp, nil
}

func main() {
    userJSON := []byte(`{"code":200,"message":"ok","data":{"name":"张三","email":"zhang@example.com"}}`)
    productJSON := []byte(`{"code":200,"message":"ok","data":{"id":1,"price":99.9}}`)

    // 同一个函数，解析不同的Data类型
    userResp, _ := ParseResponse[User](userJSON)
    fmt.Println(userResp.Data.Name) // 张三

    productResp, _ := ParseResponse[Product](productJSON)
    fmt.Println(productResp.Data.Price) // 99.9
}
```

没有泛型时，Data字段只能定义为 `interface{}`，取值时需要类型断言，既不安全也不方便。泛型让API响应的类型在编译期就确定了。

---

## 泛型切片

为切片类型添加泛型方法：

```go
type List[T any] []T

func (l List[T]) Filter(fn func(T) bool) List[T] {
    var result List[T]
    for _, v := range l {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

func (l List[T]) Each(fn func(T)) {
    for _, v := range l {
        fn(v)
    }
}

func main() {
    nums := List[int]{1, 2, 3, 4, 5, 6}

    evens := nums.Filter(func(n int) bool {
        return n%2 == 0
    })

    evens.Each(func(n int) {
        fmt.Println(n) // 2 4 6
    })
}
```

### 泛型工具函数

```go
// 去重
func Unique[T comparable](slice []T) []T {
    seen := make(map[T]bool)
    var result []T
    for _, v := range slice {
        if !seen[v] {
            seen[v] = true
            result = append(result, v)
        }
    }
    return result
}

// 分组
func GroupBy[T any, K comparable](slice []T, keyFn func(T) K) map[K][]T {
    result := make(map[K][]T)
    for _, v := range slice {
        key := keyFn(v)
        result[key] = append(result[key], v)
    }
    return result
}

func main() {
    fmt.Println(Unique([]int{1, 2, 2, 3, 3})) // [1 2 3]

    users := []User{{Name: "A", Age: 20}, {Name: "B", Age: 20}, {Name: "C", Age: 30}}
    groups := GroupBy(users, func(u User) int { return u.Age })
    // map[20:[{A 20} {B 20}] 30:[{C 30}]]
}
```

---

## 泛型Map

自定义Map类型，封装常用操作：

```go
type Map[K comparable, V any] map[K]V

func (m Map[K, V]) Keys() []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func (m Map[K, V]) Values() []V {
    values := make([]V, 0, len(m))
    for _, v := range m {
        values = append(values, v)
    }
    return values
}

func (m Map[K, V]) Filter(fn func(K, V) bool) Map[K, V] {
    result := make(Map[K, V])
    for k, v := range m {
        if fn(k, v) {
            result[k] = v
        }
    }
    return result
}

func main() {
    scores := Map[string, int]{
        "Alice": 90,
        "Bob":   60,
        "Carol": 85,
    }

    passed := scores.Filter(func(name string, score int) bool {
        return score >= 80
    })
    fmt.Println(passed.Keys()) // [Alice Carol]
}
```

---

## 泛型的限制

当前Go泛型仍有一些限制需要了解：

| 限制 | 说明 |
|------|------|
| 方法不能有类型参数 | `func (s *Stack[T]) Convert[U any]()` 不合法 |
| 无法用类型参数做类型断言 | `v.(T)` 不合法，T是类型参数时不能断言 |
| 无运算符约束 | 不能约束"必须支持 `+` 运算符"（只能枚举类型） |
| 无协变/逆变 | `List[Cat]` 不能赋值给 `List[Animal]` |
| 类型推断有限 | 某些场景需要显式指定类型参数 |

---

## 什么时候该用泛型

| 场景 | 是否用泛型 | 理由 |
|------|-----------|------|
| 通用数据结构（栈、队列、树） | ✅ 用 | 逻辑与类型无关 |
| 通用算法（排序、过滤、去重） | ✅ 用 | 避免为每种类型重复代码 |
| API响应包装 | ✅ 用 | Data字段类型各异 |
| 只有2-3种类型 | ❌ 不用 | 直接写具体类型更清晰 |
| 操作依赖具体类型行为 | ❌ 不用 | 用接口（方法约束）更合适 |
| 追求极致性能 | ⚠️ 谨慎 | 泛型可能导致单态化膨胀 |

Go官方建议：**不要为了泛型而泛型**。如果用 `interface` 和具体类型可以简洁地解决问题，就不需要泛型。

---

## 面试题精选

### 基础题

**Q1：Go泛型是什么时候引入的？解决了什么问题？**

> Go 1.18（2022年3月）引入。解决的核心问题是**类型安全的代码复用**。之前要么用 `interface{}` 丢失编译期类型检查（运行时才发现类型错误），要么为每种类型复制粘贴代码（违反DRY原则）。泛型让一份代码处理多种类型的同时保留编译期类型安全。

**Q2：any和comparable分别是什么约束？**

> `any` 是 `interface{}` 的别名，允许任意类型，但不能对值做任何操作（不能比较、不能运算）。`comparable` 是内置约束，表示支持 `==` 和 `!=` 的类型（所有基本类型、指针、channel、数组等，不包括slice、map、函数）。当泛型函数需要将值作为map的key或进行相等比较时，必须使用 `comparable`。

**Q3：`~int` 和 `int` 在类型约束中有什么区别？**

> `int` 只匹配 `int` 类型本身。`~int` 匹配所有**底层类型（underlying type）**为 `int` 的类型，包括 `type Age int`、`type Score int` 等自定义类型。实际开发中几乎总是应该用 `~int` 而非 `int`，否则自定义类型无法使用泛型函数。

### 进阶题

**Q4：以下代码为什么编译失败？如何修复？**

```go
func Add[T any](a, b T) T {
    return a + b
}
```

> 编译失败因为 `any` 约束不保证类型支持 `+` 运算符。Go泛型没有"运算符约束"，只能通过接口枚举支持加法的类型。修复：

```go
type Addable interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64 | ~string
}

func Add[T Addable](a, b T) T {
    return a + b
}
```

**Q5：泛型函数和接口多态有什么区别？分别在什么时候使用？**

> **泛型**在编译期确定具体类型（静态分发），性能好，适合"逻辑相同、类型不同"的场景（数据结构、算法）。**接口多态**通过运行时虚表分发（动态分发），适合"行为不同、调用方式相同"的场景（策略模式、依赖注入）。简单判断：如果不同类型的实现逻辑完全一样，用泛型；如果不同类型有各自的行为实现，用接口。

**Q6：为什么Go泛型的方法不能有额外的类型参数？**

```go
// 不合法
func (s *Stack[T]) Map[U any](fn func(T) U) *Stack[U] { ... }
```

> 这是Go团队有意的限制。允许方法级类型参数会使接口的实现变得极其复杂——接口定义中无法表达"实现此方法的所有可能类型参数组合"。Go选择简单性而非完备性。替代方案是用顶层泛型函数：`func MapStack[T any, U any](s *Stack[T], fn func(T) U) *Stack[U]`。

### 高级题

**Q7：Go泛型的实现方式是什么？和Java、C++的泛型有什么区别？**

> Go采用**GCShape stenciling + 字典**的混合方案：相同GCShape（内存布局）的类型共享一份代码，通过字典传递类型信息。C++模板是纯**单态化（monomorphization）**——每种类型生成一份独立代码，编译慢但运行最快。Java泛型是**类型擦除（type erasure）**——编译后所有泛型类型变成Object，运行时无类型信息。Go的方案在编译速度和运行性能之间取了折中。

**Q8：实现一个泛型缓存，支持任意key-value类型，带过期时间。**

```go
type CacheItem[V any] struct {
    Value     V
    ExpiresAt time.Time
}

type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]CacheItem[V]
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{
        items: make(map[K]CacheItem[V]),
    }
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = CacheItem[V]{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    if !ok || time.Now().After(item.ExpiresAt) {
        var zero V
        return zero, false
    }
    return item.Value, true
}

// 使用
func main() {
    cache := NewCache[string, User]()
    cache.Set("user:1", User{Name: "张三"}, 5*time.Minute)

    if user, ok := cache.Get("user:1"); ok {
        fmt.Println(user.Name)
    }
}
```

> 关键点：泛型让key和value类型在编译期确定，取值时无需类型断言；结合 `sync.RWMutex` 保证并发安全。

**Q9：泛型会影响性能吗？什么情况下需要注意？**

> Go的GCShape stenciling方案中，所有指针类型共享一份代码（因为内存布局相同），值类型按GCShape分组。这意味着：（1）指针类型的泛型函数只编译一份，性能与非泛型接口调用接近；（2）值类型（int、struct等）可能生成多份代码，但每份都是直接操作具体类型，无间接调用开销。实际影响通常很小，但在热路径上如果发现泛型版本比具体类型版本慢，可以考虑为关键类型手写特化实现。

---

## 小结

| 概念 | 语法 | 说明 |
|------|------|------|
| 泛型函数 | `func Name[T constraint](...)` | 一份函数处理多种类型 |
| 类型约束 | `interface { ~int \| ~string }` | 限制类型参数的范围 |
| 泛型结构体 | `type Name[T any] struct{...}` | 通用数据结构 |
| 泛型切片 | `type List[T any] []T` | 带方法的类型安全切片 |
| 泛型Map | `type Map[K comparable, V any] map[K]V` | 带方法的类型安全Map |

泛型的使用原则：**当你发现自己在为不同类型复制粘贴相同逻辑时，考虑用泛型**。但不要过度——简单场景下具体类型比泛型更清晰。
