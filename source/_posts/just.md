---
title: just
top_img: false
tags: 工具
abbrlink: d4ad52ee
date: 2026-02-15 11:56:25
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

## **简介**

`just` 提供一种保存和运行项目特有命令的便捷方式。

和 `make` 一样，`just` 是构建工具的一种，可以通过和 `make` 类似的语法自由构建 `justfile`，而后完成 build、run、test 等等各种任务。但与 `make` 不同的是，`just` 专注于作为**命令运行器**（command runner），而非文件构建系统，因此它不关心文件的时间戳和依赖关系，语法也更加简洁直观。

`just` 的主要优势：

- 语法比 `Makefile` 更简洁，错误信息更友好
- 支持 Linux、macOS、Windows 跨平台使用
- 可以加载 `.env` 文件中的环境变量
- 支持列出可用的 recipes（任务）
- 支持参数、默认值、条件表达式等特性
- 支持选择不同的 shell（`bash`、`powershell`、`python` 等）

---

## **安装**

### **macOS**

```bash
brew install just
```

### **Windows**

```powershell
# 使用 scoop
scoop install just

# 使用 winget
winget install Casey.Just

# 使用 cargo
cargo install just
```

### **Linux**

```bash
# Debian/Ubuntu
sudo apt install just

# Arch Linux
sudo pacman -S just

# 使用 cargo（通用方式）
cargo install just
```

安装完成后验证：

```bash
$ just --version
just 1.40.0
```

---

## **快速开始**

在项目根目录创建一个名为 `justfile`（无扩展名）的文件：

```justfile
# 构建项目
build:
    cargo build --release

# 运行项目
run:
    cargo run

# 运行测试
test:
    cargo test
```

然后在终端中运行：

```bash
$ just build    # 执行 build 任务
$ just run      # 执行 run 任务
$ just test     # 执行 test 任务
$ just          # 执行第一个任务（build）
```

> `just` 不带参数时会执行 justfile 中的**第一个** recipe。

---

## **基础语法**

### **Recipe（任务）**

Recipe 是 `just` 中的基本单位，类似于 `Makefile` 中的 target。格式为 recipe 名称后跟一个冒号，下面缩进的每一行都是要执行的命令：

```justfile
hello:
    echo "Hello, World!"
```

**注意：** `justfile` 中的缩进必须使用**4个空格**或者**Tab**，不能混用。默认情况下推荐使用4个空格。

每一行命令默认都会**先打印命令本身**，然后再执行。如果不想打印命令，可以在命令前加 `@`：

```justfile
hello:
    @echo "Hello, World!"
```

也可以反过来，在 recipe 名称前加 `@` 使整个 recipe 静默：

```justfile
@hello:
    echo "Hello, World!"
```

如果想全局静默，可以在文件开头设置：

```justfile
set quiet

hello:
    echo "Hello, World!"
```

---

### **依赖**

一个 recipe 可以依赖其他 recipe，依赖会在当前 recipe 之前执行：

```justfile
build: clean
    cargo build --release

clean:
    rm -rf target
```

执行 `just build` 时，会先执行 `clean`，再执行 `build`。

多个依赖用空格分隔：

```justfile
all: clean build test
    echo "All done!"

clean:
    rm -rf target

build:
    cargo build

test:
    cargo test
```

依赖也可以传递参数：

```justfile
default: (build "release")

build mode:
    echo "Building in {{mode}} mode"
```

---

### **参数**

Recipe 可以接受参数：

```justfile
greet name:
    echo "Hello, {{name}}!"
```

```bash
$ just greet World
Hello, World!
```

参数可以设置**默认值**：

```justfile
greet name="World":
    echo "Hello, {{name}}!"
```

```bash
$ just greet         # Hello, World!
$ just greet Alice   # Hello, Alice!
```

使用 `+` 表示接受一个或多个参数（可变参数）：

```justfile
test +files:
    cargo test {{files}}
```

```bash
$ just test test_a test_b test_c
```

使用 `*` 表示接受零个或多个参数：

```justfile
run *args:
    cargo run {{args}}
```

---

### **变量**

可以在 justfile 顶层定义变量：

```justfile
version := "1.0.0"
build_dir := "target"

build:
    echo "Building version {{version}}"
    cargo build --release --target-dir {{build_dir}}
```

变量也支持通过反引号执行命令来赋值：

```justfile
git_hash := `git rev-parse --short HEAD`
date := `date +%Y-%m-%d`

info:
    echo "Commit: {{git_hash}}, Date: {{date}}"
```

使用 `$` 可以将变量导出为环境变量：

```justfile
export DATABASE_URL := "postgres://localhost/mydb"

migrate:
    sqlx migrate run
```

也可以在单个 recipe 中导出：

```justfile
run $ENV="development":
    node server.js
```

---

### **注释**

使用 `#` 添加注释：

```justfile
# 这是一个注释
build:
    cargo build # 行内注释不会被识别为注释，会作为命令的一部分
```

在 recipe 上方的注释可以配合 `just --list` 显示为说明文档：

```justfile
# 构建项目
build:
    cargo build

# 运行所有测试
test:
    cargo test
```

```bash
$ just --list
Available recipes:
    build # 构建项目
    test  # 运行所有测试
```

---

## **条件表达式**

`just` 支持 `if/else` 条件表达式：

```justfile
os := os()

greet:
    echo {{ if os == "windows" { "Hello from Windows!" } else { "Hello from Unix!" } }}
```

支持的比较运算符：`==` 和 `!=`

也支持正则匹配：

```justfile
foo := if "hello" =~ 'hel+o' { "match" } else { "mismatch" }

test:
    echo {{foo}}
```

---

## **常用设置**

在 justfile 开头可以使用 `set` 关键字进行全局配置：

```justfile
# 使用 PowerShell 作为 shell（Windows 常用）
set shell := ["powershell", "-c"]

# 加载 .env 文件中的环境变量
set dotenv-load

# 允许使用未定义的环境变量（不报错）
set allow-duplicate-variables

# 将未知的 recipe 参数传递给默认 recipe
set fallback

# 设置在 recipe 出错时立即停止
set positional-arguments
```

常用设置项一览：

| 设置项 | 说明 |
|---|---|
| `set shell := [...]` | 指定执行命令的 shell |
| `set dotenv-load` | 自动加载 `.env` 文件 |
| `set quiet` | 全局静默，不打印命令 |
| `set positional-arguments` | 将参数作为位置参数传递给 shell |
| `set windows-shell := [...]` | 仅在 Windows 上覆盖 shell |
| `set tempdir := "..."` | 设置临时文件目录 |

---

## **内置函数**

`just` 提供了许多内置函数，以下是常用的几个：

### **系统信息**

```justfile
info:
    echo "OS: {{os()}}"              # 操作系统：linux, macos, windows
    echo "Arch: {{arch()}}"          # 架构：x86_64, aarch64
    echo "Family: {{os_family()}}"   # OS 系列：unix, windows
    echo "Num CPUs: {{num_cpus()}}"  # CPU 核心数
```

### **路径操作**

```justfile
# 不带扩展名的文件名
name := file_stem("foo/bar.txt")         # "bar"

# 获取文件扩展名
ext := extension("foo/bar.txt")          # "txt"

# 获取父目录
dir := parent_directory("foo/bar.txt")   # "foo"

# 拼接路径
full := join("foo", "bar", "baz.txt")   # "foo/bar/baz.txt"

# 获取绝对路径
abs := absolute_path("src")

# justfile 所在目录
root := justfile_directory()
```

### **字符串操作**

```justfile
trimmed := trim("  hello  ")                    # "hello"
upper := uppercase("hello")                      # "HELLO"
lower := lowercase("HELLO")                      # "hello"
replaced := replace("hello", "l", "r")           # "herro"
```

### **环境变量**

```justfile
# 获取环境变量，如果不存在则报错
home := env("HOME")

# 获取环境变量，如果不存在则使用默认值
editor := env("EDITOR", "vim")

test:
    echo "Home: {{home}}"
    echo "Editor: {{editor}}"
```

### **UUID 和随机值**

```justfile
id := uuid()
hex := sha256("hello")

show:
    echo "UUID: {{id}}"
    echo "SHA256: {{hex}}"
```

---

## **使用不同的脚本语言**

`just` 的一个强大特性是支持在 recipe 中使用不同的脚本语言。通过 shebang（`#!`）来指定：

```justfile
# 使用 Python
analyze:
    #!/usr/bin/env python3
    import json
    data = {"name": "just", "type": "command runner"}
    print(json.dumps(data, indent=2))

# 使用 Node.js
calculate:
    #!/usr/bin/env node
    const result = Array.from({length: 10}, (_, i) => i * i);
    console.log(result);

# 使用 Bash
setup:
    #!/usr/bin/env bash
    set -euo pipefail
    echo "Setting up..."
    mkdir -p build
    echo "Done!"
```

> 使用 shebang 时，recipe 中的所有行会被写入一个临时脚本文件并一起执行，而不是逐行执行。

---

## **命令行用法**

```bash
# 列出所有可用的 recipe
just --list

# 显示某个 recipe 的详细信息
just --show build

# 选择 justfile（默认自动查找当前及上级目录）
just --justfile path/to/justfile

# 设置工作目录
just --working-directory path/to/dir

# 干运行，只打印命令不执行
just --dry-run build

# 传递变量覆盖
just --set version "2.0.0" build

# 列出所有变量
just --evaluate

# 显示 justfile 的语法摘要
just --summary

# 选择要执行的 recipe（交互式，需安装 fzf）
just --choose

# 格式化 justfile
just --fmt

# 检查格式是否正确
just --fmt --check
```

---

## **实用示例**

### **前端项目**

```justfile
set dotenv-load

# 安装依赖
install:
    npm install

# 开发模式
dev:
    npm run dev

# 构建生产版本
build:
    npm run build

# 运行测试
test:
    npm run test

# 代码检查
lint:
    npm run lint

# 格式化代码
fmt:
    npx prettier --write "src/**/*.{ts,tsx}"

# 清理构建产物
clean:
    rm -rf dist node_modules/.cache
```

### **Go 项目**

```justfile
module := "myapp"
version := `git describe --tags --always`

# 构建
build:
    go build -ldflags "-X main.version={{version}}" -o bin/{{module}} .

# 运行
run *args: build
    ./bin/{{module}} {{args}}

# 测试
test:
    go test ./... -v

# 代码检查
lint:
    golangci-lint run

# 生成代码
generate:
    go generate ./...

# 交叉编译
build-all:
    GOOS=linux GOARCH=amd64 go build -o bin/{{module}}-linux-amd64 .
    GOOS=darwin GOARCH=arm64 go build -o bin/{{module}}-darwin-arm64 .
    GOOS=windows GOARCH=amd64 go build -o bin/{{module}}-windows-amd64.exe .
```

### **Docker 项目**

```justfile
image := "myapp"
tag := `git rev-parse --short HEAD`

# 构建镜像
docker-build:
    docker build -t {{image}}:{{tag}} .

# 运行容器
docker-run: docker-build
    docker run --rm -p 8080:8080 {{image}}:{{tag}}

# 推送镜像
docker-push: docker-build
    docker push {{image}}:{{tag}}

# 启动完整环境
up:
    docker compose up -d

# 停止环境
down:
    docker compose down

# 查看日志
logs service="":
    docker compose logs -f {{service}}
```

---

## **与 Makefile 的对比**

| 特性 | just | make |
|---|---|---|
| 定位 | 命令运行器 | 构建系统 |
| 文件时间戳检查 | 不支持 | 支持 |
| 参数传递 | 原生支持 | 需要变通 |
| 错误提示 | 友好 | 晦涩 |
| `.env` 支持 | 内置 | 需要额外处理 |
| 跨平台 | 良好 | 依赖环境 |
| 列出任务 | `just --list` | 需要额外实现 |
| 多语言 shebang | 支持 | 有限支持 |

如果你的需求是管理和运行项目中的各种命令，而不是像 C/C++ 项目那样需要基于文件依赖的增量编译，那么 `just` 是比 `make` 更好的选择。
