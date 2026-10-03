---
description: ""
title: "Go 语言基础语法速通"
draft: false
date: "2026-09-07T08:02:10+08:00"
slug: "go"
categories:
 - Go
tags:
 - 
image: ""
---

---

# 🚀 Go 语言基础语法速通

> 目标：快速掌握 Go 中低阶语法，重点标注与 Java/Python 的不同之处。

---

## 1. 程序基本结构

```go
package main          // 每个文件必须属于一个包，main 包表示可执行程序

import (
    "fmt"             // 导入标准库，多个包用括号包裹
    "os"
)

func main() {         // 程序入口，无返回值，无参数
    fmt.Println("Hello, Go!")
}
```

**与 Java/Python 的差异：**
- 无类概念，程序入口是 `func main()`，不是 `public static void main(String[] args)` 或 `if __name__ == "__main__":`
- 包名写在文件最顶部，所有代码必须属于某个包

---

## 2. 变量声明

### 2.1 三种声明方式

```go
// 方式1：完整声明（类似 Java）
var name string = "Alice"
var age int = 25

// 方式2：类型推断（类似 Java var / Python）
var city = "New York"     // 编译器自动推断为 string

// 方式3：短变量声明（★ Go 特有，仅函数内可用）
country := "USA"          // 最常用！
x, y := 10, 20            // 多变量同时声明
```

### 2.2 批量声明

```go
var (
    id      int     // 零值：0
    status  string  // 零值：""
    created bool    // 零值：false
)
```

### 2.3 零值初始化（★ Go 特有）

| 类型 | 零值 |
|------|------|
| `int/float` | `0` |
| `string` | `""` |
| `bool` | `false` |
| 指针/切片/map/channel/接口 | `nil` |

> **与 Java 对比**：Java 局部变量必须手动初始化，Go 会自动赋零值。  
> **与 Python 对比**：Python 无零值概念，未赋值变量不可用。

### 2.4 匿名变量 `_`

```go
n, _ := someFunc()    // 忽略第二个返回值，类似 Python 的 `_, n = someFunc()`
```

---

## 3. 常量与枚举

```go
const Pi = 3.14159

// 批量常量
const (
    StatusOK       = 200
    StatusNotFound = 404
)

// iota 枚举（★ Go 特有）
const (
    Sunday = iota   // 0
    Monday          // 1
    Tuesday         // 2
)

// iota 高级用法：位运算枚举
const (
    _  = iota
    KB = 1 << (10 * iota)  // 1024
    MB                     // 1048576
    GB                     // 1073741824
)
```

> **与 Java 对比**：Java 用 `enum` 关键字，Go 用 `const + iota`。  
> **与 Python 对比**：Python 无原生枚举（3.4+ 才有 `enum` 模块）。

---

## 4. 基本数据类型

| Go 类型 | 说明 | 对应 Java | 对应 Python |
|---------|------|-----------|-------------|
| `bool` | 布尔 | `boolean` | `bool` |
| `string` | 字符串（不可变） | `String` | `str` |
| `int` / `int8` / `int16` / `int32` / `int64` | 有符号整数 | `byte`/`short`/`int`/`long` | `int` |
| `uint` / `uint8` / `uint16` / `uint32` / `uint64` | 无符号整数 | 无 | 无 |
| `float32` / `float64` | 浮点数 | `float`/`double` | `float` |
| `complex64` / `complex128` | 复数 | 无 | `complex` |
| `byte` | `uint8` 别名 | `byte` | 无 |
| `rune` | `int32` 别名，Unicode 码点 | `char` | 无 |

> **注意**：Go 的 `int` 大小取决于系统架构（32/64位），不像 Java 固定 32 位。

---

## 5. 运算符

### 5.1 算术运算符

```go
a, b := 10, 3

fmt.Println(a + b)   // 13
fmt.Println(a - b)   // 7
fmt.Println(a * b)   // 30
fmt.Println(a / b)   // 3  ← 整数除法！（类似 Java，Python3 会得到 3.33）
fmt.Println(a % b)   // 1  ← 取余

// 复合赋值
x := 5
x += 3   // x = 8

// 自增/自减（★ Go 特殊）
y := 10
y++      // y = 11
y--      // y = 10
// 注意：Go 中 y++ 是语句，不是表达式！不能写 `z := y++`
```

> **与 Java 对比**：运算符基本一致，但 `++`/`--` 不能作为表达式。  
> **与 Python 对比**：Python 无 `++`/`--`，Go 有但不能用于表达式。

### 5.2 比较运算符

```go
fmt.Println(a == b)  // false
fmt.Println(a != b)  // true
fmt.Println(a < b)   // true
fmt.Println(a >= b)  // false
```

> **与 Python 对比**：Go 无 `is`（身份比较），用 `==` 即可。字符串也可用 `==` 比较内容。

### 5.3 逻辑运算符

```go
c1, c2 := true, false

fmt.Println(c1 && c2)  // false  （与 Java 相同）
fmt.Println(c1 || c2)  // true   （与 Java 相同）
fmt.Println(!c1)       // false  （与 Java 相同）
```

> **与 Python 对比**：Python 用 `and`/`or`/`not`，Go 用 `&&`/`||`/`!`（与 Java/C 相同）。

### 5.4 位运算符

```go
a, b := 12, 10   // 1100, 1010

fmt.Println(a & b)   // 8  (1000)  按位与
fmt.Println(a | b)   // 14 (1110)  按位或
fmt.Println(a ^ b)   // 6  (0110)  按位异或
fmt.Println(^a)      // -13       按位取反（★ 注意不是 ~a）
fmt.Println(a << 2)  // 48        左移
fmt.Println(a >> 2)  // 3         右移
```

> **与 Java 对比**：基本一致，但取反用 `^`（一元）而不是 `~`。  
> **与 Python 对比**：Python 支持任意精度整数，Go 固定位宽。

---

## 6. 输入输出

### 6.1 输出

```go
fmt.Print("无换行")           // 不自动换行
fmt.Println("带换行")         // 自动换行
fmt.Printf("name: %s, age: %d\n", "Go", 14)  // 格式化输出

// 常用格式化动词
// %v   默认格式
// %+v  带字段名（结构体）
// %#v  Go 语法格式
// %T   类型
// %d   十进制整数
// %f   浮点数
// %.2f 保留2位小数
// %s   字符串
// %q   带引号字符串
// %p   指针
// %b   二进制
// %x   十六进制
// %t   布尔值

// 返回格式化字符串（类似 Python f-string / Java String.format）
s := fmt.Sprintf("score: %d", 100)
```

### 6.2 输入

```go
var name string
var age int

fmt.Scan(&name)           // 以空白字符分隔，需传地址（&）
fmt.Scanln(&name)         // 以换行符分隔
fmt.Scanf("%s %d", &name, &age)  // 按格式读取，类似 C 的 scanf
```

> **与 Java 对比**：类似 `Scanner`，但需显式传地址 `&`。  
> **与 Python 对比**：Python 用 `input()` 返回字符串，Go 需指定类型和地址。

---

## 7. 控制结构

### 7.1 if 语句

```go
score := 85

if score >= 90 {                    // ★ 条件无括号！
    fmt.Println("A")
} else if score >= 80 {
    fmt.Println("B")
} else {
    fmt.Println("F")
}

// if with initialization（★ Go 特有）
if age := 25; age >= 18 {           // 在 if 中声明变量，作用域仅在 if 块内
    fmt.Println("Adult")
}
```

> **与 Java 对比**：条件无括号，大括号必须写且换行。  
> **与 Python 对比**：用 `{}` 代替缩进，无 `elif` 用 `else if`。

### 7.2 switch 语句

```go
day := "Monday"

switch day {
case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
    fmt.Println("Weekday")          // ★ 默认不穿透，无需 break！
case "Saturday", "Sunday":
    fmt.Println("Weekend")
default:
    fmt.Println("Invalid")
}

// switch with initialization
switch hour := 14; {
case hour < 12:
    fmt.Println("Morning")
case hour < 18:
    fmt.Println("Afternoon")
default:
    fmt.Println("Evening")
}
```

> **与 Java 对比**：Go 的 `switch` 默认不穿透（无需 `break`），case 可以是表达式。  
> **与 Python 对比**：Python 3.10+ 才有 `match-case`，Go 一直支持。

### 7.3 循环语句（★ Go 只有 for！）

```go
// 标准 for 循环（类似 Java/C）
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// while 风格（★ Go 无 while 关键字）
count := 0
for count < 3 {
    fmt.Println(count)
    count++
}

// 无限循环（★ Go 无 do-while）
for {
    fmt.Println("infinite")
    break
}

// range 循环（★ Go 特有，类似 Python 的 for-in）
fruits := []string{"apple", "banana", "orange"}

for _, fruit := range fruits {      // 只取值，忽略索引
    fmt.Println(fruit)
}

for index, fruit := range fruits {  // 取索引和值
    fmt.Printf("%d: %s\n", index, fruit)
}

// 遍历 map
ages := map[string]int{"Alice": 25, "Bob": 30}
for name, age := range ages {
    fmt.Printf("%s: %d\n", name, age)
}

// 遍历字符串（按 rune）
for i, char := range "Hello 世界" {
    fmt.Printf("pos %d: %c\n", i, char)
}

// 带标签的 break/continue
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break outer    // 跳出外层循环
            }
        }
    }
```

> **与 Java 对比**：Go 无 `while`/`do-while`，全部用 `for` 变体。  
> **与 Python 对比**：Go 的 `range` 类似 Python 的 `enumerate()` 和 `items()`。

---

## 8. 函数

### 8.1 基本定义

```go
// 基本函数（参数类型在后！）
func add(a int, b int) int {
    return a + b
}

// 同类型参数简写
func multiply(a, b int) int {
    return a * b
}

// 多返回值（★ Go 核心特性！）
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// 命名返回值
func calculate(a, b int) (sum, diff int) {
    sum = a + b
    diff = a - b
    return          // naked return，自动返回命名变量
}

// 可变参数
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// 函数作为参数/返回值（一等公民）
func apply(a, b int, op func(int, int) int) int {
    return op(a, b)
}

// 匿名函数
greet := func(name string) string {
    return "Hello, " + name
}
```

> **与 Java 对比**：参数类型在后，支持多返回值（Java 需用数组/对象包装），无函数重载。  
> **与 Python 对比**：无默认参数、无关键字参数、无装饰器。

### 8.2 defer 语句（★ Go 特有）

```go
func readFile(path string) {
    f, err := os.Open(path)
    if err != nil {
        return
    }
    defer f.Close()    // 函数返回时自动执行，LIFO 顺序

    // 读取文件...
}
```

> **注意**：`defer` 的参数在声明时求值，不是执行时。

---

## 9. 数组、切片与 Map

### 9.1 数组（固定长度）

```go
var arr [5]int           // [0 0 0 0 0]
arr2 := [3]int{1, 2, 3}
arr3 := [...]int{1, 2, 3, 4}  // 编译器推断长度
```

### 9.2 切片（动态数组，★ 最常用！）

```go
s := []int{1, 2, 3}      // 切片字面量
s2 := make([]int, 5)     // 长度5，容量5
s3 := make([]int, 3, 10) // 长度3，容量10

s = append(s, 4, 5)      // 追加元素
sub := s[1:3]            // 切片 [2, 3]
```

> **与 Java 对比**：类似 `ArrayList`，但语法更简洁。  
> **与 Python 对比**：类似 Python 列表，但类型固定。

### 9.3 Map（哈希表）

```go
m := make(map[string]int)
m["Alice"] = 25
m["Bob"] = 30

age, ok := m["Alice"]    // ok 为 true 表示存在
if ok {
    fmt.Println(age)
}

delete(m, "Bob")         // 删除键
```

> **与 Java 对比**：类似 `HashMap`，但直接访问不存在的键返回零值，需用 `ok` 判断。  
> **与 Python 对比**：类似 `dict`，但类型固定。

---

## 10. 结构体与方法

### 10.1 结构体（替代 Java 的类）

```go
type Person struct {
    Name string
    Age  int
}

p := Person{Name: "Alice", Age: 25}
p2 := Person{"Bob", 30}    // 按字段顺序

// 指针访问
p.Age = 26
```

### 10.2 方法

```go
// 值接收者
func (p Person) Greet() string {
    return "Hello, I'm " + p.Name
}

// 指针接收者（可修改原对象）
func (p *Person) HaveBirthday() {
    p.Age++
}

p := Person{"Alice", 25}
fmt.Println(p.Greet())      // Hello, I'm Alice
p.HaveBirthday()
fmt.Println(p.Age)          // 26
```

> **与 Java 对比**：Go 无 `class`，用 `struct + 方法` 实现面向对象。无继承，用组合代替。  
> **与 Python 对比**：类似 Python 类，但方法需显式绑定到类型。

---

## 11. 接口

```go
type Speaker interface {
    Speak() string
}

type Dog struct{ Name string }
func (d Dog) Speak() string { return "Woof!" }

// ★ 隐式实现！无需 implements 关键字
var s Speaker = Dog{Name: "Buddy"}
fmt.Println(s.Speak())
```

> **与 Java 对比**：Go 接口是隐式实现，无需 `implements`。  
> **与 Python 对比**：类似 Python 的鸭子类型，但有编译时检查。

---

## 12. 并发基础（Go 核心特性）

```go
// Goroutine（轻量级线程）
go func() {
    fmt.Println("Running in goroutine")
}()

// Channel（通信管道）
ch := make(chan int)
go func() {
    ch <- 42    // 发送
}()
value := <-ch   // 接收
```

> 并发是 Go 的王牌特性，但属于高阶内容，这里仅作引入。

---

## 13. 错误处理（★ Go 风格）

```go
// Go 无 try-catch，用多返回值传递错误
file, err := os.Open("test.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

// 自定义错误
err := errors.New("something went wrong")
err2 := fmt.Errorf("wrapped: %w", err)  // 包装错误
```

> **与 Java 对比**：无 `try-catch-finally`，错误是值，必须显式处理。  
> **与 Python 对比**：无 `raise`/`except`，用返回值传递错误。

---

## 14. 常用标准库速查

```go
import (
    "fmt"       // 格式化 I/O
    "os"        // 操作系统接口
    "strings"   // 字符串操作
    "strconv"   // 字符串转换
    "time"      // 时间处理
    "math"      // 数学函数
    "sort"      // 排序
    "errors"    // 错误处理
)
```

---

## 📌 快速对比表

| 特性 | Go | Java | Python |
|------|-----|------|--------|
| 变量声明 | `var x int` / `x := 1` | `int x = 1;` | `x = 1` |
| 类型推断 | `var x = 1` | `var x = 1;` | 动态类型 |
| 常量 | `const` + `iota` | `final` + `enum` | 无（大写约定） |
| 自增 | `x++`（语句） | `++x`/`x++` | 无 |
| 循环 | 只有 `for` | `for`/`while`/`do-while` | `for`/`while` |
| switch | 默认不穿透 | 需 `break` | 3.10+ `match-case` |
| 函数多返回值 | ✅ 原生支持 | ❌ 需包装 | ✅ 元组 |
| 函数重载 | ❌ 不支持 | ✅ 支持 | ❌ 不支持 |
| 默认参数 | ❌ 不支持 | ✅ 支持 | ✅ 支持 |
| 异常处理 | `error` 返回值 | `try-catch` | `try-except` |
| 类/继承 | `struct` + 组合 | `class` + 继承 | `class` + 继承 |
| 接口 | 隐式实现 | 显式 `implements` | 鸭子类型 |
| 并发 | Goroutine + Channel | Thread + Executor | Thread / asyncio |
| 泛型 | Go 1.18+ | ✅ | ✅ |

---

> 💡 **学习建议**：Go 的设计哲学是"少即是多"。摒弃了类继承、异常、泛型（早期）、函数重载等特性，用更简洁的方式解决问题。建议多写代码，熟悉 `:=`、`range`、`defer` 等 Go 特有的语法糖。

希望这份教程对你有帮助！🎉