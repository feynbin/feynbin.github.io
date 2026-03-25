---
title: Go语言反射
top_img: false
tags: golang
abbrlink: b7d9e4f
date: 2026-03-25 20:00:00
updated:
categories:
keywords:
description:
cover:
highlight_shrink:
---

# Go语言反射

> 本文是Go语言学习系列的第十四篇。前十三篇：[《golang简介》](/posts/d0ea244/)、[《Go语言变量》](/posts/a1b2c3d/)、[《Go语言数组、切片与Map》](/posts/b3f5e78/)、[《Go语言判断语句》](/posts/c4d6e9f/)、[《Go语言循环语句》](/posts/e5f7a2b/)、[《Go语言函数》](/posts/f6a8b3c/)、[《Go语言init与defer》](/posts/a7c9d4e/)、[《Go语言结构体》](/posts/b8d1e5f/)、[《Go语言自定义类型与接口》](/posts/c9e2f6a/)、[《Go语言协程与信道》](/posts/d3f4a7b/)、[《Go语言线程安全与sync.Map》](/posts/e4a8b2c/)、[《Go语言错误与异常处理》](/posts/f5b9c3d/)、[《Go语言泛型》](/posts/a6c0d4e/)，建议先行阅读。

反射（reflection）允许程序在**运行时**检查变量的类型、读取值、修改值，甚至动态调用方法。它很强大，但也意味着更高的复杂度、更差的可读性和更弱的类型安全。因此Go社区的态度一直很明确：**能不用反射就不用，必须动态处理时再用反射。**

---

## 什么是反射

普通代码在编译期就知道类型：

```go
var n int = 10
fmt.Println(n + 1)
```

但有些场景在编译期并不知道实际类型，比如：

- 通用ORM：根据结构体字段生成SQL
- JSON/配置解析：根据tag映射字段
- 框架中间件：动态调用用户传入的方法
- 通用日志/调试工具：打印任意类型的值

这时就需要 `reflect` 包在运行时获取类型信息。

Go反射最核心的两个类型：

- `reflect.Type`：描述"类型"
- `reflect.Value`：描述"值"

```go
var x float64 = 3.14

t := reflect.TypeOf(x)
v := reflect.ValueOf(x)

fmt.Println(t)        // float64
fmt.Println(t.Kind()) // float64
fmt.Println(v)        // 3.14
fmt.Println(v.Kind()) // float64
```

可以简单理解为：

```text
TypeOf 关注：这是什么类型
ValueOf 关注：这里面装的是什么值
```

---

## reflect.TypeOf：获取类型信息

### 获取具体类型

```go
var a int = 10
var b string = "hello"
var c []int = []int{1, 2, 3}

fmt.Println(reflect.TypeOf(a)) // int
fmt.Println(reflect.TypeOf(b)) // string
fmt.Println(reflect.TypeOf(c)) // []int
```

### Kind 和 Type 的区别

这是反射中最容易混淆的地方之一。

- `Type` 表示完整类型，如 `main.User`、`[]int`、`map[string]int`
- `Kind` 表示底层类别，如 `struct`、`slice`、`map`

```go
type MyInt int

var x MyInt = 100
t := reflect.TypeOf(x)

fmt.Println(t)        // main.MyInt
fmt.Println(t.Kind()) // int
```

所以：

```text
Type 更具体
Kind 更抽象
```

### 通过 Kind 做类型判断

如果只关心一个值属于哪一大类，通常用 `Kind()`：

```go
func checkType(v any) {
	t := reflect.TypeOf(v)

	switch t.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		fmt.Println("这是整数类型")
	case reflect.String:
		fmt.Println("这是字符串类型")
	case reflect.Struct:
		fmt.Println("这是结构体类型")
	case reflect.Slice:
		fmt.Println("这是切片类型")
	default:
		fmt.Println("其他类型:", t)
	}
}
```

如果需要判断是否是某个具体类型，则直接比较 `Type`：

```go
type User struct{}

var u User
if reflect.TypeOf(u) == reflect.TypeOf(User{}) {
    fmt.Println("u 的具体类型就是 User")
}
```

### 指针类型与 Elem

反射遇到指针时，经常要配合 `Elem()` 取出其指向的元素类型：

```go
type User struct {
    Name string
}

u := &User{Name: "Alice"}
t := reflect.TypeOf(u)

fmt.Println(t)           // *main.User
fmt.Println(t.Kind())    // ptr
fmt.Println(t.Elem())    // main.User
fmt.Println(t.Elem().Kind()) // struct
```

---

## reflect.ValueOf：获取值信息

`reflect.Value` 可以进一步读取变量内部的数据。

```go
var x int = 42
v := reflect.ValueOf(x)

fmt.Println(v.Int())      // 42
fmt.Println(v.Kind())     // int
fmt.Println(v.Interface()) // 42
```

### 常见取值方法

不同类型要用不同的方法取值：

```go
fmt.Println(reflect.ValueOf(10).Int())         // int/int8/int16/int32/int64
fmt.Println(reflect.ValueOf("go").String())    // string
fmt.Println(reflect.ValueOf(true).Bool())      // bool
fmt.Println(reflect.ValueOf(3.14).Float())     // float32/float64
fmt.Println(reflect.ValueOf([]int{1, 2}).Len()) // slice/map/array/string 长度
```

### 还原回普通接口值

反射值可以通过 `Interface()` 还原成 `any`，然后再做类型断言：

```go
v := reflect.ValueOf(100)
x := v.Interface()

num, ok := x.(int)
if ok {
    fmt.Println(num) // 100
}
```

---

## 通过反射修改值

这是反射的重点，也是最容易踩坑的部分。

### 直接修改为什么会失败

```go
var x int = 10
v := reflect.ValueOf(x)

fmt.Println(v.CanSet()) // false
// v.SetInt(20)         // panic: reflect.Value.SetInt using unaddressable value
```

原因很简单：`reflect.ValueOf(x)` 拿到的是 `x` 的**副本**，不是原变量本身，所以不能修改。

### 正确做法：传指针，再用 Elem

```go
var x int = 10
v := reflect.ValueOf(&x) // 传入指针

fmt.Println(v.Kind())     // ptr
fmt.Println(v.Elem().CanSet()) // true

v.Elem().SetInt(20)
fmt.Println(x) // 20
```

可以记一个规则：

```text
想改值：必须拿到地址（指针） + Elem()
```

### CanSet 和 CanAddr

两个常见判断方法：

- `CanAddr()`：是否可以取地址
- `CanSet()`：是否可以修改

```go
var x int = 10
v1 := reflect.ValueOf(x)
v2 := reflect.ValueOf(&x).Elem()

fmt.Println(v1.CanAddr(), v1.CanSet()) // false false
fmt.Println(v2.CanAddr(), v2.CanSet()) // true true
```

### 修改不同类型的值

```go
name := "Tom"
age := 18
score := 95.5

reflect.ValueOf(&name).Elem().SetString("Jack")
reflect.ValueOf(&age).Elem().SetInt(20)
reflect.ValueOf(&score).Elem().SetFloat(99.9)

fmt.Println(name, age, score) // Jack 20 99.9
```

> `SetInt` 只能用于整数Kind，`SetString` 只能用于字符串Kind，类型不匹配会直接 panic。

---

## 结构体反射

结构体是反射最常见的应用场景，因为字段、tag、方法都能在运行时处理。

### 获取结构体类型和字段信息

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

u := User{Name: "Alice", Age: 25}
t := reflect.TypeOf(u)

fmt.Println(t.Name())      // User
fmt.Println(t.Kind())      // struct
fmt.Println(t.NumField())  // 2

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    fmt.Println("字段名:", field.Name)
    fmt.Println("字段类型:", field.Type)
    fmt.Println("tag:", field.Tag.Get("json"))
}
```

输出大致如下：

```text
字段名: Name
字段类型: string
tag: name
字段名: Age
字段类型: int
tag: age
```

### 根据字段名获取字段

```go
type User struct {
    Name string
    Age  int
}

u := User{Name: "Alice", Age: 25}
v := reflect.ValueOf(u)

nameField := v.FieldByName("Name")
fmt.Println(nameField.String()) // Alice
```

如果字段不存在，返回的是无效值：

```go
f := v.FieldByName("Email")
fmt.Println(f.IsValid()) // false
```

所以动态取字段时，最好先判断 `IsValid()`。

---

## 通过反射修改结构体字段

### 修改结构体字段的基本写法

```go
type User struct {
    Name string
    Age  int
}

u := User{Name: "Alice", Age: 25}
v := reflect.ValueOf(&u).Elem()

v.FieldByName("Name").SetString("Bob")
v.FieldByName("Age").SetInt(30)

fmt.Println(u) // {Bob 30}
```

这和修改普通变量一样，本质上仍然要求：

- 必须传入结构体指针
- 必须 `Elem()`
- 字段必须可设置

### 未导出字段不能随便改

```go
type User struct {
    Name string
    age  int
}

u := User{Name: "Alice", age: 25}
v := reflect.ValueOf(&u).Elem()

fmt.Println(v.FieldByName("Name").CanSet()) // true
fmt.Println(v.FieldByName("age").CanSet())  // false
```

虽然 `age` 字段存在，但它是未导出字段，反射默认不允许直接修改。强行 `SetInt` 会 panic。

> 这也是Go封装性的体现：反射不是"万能后门"，依然要遵守导出规则。

### 遍历并修改结构体中的字段

```go
type User struct {
    Name string
    Age  int
}

u := User{Name: "Alice", Age: 25}
v := reflect.ValueOf(&u).Elem()

for i := 0; i < v.NumField(); i++ {
    field := v.Field(i)

    switch field.Kind() {
    case reflect.String:
        field.SetString("updated")
    case reflect.Int:
        field.SetInt(100)
    }
}

fmt.Println(u) // {updated 100}
```

这类写法常见于：

- 默认值填充
- 配置绑定
- 结构体字段批量处理

---

## 调用结构体方法

反射除了读字段、改字段，还能动态调用方法。

### 调用无参方法

```go
type User struct {
    Name string
}

func (u User) SayHello() {
    fmt.Println("Hello,", u.Name)
}

u := User{Name: "Alice"}
v := reflect.ValueOf(u)

method := v.MethodByName("SayHello")
fmt.Println(method.IsValid()) // true

method.Call(nil) // Hello, Alice
```

### 调用带参数的方法

```go
type User struct {
    Name string
}

func (u User) Greet(prefix string) string {
    return prefix + ", " + u.Name
}

u := User{Name: "Alice"}
v := reflect.ValueOf(u)

method := v.MethodByName("Greet")
result := method.Call([]reflect.Value{
    reflect.ValueOf("Hi"),
})

fmt.Println(result[0].String()) // Hi, Alice
```

### 指针接收者方法的调用

如果方法是指针接收者，反射时通常也要传指针：

```go
type User struct {
    Name string
}

func (u *User) SetName(name string) {
    u.Name = name
}

u := User{Name: "Alice"}
v := reflect.ValueOf(&u)

method := v.MethodByName("SetName")
method.Call([]reflect.Value{reflect.ValueOf("Bob")})

fmt.Println(u.Name) // Bob
```

如果你拿的是值 `reflect.ValueOf(u)`，往往找不到只定义在 `*User` 上的方法。

---

## 结构体 tag 与反射

之前在结构体文章里提到过 tag，本质上它就是给反射读取的元信息。

```go
type User struct {
    Name string `json:"name" db:"user_name"`
    Age  int    `json:"age" db:"user_age"`
}

func main() {
    t := reflect.TypeOf(User{})

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Println(field.Name, field.Tag.Get("json"), field.Tag.Get("db"))
    }
}
```

这也是为什么：

- `encoding/json` 能按 `json:"name"` 序列化
- ORM 能按 `db:"user_name"` 生成数据库字段映射
- 参数校验库能按 `validate:"required"` 做校验

这些框架大量依赖反射读取结构体元数据。

---

## 一个实战示例：打印结构体信息

下面写一个简单的通用函数，传入任意结构体，打印字段名、类型、值和tag：

```go
func PrintStructFields(input any) {
    t := reflect.TypeOf(input)
    v := reflect.ValueOf(input)

    if t.Kind() == reflect.Ptr {
        t = t.Elem()
        v = v.Elem()
    }

    if t.Kind() != reflect.Struct {
        fmt.Println("input 不是结构体")
        return
    }

    for i := 0; i < t.NumField(); i++ {
        fieldType := t.Field(i)
        fieldValue := v.Field(i)

        fmt.Printf("字段: %s, 类型: %s, 值: %v, json tag: %s\n",
            fieldType.Name,
            fieldType.Type,
            fieldValue.Interface(),
            fieldType.Tag.Get("json"),
        )
    }
}

type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func main() {
    u := User{Name: "Alice", Age: 25}
    PrintStructFields(u)
}
```

这个例子本身不复杂，但它已经展示了反射在框架中的基本套路：

1. 先拿到 `Type` 和 `Value`
2. 判断是否为指针，如果是就 `Elem()`
3. 判断是否为结构体
4. 遍历字段，读取字段名、类型、值、tag

---

## 反射的常见坑

### 1. nil 与零值问题

```go
var p *int = nil
v := reflect.ValueOf(p)

fmt.Println(v.Kind())   // ptr
fmt.Println(v.IsNil())  // true
```

但如果是一个真正的空接口零值：

```go
var x any = nil
fmt.Println(reflect.TypeOf(x)) // <nil>
```

所以反射前经常需要先判断是否为 nil。

### 2. 对错误Kind调用错误方法

```go
v := reflect.ValueOf("hello")
// fmt.Println(v.Int()) // panic
```

`String()`、`Int()`、`Bool()` 这些方法都必须和对应 Kind 匹配。

### 3. 修改值时忘记传指针

```go
u := User{Name: "Alice"}
v := reflect.ValueOf(u)
fmt.Println(v.CanSet()) // false
```

这是最常见的错误，没有之一。

### 4. 反射代码性能较差

反射要做运行时类型检查、动态分发、装箱拆箱，比普通代码慢得多。对于高频热点路径，应尽量避免反射。

### 5. 可读性下降

```go
v.FieldByName("Age").SetInt(18)
```

这种代码不如：

```go
u.Age = 18
```

直观。能直接写业务代码时，不要为了"通用"强行上反射。

---

## 什么时候该用反射

适合使用反射的场景：

- 框架/库开发，需要处理任意类型
- 读取结构体 tag
- 做通用序列化、映射、拷贝、依赖注入
- 编写调试、日志、测试工具

不适合使用反射的场景：

- 普通业务代码，类型已知
- 性能敏感的热路径
- 只是为了少写几行重复代码

简单判断标准：

```text
如果编译期就知道类型，优先普通代码或泛型
如果运行时才知道类型，再考虑反射
```

---

## 面试高频题

### Q1：Go反射的核心类型是什么？

**答**：Go反射的核心类型是 `reflect.Type` 和 `reflect.Value`。`Type` 用来描述类型信息，比如类型名、Kind、字段信息、方法信息；`Value` 用来描述值本身，可以读取具体数据，在满足条件时也可以修改值。通常通过 `reflect.TypeOf(x)` 和 `reflect.ValueOf(x)` 获取。

### Q2：Type 和 Kind 有什么区别？

**答**：`Type` 是完整类型，包含具体类型名，例如 `main.User`、`[]int`、`map[string]int`。`Kind` 是底层类别，例如 `struct`、`slice`、`map`、`int`。比如自定义类型 `type MyInt int`，它的 `Type` 是 `main.MyInt`，但 `Kind` 是 `int`。判断大类一般用 `Kind`，判断具体类型一般比较 `Type`。

### Q3：为什么 `reflect.ValueOf(x)` 不能直接修改 x？

**答**：因为 `reflect.ValueOf(x)` 拿到的是值的副本，不是原变量本身，所以 `CanSet()` 为 false。要修改原值，必须传指针：`reflect.ValueOf(&x).Elem()`。只有拿到可寻址、可设置的值后，才能调用 `SetInt`、`SetString` 等方法。

### Q4：CanSet 和 CanAddr 有什么区别？

**答**：`CanAddr()` 表示该值是否可取地址，`CanSet()` 表示该值是否可修改。一般来说，可修改的值通常也可取地址，但可取地址不一定代表一定可修改。最常见的可设置值是通过指针 `Elem()` 得到的变量或结构体导出字段。

### Q5：如何通过反射修改结构体字段？

**答**：必须传入结构体指针，再通过 `Elem()` 拿到结构体本体，然后用 `FieldByName` 或 `Field(i)` 获取字段，最后调用对应的 `Set` 方法。例如：

```go
u := User{Name: "Alice", Age: 25}
v := reflect.ValueOf(&u).Elem()
v.FieldByName("Name").SetString("Bob")
v.FieldByName("Age").SetInt(30)
```

前提是字段必须是导出的，并且字段本身 `CanSet()` 为 true。

### Q6：为什么未导出字段不能通过反射直接修改？

**答**：因为Go的反射仍然遵守语言的封装规则。未导出字段在包外本来就不可访问，反射不会绕过这个限制。即使拿到了字段，也通常 `CanSet()` 为 false，强行设置会 panic。这是为了保证包的封装性不被反射破坏。

### Q7：如何通过反射调用结构体方法？

**答**：先通过 `reflect.ValueOf(obj)` 拿到值，再用 `MethodByName("方法名")` 获取方法，最后用 `Call([]reflect.Value{...})` 调用。无参方法传 `nil`，有参方法传参数切片。若方法是指针接收者，通常要对指针做反射，例如 `reflect.ValueOf(&obj)`。

### Q8：反射和泛型有什么区别？

**答**：泛型是在**编译期**确定类型参数，保留类型安全，性能更好，适合"逻辑相同、类型不同"的场景。反射是在**运行时**检查类型和值，灵活性更强，但性能更差、代码更复杂。能在编译期解决的问题优先用泛型；只有运行时才知道类型时，才使用反射。

### Q9：反射常见的应用场景有哪些？

**答**：最常见的是框架和库开发，例如 JSON 序列化、ORM 映射、配置绑定、依赖注入、参数校验、测试工具和调试打印。它们共同特点是：需要处理任意结构体或任意类型，而这些信息只有在运行时才能确定。

### Q10：反射的缺点是什么？

**答**：主要有四点。第一，性能开销更大，运行时检查和动态调用都比直接代码慢。第二，可读性差，逻辑不直观。第三，容易出现 panic，比如 Kind 不匹配、字段不存在、值不可设置。第四，类型错误从编译期推迟到运行时，调试成本更高。

### Q11：下面代码为什么会 panic？

```go
var x int = 10
v := reflect.ValueOf(x)
v.SetInt(20)
```

**答**：因为 `v` 是 `x` 的副本，不可设置，`CanSet()` 为 false，所以调用 `SetInt` 会 panic。正确写法是：

```go
v := reflect.ValueOf(&x).Elem()
v.SetInt(20)
```

### Q12：反射中 `Elem()` 的作用是什么？

**答**：`Elem()` 用于获取指针、接口、切片元素等包装内部的实际值。在反射修改变量时最常见的用法是对指针取 `Elem()`，例如 `reflect.ValueOf(&x).Elem()` 得到变量 `x` 本身。没有 `Elem()`，你拿到的只是指针值，不能直接按目标类型修改内部数据。

---

## 小结

| 概念 | 要点 |
|------|------|
| `reflect.TypeOf` | 获取类型信息 |
| `reflect.ValueOf` | 获取值信息 |
| `Type` vs `Kind` | Type 是具体类型，Kind 是底层类别 |
| 读取值 | `Int()`、`String()`、`Bool()`、`Interface()` |
| 修改值 | 必须传指针，再 `Elem()` |
| `CanSet()` | 判断值是否可以修改 |
| 结构体反射 | 可读取字段、tag、方法 |
| 修改结构体字段 | 字段必须导出且值可设置 |
| 方法调用 | `MethodByName` + `Call` |
| 使用原则 | 能不用反射就不用，运行时动态处理再用 |

反射是Go里非常重要的一块能力，但它更像一种"底层工具"而不是日常业务开发的默认方案。真正写业务时，优先考虑具体类型、接口和泛型；当你要做框架、通用组件、序列化或元编程时，反射才会真正发挥价值。
