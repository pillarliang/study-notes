# Chapter 5：函数（Functions）

本章把 Go 的视角从"控制流"推进到"代码复用"。Go 的函数与其他支持一等函数的语言（C、Python、Ruby、JavaScript）相似，但在多返回值、命名返回值、`defer`、值传递语义上有自己的设计取舍。理解这些差异，比记住语法更重要。

---

## 1. 声明与调用

**语法骨架：**

```go
func 函数名(参数1 类型1, 参数2 类型2, ...) 返回类型 {
    // 函数体
    return 返回值
}
```

下面分别展开 4 个组成部分、约束规则，以及同类型参数的简写。

### 1.1 函数的四个组成部分

Go 函数声明由四部分组成：

| 部分 | 说明 |
| --- | --- |
| 关键字 `func` | 固定起始 |
| 函数名 | 包内唯一 |
| 输入参数 | 括号内，**名字在前、类型在后**，逗号分隔 |
| 返回类型 | 写在参数右括号与函数体左花括号之间 |

```go
func div(num int, denom int) int {
    if denom == 0 {
        return 0
    }
    return num / denom
}
```

### 1.2 关键约束

- Go 是强类型语言，**必须**显式标注参数类型。
- 有返回值的函数**必须**有 `return`；无返回值的函数仅在需要提前退出时才写 `return`。
- 无参数时写空括号 `()`；无返回值时返回类型位置直接留空。

### 1.3 同类型参数的简写

连续多个同类型参数可只写一次类型：

```go
func div(num, denom int) int { ... }
```

---

## 2. 模拟命名参数与可选参数

**Go 没有命名参数和可选参数。** 调用时必须按顺序提供所有参数。

需要时的标准做法：定义一个 struct，把"可选/可命名"语义放进字段里。

```go
type MyFuncOpts struct {
    FirstName string
    LastName  string
    Age       int
}

func MyFunc(opts MyFuncOpts) error { ... }

MyFunc(MyFuncOpts{LastName: "Patel", Age: 50})
MyFunc(MyFuncOpts{FirstName: "Joe", LastName: "Smith"})
```

> **设计原则**：参数超过几个就该警觉。命名/可选参数主要在参数过多时才显出价值，而参数过多本身就是函数职责过宽的信号。

---

## 3. 变长参数（Variadic）与切片

**语法骨架：**

```go
// 声明
func 函数名(普通参数 T1, vs ...T) 返回类型 {
    // 函数体内 vs 是 []T 切片
}

// 调用
fn(普通参数, v1, v2, v3)         // 直接列出多个值
fn(普通参数, slice...)            // 把 slice 展开（... 放在变量/字面量后）
fn(普通参数)                      // 传 0 个变长参数也合法
```

### 3.1 规则

- 用 `...T` 表示变长参数。
- 变长参数**必须是参数列表中最后一个**。
- 在函数内部，变长参数是一个 `[]T` 切片，可以为零长度。

```go
func addTo(base int, vals ...int) []int {
    out := make([]int, 0, len(vals))
    for _, v := range vals {
        out = append(out, base+v)
    }
    return out
}
```

### 3.2 调用方式

```go
addTo(3)                        // 传 0 个
addTo(3, 2)                     // 传 1 个
addTo(3, 2, 4, 6, 8)            // 传多个
a := []int{4, 3}
addTo(3, a...)                  // 把 slice 展开传入
addTo(3, []int{1, 2, 3, 4, 5}...)
```

> 把 slice 当作变长参数传入时，**`...` 必须放在变量或字面量之后**，否则编译错误。

---

## 4. 多返回值

**语法骨架：**

```go
// 声明：返回类型必须用 () 括起来
func 函数名(参数...) (T1, T2, T3) {
    return v1, v2, v3                // 逗号分隔，不加括号
}

// 调用：必须逐个接收（与 Python 不同，不会自动打包成 tuple）
v1, v2, v3 := 函数名(...)
v1, _, v3 := 函数名(...)              // 用 _ 忽略不需要的
```

### 4.1 语法

返回多个值时，**返回类型必须用括号包起来**，`return` 也必须返回全部值（用逗号分隔，**不要**给返回值加括号）：

```go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

**返回类型何时需要括号：**

| 场景          | 是否需要括号  | 示例                                                      |
| ----------- | ------- | ------------------------------------------------------- |
| 单返回值，不命名    | 可省      | `func f() int`                                          |
| 单返回值，命名     | **必须加** | `func f() (n int)`                                      |
| 多返回值，不论是否命名 | **必须加** | `func f() (int, error)` / `func f() (n int, err error)` |


调用：

```go
result, remainder, err := divAndRemainder(5, 2)
if err != nil {
    fmt.Println(err)
    os.Exit(1)
}
```

### 4.2 与 Python 的区别

Python 的"多返回值"其实是**返回一个 tuple**，可以解构、也可以整体接收。Go 不是 —— Go 的多返回值是真的多个独立值，**必须逐个接收**，否则编译错误。

```python
v = div_and_remainder(5, 2)   # Python 合法：v = (2, 1)
```

```go
v := divAndRemainder(5, 2)    // Go 编译错误
```

### 4.3 约定

- `error` 永远是返回值列表中的**最后一个**。
- 函数成功时返回 `nil` 作为 error。

### 4.4 忽略返回值

不需要的返回值赋给 `_`：

```go
result, _, err := divAndRemainder(5, 2)
```

也可以**完全省略接收**（如 `fmt.Println` 实际上有返回值，惯例是不接收）。但除非是这种约定俗成的场景，否则应显式用 `_` 表明意图。

---

## 5. 命名返回值（Named Return Values）

**语法骨架：**

```go
func 函数名(参数...) (name1 T1, name2 T2, err error) {
    // name1, name2, err 已经预声明并初始化为零值
    name1 = ...
    return name1, name2, err         // 显式 return，推荐
    // return                         // naked return，不推荐（见 §5.4）
}
```

括号规则见 §4.1：只要返回值带名字，哪怕只有一个，也必须加括号。

### 5.1 机制

在返回类型列表中给返回值起名，相当于在函数体开头**预声明这些变量并初始化为零值**：

```go
func divAndRemainder(num, denom int) (result int, remainder int, err error) {
    if denom == 0 {
        err = errors.New("cannot divide by zero")
        return result, remainder, err
    }
    result, remainder = num/denom, num%denom
    return result, remainder, err
}
```

- 只想给部分返回值起名时，对其余位置写 `_`。
- 命名仅在函数体内有效，**不约束调用方**接收时使用什么变量名。

### 5.2 两个潜在陷阱

**陷阱 1：shadowing。** 在内层 block 用 `:=` 时容易意外新建同名变量，遮住外层的命名返回值，最终返回的不是想返回的那个。

**陷阱 2：可以"言行不一"。** 函数体里给命名返回值赋了值，`return` 时写不同的字面量，**实际返回的是 `return` 语句中的字面量**。编译器会把 `return` 表达式赋给命名返回值再返回。

```go
func divAndRemainderConfusing(num, denom int) (result int, remainder int, err error) {
    result, remainder = 20, 30          // 这两行被覆盖
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil // 真正返回的是这一行的值
}
```

调用 `divAndRemainderConfusing(5, 2)` 输出 `2 1`，而非 `20 30`。

### 5.3 何时该用

命名返回值的真正用途只有一处：**配合 `defer` 修改/读取返回值**（见 §10）。其他场景下，文档作用有限，反而带来 shadowing 和"言行不一"两类隐患。

### 5.4 空 `return`（Naked Return）—— 不要用

命名返回值允许写不带表达式的 `return`，自动返回当前命名变量的值：

```go
func divAndRemainder(num, denom int) (result int, remainder int, err error) {
    if denom == 0 {
        err = errors.New("cannot divide by zero")
        return                  // naked return
    }
    result, remainder = num/denom, num%denom
    return
}
```

> **原则**：函数有返回值时**永远不要用 naked return**。读者必须回溯整个函数体才能搞清楚"实际返回的是什么"，可读性大幅下降。

---

## 6. 函数即值（Functions Are Values）

### 6.1 函数签名就是类型

函数类型由 `func` 关键字加参数类型与返回类型构成，称为**签名（signature）**。任何参数数量、类型、返回类型都一致的函数，都满足同一个签名。

```go
var myFuncVariable func(string) int
```

`myFuncVariable` 可以接受任何 `func(string) int` 签名的函数。

### 6.2 零值是 `nil`

函数变量的零值是 `nil`，调用 `nil` 函数会 **panic**。

### 6.3 用 map 把函数当数据

由于函数是值，可以放进 map、slice，按 key 查找后调用。

```go
var opMap = map[string]func(int, int) int{
    "+": func(i, j int) int { return i + j },
    "-": func(i, j int) int { return i - j },
    "*": func(i, j int) int { return i * j },
    "/": func(i, j int) int { return i / j },
}

op := "+"
if f, ok := opMap[op]; ok {
    fmt.Println(f(2, 3))
}
```

### 6.4 别写脆弱程序

书里给出的简易计算器示例，22 行核心循环里有 16 行在做错误检查与数据校验。**错误处理是专业代码与业余代码的分水岭**，不要为了短而省略。

---

## 7. 函数类型声明

可以用 `type` 给函数类型起名，便于复用与文档化：

```go
type opFuncType func(int, int) int

var opMap = map[string]opFuncType{
    // 函数本身完全不用改，只要签名匹配就能赋值
}
```

任何 `func(int, int) int` 的函数都自动满足 `opFuncType`。后续在第 7 章会看到，函数类型还能成为"通向 interface 的桥梁"。

---

## 8. 匿名函数（Anonymous Functions）

**语法骨架：**

```go
// 形式 1：赋给变量后调用
f := func(参数) 返回类型 {
    // 函数体
}
f(args)

// 形式 2：就地定义、就地调用（IIFE）
func(参数) 返回类型 {
    // 函数体
}(args)
```

匿名函数 = `func` 关键字 + 参数 + 返回值 + 函数体，**没有名字**。

```go
f := func(j int) {
    fmt.Println("printing", j, "from inside of an anonymous function")
}
for i := 0; i < 5; i++ {
    f(i)
}
```

也可以**就地定义、就地调用**（IIFE 风格）：

```go
for i := 0; i < 5; i++ {
    func(j int) {
        fmt.Println("printing", j, "from inside of an anonymous function")
    }(i)
}
```

> 立即执行的匿名函数几乎没有意义（直接写代码即可）。匿名函数真正派上用场的两个场景：**`defer` 语句**与**启动 goroutine**。

### 包级匿名函数

也可以在包级别用 `var` 把匿名函数赋给变量。**这与普通 `func` 声明的关键差异是：包级匿名函数变量的值可以被重新赋值**。能力即风险——包级状态应当不可变，否则数据流难以追踪。

---

## 9. 闭包（Closures）

### 9.1 定义

声明在另一个函数内部的函数，能**读取与修改外层函数的变量**，称为闭包。

```go
func main() {
    a := 20
    f := func() {
        fmt.Println(a)
        a = 30                  // 修改外层的 a
    }
    f()
    fmt.Println(a)              // 30
}
```

注意：使用 `=` 是修改外层变量；换成 `a := 30` 则是在闭包内**新建**一个 a，shadow 外层。多变量赋值时尤其要看清楚是 `=` 还是 `:=`。

### 9.2 闭包的两类用途

**用途 1：限制作用域。** 把只在当前函数内被多次调用的子函数声明为闭包，避免污染包级命名空间。

**用途 2：把函数内的局部状态"带出去"。** 闭包能把外层局部变量"封装"进函数里，再把这个函数传给别人或 return 出去 —— 这才是闭包真正强大之处。

### 9.3 把闭包传给其他函数：排序

`sort.Slice` 接受任意 slice 与一个比较函数，比较函数通过闭包**捕获**被排序的 slice：

```go
people := []Person{
    {"Pat", "Patterson", 37},
    {"Tracy", "Bobdaughter", 23},
    {"Fred", "Fredson", 18},
}

sort.Slice(people, func(i, j int) bool {
    return people[i].LastName < people[j].LastName
})

sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
```

闭包内的 `people` 不是参数，而是从外层"捕获"过来的。同一份数据，用不同闭包就能得到不同排序。`sort.Slice` 会**就地修改** slice（与 §11 的值传递语义有关）。

### 9.4 从函数返回闭包

```go
func makeMult(base int) func(int) int {
    return func(factor int) int {
        return base * factor
    }
}

twoBase := makeMult(2)
threeBase := makeMult(3)
for i := 0; i < 3; i++ {
    fmt.Println(twoBase(i), threeBase(i))
}
// 0 0
// 2 3
// 4 6
```

`twoBase` 和 `threeBase` 各自捕获了一份独立的 `base`。返回闭包是构建中间件、`sort.Search`、资源清理等模式的基础。

> 接受函数为参数 / 返回函数的函数，称为 **higher-order function（高阶函数）**。

---

## 10. `defer`

**语法骨架：**

```go
defer 函数调用表达式              // 后面必须是"调用"，不能只写函数引用

// 几种典型形式：
defer f.Close()                   // 方法调用
defer cleanupFunc(args)           // 普通函数调用
defer func() { ... }()            // 匿名函数立即调用（常用于读写命名返回值）
```

### 10.1 用途

释放临时资源（文件、网络连接、锁、数据库事务等）。`defer` 把清理代码"贴在"函数上，无论函数从哪条路径退出，都会执行。

```go
f, err := os.Open(os.Args[1])
if err != nil {
    log.Fatal(err)
}
defer f.Close()
```

### 10.2 关键规则

| 规则 | 说明 |
| --- | --- |
| 接受 函数 / 方法 / 闭包 | 三者皆可 |
| 多个 `defer` 按 **LIFO** 执行 | 最后注册的先执行 |
| 参数**立即求值**，函数体延迟执行 | `defer f(x)` 中 `x` 在 defer 当下取值，存起来等函数运行时使用 |
| `defer` 函数体在 `return` **之后**执行 | 这是它能修改命名返回值的前提 |
| 函数后必须有括号 | `defer f.Close()` 而不是 `defer f.Close` |

### 10.3 典型示例：参数立即求值 + LIFO

```go
func deferExample() int {
    a := 10
    defer func(val int) { fmt.Println("first:", val) }(a)
    a = 20
    defer func(val int) { fmt.Println("second:", val) }(a)
    a = 30
    fmt.Println("exiting:", a)
    return a
}
// 输出：
// exiting: 30
// second: 20
// first: 10
```

两个匿名函数都把 `a` 当时的值（10、20）"冻结"在自己的 `val` 参数里。

### 10.4 用 `defer` 修改返回值（命名返回值的真正用途）

`defer` 中的闭包能读取与修改**命名返回值**。这是处理事务回滚 / 提交的标准模式：

```go
func DoSomeInserts(ctx context.Context, db *sql.DB, value1, value2 string) (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if err == nil {
            err = tx.Commit()       // 可能改写 err
        }
        if err != nil {
            tx.Rollback()
        }
    }()
    _, err = tx.ExecContext(ctx, "INSERT INTO FOO (val) values $1", value1)
    if err != nil {
        return err
    }
    return nil
}
```

闭包读 `err` 决定 commit 还是 rollback，再把 commit 的错误写回 `err`。**没有命名返回值就拿不到这个 `err`**。

### 10.5 "返回闭包做清理"惯用法

> 当一个函数获取了资源，**最好同时返回一份清理用的闭包**。让调用方对这份资源的清理责任显式可见。

```go
func getFile(name string) (*os.File, func(), error) {
    file, err := os.Open(name)
    if err != nil {
        return nil, nil, err
    }
    return file, func() { file.Close() }, nil
}

f, closer, err := getFile(os.Args[1])
if err != nil {
    log.Fatal(err)
}
defer closer()
```

由于 Go 不允许未使用的变量，`closer` 必须被显式调用，否则编译失败 —— 这相当于编译器在提醒"别忘了 defer"。

### 10.6 与 try/finally 的对比

Java/Ruby/Python 用 try-finally / begin-rescue-ensure 等"块结构"管理资源，会增加缩进层级。研究表明，影响代码复杂度的主要因素是**嵌套深度**与**缺乏结构**。`defer` 把清理代码与资源获取放在同一缩进层级，可读性更佳。

---

## 11. Go 是值传递（Call by Value）

### 11.1 基本规则

> Go **永远**把参数复制一份后传入函数。函数内对参数的修改不会影响调用方。

```go
type person struct {
    age  int
    name string
}

func modifyFails(i int, s string, p person) {
    i = i * 2
    s = "Goodbye"
    p.name = "Bob"
}

func main() {
    p := person{}
    i, s := 2, "Hello"
    modifyFails(i, s, p)
    fmt.Println(i, s, p)        // 2 Hello {0 }
}
```

对**基本类型**和 **struct**，行为如直觉所料 —— 函数无法修改外部变量。

### 11.2 map 与 slice 的特殊行为

被调函数：

```go
func modMap(m map[int]string) {
    m[2] = "hello"
    m[3] = "goodbye"
    delete(m, 1)             // 内建函数，从 map 删除 key 为 1 的项
}

func modSlice(s []int) {
    for k, v := range s {
        s[k] = v * 2
    }
    s = append(s, 10)        // 这一步对调用方不可见。是否"分家"取决于 append 当时是否触发扩容；扩容则永远分家，未扩容则继续共享底层数组。
}
```

调用方：

```go
func main() {
    m := map[int]string{
        1: "first",
        2: "second",
    }
    modMap(m)
    fmt.Println(m)           // map[2:hello 3:goodbye]

    s := []int{1, 2, 3}
    modSlice(s)
    fmt.Println(s)           // [2 4 6]
}
```

输出对照：

| 外层原值 | 函数内做的事 | 外层调用后 | 是否生效 |
| --- | --- | --- | --- |
| `m = {1:"first", 2:"second"}` | 写入 key 2、3，删除 key 1 | `{2:"hello", 3:"goodbye"}` | ✅ 全部生效 |
| `s = [1, 2, 3]` | 元素 ×2，再 `append(s, 10)` | `[2, 4, 6]` | 元素修改 ✅；append 增长 ❌ |

| 类型 | 函数内的修改是否影响外部 |
| --- | --- |
| 基本类型、struct | 否（一律复制） |
| map | 是（任意修改都生效） |
| slice 元素修改 | 是 |
| slice 长度变化（`append` 等） | 否 |

### 11.3 原因与统一解释

map 与 slice 在底层都是**用指针实现的复合结构**。复制的是结构本身（包含底层数组指针），但底层数据共享，所以"内部修改"会被看到；而 `append` 可能让 slice 重新指向新底层数组，这一变化只发生在副本身上，外部看不到。**append 一旦触发扩容，本地 slice 与外层 slice 就"分家"了**——之后函数内对这个 slice 的任何元素修改都只动新数组，外层永远看不到。

> **Go 中所有类型都是值类型，只是有些"值"内部包含指针。**

### 11.4 设计含义

值传递让数据流更易追踪：除非传入 slice / map / 指针，调用方可以确信函数不会改自己的变量。这是 Go 推崇"返回新值而非修改入参"的语言层基础。需要可变共享时，下章的指针登场。

---

## 12. 小结

本章的关键脉络：

1. 函数是**值**，签名即类型 —— 解锁高阶函数、map 派发、中间件等模式。
2. 多返回值 + `error` 末位约定，是 Go 错误处理的语法基石。
3. 命名返回值与 naked return 多数时候是诱导陷阱，**唯一真正用途**是配合 `defer` 修改返回值。
4. 闭包让函数把局部状态"打包"带走，是排序、资源清理、状态机等模式的核心机制。
5. `defer` 把资源清理与获取写在同一处，参数立即求值、函数体 LIFO、`return` 后执行 —— 三条规则覆盖几乎全部用法。
6. 值传递是 Go 的统一语义；map 与 slice 看似"引用传递"，实质是结构内部含指针。

下一章进入指针，重新审视"何时该用引用语义"。
