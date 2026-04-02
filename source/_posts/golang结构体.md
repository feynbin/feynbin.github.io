---
title: Go语言结构体
top_img: false
tags: golang
abbrlink: b8d1e5f
date: 2026-03-22 20:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言结构体

Go没有 `class` 关键字，但结构体（struct）承担了面向对象中"类"的角色。结构体定义数据结构，方法定义行为，组合替代继承——这是Go的面向对象哲学。

---

## 定义结构体

使用 `type` + `struct` 定义结构体：

```go
type User struct {
    Name string
    Age  int
    Email string
}
```

**命名规范**：
- 结构体名首字母大写（如 `User`）→ 可被其他包访问（导出）
- 字段名首字母大写（如 `Name`）→ 字段可被其他包访问
- 字段名首字母小写（如 `name`）→ 字段仅包内可见

```go
type User struct {
    Name  string  // 导出字段：其他包可以访问
    phone string  // 未导出字段：仅当前包可以访问
}
```

---

## 创建与初始化

### 1. 零值初始化

声明后所有字段自动初始化为零值：

```go
var u User
fmt.Println(u) // { 0 }  Name="", Age=0, Email=""
```

### 2. 字面量——按字段名（推荐）

```go
u := User{
    Name:  "Alice",
    Age:   25,
    Email: "alice@example.com",
}
```

按字段名初始化可以只指定部分字段，未指定的为零值：

```go
u := User{Name: "Bob"} // Age=0, Email=""
```

### 3. 字面量——按顺序

```go
u := User{"Alice", 25, "alice@example.com"}
```

必须按定义顺序给出**所有**字段值，不能省略。一旦结构体增加字段，所有使用此方式的代码都需要修改，**不推荐使用**。

### 4. new——返回指针

```go
u := new(User)       // 返回 *User，所有字段为零值
u.Name = "Charlie"
```

### 5. 取地址初始化——最常用

```go
u := &User{
    Name: "Alice",
    Age:  25,
}
// u 的类型是 *User
```

这种写法等价于 `new(User)` + 赋值，是实际开发中**最常用**的创建方式。

---

## 结构体方法

Go通过**方法接收者（receiver）** 将函数绑定到结构体上，类似其他语言的 `this` 或 `self`：

```go
type User struct {
    Name string
    Age  int
}

// 值接收者
func (u User) Greet() string {
    return "Hello, I'm " + u.Name
}

u := User{Name: "Alice"}
fmt.Println(u.Greet()) // Hello, I'm Alice
```

`(u User)` 就是接收者，`u` 相当于其他语言中的 `this`/`self`，指代调用方法的那个实例。

### 值接收者 vs 指针接收者

这是结构体方法最核心的区别：

```go
// 值接收者：操作的是副本，不影响原结构体
func (u User) SetNameByValue(name string) {
    u.Name = name // 修改的是副本
}

// 指针接收者：操作的是原结构体
func (u *User) SetName(name string) {
    u.Name = name // 修改原结构体
}
```

```go
u := User{Name: "Alice"}

u.SetNameByValue("Bob")
fmt.Println(u.Name) // Alice（未改变）

u.SetName("Bob")
fmt.Println(u.Name) // Bob（已改变）
```

### 如何选择接收者类型？

| 场景 | 选择 | 原因 |
|------|------|------|
| 需要修改结构体字段 | 指针接收者 | 值接收者修改的是副本 |
| 结构体较大 | 指针接收者 | 避免每次调用复制整个结构体 |
| 结构体很小且只读 | 值接收者 | 更安全，无副作用 |
| 实现某个接口 | 保持一致 | 同一结构体的方法建议统一用一种接收者 |

> **实际开发建议**：如果拿不准，用指针接收者。大多数方法都需要修改状态或者结构体较大，指针接收者是更安全的默认选择。

### Go 的自动取址/解引用

Go编译器会自动处理值和指针之间的方法调用，不需要手动转换：

```go
u := User{Name: "Alice"}
u.SetName("Bob")   // u 是值，但Go自动取址 (&u).SetName("Bob")

p := &User{Name: "Alice"}
p.Greet()           // p 是指针，但Go自动解引用 (*p).Greet()
```

虽然编译器帮你做了转换，但理解背后的机制很重要——在接口实现中，这种自动转换有限制（后续接口文章会详细讲解）。

---

## 结构体指针

### 为什么需要指针？

结构体是**值类型**，赋值和传参都会完整复制：

```go
u1 := User{Name: "Alice", Age: 25}
u2 := u1      // 完整复制
u2.Name = "Bob"
fmt.Println(u1.Name) // Alice（u1 不受影响）
```

如果需要在函数中修改结构体，或者避免大结构体的复制开销，使用指针：

```go
func birthday(u *User) {
    u.Age++ // 通过指针修改原结构体
}

u := &User{Name: "Alice", Age: 25}
birthday(u)
fmt.Println(u.Age) // 26
```

### 指针访问字段的语法糖

Go中通过指针访问字段不需要 `->` 或 `(*p).Field`，直接用 `.` 即可：

```go
u := &User{Name: "Alice"}

// 以下两种写法等价
fmt.Println((*u).Name) // 标准写法
fmt.Println(u.Name)    // 语法糖，Go自动解引用
```

---

## 结构体 Tag

Tag 是附加在结构体字段上的元信息字符串，运行时可以通过反射读取。最常用于 JSON 序列化/反序列化。

### 基本语法

```go
type User struct {
    Name  string `json:"name"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}
```

反引号内的 `json:"name"` 就是 tag。序列化时字段名会按 tag 中的名称输出：

```go
u := User{Name: "Alice", Age: 25, Email: "alice@example.com"}
data, _ := json.Marshal(u)
fmt.Println(string(data))
// {"name":"Alice","age":25,"email":"alice@example.com"}
```

不加 tag 时，JSON 的 key 就是字段名本身（大写开头）：

```go
// 无 tag 时输出
// {"Name":"Alice","Age":25,"Email":"alice@example.com"}
```

### 忽略字段（-）

使用 `json:"-"` 让字段在序列化时**完全忽略**，常用于密码、token等敏感信息：

```go
type User struct {
    Name     string `json:"name"`
    Password string `json:"-"` // 序列化时忽略，不会出现在 JSON 中
}

u := User{Name: "Alice", Password: "secret123"}
data, _ := json.Marshal(u)
fmt.Println(string(data))
// {"name":"Alice"}  —— Password 不会出现
```

### 零值忽略（omitempty）

使用 `omitempty` 选项，当字段值为零值时不输出该字段：

```go
type User struct {
    Name  string `json:"name"`
    Age   int    `json:"age,omitempty"`
    Email string `json:"email,omitempty"`
    Phone string `json:"phone,omitempty"`
}

u := User{Name: "Alice", Age: 25}
data, _ := json.Marshal(u)
fmt.Println(string(data))
// {"name":"Alice","age":25}
// Email 和 Phone 为零值("")，被省略
```

### 常用 tag 汇总

```go
type User struct {
    Name     string `json:"name"`               // 重命名
    Password string `json:"-"`                   // 忽略
    Age      int    `json:"age,omitempty"`       // 零值时省略
    Email    string `json:"email,omitempty"`     // 重命名 + 零值省略
    Score    int    `json:"score,string"`        // 序列化为字符串 "85" 而非 85
}
```

### 多种 tag 并存

一个字段可以有多种 tag，用空格分隔：

```go
type User struct {
    Name string `json:"name" db:"user_name" xml:"name" validate:"required"`
}
```

| tag | 用途 |
|-----|------|
| `json` | JSON 序列化/反序列化 |
| `db` | 数据库 ORM 映射 |
| `xml` | XML 序列化 |
| `yaml` | YAML 序列化 |
| `validate` | 参数校验 |
| `form` | HTTP 表单绑定 |

### tag 的实际应用——API 响应

```go
type APIResponse struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

type UserVO struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email,omitempty"`
    Password string `json:"-"` // 永远不返回给前端
    Phone    string `json:"-"` // 敏感信息不暴露
}
```

---

## 结构体组合（Embedding）

Go没有继承，而是通过**组合**实现代码复用。将一个结构体作为另一个结构体的匿名字段嵌入，就可以直接访问被嵌入结构体的字段和方法。

### 基本用法

```go
type Animal struct {
    Name string
    Age  int
}

func (a *Animal) Eat() {
    fmt.Printf("%s is eating\n", a.Name)
}

type Dog struct {
    Animal // 匿名嵌入——组合
    Breed  string
}
```

```go
d := Dog{
    Animal: Animal{Name: "Buddy", Age: 3},
    Breed:  "Golden Retriever",
}

// 直接访问 Animal 的字段和方法，不需要 d.Animal.Name
fmt.Println(d.Name)  // Buddy
fmt.Println(d.Age)   // 3
d.Eat()              // Buddy is eating

// 也可以显式访问
fmt.Println(d.Animal.Name) // Buddy
```

### 方法重写（Override）

外层结构体可以定义同名方法，覆盖嵌入结构体的方法：

```go
func (d *Dog) Eat() {
    fmt.Printf("%s is eating dog food\n", d.Name)
}

d := Dog{Animal: Animal{Name: "Buddy"}}
d.Eat()        // Buddy is eating dog food（调用 Dog 的方法）
d.Animal.Eat() // Buddy is eating（显式调用 Animal 的方法）
```

### 多层组合

```go
type Animal struct {
    Name string
}

func (a *Animal) Breathe() {
    fmt.Printf("%s is breathing\n", a.Name)
}

type Dog struct {
    Animal
    Breed string
}

func (d *Dog) Bark() {
    fmt.Printf("%s is barking\n", d.Name)
}

type GuideDog struct {
    Dog
    Handler string
}

func (g *GuideDog) Guide() {
    fmt.Printf("%s is guiding %s\n", g.Name, g.Handler)
}
```

```go
g := GuideDog{
    Dog: Dog{
        Animal: Animal{Name: "Rex"},
        Breed:  "Labrador",
    },
    Handler: "John",
}

g.Breathe() // Rex is breathing（来自 Animal）
g.Bark()    // Rex is barking（来自 Dog）
g.Guide()   // Rex is guiding John（自己的方法）
g.Name      // Rex（穿透两层直接访问）
```

### 组合 vs 继承

| 特性 | Go 组合 | Java/Python 继承 |
|------|--------|-----------------|
| 关键字 | 无，匿名嵌入 | `extends` / `:` |
| 关系 | has-a（拥有） | is-a（是一个） |
| 多继承 | 支持嵌入多个结构体 | Java 不支持多继承 |
| 方法调用 | 编译期确定 | 运行时多态（虚方法表） |
| 字段提升 | 嵌入字段自动提升到外层 | 通过继承链查找 |
| 耦合度 | 低耦合 | 高耦合 |

> Go 的设计理念是"组合优于继承"。组合更灵活——你可以随时添加或移除嵌入的结构体，而继承一旦确定就很难改变。

---

## 结构体比较

结构体是否可比较，取决于其**所有字段**是否都可比较：

```go
type Point struct {
    X, Y int
}

p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true

type User struct {
    Name    string
    Friends []string // 切片不可比较
}
// u1 == u2  // 编译错误：struct containing []string cannot be compared
```

如果结构体含有切片、map 等不可比较字段，需要使用 `reflect.DeepEqual` 或自定义比较方法。

---

## 面试高频题

### Q1：结构体是值类型还是引用类型？

**答**：值类型。赋值和传参都会完整复制整个结构体的所有字段。如果结构体较大或需要在函数中修改，应传指针。但要注意，如果结构体字段中包含引用类型（切片、map、指针），复制的只是引用本身，底层数据仍然共享。

### Q2：值接收者和指针接收者有什么区别？

**答**：值接收者操作的是结构体的副本，修改不影响原结构体；指针接收者操作的是原结构体，修改直接生效。另外，值接收者每次调用都会复制结构体，对于大结构体有性能开销。实际开发中，如果方法需要修改状态或结构体较大，用指针接收者；同一结构体的方法建议统一用一种接收者类型。

### Q3：下面代码输出什么？

```go
type User struct {
    Name string
}

func (u User) SetName(name string) {
    u.Name = name
}

func main() {
    u := User{Name: "Alice"}
    u.SetName("Bob")
    fmt.Println(u.Name)
}
```

**答**：输出 `Alice`。`SetName` 使用值接收者，`u` 在方法内部是副本，修改不影响原结构体。改为指针接收者 `func (u *User) SetName(name string)` 才能修改成功。

### Q4：结构体 tag 中 `json:"-"` 和 `json:",omitempty"` 的区别？

**答**：`json:"-"` 表示该字段在序列化和反序列化时**完全忽略**，无论值是什么都不会出现在 JSON 中，适用于密码等敏感字段。`json:",omitempty"` 表示当字段值为零值（0、""、nil、false、空切片、空map）时才省略，有值时正常输出。两者可以结合场景选用：机密数据用 `-`，可选数据用 `omitempty`。

### Q5：Go 的组合和 Java 的继承有什么区别？

**答**：Go 的组合是 has-a 关系，Java 的继承是 is-a 关系。Go 通过匿名嵌入结构体实现字段和方法的提升，可以同时嵌入多个结构体（类似多继承）；Java 只支持单继承。Go 的方法调用在编译期确定，没有运行时多态（虚方法表）；Java 通过继承链实现运行时多态。Go 的组合耦合度更低，可以随时添加或移除嵌入结构体，而继承关系一旦确定就很难改变。

### Q6：两个相同结构体的值可以比较吗？

**答**：取决于所有字段是否都可比较。如果所有字段都是可比较类型（int、string、bool、数组、指针等），结构体可以用 `==` 比较。如果包含切片、map、函数等不可比较字段，编译器会报错。不可比较的结构体需要用 `reflect.DeepEqual` 或自定义方法比较。

### Q7：下面代码输出什么？

```go
type Base struct {
    Name string
}

func (b *Base) Show() {
    fmt.Println("Base:", b.Name)
}

type Child struct {
    Base
}

func (c *Child) Show() {
    fmt.Println("Child:", c.Name)
}

func main() {
    c := Child{Base: Base{Name: "test"}}
    c.Show()
    c.Base.Show()
}
```

**答**：输出 `Child: test` 和 `Base: test`。`c.Show()` 调用的是 Child 自己的 Show 方法（外层覆盖内层）。`c.Base.Show()` 显式调用被嵌入的 Base 的 Show 方法。这类似于其他语言中 override 后通过 `super` 调用父类方法。

### Q8：下面的 JSON 序列化输出什么？

```go
type User struct {
    Name     string `json:"name"`
    Age      int    `json:"age,omitempty"`
    Password string `json:"-"`
    Phone    string `json:"phone,omitempty"`
}

u := User{Name: "Alice", Age: 0, Password: "123456", Phone: ""}
data, _ := json.Marshal(u)
fmt.Println(string(data))
```

**答**：输出 `{"name":"Alice"}`。`Password` 使用 `json:"-"` 被完全忽略。`Age` 为 0（int 的零值），`omitempty` 生效，被省略。`Phone` 为 ""（string 的零值），`omitempty` 生效，被省略。只有 `Name` 有非零值，正常输出。

### Q9：下面代码有什么问题？

```go
type Config struct {
    Items map[string]string
}

func main() {
    c1 := Config{Items: map[string]string{"a": "1"}}
    c2 := c1
    c2.Items["b"] = "2"
    fmt.Println(c1.Items)
}
```

**答**：`c1.Items` 输出 `map[a:1 b:2]`，c1 被意外修改了。虽然结构体赋值是值复制，但 map 字段复制的只是 map 的指针，c1 和 c2 的 Items 指向同一个底层哈希表。修改 c2 的 Items 会影响 c1。要实现深拷贝，需要手动复制 map 的内容：

```go
c2 := Config{Items: make(map[string]string)}
for k, v := range c1.Items {
    c2.Items[k] = v
}
```

### Q10：结构体方法中的接收者变量名有什么惯例？

**答**：Go 惯例是使用结构体类型名的**首字母小写**作为接收者变量名，而不是用 `this` 或 `self`：

```go
func (u *User) GetName() string { return u.Name }     // User → u
func (c *Config) Load() error { ... }                  // Config → c
func (db *Database) Query() { ... }                    // Database → db
```

同一结构体的所有方法应使用相同的接收者变量名，保持一致性。用 `this`/`self` 虽然不会报错，但不符合Go社区惯例。

---

## 小结

| 概念 | 要点 |
|------|------|
| 定义 | `type Name struct {}`，大小写控制导出 |
| 初始化 | 按字段名初始化（推荐）、`&Type{}` 取地址初始化（最常用） |
| 值类型 | 赋值和传参完整复制，含引用类型字段时需注意浅拷贝 |
| 方法 | 通过接收者绑定，`(u User)` 值接收者 / `(u *User)` 指针接收者 |
| 接收者选择 | 需要修改或结构体大 → 指针；小且只读 → 值 |
| 指针语法糖 | `p.Field` 等价于 `(*p).Field`，自动解引用 |
| Tag | 元信息字符串，`json:"-"` 忽略、`omitempty` 零值省略 |
| 组合 | 匿名嵌入替代继承，字段和方法自动提升 |
| 方法重写 | 外层同名方法覆盖内层，显式调用内层用 `x.Base.Method()` |
| 比较 | 所有字段可比较则结构体可比较，否则用 `reflect.DeepEqual` |

下一篇将介绍Go的**接口（Interface）**。
