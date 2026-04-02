---
title: Go语言文件操作
top_img: false
tags: golang
abbrlink: a7d1e5f
date: 2026-03-23 21:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言文件操作

文件操作是实际开发中的高频需求——配置读取、日志写入、数据导入导出都离不开它。Go标准库通过 `os`、`io`、`bufio` 三个包提供了从底层到高层的完整文件操作能力。

---

## 文件读取

### 一次性读取：os.ReadFile

适合读取**小文件**（配置文件、模板等），将整个文件内容一次性加载到内存：

```go
func main() {
    data, err := os.ReadFile("config.json")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(data))
}
```

> **注意**：`os.ReadFile` 是Go 1.16引入的，替代了已废弃的 `ioutil.ReadFile`。文件过大时会占用大量内存，不适合处理GB级文件。

### 分片读取：Read + 固定缓冲区

手动控制每次读取的字节数，适合处理**大文件**或需要流式处理的场景：

```go
func main() {
    file, err := os.Open("large.dat")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    buf := make([]byte, 1024) // 每次读取1KB
    for {
        n, err := file.Read(buf)
        if n > 0 {
            // 处理 buf[:n]，注意用 n 而不是 len(buf)
            fmt.Print(string(buf[:n]))
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
    }
}
```

> **关键点**：`Read` 返回实际读取的字节数 `n`，最后一次读取可能不足1024字节，必须用 `buf[:n]` 而非 `buf`。先处理数据再检查 `err`，因为 `io.EOF` 时 `n` 可能大于0。

### 带缓冲读取：bufio.Reader

`bufio.Reader` 在底层维护一个缓冲区（默认4KB），减少系统调用次数，提升读取性能：

```go
func main() {
    file, err := os.Open("data.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    reader := bufio.NewReader(file)
    buf := make([]byte, 512)
    for {
        n, err := reader.Read(buf)
        if n > 0 {
            fmt.Print(string(buf[:n]))
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
    }
}
```

> `bufio.NewReaderSize(file, 8192)` 可以自定义缓冲区大小。

### 按行读取：bufio.Scanner

处理文本文件最常用的方式，自动按行分割：

```go
func main() {
    file, err := os.Open("log.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    lineNum := 0
    for scanner.Scan() {
        lineNum++
        line := scanner.Text() // 不含换行符
        fmt.Printf("%d: %s\n", lineNum, line)
    }
    // 扫描结束后检查是否有非EOF错误
    if err := scanner.Err(); err != nil {
        log.Fatal(err)
    }
}
```

> **陷阱**：`bufio.Scanner` 默认单行最大64KB（`bufio.MaxScanTokenSize`）。如果文件中有超长行，需要手动扩大缓冲区：

```go
scanner := bufio.NewScanner(file)
scanner.Buffer(make([]byte, 0, 1024*1024), 1024*1024) // 最大1MB/行
```

### 按分隔符读取：bufio.Reader.ReadString

按指定分隔符读取，分隔符会包含在返回结果中：

```go
func main() {
    file, err := os.Open("data.csv")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    reader := bufio.NewReader(file)
    for {
        // 读到分号为止（包含分号本身）
        token, err := reader.ReadString(';')
        if len(token) > 0 {
            token = strings.TrimRight(token, ";")
            fmt.Println(token)
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
    }
}
```

也可以用 `bufio.Scanner` 自定义分割函数：

```go
scanner := bufio.NewScanner(file)
scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
    // 按分号分割
    if i := bytes.IndexByte(data, ';'); i >= 0 {
        return i + 1, data[:i], nil
    }
    if atEOF && len(data) > 0 {
        return len(data), data, nil
    }
    return 0, nil, nil
})

for scanner.Scan() {
    fmt.Println(scanner.Text())
}
```

### 读取方式对比

| 方式 | 适用场景 | 内存占用 | 性能 |
|------|----------|----------|------|
| `os.ReadFile` | 小文件（<几MB） | 整个文件大小 | 简单快速 |
| `file.Read` | 大文件、二进制流 | 缓冲区大小 | 系统调用频繁 |
| `bufio.Reader` | 大文件、需减少IO | 缓冲区大小（默认4KB） | 减少系统调用 |
| `bufio.Scanner` | 文本按行处理 | 单行大小 | 文本处理首选 |
| `ReadString` | 按分隔符切分 | 单片段大小 | 灵活切分 |

---

## 文件写入

### 一次性写入：os.WriteFile

适合写入小文件，会创建或覆盖目标文件：

```go
func main() {
    content := []byte("Hello, Go文件操作!\n")
    // 0644：Owner读写，Group和Others只读
    err := os.WriteFile("output.txt", content, 0644)
    if err != nil {
        log.Fatal(err)
    }
}
```

### 打开文件写入：os.OpenFile

需要更精细的控制（追加写入、读写模式等）时，使用 `os.OpenFile`：

```go
func main() {
    // O_WRONLY: 只写 | O_CREATE: 不存在则创建 | O_APPEND: 追加
    file, err := os.OpenFile("app.log", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0644)
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    _, err = file.WriteString("2026-03-23 服务启动\n")
    if err != nil {
        log.Fatal(err)
    }
}
```

### 文件打开模式详解

`os.OpenFile` 的第二个参数是文件标志（flag），可以组合使用：

| 标志 | 值 | 说明 |
|------|-----|------|
| `os.O_RDONLY` | 0 | 只读（默认） |
| `os.O_WRONLY` | 1 | 只写 |
| `os.O_RDWR` | 2 | 读写 |
| `os.O_CREATE` | - | 文件不存在时创建 |
| `os.O_TRUNC` | - | 打开时清空文件内容 |
| `os.O_APPEND` | - | 写入追加到文件末尾 |
| `os.O_EXCL` | - | 与O_CREATE一起使用，文件已存在则报错 |
| `os.O_SYNC` | - | 同步IO，每次写入都刷盘 |

常用组合：

```go
// 覆盖写入（最常用）
os.OpenFile("f.txt", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)

// 追加写入（日志场景）
os.OpenFile("f.log", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0644)

// 创建新文件，已存在则报错（防止覆盖）
os.OpenFile("f.txt", os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0644)

// 读写模式
os.OpenFile("f.txt", os.O_RDWR|os.O_CREATE, 0644)
```

### 文件权限说明

第三个参数是Unix文件权限（`os.FileMode`），用八进制表示：

```
0644 = Owner读写(6) + Group只读(4) + Others只读(4)
0755 = Owner全部(7) + Group读+执行(5) + Others读+执行(5)
0600 = Owner读写(6) + 其余无权限
```

> Windows上权限参数的效果有限，但为了跨平台兼容性，建议始终设置合理的权限值。

### 带缓冲写入：bufio.Writer

高频写入场景下，`bufio.Writer` 减少系统调用，显著提升性能：

```go
func main() {
    file, err := os.Create("output.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    writer := bufio.NewWriter(file)
    for i := 0; i < 10000; i++ {
        fmt.Fprintf(writer, "第%d行数据\n", i)
    }
    // 必须调用Flush，将缓冲区剩余数据写入文件
    if err := writer.Flush(); err != nil {
        log.Fatal(err)
    }
}
```

> **关键**：`Flush()` 必须调用，否则缓冲区中未满的数据不会写入文件。结合 `defer` 时注意错误处理——`defer writer.Flush()` 会忽略错误。

---

## 文件复制

### 使用 io.Copy

最简洁高效的方式，底层会尝试使用零拷贝优化（如Linux的sendfile）：

```go
func CopyFile(src, dst string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("打开源文件失败: %w", err)
    }
    defer srcFile.Close()

    dstFile, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("创建目标文件失败: %w", err)
    }
    defer dstFile.Close()

    _, err = io.Copy(dstFile, srcFile)
    if err != nil {
        return fmt.Errorf("复制失败: %w", err)
    }

    // 确保数据刷盘
    return dstFile.Sync()
}
```

### 带进度的大文件复制

利用 `io.TeeReader` 或自定义 `io.Reader` 实现复制进度回调：

```go
type ProgressReader struct {
    reader    io.Reader
    total     int64
    current   int64
    onProgress func(percent float64)
}

func (pr *ProgressReader) Read(p []byte) (int, error) {
    n, err := pr.reader.Read(p)
    pr.current += int64(n)
    if pr.onProgress != nil && pr.total > 0 {
        pr.onProgress(float64(pr.current) / float64(pr.total) * 100)
    }
    return n, err
}

func CopyWithProgress(src, dst string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer srcFile.Close()

    info, err := srcFile.Stat()
    if err != nil {
        return err
    }

    dstFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer dstFile.Close()

    pr := &ProgressReader{
        reader: srcFile,
        total:  info.Size(),
        onProgress: func(percent float64) {
            fmt.Printf("\r复制进度: %.1f%%", percent)
        },
    }

    _, err = io.Copy(dstFile, pr)
    fmt.Println()
    return err
}
```

---

## 目录操作

### 创建目录

```go
// 创建单层目录
err := os.Mkdir("logs", 0755)

// 递归创建多层目录（类似 mkdir -p）
err := os.MkdirAll("data/cache/images", 0755)
```

### 读取目录内容

```go
func main() {
    entries, err := os.ReadDir(".")
    if err != nil {
        log.Fatal(err)
    }
    for _, entry := range entries {
        info, _ := entry.Info()
        if entry.IsDir() {
            fmt.Printf("[目录] %s\n", entry.Name())
        } else {
            fmt.Printf("[文件] %s (%d bytes)\n", entry.Name(), info.Size())
        }
    }
}
```

> `os.ReadDir` 是Go 1.16引入的，比旧的 `ioutil.ReadDir` 更高效——它返回 `DirEntry` 而非 `FileInfo`，只在需要时才调用 `Info()` 获取文件详细信息。

### 遍历目录树：filepath.WalkDir

递归遍历目录及其所有子目录：

```go
func main() {
    err := filepath.WalkDir(".", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        // 跳过隐藏目录
        if d.IsDir() && strings.HasPrefix(d.Name(), ".") {
            return filepath.SkipDir
        }
        // 只打印.go文件
        if !d.IsDir() && strings.HasSuffix(d.Name(), ".go") {
            fmt.Println(path)
        }
        return nil
    })
    if err != nil {
        log.Fatal(err)
    }
}
```

> `filepath.WalkDir`（Go 1.16）比旧的 `filepath.Walk` 性能更好，因为它不会为每个文件调用 `Stat`。

### 其他常用操作

```go
// 删除文件或空目录
os.Remove("temp.txt")

// 递归删除目录及内容
os.RemoveAll("temp_dir")

// 重命名/移动
os.Rename("old.txt", "new.txt")

// 获取文件信息
info, err := os.Stat("file.txt")
if err != nil {
    if os.IsNotExist(err) {
        fmt.Println("文件不存在")
    }
}
fmt.Printf("大小: %d, 修改时间: %v, 是否目录: %v\n",
    info.Size(), info.ModTime(), info.IsDir())

// 判断文件是否存在
func FileExists(path string) bool {
    _, err := os.Stat(path)
    return !os.IsNotExist(err)
}

// 获取/创建临时文件
tmpFile, err := os.CreateTemp("", "prefix-*.txt")
defer os.Remove(tmpFile.Name()) // 用完清理

// 获取/创建临时目录
tmpDir, err := os.MkdirTemp("", "myapp-*")
defer os.RemoveAll(tmpDir)
```

---

## 面试题精选

### 基础题

**Q1：os.ReadFile 和 os.Open + Read 有什么区别？分别适合什么场景？**

> `os.ReadFile` 一次性将整个文件读入 `[]byte`，代码简洁但内存占用等于文件大小，适合小文件（配置、模板等）。`os.Open` + `Read` 可以分片读取，每次只加载缓冲区大小的数据，适合大文件或流式处理。选择标准：文件大小是否可控——如果确定文件不会很大，用 `ReadFile`；否则用分片读取。

**Q2：bufio.Scanner 的默认行大小限制是多少？超过会怎样？**

> 默认最大64KB（`bufio.MaxScanTokenSize = 64 * 1024`）。超过限制时 `Scan()` 返回 `false`，`Err()` 返回 `bufio.ErrTooLong`。解决方式是调用 `scanner.Buffer(buf, maxSize)` 扩大缓冲区。这是一个常见的生产问题——处理日志文件时可能遇到超长行。

**Q3：os.O_APPEND 和 O_TRUNC 的区别是什么？同时使用会怎样？**

> `O_APPEND` 每次写入追加到文件末尾，保留原内容。`O_TRUNC` 打开时立即清空文件内容。同时使用时 `O_TRUNC` 先清空文件，后续写入追加到末尾——等同于先清空再从头写，实际效果和只用 `O_TRUNC` 一样。日志文件用 `O_APPEND`，覆盖写入用 `O_TRUNC`。

### 进阶题

**Q4：以下代码有什么问题？**

```go
func ReadLines(path string) ([]string, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    var lines []string
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        lines = append(lines, scanner.Text())
    }
    return lines, nil
}
```

> 缺少对 `scanner.Err()` 的检查。`scanner.Scan()` 返回 `false` 可能是正常EOF，也可能是读取错误（如I/O故障、行超长）。修复：在 `return` 前检查 `scanner.Err()`，有错误时返回该错误而非 `nil`。

```go
if err := scanner.Err(); err != nil {
    return nil, err
}
return lines, nil
```

**Q5：为什么 bufio.Writer 必须调用 Flush？defer file.Close() 不会自动刷缓冲吗？**

> `bufio.Writer` 在应用层维护缓冲区，`file.Close()` 只关闭操作系统文件描述符，不知道 `bufio.Writer` 缓冲区中还有未写入的数据。不调用 `Flush()` 会导致**数据丢失**——缓冲区中未满的最后一批数据不会写入文件。正确做法是在 `Close()` 前显式 `Flush()`，并检查其返回错误。

**Q6：io.Copy 底层是怎么工作的？为什么说它比手动 Read/Write 循环更好？**

> `io.Copy` 内部维护一个32KB的缓冲区进行循环读写，但关键在于它会检查源和目标是否实现了 `io.WriterTo` 或 `io.ReaderFrom` 接口。如果实现了，会直接调用这些优化方法——例如 `*os.File` 实现了 `ReadFrom`，在Linux上底层使用 `sendfile` 系统调用，实现**零拷贝**传输（数据不经过用户空间），性能远超手动循环。

### 高级题

**Q7：如何安全地写入文件，避免写入中途崩溃导致文件损坏？**

> 使用**原子写入**模式：先写入临时文件，完成后重命名。重命名在大多数文件系统上是原子操作。

```go
func AtomicWriteFile(path string, data []byte, perm os.FileMode) error {
    // 临时文件与目标文件同目录，确保在同一文件系统
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil {
        return err
    }
    tmpPath := tmp.Name()

    // 写入失败时清理临时文件
    defer func() {
        if err != nil {
            os.Remove(tmpPath)
        }
    }()

    if _, err = tmp.Write(data); err != nil {
        tmp.Close()
        return err
    }
    // Sync确保数据落盘
    if err = tmp.Sync(); err != nil {
        tmp.Close()
        return err
    }
    if err = tmp.Close(); err != nil {
        return err
    }
    // 原子重命名
    return os.Rename(tmpPath, path)
}
```

> 这是数据库WAL、配置热更新等场景的标准做法。

**Q8：并发写入同一文件需要注意什么？**

> 多个goroutine写同一文件会导致数据交错。解决方案有三种：（1）用 `sync.Mutex` 加锁保护写入操作；（2）用一个专门的goroutine负责写入，其他goroutine通过channel发送数据；（3）使用 `O_APPEND` 模式——POSIX规范保证 `O_APPEND` 写入是原子的（前提是单次写入不超过 `PIPE_BUF` 大小，通常4KB）。标准库 `log` 包内部就是用 `Mutex` 保护写入。

**Q9：filepath.WalkDir 和 os.ReadDir 有什么区别？遍历大目录时有什么性能考虑？**

> `os.ReadDir` 读取单层目录，返回排序后的 `[]DirEntry`。`filepath.WalkDir` 递归遍历整个目录树，按字典序深度优先访问。性能考虑：（1）`WalkDir` 比旧的 `Walk` 快，因为使用 `DirEntry` 避免了每个文件一次 `Stat` 调用；（2）遍历超大目录（百万文件）时，`WalkDir` 占用内存较少，因为它逐个处理而非全部加载；（3）如果只需查找特定文件，可以在回调中用 `filepath.SkipDir` 跳过不相关的子目录来减少遍历范围。

---

## 小结

| 操作 | 推荐方式 | 说明 |
|------|----------|------|
| 读小文件 | `os.ReadFile` | 一次性读取，简洁 |
| 读大文件 | `bufio.Reader` / `file.Read` | 分片读取，控制内存 |
| 按行读取 | `bufio.Scanner` | 文本处理首选 |
| 写小文件 | `os.WriteFile` | 一次性写入 |
| 追加写入 | `os.OpenFile` + `O_APPEND` | 日志场景 |
| 高频写入 | `bufio.Writer` + `Flush` | 减少系统调用 |
| 文件复制 | `io.Copy` | 可能零拷贝优化 |
| 遍历目录 | `filepath.WalkDir` | 递归高效 |
| 安全写入 | 临时文件 + `Rename` | 防崩溃损坏 |

文件操作的核心原则：**打开的文件必须关闭（defer Close）、写入的数据必须落盘（Sync/Flush）、错误必须检查**。
