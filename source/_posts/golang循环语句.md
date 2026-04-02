---
title: Go语言循环语句
top_img: false
tags: golang
abbrlink: e5f7a2b
date: 2026-03-22 14:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言循环语句

Go只有一个循环关键字——`for`。没有 `while`、没有 `do-while`，但通过 `for` 的不同写法可以覆盖所有循环场景。简洁统一，这就是Go的风格。

---

## for 循环的四种写法

### 1. 经典三段式

与C/Java的 `for` 结构一致，由初始化、条件、后置语句三部分组成：

```go
for i := 0; i < 5; i++ {
    fmt.Println(i)
}
// 输出: 0 1 2 3 4
```

与 `if` 一样，条件不需要小括号，花括号必须有。

三个部分都可以省略，但分号不能省：

```go
i := 0
for ; i < 5; i++ {
    fmt.Println(i)
}
```

### 2. while 模式

Go没有 `while` 关键字，但省略初始化和后置语句后，`for` 就是 `while`：

```go
n := 1
for n < 100 {
    n *= 2
}
fmt.Println(n) // 128
```

等价于其他语言的：

```java
// Java
while (n < 100) {
    n *= 2;
}
```

### 3. 无限循环

省略所有部分，`for` 就是无限循环：

```go
for {
    fmt.Println("运行中...")
    // 需要 break 或 return 退出，否则永远执行
}
```

等价于其他语言的 `while(true)`。在服务端编程中很常见，比如持续监听请求、消费消息队列等。

### 4. do-while 模式

Go没有 `do-while`，但可以用无限循环 + 尾部条件判断来模拟——**循环体至少执行一次**：

```go
i := 0
for {
    fmt.Println(i)
    i++
    if i >= 5 {
        break
    }
}
// 输出: 0 1 2 3 4
```

等价于其他语言的：

```java
// Java
do {
    System.out.println(i);
    i++;
} while (i < 5);
```

---

## for range 遍历

`for range` 是Go遍历集合的标准方式，适用于数组、切片、map、字符串和channel。

### 遍历数组和切片

```go
nums := []int{10, 20, 30, 40, 50}

// 同时获取索引和值
for index, value := range nums {
    fmt.Printf("索引: %d, 值: %d\n", index, value)
}

// 只需要索引
for i := range nums {
    fmt.Println(i)
}

// 只需要值，用 _ 忽略索引
for _, v := range nums {
    fmt.Println(v)
}
```

### 遍历 map

```go
m := map[string]int{
    "Go":     1,
    "Rust":   2,
    "Python": 3,
}

for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// 只需要 key
for key := range m {
    fmt.Println(key)
}
```

**注意**：map的遍历顺序是**随机的**，每次运行结果可能不同。如果需要有序遍历，先对key排序：

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
```

### 遍历字符串

`for range` 遍历字符串时，按**UTF-8字符（rune）** 遍历，而非按字节：

```go
s := "Go语言"

// range 按 rune 遍历
for i, ch := range s {
    fmt.Printf("字节位置: %d, 字符: %c\n", i, ch)
}
// 字节位置: 0, 字符: G
// 字节位置: 1, 字符: o
// 字节位置: 2, 字符: 语    （占3个字节，下一个位置是5）
// 字节位置: 5, 字符: 言

// 对比：按字节遍历
for i := 0; i < len(s); i++ {
    fmt.Printf("%d: %x\n", i, s[i]) // 输出原始字节
}
```

索引 `i` 是该字符在字符串中的**字节起始位置**，不是第几个字符。中文字符在UTF-8中占3个字节，所以索引会"跳跃"。

---

## break 和 continue

### break：终止循环

```go
for i := 0; i < 10; i++ {
    if i == 5 {
        break // 立即退出整个 for 循环
    }
    fmt.Println(i)
}
// 输出: 0 1 2 3 4
```

### continue：跳过本次迭代

```go
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue // 跳过偶数，直接进入下一次循环
    }
    fmt.Println(i)
}
// 输出: 1 3 5 7 9
```

### 嵌套循环中的 break：标签（Label）

默认情况下，`break` 和 `continue` 只作用于最内层循环。要跳出外层循环，需要使用**标签**：

```go
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break outer // 直接跳出外层循环
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
// 输出:
// i=0, j=0
// i=0, j=1
// i=0, j=2
// i=1, j=0
```

不使用标签时：

```go
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i == 1 && j == 1 {
            break // 只跳出内层循环，外层继续
        }
        fmt.Printf("i=%d, j=%d\n", i, j)
    }
}
// i=2 的循环仍然会执行
```

`continue` 同样支持标签，跳到外层循环的下一次迭代：

```go
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                continue outer // 跳过外层本次迭代，i++
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
// 输出:
// i=0, j=0
// i=1, j=0
// i=2, j=0
```

---

## for range 的常见陷阱

### 陷阱一：循环变量是副本

`range` 会将每个元素**复制**到循环变量中，修改循环变量不会影响原集合：

```go
type User struct{ Name string }

users := []User{{"Alice"}, {"Bob"}, {"Charlie"}}
for _, u := range users {
    u.Name = "X" // 修改的是副本，无效
}
fmt.Println(users) // [{Alice} {Bob} {Charlie}]
```

正确做法——通过索引修改：

```go
for i := range users {
    users[i].Name = "X" // 直接通过索引修改原切片
}
fmt.Println(users) // [{X} {X} {X}]
```

### 陷阱二：遍历中取地址

Go 1.22之前，循环变量在整个循环中是同一个变量，每次迭代覆盖值。取地址会导致所有指针指向同一个变量：

```go
// Go 1.21及之前的行为
nums := []int{1, 2, 3}
ptrs := make([]*int, 0)
for _, v := range nums {
    ptrs = append(ptrs, &v) // 所有指针指向同一个 v
}
for _, p := range ptrs {
    fmt.Println(*p) // 3 3 3（全是最后一个值）
}
```

**Go 1.22起**，每次迭代的循环变量是独立的，上面的代码会正确输出 `1 2 3`。但为了兼容性和代码清晰度，建议仍然使用局部变量或索引：

```go
for _, v := range nums {
    v := v // 显式创建局部副本（Go 1.22前的惯用做法）
    ptrs = append(ptrs, &v)
}
```

### 陷阱三：遍历 map 时删除元素

Go允许在遍历 map 时删除元素，这是安全的：

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {
    if v == 2 {
        delete(m, k) // 安全
    }
}
```

但在遍历 map 时**新增**元素，新元素可能出现也可能不出现在后续迭代中，行为是不确定的。应避免在遍历中新增。

---

## 性能提示

### 提前获取长度

遍历切片时，`len()` 在每次循环条件判断时都会被调用。虽然对切片来说开销很小（内联优化），但在某些特殊场景下提前取出可读性更好：

```go
// 常规写法，通常足够
for i := 0; i < len(s); i++ { }

// 明确表达"长度不会变"的意图
n := len(s)
for i := 0; i < n; i++ { }
```

### 预分配切片容量

如果在循环中向切片 append，提前分配容量可以避免多次扩容：

```go
// 不推荐：多次扩容
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// 推荐：预分配
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}
```

---

## 面试高频题

### Q1：Go有几种循环？为什么没有 while？

**答**：Go只有 `for` 一种循环关键字，通过不同写法覆盖所有场景：三段式 `for i := 0; i < n; i++`、while模式 `for condition {}`、无限循环 `for {}`、以及 `for range` 遍历集合。不提供 `while` 和 `do-while` 是Go"一种事情只有一种做法"的设计哲学——减少选择成本，降低心智负担。

### Q2：for range 遍历切片时，修改元素能生效吗？

```go
nums := []int{1, 2, 3}
for _, v := range nums {
    v *= 2
}
fmt.Println(nums)
```

**答**：输出 `[1 2 3]`，修改不生效。`range` 将每个元素复制到循环变量 `v` 中，修改 `v` 只是修改副本。要修改原切片，必须用索引：`for i := range nums { nums[i] *= 2 }`。如果切片存储的是指针类型，通过 `v` 修改指向的对象是可以生效的。

### Q3：for range 遍历 map 的顺序是固定的吗？

**答**：不固定。Go的map遍历顺序是**随机的**，这是运行时故意引入的随机化，防止开发者依赖遍历顺序。即使map内容不变，两次遍历的顺序也可能不同。如果需要有序遍历，应先将key取出到切片中排序，再按排序后的key遍历。

### Q4：下面代码输出什么？

```go
s := "Go语言"
count := 0
for range s {
    count++
}
fmt.Println(count)
```

**答**：输出 `4`。`for range` 遍历字符串时按UTF-8字符（rune）遍历，不是按字节。`"Go语言"` 包含4个字符（G、o、语、言），虽然占8个字节（2+3+3），但 `range` 迭代4次。如果用 `len(s)` 得到的是字节数8。

### Q5：break 和 continue 在嵌套循环中的行为？

**答**：默认只作用于最内层循环。`break` 终止最内层循环，`continue` 跳过最内层循环的当前迭代。如果需要控制外层循环，使用标签（label）：在外层循环前定义标签如 `outer:`，然后用 `break outer` 跳出外层循环，或 `continue outer` 跳到外层循环的下一次迭代。

### Q6：Go 1.22 对循环变量做了什么改变？

**答**：Go 1.22之前，`for range` 的循环变量在整个循环中是**同一个变量**，每次迭代覆盖其值。这导致在闭包或取地址时，所有引用都指向最后一次迭代的值，是Go最常见的 bug 之一。Go 1.22起，每次迭代的循环变量是**独立的新变量**，闭包和取地址都能正确捕获当前迭代的值。这是一个不向后兼容的语义变更，通过 `go.mod` 中的Go版本声明来控制生效。

### Q7：在 for range 遍历切片时 append 元素，会发生什么？

```go
s := []int{1, 2, 3}
for _, v := range s {
    s = append(s, v*10)
}
fmt.Println(s)
```

**答**：输出 `[1 2 3 10 20 30]`，只循环3次。`for range` 在开始时就确定了遍历的长度（基于进入循环时的切片长度），循环过程中 append 的元素不会影响迭代次数。这与直接用 `for i := 0; i < len(s); i++` 不同——后者每次检查 `len(s)` 的当前值，会导致无限循环。

### Q8：下面的无限循环怎么安全退出？

```go
for {
    data, err := readFromQueue()
    if err != nil {
        // 怎么处理？
    }
    process(data)
}
```

**答**：常见的退出方式有三种。第一，`break` 直接退出循环。第二，`return` 直接退出函数。第三，配合 `context` 实现优雅退出：

```go
for {
    select {
    case <-ctx.Done():
        return // 收到取消信号，优雅退出
    default:
        data, err := readFromQueue()
        if err != nil {
            log.Println(err)
            continue
        }
        process(data)
    }
}
```

在服务端编程中，推荐使用 `context` 方式，它支持超时控制和级联取消。

---

## 小结

| 概念 | 要点 |
|------|------|
| for 三段式 | `for init; condition; post {}`，唯一的循环关键字 |
| while 模式 | `for condition {}`，省略初始化和后置语句 |
| 无限循环 | `for {}`，配合 break/return/context 退出 |
| do-while 模式 | `for { ... if cond { break } }`，循环体至少执行一次 |
| for range | 遍历数组、切片、map、字符串、channel 的标准方式 |
| 字符串遍历 | `range` 按 rune 遍历，索引是字节位置 |
| break/continue | 默认作用于最内层循环，用标签控制外层循环 |
| 循环变量陷阱 | range 的循环变量是副本；Go 1.22 起每次迭代独立 |
| map 遍历顺序 | 随机的，需要有序则先排序 key |

下一篇将介绍Go的**函数与方法**。
