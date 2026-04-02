---
title: Go语言数组、切片与Map
top_img: false
tags: golang
abbrlink: b3f5e78
date: 2026-03-19 11:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言数组、切片与Map

Go语言中最常用的三种集合类型：**数组（Array）** 是固定长度的值类型，**切片（Slice）** 是基于数组的动态引用类型，**Map** 是键值对的哈希表。日常开发中切片和Map使用频率远高于数组。

---

## 数组（Array）

数组是**固定长度**、**同一类型**元素的集合。长度是类型的一部分——`[3]int` 和 `[5]int` 是不同的类型。

### 声明与初始化

```go
// 声明后自动初始化为零值
var a [3]int              // [0, 0, 0]

// 声明并初始化
b := [3]int{1, 2, 3}

// 让编译器根据元素个数推断长度
c := [...]int{1, 2, 3, 4} // 长度为4

// 指定索引位置初始化
d := [5]int{1: 10, 3: 30} // [0, 10, 0, 30, 0]
```

### 数组与索引：为什么随机访问是O(1)

数组在内存中是一段**连续的**、等大小的空间。正因为连续且等大小，CPU可以通过一个公式直接算出任意元素的地址，无需遍历：

```
元素地址 = 首地址 + 索引 × 元素大小
```

以 `[5]int32{10, 20, 30, 40, 50}` 为例，假设首地址为 `0x100`，`int32` 占4字节：

```
索引    计算方式              地址      值
 0     0x100 + 0×4 = 0x100   0x100    10
 1     0x100 + 1×4 = 0x104   0x104    20
 2     0x100 + 2×4 = 0x108   0x108    30
 3     0x100 + 3×4 = 0x10C   0x10C    40
 4     0x100 + 4×4 = 0x110   0x110    50
```

不管访问第1个还是第10000个元素，都只需要一次乘法和一次加法，时间复杂度恒为 **O(1)**。这也是数组相对于链表最核心的优势。

> **延伸**：切片的随机访问也是O(1)，因为切片底层就是数组。`s[i]` 实际上是 `底层数组指针 + i × 元素大小`。

**索引从0开始的原因**：如果从0开始，公式是 `首地址 + i × 大小`；如果从1开始，就变成 `首地址 + (i-1) × 大小`，每次多一次减法。从0开始让偏移量计算更直接——索引本质上就是相对于首地址的**偏移量**。

### 数组是值类型

这是Go数组与大多数语言最关键的区别——**赋值和传参都会复制整个数组**：

```go
a := [3]int{1, 2, 3}
b := a        // 完整复制，b是独立副本
b[0] = 99
fmt.Println(a) // [1 2 3]  —— a不受影响
fmt.Println(b) // [99 2 3]
```

```go
func modify(arr [3]int) {
    arr[0] = 99 // 修改的是副本，不影响原数组
}

a := [3]int{1, 2, 3}
modify(a)
fmt.Println(a) // [1 2 3]
```

> **实际开发中很少直接使用数组**，因为固定长度不够灵活，值传递在大数组时有性能开销。绝大多数场景使用切片。

### 遍历

```go
arr := [3]string{"Go", "Rust", "Python"}

// 方式一：索引遍历
for i := 0; i < len(arr); i++ {
    fmt.Println(i, arr[i])
}

// 方式二：range（推荐）
for index, value := range arr {
    fmt.Println(index, value)
}

// 只需要值，忽略索引
for _, value := range arr {
    fmt.Println(value)
}
```

---

## 切片（Slice）

切片是Go中最常用的数据结构之一。它是对底层数组的一个**引用视图**，具有动态长度。

### 底层结构

理解切片的关键在于其运行时结构（`reflect.SliceHeader`）：

```go
type SliceHeader struct {
    Data uintptr  // 指向底层数组的指针
    Len  int      // 当前元素个数
    Cap  int      // 底层数组从Data开始的容量
}
```

一个切片由三部分组成：**指针**（指向底层数组某个位置）、**长度**（len）、**容量**（cap）。

### 创建切片

```go
// 1. 从数组或切片截取
arr := [5]int{1, 2, 3, 4, 5}
s1 := arr[1:4]    // [2, 3, 4]  len=3, cap=4

// 2. 字面量（最常用）
s2 := []int{1, 2, 3}

// 3. make：指定长度和容量
s3 := make([]int, 3)       // len=3, cap=3, 元素为零值 [0,0,0]
s4 := make([]int, 3, 10)   // len=3, cap=10

// 4. 声明nil切片
var s5 []int               // nil切片, len=0, cap=0
```

### 切片截取的细节

从数组或切片截取时，新切片与原数据**共享底层数组**：

```go
arr := [5]int{1, 2, 3, 4, 5}
s := arr[1:3] // [2, 3]

s[0] = 99
fmt.Println(arr) // [1, 99, 3, 4, 5] —— 原数组被修改了
```

可通过三索引切片限制容量，避免意外修改：

```go
s := arr[1:3:3] // [2, 3]  len=2, cap=2（容量被限制）
```

### append与扩容

`append` 向切片追加元素。当容量不足时，Go会分配新的底层数组：

```go
s := make([]int, 0, 3)
s = append(s, 1, 2, 3) // len=3, cap=3，未扩容
s = append(s, 4)        // len=4, cap=6，扩容了，底层数组已更换
```

**扩容规则**（Go 1.18+ 使用平滑增长策略）：

| 当前容量 | 扩容行为 |
|---------|---------|
| < 256 | 约2倍扩容 |
| >= 256 | 按 `newcap += (newcap + 3*256) / 4` 平滑增长，增长因子逐渐从2倍趋向1.25倍 |

> 最终容量还会进行内存对齐调整，实际分配可能略大于计算值。

**关键点**：`append` 可能返回新的切片头，必须用返回值接收：

```go
s = append(s, elem) // 正确
append(s, elem)     // 错误：返回值被丢弃，原s可能未更新
```

### copy

`copy` 在两个切片之间复制元素，返回实际复制的个数（取两者长度的较小值）：

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)
n := copy(dst, src) // n=3, dst=[1, 2, 3]
```

当需要一份完全独立的副本时：

```go
original := []int{1, 2, 3}
clone := make([]int, len(original))
copy(clone, original)
// 或者使用 append 的惯用写法：
clone2 := append([]int(nil), original...)
```

### nil切片与空切片

```go
var s1 []int           // nil切片：s1 == nil, len=0, cap=0
s2 := []int{}          // 空切片：s2 != nil, len=0, cap=0
s3 := make([]int, 0)   // 空切片：s3 != nil, len=0, cap=0
```

三者对 `len`、`cap`、`append`、`range` 的行为完全一致，区别仅在于 `== nil` 的判断。JSON序列化时，nil切片编码为 `null`，空切片编码为 `[]`。

---

## Map

Map是Go的内置哈希表实现，存储无序的键值对。

### 创建与基本操作

```go
// 1. make创建（推荐）
m := make(map[string]int)

// 2. 字面量
m := map[string]int{
    "Go":     1,
    "Rust":   2,
    "Python": 3,
}

// 3. 声明nil map（只读，写入会panic）
var m map[string]int // nil map
```

**增删改查**：

```go
m := make(map[string]int)

// 增 / 改
m["Go"] = 1
m["Go"] = 2 // 覆盖

// 查
val := m["Go"]       // 存在则返回值，不存在返回零值

// 推荐写法：comma ok模式
val, ok := m["Rust"]
if !ok {
    fmt.Println("key不存在")
}

// 删
delete(m, "Go") // key不存在时不会panic
```

### 遍历

Map的遍历顺序是**随机的**，每次运行结果可能不同（这是Go故意为之，防止依赖遍历顺序）：

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

for key, value := range m {
    fmt.Println(key, value)
}
```

如果需要有序遍历，先对key排序：

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### key的要求

Map的key必须是**可比较的类型**（支持 `==` 运算符）：

| 可以作为key | 不可以作为key |
|------------|-------------|
| int、string、float、bool | slice |
| 数组（固定长度） | map |
| 指针、channel | 函数 |
| 只含可比较字段的struct | 含slice/map字段的struct |

> **注意**：float作为key虽然合法，但由于精度问题应避免使用。

### Map并发不安全

Map在多个goroutine同时读写时会直接panic（`concurrent map read and map write`），不是数据错乱，是直接崩溃。

```go
// 错误示例：并发读写会panic
m := make(map[int]int)
go func() {
    for i := 0; i < 1000; i++ {
        m[i] = i
    }
}()
go func() {
    for i := 0; i < 1000; i++ {
        _ = m[i]
    }
}()
```

**解决方案**：

```go
// 方案一：sync.RWMutex
var mu sync.RWMutex
mu.Lock()
m[key] = value
mu.Unlock()

mu.RLock()
v := m[key]
mu.RUnlock()

// 方案二：sync.Map（适合读多写少场景）
var sm sync.Map
sm.Store("key", "value")
val, ok := sm.Load("key")
sm.Delete("key")
```

| 方案 | 适用场景 | 特点 |
|-----|---------|------|
| `sync.RWMutex` + `map` | 通用场景 | 灵活，读写比例均衡时性能好 |
| `sync.Map` | 读多写少、key稳定 | 无需初始化，读操作几乎无锁 |

---

## 三者对比

| 特性 | 数组 | 切片 | Map |
|-----|------|------|-----|
| 长度 | 固定，编译期确定 | 动态，运行时可变 | 动态 |
| 类型性质 | 值类型 | 引用类型（底层指针） | 引用类型（底层指针） |
| 传参行为 | 完整复制 | 复制切片头（共享底层数组） | 复制map指针（共享数据） |
| 零值 | 各元素为类型零值 | `nil` | `nil` |
| 比较 | 可用 `==` 比较 | 不可比较（只能与nil比） | 不可比较（只能与nil比） |

---

## 面试高频题

### Q1：数组和切片有什么区别？

**答**：核心区别有三点。第一，数组长度固定且是类型的一部分，`[3]int` 和 `[5]int` 是不同类型；切片长度动态可变。第二，数组是值类型，赋值和传参会复制整个数组；切片本质是一个包含指针、长度、容量的结构体，赋值和传参只复制这个结构体头，底层数组是共享的。第三，实际开发中几乎不直接使用数组，切片是绝对主力。

### Q2：切片的扩容机制是怎样的？

**答**：当 `append` 时容量不足，Go会分配新的底层数组。Go 1.18之前规则是：长度小于1024时2倍扩容，大于等于1024时1.25倍。Go 1.18之后改为平滑增长策略：容量小于256时约2倍，大于等于256时按公式 `newcap += (newcap + 3*256) / 4` 逐渐从2倍过渡到1.25倍，避免了在1024处的跳变。最终容量还会根据内存对齐进行调整。

### Q3：下面代码输出什么？为什么？

```go
func main() {
    s := []int{1, 2, 3}
    modify(s)
    fmt.Println(s)
}

func modify(s []int) {
    s[0] = 99
    s = append(s, 4)
    s[1] = 88
}
```

**答**：输出 `[99 2 3]`。`s[0] = 99` 修改的是共享的底层数组，所以外部可见。`append` 之后发生了扩容（len=3, cap=3，追加触发扩容），`s` 指向了新的底层数组，之后的 `s[1] = 88` 修改的是新数组，对外部的 `s` 不可见。

### Q4：nil切片和空切片有什么区别？

**答**：`var s []int`（nil切片）和 `s := []int{}`（空切片）在 `len`、`cap`、`for range`、`append` 等操作上行为完全一致。区别在于：nil切片底层指针为nil，`s == nil` 为true；空切片底层指针非nil，`s == nil` 为false。在JSON序列化时，nil切片输出 `null`，空切片输出 `[]`。一般建议：如果表示"还没有数据"用nil切片，如果表示"有数据但恰好是空的"用空切片。

### Q5：Map的key可以是哪些类型？为什么slice不能做key？

**答**：Map的key必须是可比较类型，即支持 `==` 和 `!=` 操作。可用的类型包括：int、string、bool、float、数组、指针、channel、以及所有字段都可比较的struct。slice、map、函数不能做key，因为Go没有为它们定义相等性比较——slice和map的内容可变，没有稳定的哈希值，强行比较语义不明确。

### Q6：Map为什么并发不安全？怎么解决？

**答**：Go的map在运行时会检测并发读写，一旦检测到直接panic，而不是返回错误数据。这是故意为之的设计——大多数场景不需要并发安全的map，加锁会带来不必要的性能开销。需要并发安全时有两种方案：一是 `sync.RWMutex` 配合普通map，适合通用场景；二是 `sync.Map`，内部使用了读写分离+原子操作的优化，适合读多写少、key相对稳定的场景。

### Q7：用range遍历切片时修改元素，能生效吗？

```go
type User struct{ Name string }

users := []User{{"A"}, {"B"}, {"C"}}
for _, u := range users {
    u.Name = "X"
}
fmt.Println(users)
```

**答**：输出 `[{A} {B} {C}]`，修改不生效。因为 `range` 会将每个元素**复制**到循环变量 `u` 中，修改 `u` 不影响原切片。要修改原切片，应使用索引：

```go
for i := range users {
    users[i].Name = "X" // 直接通过索引修改
}
```

如果切片存储的是指针 `[]*User`，则通过 `u.Name = "X"` 可以生效，因为复制的是指针，指向同一个对象。

---

## 小结

| 概念 | 要点 |
|------|------|
| 数组 | 固定长度、值类型、赋值即复制、实际开发很少直接使用 |
| 切片 | 动态长度、引用底层数组、三要素（指针+len+cap）、append可能扩容 |
| 扩容 | <256约2倍，>=256平滑增长趋向1.25倍，最终内存对齐 |
| Map | 无序键值对、key必须可比较、并发不安全、nil map写入会panic |
| 并发Map | `sync.RWMutex` + map（通用）或 `sync.Map`（读多写少） |

下一篇将介绍Go的**结构体（Struct）与方法**。
