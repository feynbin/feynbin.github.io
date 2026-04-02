---
title: Go语言线程安全与sync.Map
top_img: false
tags: golang
abbrlink: e4a8b2c
date: 2026-03-23 16:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言线程安全与sync.Map

当多个goroutine同时读写共享数据时，如果不加保护就会产生**数据竞争（Data Race）**，导致结果不可预期甚至程序崩溃。Go通过 `sync` 包提供了互斥锁、读写锁和并发安全的Map等工具来解决这个问题。

---

## 为什么需要线程安全

先看一个不安全的例子：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // 多个goroutine同时读写，存在数据竞争
        }()
    }

    wg.Wait()
    fmt.Println("counter:", counter) // 结果不是1000，每次运行可能不同
}
```

`counter++` 不是原子操作，它包含三步：读取值、加1、写回值。多个goroutine并发执行时，可能出现：

```
goroutine A 读取 counter = 5
goroutine B 读取 counter = 5   ← 读到了过期值
goroutine A 写入 counter = 6
goroutine B 写入 counter = 6   ← 覆盖了A的结果，丢失一次操作
```

> 使用 `go run -race main.go` 可以开启竞态检测器，程序运行时会报告数据竞争。

---

## sync.Mutex：互斥锁

`sync.Mutex` 是最基本的锁，保证同一时刻只有一个goroutine能访问临界区：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()   // 加锁：其他goroutine到这里会阻塞等待
            counter++
            mu.Unlock() // 解锁：允许下一个goroutine进入
        }()
    }

    wg.Wait()
    fmt.Println("counter:", counter) // 始终输出1000
}
```

### 使用defer确保解锁

如果临界区内的代码可能panic或有多个return路径，推荐用 `defer` 解锁：

```go
mu.Lock()
defer mu.Unlock()
// 即使这里panic，锁也会被释放
doSomething()
```

### 用结构体封装锁

实际项目中，通常将锁和它保护的数据封装在一起：

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

> **易错点**：`sync.Mutex` 不可复制。如果结构体包含Mutex，方法接收者必须用指针，传参也必须传指针。

---

## sync.RWMutex：读写锁

`sync.Mutex` 不区分读写——即使多个goroutine只是读取数据，也要互相等待。`sync.RWMutex` 解决了这个问题：

| 操作 | 方法 | 并发规则 |
|------|------|----------|
| 读锁 | `RLock()` / `RUnlock()` | 多个读锁可以共存 |
| 写锁 | `Lock()` / `Unlock()` | 写锁独占，与所有读锁/写锁互斥 |

```go
type SafeCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()         // 读锁：允许并发读
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()          // 写锁：独占访问
    defer c.mu.Unlock()
    c.data[key] = value
}
```

**适用场景**：读多写少的情况下，RWMutex比Mutex性能更好。如果读写频率接近，RWMutex的额外开销反而可能更慢。

---

## 为什么普通Map不是线程安全的

Go的内建 `map` 不支持并发读写，并发操作会直接panic：

```go
func main() {
    m := make(map[int]int)
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            m[n] = n // fatal error: concurrent map writes
        }(i)
    }

    wg.Wait()
}
```

这不是数据竞争的"可能出错"，而是Go运行时**主动检测并panic**，因为并发写map会破坏其内部数据结构。

### 方案一：Mutex + Map

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Store(key string, value int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}

func (s *SafeMap) Load(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    val, ok := s.m[key]
    return val, ok
}

func (s *SafeMap) Delete(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.m, key)
}
```

### 方案二：sync.Map（标准库提供）

---

## sync.Map

`sync.Map` 是Go标准库提供的并发安全Map，无需额外加锁：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var m sync.Map

    // Store：存储键值对
    m.Store("name", "Go")
    m.Store("version", 1.22)

    // Load：读取值
    val, ok := m.Load("name")
    if ok {
        fmt.Println("name:", val) // name: Go
    }

    // LoadOrStore：存在则返回已有值，不存在则存入
    actual, loaded := m.LoadOrStore("name", "Rust")
    fmt.Println(actual, loaded) // Go true（已存在，返回旧值）

    actual, loaded = m.LoadOrStore("lang", "Go")
    fmt.Println(actual, loaded) // Go false（不存在，存入新值）

    // Delete：删除键值对
    m.Delete("version")

    // Range：遍历所有键值对
    m.Store("a", 1)
    m.Store("b", 2)
    m.Range(func(key, value any) bool {
        fmt.Printf("%v: %v\n", key, value)
        return true // 返回false可提前终止遍历
    })
}
```

### sync.Map的完整API

| 方法 | 签名 | 说明 |
|------|------|------|
| `Store` | `Store(key, value any)` | 存储键值对 |
| `Load` | `Load(key any) (value any, ok bool)` | 读取，ok表示是否存在 |
| `Delete` | `Delete(key any)` | 删除键值对 |
| `LoadOrStore` | `LoadOrStore(key, value any) (actual any, loaded bool)` | 存在返回旧值，不存在则存入 |
| `LoadAndDelete` | `LoadAndDelete(key any) (value any, loaded bool)` | 读取并删除 |
| `Range` | `Range(f func(key, value any) bool)` | 遍历，回调返回false停止 |
| `CompareAndSwap` | `CompareAndSwap(key, old, new any) (swapped bool)` | 原子比较并交换（Go 1.20+） |
| `CompareAndDelete` | `CompareAndDelete(key, old any) (deleted bool)` | 原子比较并删除（Go 1.20+） |
| `Swap` | `Swap(key, value any) (previous any, loaded bool)` | 原子交换值（Go 1.20+） |

### sync.Map的实现原理

`sync.Map` 内部维护两个数据结构：

```
sync.Map
├── read  (atomic.Value → readOnly map)  // 无锁读取，用atomic保证可见性
└── dirty (map + Mutex)                   // 需要加锁访问
```

- **read map**：只读的map，读取时不需要加锁，通过 `atomic.Value` 实现无锁访问
- **dirty map**：包含所有数据（read中的 + 新写入的），读写需要加Mutex锁

工作流程：
1. **Load**：先查read map（无锁），找到直接返回；未找到再加锁查dirty map
2. **Store**：如果key已在read map中，直接原子更新；否则加锁写入dirty map
3. **提升（promote）**：当read未命中次数达到阈值（dirty的长度），dirty会被提升为新的read map

> 本质上就是一种**读写分离 + 延迟提升**的策略，底层仍然依赖Mutex，但通过减少锁的使用频率来提升读性能。

### sync.Map vs Mutex+Map：如何选择

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 读多写少，key相对稳定 | `sync.Map` | read map命中率高，几乎无锁 |
| 读写频率相当 | `Mutex + Map` | sync.Map的两层结构反而有额外开销 |
| 需要已知类型的Map | `Mutex + Map` | sync.Map的key和value都是 `any`，缺少类型安全 |
| key不断新增删除 | `Mutex + Map` | dirty频繁提升，sync.Map优势消失 |
| 多个goroutine操作不同的key | `sync.Map` | 官方文档推荐的典型场景 |

---

## sync.Once：只执行一次

顺带介绍 `sync.Once`，它保证某个操作在并发环境下只执行一次，常用于单例初始化：

```go
var (
    instance *Database
    once     sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        instance = &Database{} // 无论多少goroutine调用，只执行一次
        instance.Connect("localhost:3306")
    })
    return instance
}
```

---

## 原子操作：sync/atomic

对于简单的数值操作，`sync/atomic` 包比Mutex更高效：

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    var counter int64
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1) // 原子加1，无需锁
        }()
    }

    wg.Wait()
    fmt.Println("counter:", atomic.LoadInt64(&counter)) // 1000
}
```

常用原子操作：

| 函数 | 说明 |
|------|------|
| `atomic.AddInt64(&v, delta)` | 原子加 |
| `atomic.LoadInt64(&v)` | 原子读 |
| `atomic.StoreInt64(&v, new)` | 原子写 |
| `atomic.CompareAndSwapInt64(&v, old, new)` | CAS操作 |
| `atomic.Value` | 原子存取任意类型值 |

**选择原则**：简单数值用atomic，复杂数据结构用Mutex。

---

## 易错点总结

| 问题 | 后果 | 解决方案 |
|------|------|----------|
| 并发读写普通map | panic | 用sync.Map或Mutex+Map |
| 忘记Unlock | 死锁，所有goroutine阻塞 | 用 `defer mu.Unlock()` |
| Mutex值复制 | 锁失效，数据竞争 | 始终传指针 |
| Lock后再Lock（同一goroutine） | 死锁（Mutex不可重入） | 检查调用链，避免嵌套加锁 |
| RWMutex写锁中调用读锁 | 死锁 | 写锁内直接访问数据，不要再加读锁 |
| sync.Map存入后类型断言失败 | panic | 约定好value类型，或使用泛型封装 |

---

## 面试题精选

### 基础题

**Q1：Go的map为什么不是线程安全的？**

> Go团队出于性能考虑，没有为内建map加锁。大多数使用场景不需要并发访问，加锁会带来不必要的性能开销。Go运行时会检测并发读写map的行为并主动panic（`fatal error: concurrent map writes`），而不是产生静默的数据损坏，这是一种"fail fast"的设计哲学。

**Q2：Mutex和RWMutex有什么区别？分别适用于什么场景？**

> `Mutex` 是互斥锁，无论读写都互斥；`RWMutex` 是读写锁，允许多个读锁共存，但写锁独占。读多写少时RWMutex性能更好（多个读操作可以并行），读写频率相当时Mutex可能更好（RWMutex维护读者计数有额外开销）。

**Q3：以下代码有什么问题？**

```go
type User struct {
    mu   sync.Mutex
    name string
}

func (u User) GetName() string {
    u.mu.Lock()
    defer u.mu.Unlock()
    return u.name
}
```

> 问题在于方法接收者是值类型 `User` 而不是指针 `*User`。每次调用 `GetName()` 会复制整个结构体（包括Mutex），导致每次锁的是不同的副本，锁完全失效。应改为 `func (u *User) GetName() string`。

### 进阶题

**Q4：sync.Map的内部实现原理是什么？为什么读多写少时性能好？**

> sync.Map内部维护read和dirty两个map。read通过atomic.Value存取，读操作无需加锁；dirty包含所有数据，需要Mutex保护。读操作优先查read（无锁），命中则直接返回；未命中时才加锁查dirty。当未命中次数达到dirty长度时，dirty被原子提升为read。所以在读多写少、key稳定的场景下，绝大多数读操作都命中read，几乎不需要加锁。

**Q5：sync.Map和加锁Map分别在什么场景下使用？**

> sync.Map适合两种场景（官方文档明确指出）：（1）key一旦写入很少删除或更新（缓存类场景）；（2）多个goroutine读写不同的key集合（分区式访问）。其他场景建议用Mutex/RWMutex + 普通map，原因：sync.Map的key/value是 `any` 类型缺少类型安全，且在写入频繁时dirty频繁重建性能反而更差。

**Q6：如何实现一个线程安全的LRU缓存？**

```go
type LRUCache struct {
    mu       sync.Mutex
    capacity int
    items    map[string]*list.Element
    order    *list.List // 最近使用的在前面
}

type entry struct {
    key   string
    value any
}

func (c *LRUCache) Get(key string) (any, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem)
        return elem.Value.(*entry).value, true
    }
    return nil, false
}

func (c *LRUCache) Put(key string, value any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem)
        elem.Value.(*entry).value = value
        return
    }
    elem := c.order.PushFront(&entry{key, value})
    c.items[key] = elem
    if c.order.Len() > c.capacity {
        oldest := c.order.Back()
        c.order.Remove(oldest)
        delete(c.items, oldest.Value.(*entry).key)
    }
}
```

> 关键点：用Mutex而非RWMutex，因为Get操作也会修改链表顺序（MoveToFront），本质上是写操作。

### 高级题

**Q7：以下代码会发生什么？如何修复？**

```go
var mu sync.Mutex

func A() {
    mu.Lock()
    defer mu.Unlock()
    B()
}

func B() {
    mu.Lock() // ？
    defer mu.Unlock()
    fmt.Println("hello")
}
```

> 死锁。goroutine在A中持有锁，调用B时再次Lock同一个Mutex，但Go的Mutex**不可重入**（同一goroutine也不能重复Lock），第二次Lock会永久阻塞。修复方式：（1）拆分为内部不加锁的 `bInternal()` 函数供A调用，B调用时自己加锁；（2）重新设计接口避免嵌套加锁。

**Q8：atomic和Mutex有什么区别？什么时候用atomic？**

> atomic基于CPU指令（CAS等）实现，无需系统调用和上下文切换，性能比Mutex高一个数量级。但atomic只能处理简单的数值操作（加减、读写、CAS），无法保护复杂的多步操作。选择原则：单个变量的简单操作用atomic，多变量或多步的复合操作用Mutex。`atomic.Value` 可以原子存取任意类型，但仅适合"整体替换"语义。

**Q9：如何用 `go run -race` 检测数据竞争？它的原理是什么？**

> `go run -race` 启用竞态检测器，它在编译时为每次内存访问插入检测代码，运行时记录所有goroutine对共享变量的访问顺序。如果检测到两个goroutine在没有同步机制的情况下访问同一变量且至少一个是写操作，就会报告数据竞争。注意：（1）它只能检测运行时实际发生的竞争，不是静态分析；（2）会显著降低性能（2-10倍），通常在测试阶段使用 `go test -race`；（3）检测到竞争即表示代码有bug，应该100%修复。

---

## 小结

| 工具 | 适用场景 | 特点 |
|------|----------|------|
| `sync.Mutex` | 通用互斥 | 简单可靠，读写都互斥 |
| `sync.RWMutex` | 读多写少 | 读锁并发，写锁独占 |
| `sync.Map` | 并发Map（读多写少） | 无需手动加锁，内部读写分离 |
| `sync.Once` | 单次初始化 | 保证只执行一次 |
| `sync/atomic` | 简单数值操作 | 基于CPU指令，最高性能 |

并发安全的核心原则：**识别共享数据 → 选择合适的保护机制 → 最小化临界区范围**。

下一篇我们将学习Go语言的错误处理机制。
