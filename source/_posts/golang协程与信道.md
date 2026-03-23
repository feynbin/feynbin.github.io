---
title: Go语言协程与信道
top_img: false
tags: golang
abbrlink: d3f4a7b
date: 2026-03-23 14:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言协程与信道

> 本文是Go语言学习系列的第十篇。前九篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)、[《Go语言函数》](/posts/f6a8b3c/)、[《Go语言init与defer》](/posts/a7c9d4e/)、[《Go语言结构体》](/posts/b8d1e5f/)、[《Go语言自定义类型与接口》](/posts/c9e2f6a/)，建议先行阅读。

并发是Go语言的核心竞争力。Go通过goroutine（协程）和channel（信道）提供了一套简洁而强大的并发编程模型，遵循CSP（Communicating Sequential Processes）理念——**不要通过共享内存来通信，而要通过通信来共享内存**。

---

## 什么是Goroutine（协程）

Goroutine是Go运行时管理的轻量级线程。与操作系统线程相比：

| 特性 | OS线程 | Goroutine |
|------|--------|-----------|
| 栈大小 | 固定，通常1-8MB | 初始2KB，按需增长 |
| 创建开销 | 大（系统调用） | 小（用户态） |
| 调度 | 内核调度 | Go运行时调度（GMP模型） |
| 数量上限 | 通常数千个 | 轻松支持数十万个 |

Go运行时使用**GMP调度模型**：G（Goroutine）、M（Machine，即OS线程）、P（Processor，逻辑处理器）。多个G被调度到少量M上执行，P控制并发度（默认等于CPU核心数）。

---

## 创建Goroutine

使用 `go` 关键字即可启动一个协程：

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    go sayHello("Go") // 启动一个goroutine
    go sayHello("World")

    // 主goroutine需要等待，否则程序直接退出
    time.Sleep(time.Second)
}
```

也可以用匿名函数：

```go
go func(msg string) {
    fmt.Println(msg)
}("Hello from anonymous goroutine")
```

> **注意**：`main()` 函数本身运行在主goroutine中。当主goroutine退出时，所有子goroutine会被强制终止，不论是否执行完毕。上面用 `time.Sleep` 等待是不可靠的做法，正式代码应使用 `sync.WaitGroup` 或 channel。

---

## sync.WaitGroup：等待协程完成

`sync.WaitGroup` 用于等待一组goroutine全部执行完毕：

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // 完成时计数器减1
    fmt.Printf("Worker %d started\n", id)
    // 模拟工作...
    fmt.Printf("Worker %d finished\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1) // 计数器加1
        go worker(i, &wg)
    }

    wg.Wait() // 阻塞，直到计数器归零
    fmt.Println("All workers done")
}
```

三个核心方法：

| 方法 | 作用 |
|------|------|
| `Add(n)` | 计数器加n（通常在启动goroutine前调用） |
| `Done()` | 计数器减1（等价于 `Add(-1)`） |
| `Wait()` | 阻塞直到计数器归零 |

> **易错点**：`wg` 必须以指针传递给goroutine，否则每个goroutine拿到的是副本，`Done()` 不会影响原始计数器。

---

## 如何"关闭"一个Goroutine

Go没有提供直接杀死goroutine的API——这是**设计上的选择**，强制终止可能导致资源泄漏。正确的做法是**通知goroutine自行退出**，常见方式有三种：

### 方式一：使用channel通知退出

```go
func worker(stop <-chan struct{}) {
    for {
        select {
        case <-stop:
            fmt.Println("收到退出信号，清理资源...")
            return
        default:
            fmt.Println("工作中...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    stop := make(chan struct{})
    go worker(stop)

    time.Sleep(2 * time.Second)
    close(stop) // 通知退出
    time.Sleep(time.Second)
}
```

### 方式二：使用context（推荐）

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("收到取消信号:", ctx.Err())
            return
        default:
            fmt.Println("工作中...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx)

    time.Sleep(2 * time.Second)
    cancel() // 取消context，通知goroutine退出
    time.Sleep(time.Second)
}
```

`context` 是生产环境中管理goroutine生命周期的标准方式，支持取消、超时、传值等能力。

---

## Channel（信道）

Channel是goroutine之间通信的管道，是Go并发模型的核心。

### 创建和使用

```go
// 创建channel
ch := make(chan int)    // 无缓冲channel
ch := make(chan int, 5) // 缓冲大小为5的channel

// 发送数据
ch <- 42

// 接收数据
value := <-ch
```

### 无缓冲Channel

无缓冲channel要求发送和接收同时就绪，否则会阻塞：

```go
func main() {
    ch := make(chan string)

    go func() {
        ch <- "hello" // 发送方阻塞，直到有人接收
    }()

    msg := <-ch // 接收方阻塞，直到有人发送
    fmt.Println(msg)
}
```

无缓冲channel天然提供了同步机制——发送和接收像一次"握手"。

### 有缓冲Channel

缓冲channel在缓冲区未满时不会阻塞发送方：

```go
ch := make(chan int, 3)

ch <- 1 // 不阻塞
ch <- 2 // 不阻塞
ch <- 3 // 不阻塞
// ch <- 4 // 阻塞！缓冲区已满

fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
```

| 操作 | 无缓冲 | 有缓冲 |
|------|--------|--------|
| 发送 | 阻塞直到有接收者 | 缓冲区满时阻塞 |
| 接收 | 阻塞直到有发送者 | 缓冲区空时阻塞 |

### Channel方向

可以限制channel的方向，增强类型安全：

```go
func producer(out chan<- int) { // 只能发送
    for i := 0; i < 5; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) { // 只能接收
    for v := range in {
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```

### 关闭Channel

使用 `close()` 关闭channel，关闭后：
- 不能再发送数据（panic）
- 可以继续接收已缓冲的数据
- 缓冲区为空后，接收返回零值

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch)

fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
v, ok := <-ch     // v=0, ok=false（channel已关闭且为空）
```

使用 `range` 遍历channel，会在channel关闭后自动退出：

```go
for v := range ch {
    fmt.Println(v)
}
```

> **原则**：只由发送方关闭channel，接收方不应关闭。

---

## Select：多路复用

`select` 语句用于同时监听多个channel操作，类似于 `switch`，但专门用于channel：

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "来自ch1"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "来自ch2"
    }()

    // 等待第一个就绪的channel
    select {
    case msg := <-ch1:
        fmt.Println(msg)
    case msg := <-ch2:
        fmt.Println(msg)
    }
}
```

### select的特性

- **随机选择**：多个case同时就绪时，随机选一个执行（避免饥饿）
- **阻塞**：没有case就绪且无default时，select会阻塞
- **非阻塞**：添加 `default` 分支可实现非阻塞操作

```go
// 非阻塞接收
select {
case msg := <-ch:
    fmt.Println("收到:", msg)
default:
    fmt.Println("没有数据，继续做别的事")
}
```

### 用select实现多channel数据收集

```go
func fanIn(ch1, ch2 <-chan string) <-chan string {
    merged := make(chan string)
    go func() {
        defer close(merged)
        for ch1 != nil || ch2 != nil {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil // channel关闭，置为nil避免重复触发
                    continue
                }
                merged <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                merged <- v
            }
        }
    }()
    return merged
}
```

---

## 协程超时处理

在实际开发中，必须对协程操作设置超时，避免goroutine永久阻塞导致泄漏。

### 方式一：time.After

```go
select {
case result := <-ch:
    fmt.Println("收到结果:", result)
case <-time.After(3 * time.Second):
    fmt.Println("操作超时！")
}
```

### 方式二：context.WithTimeout（推荐）

```go
func fetchData(ctx context.Context) (string, error) {
    ch := make(chan string, 1)

    go func() {
        // 模拟耗时操作
        time.Sleep(5 * time.Second)
        ch <- "数据"
    }()

    select {
    case result := <-ch:
        return result, nil
    case <-ctx.Done():
        return "", ctx.Err() // context.DeadlineExceeded
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel() // 始终调用cancel释放资源

    result, err := fetchData(ctx)
    if err != nil {
        fmt.Println("错误:", err) // 输出: 错误: context deadline exceeded
        return
    }
    fmt.Println(result)
}
```

### 方式三：context.WithDeadline

与 `WithTimeout` 类似，但指定的是绝对时间点而非相对时长：

```go
deadline := time.Now().Add(3 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
```

---

## 常见并发模式

### Worker Pool（工作池）

```go
func workerPool(jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        results <- job * 2 // 处理任务
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    var wg sync.WaitGroup

    // 启动3个worker
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go workerPool(jobs, results, &wg)
    }

    // 发送任务
    for i := 1; i <= 10; i++ {
        jobs <- i
    }
    close(jobs)

    // 等待所有worker完成后关闭results
    go func() {
        wg.Wait()
        close(results)
    }()

    for r := range results {
        fmt.Println(r)
    }
}
```

### 限流器（Rate Limiter）

```go
func main() {
    // 每200ms允许一次操作
    limiter := time.NewTicker(200 * time.Millisecond)
    defer limiter.Stop()

    requests := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        requests <- i
    }
    close(requests)

    for req := range requests {
        <-limiter.C // 等待令牌
        fmt.Println("处理请求", req, time.Now().Format("15:04:05.000"))
    }
}
```

---

## 易错点总结

| 问题 | 后果 | 解决方案 |
|------|------|----------|
| 忘记关闭channel | `range` 死锁 | 发送方负责 `close()` |
| 向已关闭channel发送 | panic | 只关闭一次，只由发送方关闭 |
| WaitGroup用值传递 | `Wait()` 永远不返回 | 传指针 `&wg` |
| goroutine泄漏 | 内存增长 | 用context或channel通知退出 |
| 循环变量捕获 | 所有goroutine用同一个值 | 通过参数传递或局部变量 |
| 无缓冲channel读写同一goroutine | 死锁 | 用缓冲channel或分开goroutine |

循环变量捕获的典型错误：

```go
// ❌ 错误写法（Go 1.22之前）
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // 可能全部打印5
    }()
}

// ✅ 正确写法：通过参数传递
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}

// ✅ Go 1.22+：循环变量语义已修改，每次迭代创建新变量
```

---

## 面试题精选

### 基础题

**Q1：goroutine和操作系统线程有什么区别？**

> 核心区别在三方面：（1）**栈大小**：goroutine初始栈仅2KB且可动态增长，OS线程固定1-8MB；（2）**调度方式**：goroutine由Go运行时在用户态调度（GMP模型），OS线程由内核调度，上下文切换开销更大；（3）**数量**：一个Go程序可轻松运行数十万goroutine，而OS线程通常只能创建数千个。

**Q2：以下代码输出什么？为什么？**

```go
func main() {
    for i := 0; i < 3; i++ {
        go func() {
            fmt.Println(i)
        }()
    }
    time.Sleep(time.Second)
}
```

> 在Go 1.22之前，很可能输出三个 `3`，因为闭包捕获的是变量 `i` 的引用，循环结束时 `i=3`。Go 1.22修改了循环变量语义，每次迭代创建新变量，输出 `0 1 2`（乱序）。

**Q3：无缓冲channel和有缓冲channel的区别？**

> 无缓冲channel发送和接收必须同时就绪（同步通信），像面对面交接；有缓冲channel在缓冲区未满时发送不阻塞（异步通信），像往信箱里投递。无缓冲channel适合同步协调，有缓冲channel适合解耦生产者和消费者的速率。

### 进阶题

**Q4：如何优雅地关闭一个goroutine？列举你知道的方式。**

> 三种方式：（1）用 `close(stopCh)` 发送退出信号，goroutine通过 `select` 监听；（2）用 `context.WithCancel` 创建可取消的context（推荐），调用 `cancel()` 通知退出；（3）用 `context.WithTimeout` / `WithDeadline` 实现超时自动退出。核心原则：**不要强杀goroutine，让它自己退出**。

**Q5：下面代码有什么问题？如何修复？**

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        go func(wg sync.WaitGroup) {
            wg.Add(1)
            defer wg.Done()
            fmt.Println(i)
        }(wg)
    }
    wg.Wait()
}
```

> 三个问题：（1）`wg` 是值传递，每个goroutine操作的是副本，`wg.Wait()` 立即返回；（2）`wg.Add(1)` 应该在 `go` 之前调用，放在goroutine内部存在竞态——`Wait()` 可能在 `Add` 之前执行；（3）闭包直接引用 `i`（Go 1.22之前会有问题）。修复：传指针 `&wg`，在 `go` 前调用 `Add`，`i` 作为参数传入。

**Q6：什么是goroutine泄漏？如何检测和避免？**

> goroutine泄漏指goroutine被创建后永远不会退出（通常因为阻塞在channel操作或锁上），导致内存持续增长。检测方式：（1）`runtime.NumGoroutine()` 监控数量变化；（2）使用 `pprof` 分析goroutine栈；（3）测试中用 `goleak` 库。避免方式：始终确保goroutine有退出路径——使用context控制生命周期，用 `select` + 超时防止永久阻塞。

**Q7：select中多个case同时就绪会怎样？**

> Go会**随机**选择一个就绪的case执行，这是语言规范定义的行为，目的是避免饥饿——确保每个channel都有公平的机会被处理。不是按代码顺序，也不是轮询。

### 高级题

**Q8：实现一个带超时的并发请求函数，同时请求3个URL，返回最快的结果，超过2秒全部取消。**

```go
func fetchFirst(urls []string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    ch := make(chan string, len(urls)) // 缓冲避免goroutine泄漏

    for _, url := range urls {
        go func(u string) {
            // 用ctx创建请求，超时自动取消
            req, _ := http.NewRequestWithContext(ctx, "GET", u, nil)
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)
            ch <- string(body)
        }(url)
    }

    select {
    case result := <-ch:
        return result, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}
```

> 关键点：channel必须有缓冲（`len(urls)`），否则未被选中的goroutine因发送阻塞而泄漏；使用 `NewRequestWithContext` 将context传入HTTP请求，cancel后请求自动中断。

**Q9：如何用channel实现一个信号量（限制并发数）？**

```go
type Semaphore chan struct{}

func NewSemaphore(max int) Semaphore {
    return make(Semaphore, max)
}

func (s Semaphore) Acquire() { s <- struct{}{} }
func (s Semaphore) Release() { <-s }

// 使用
sem := NewSemaphore(3) // 最多3个并发
for i := 0; i < 10; i++ {
    sem.Acquire()
    go func(id int) {
        defer sem.Release()
        // 执行工作...
    }(i)
}
```

> 利用缓冲channel的容量限制并发：缓冲区满时 `Acquire` 阻塞，有goroutine `Release` 后才能继续。

---

## 小结

| 概念 | 核心用途 | 关键API |
|------|----------|---------|
| goroutine | 并发执行 | `go func()` |
| WaitGroup | 等待一组goroutine完成 | `Add` / `Done` / `Wait` |
| channel | goroutine间通信 | `make(chan T)` / `<-` / `close` |
| select | 多channel复用 | `select { case ... }` |
| context | 生命周期管理 | `WithCancel` / `WithTimeout` |

记住Go并发的核心哲学：**Do not communicate by sharing memory; instead, share memory by communicating.**

下一篇我们将学习Go语言的错误处理机制。
