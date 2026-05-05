# 第 7 章 类型、方法与接口（Types, Methods, and Interfaces）

Go 是一门静态类型语言，既有内置类型也允许自定义类型，并支持给类型挂方法。但在面向对象语义上，Go 走了一条与主流语言不同的路线：**鼓励组合，避免继承；接口隐式实现，而非显式声明**。本章围绕"类型 → 方法 → 接口 → 依赖注入"层层展开。

---

## 1. 类型声明

### 1.1 语法骨架

```go
type 类型名 底层类型
```

- 底层类型可以是：内置类型、struct 字面量、函数签名、复合类型字面量。
- 同一类型可在任意 block 声明（package 级以下也可），但作用域受 block 限制；跨包访问只能用导出类型（见第 10 章）。

### 1.2 示例

```go
type Person struct {           // 底层是 struct 字面量
    FirstName string
    LastName  string
    Age       int
}

type Score        int                       // 底层是内置类型
type Converter    func(string) Score        // 底层是函数类型
type TeamScores   map[string]Score          // 底层是 map 类型
```

### 1.3 抽象类型与具体类型

- **抽象类型（abstract type）**：只规定要做什么，不规定怎么做（如接口）。
- **具体类型（concrete type）**：既规定做什么，也规定怎么做（既有数据存储方式，也有方法实现）。
- Go 中所有类型非抽象即具体；不像 Java 那样有"抽象类"或带默认方法的接口这种混合形态。

---

## 2. 方法

### 2.1 语法骨架

```go
func (接收者名 接收者类型) 方法名(参数) 返回值 { ... }
```

- 接收者（receiver）位于 `func` 与方法名之间，惯例命名为类型名首字母小写，**不要用 `this` 或 `self`**。
- 方法只能定义在 **package block** 上，函数则可以在任意 block。
- 方法不能重载：同一类型上不能有同名方法，但不同类型间方法名可以重复。
- 方法必须与类型在同一包中，因此**不能给第三方类型加方法**。

### 2.2 示例

```go
func (p Person) String() string {
    return fmt.Sprintf("%s %s, age %d", p.FirstName, p.LastName, p.Age)
}

p := Person{FirstName: "Fred", LastName: "Fredson", Age: 52}
output := p.String()
```

### 2.3 何时用方法、何时用函数

判断标准：**逻辑是否依赖外部状态**。

- 依赖启动时配置、可变状态 → 把状态放进 struct，写成方法。
- 仅依赖入参 → 写成函数。
- 包级别变量应当尽量保持不可变；可变状态需收纳进类型。

---

## 3. 指针接收者与值接收者

### 3.1 选择规则

| 情况                     | 接收者类型             |
| ------------------------ | ---------------------- |
| 方法需要修改接收者       | **必须** 指针接收者    |
| 方法需处理 `nil` 接收者  | **必须** 指针接收者    |
| 方法不修改接收者         | 可以用值接收者         |
| 类型已存在指针接收者方法 | 推荐**所有方法**统一用指针接收者，保持一致性 |

### 3.2 自动取址 / 自动解引用

```go
var c Counter
c.Increment()              // Counter 是值类型，但 Increment 是 *Counter 方法
                            // Go 自动改写为 (&c).Increment()

c := &Counter{}
c.String()                 // 指针上调用值接收者方法
                            // Go 自动改写为 (*c).String()
```

> ⚠️ 这种自动转换只是**语法糖**，与 method set 概念是两回事。

### 3.3 函数参数中的"拷贝陷阱"

值传给函数后，得到的是副本；在副本上调用指针接收者方法，**修改不会回传**：

```go
func doUpdateWrong(c Counter)  { c.Increment() } // 改的是 c 的副本
func doUpdateRight(c *Counter) { c.Increment() } // 改的是 main 中的 c

var c Counter
doUpdateWrong(c)        // main 中 c 不变
doUpdateRight(&c)       // main 中 c 被修改
```

### 3.4 method set 概念

- 指针实例的 method set 包含 **指针接收者方法 + 值接收者方法**。
- 值实例的 method set 仅包含 **值接收者方法**。
- 该规则在判断"类型是否实现接口"时至关重要（见 §7.2）。

### 3.5 nil 接收者

- 值接收者上调用 `nil` → panic（无值可拷）。
- 指针接收者上调用 `nil` → 方法被实际调用，由方法体决定是否处理 `nil`。

允许 `nil` 接收者的典型例子：二叉树插入。

```go
type IntTree struct {
    val         int
    left, right *IntTree
}

func (it *IntTree) Insert(val int) *IntTree {
    if it == nil {
        return &IntTree{val: val}
    }
    if val < it.val {
        it.left = it.left.Insert(val)
    } else if val > it.val {
        it.right = it.right.Insert(val)
    }
    return it
}

func (it *IntTree) Contains(val int) bool {
    switch {
    case it == nil:        return false
    case val < it.val:     return it.left.Contains(val)
    case val > it.val:     return it.right.Contains(val)
    default:               return true
    }
}
```

> 注意：指针接收者类似指针参数——传进来的是指针副本。让 `nil` 接收者方法把"原始变量"变成非 `nil` 是做不到的（修改的是指针副本）。

### 3.6 不要写 getter/setter

Go 鼓励直接访问字段。仅当为满足接口、需要原子更新多字段或更新逻辑非平凡时才写方法。

---

## 4. 方法即函数

方法可被当作函数值使用，这为依赖注入打下基础。

### 4.1 method value（绑定实例）

```go
type Adder struct{ start int }
func (a Adder) AddTo(val int) int { return a.start + val }

myAdder := Adder{start: 10}
f1 := myAdder.AddTo       // f1 类型: func(int) int
fmt.Println(f1(10))       // 20
```

method value 像闭包，捕获了实例的字段。

### 4.2 method expression（不绑定实例）

```go
f2 := Adder.AddTo         // f2 类型: func(Adder, int) int
fmt.Println(f2(myAdder, 15)) // 25
```

接收者作为第一个参数显式传入。

---

## 5. 类型声明不是继承

### 5.1 同底层类型 ≠ 父子关系

```go
type Score int
type HighScore Score
```

`HighScore` 与 `Score` 仅共享底层类型，**没有继承关系**：

- 两者之间赋值需显式 `类型转换`。
- `Score` 上的方法对 `HighScore` 不可见。
- 但都能与字面量、底层类型常量做兼容运算。

```go
var i int = 300
var s Score = 100
var hs HighScore = 200

hs = s         // 编译错误
s  = i         // 编译错误
s  = Score(i)        // ok
hs = HighScore(s)    // ok

scoreWithBonus := s + 100   // ok，结果仍是 Score
```

### 5.2 类型即文档

声明 `Percentage` 比直接用 `int` 更清晰，编译器还能阻止错误用法。**当数据相同但操作集合不同时，应定义不同类型**。

---

## 6. iota 与枚举

### 6.1 适用前提

Go 没有原生 enum；用 `iota` 给一组常量自动递增赋值。

> 仅当**值是什么不重要、只关心是否能区分**时才用 `iota`。一旦数值会序列化到外部系统（数据库、协议字段），就**显式写值**。

### 6.2 语法骨架

```go
type MailCategory int      // 先定义底层 int 的类型

const (
    Uncategorized MailCategory = iota   // 0
    Personal                             // 1
    Spam                                 // 2
    Social                               // 3
    Advertisements                       // 4
)
```

- `iota` 在每个 `const` 块从 0 重置。
- 块内每行 `iota` 自增，**无论该行是否使用**。
- 后续行无显式赋值会沿用上一行的 *表达式模板* 与类型。

### 6.3 易错示例

```go
const (
    Field1 = 0
    Field2 = 1 + iota   // iota=1 → 2
    Field3 = 20
    Field4              // 复用上一行模板 → 20（与 iota 无关）
    Field5 = iota       // iota=4 → 4
)
// 输出：0 2 20 20 4
```

### 6.4 位标志惯用法

```go
type BitField int
const (
    Field1 BitField = 1 << iota // 1
    Field2                      // 2
    Field3                      // 4
    Field4                      // 8
)
```

强大但脆弱：中间插入新常量会让所有后续值漂移。

---

## 7. 嵌入实现组合

### 7.1 语法

在 struct 中只写类型名、不给字段名 → **嵌入字段**（embedded field）：

```go
type Employee struct {
    Name string
    ID   string
}
func (e Employee) Description() string { return fmt.Sprintf("%s (%s)", e.Name, e.ID) }

type Manager struct {
    Employee                  // 嵌入字段，无字段名
    Reports []Employee
}
```

### 7.2 字段/方法提升（promotion）

被嵌入类型的字段与方法**被提升到外层**，可直接访问：

```go
m := Manager{Employee: Employee{Name: "Bob", ID: "12345"}}
m.ID            // "12345"
m.Description() // "Bob (12345)"
```

> 嵌入对象不限于 struct，任何类型都可被嵌入。

### 7.3 同名遮蔽

外层与嵌入字段同名时，外层优先；要访问被遮蔽的字段须显式写嵌入类型名：

```go
type Inner struct{ X int }
type Outer struct {
    Inner
    X int
}

o := Outer{Inner: Inner{X: 10}, X: 20}
o.X         // 20
o.Inner.X   // 10
```

### 7.4 嵌入 ≠ 继承

**关键区别**：

- 不能把 `Manager` 赋给 `Employee` 类型的变量（编译错误）。
- 没有动态分派（dynamic dispatch）。嵌入字段的方法调用另一个方法时，调用的永远是**嵌入类型自己的方法**，不会因外层重写而改变。

```go
type Inner struct{ A int }
func (i Inner) IntPrinter(val int) string  { return fmt.Sprintf("Inner: %d", val) }
func (i Inner) Double() string             { return i.IntPrinter(i.A * 2) }   // 调用的永远是 Inner.IntPrinter

type Outer struct {
    Inner
    S string
}
func (o Outer) IntPrinter(val int) string { return fmt.Sprintf("Outer: %d", val) }

o := Outer{Inner: Inner{A: 10}, S: "Hello"}
o.Double()   // 输出 "Inner: 20"，不是 "Outer: 20"
```

但**嵌入字段的方法会计入外层 method set**——这就让外层自动满足接口。

---

## 8. 接口

### 8.1 语法骨架

```go
type 接口名 interface {
    方法签名1
    方法签名2
}
```

- 接口列出"具体类型必须实现的方法集合"，即接口的 method set。
- 通常以 `-er` 结尾命名：`Reader`、`Writer`、`Stringer`、`Closer`。
- 标准库 `fmt.Stringer` 定义：

```go
type Stringer interface {
    String() string
}
```

### 8.2 隐式实现：类型安全的鸭子类型

**核心特性**：具体类型不需要声明 `implements`，只要 method set 包含接口的全部方法，就自动满足该接口。

- 兼具静态语言的**类型安全**与动态语言的**解耦灵活**。
- 接口由**调用方**定义，描述"我需要什么能力"，而非提供方宣称"我能做什么"。

```go
// LogicProvider 是具体实现方，不显式声明实现 Logic
type LogicProvider struct{}
func (lp LogicProvider) Process(data string) string { /* ... */ }

// 调用方定义接口
type Logic interface { Process(data string) string }

type Client struct{ L Logic }
func (c Client) Program() { c.L.Process(/* data */) }
```

### 8.3 method set 决定可不可赋

```go
type Counter struct{ /* ... */ }
func (c *Counter) Increment()      { /* 指针接收者 */ }
func (c  Counter) String() string  { /* 值接收者   */ }

type Incrementer interface { Increment() }

var ptrC = &Counter{}      // 指针实例：method set 含 Increment + String
var valC = Counter{}       // 值实例：method set 仅含 String

var inc Incrementer
inc = ptrC                 // ok
inc = valC                 // 编译错误：Counter 不实现 Incrementer
```

### 8.4 接口嵌入

接口里也可以嵌入接口：

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Closer interface { Close() error }

type ReadCloser interface {
    Reader
    Closer
}
```

---

## 9. "接受接口，返回结构体"

### 9.1 原则

- **入参用接口**：声明所需能力，灵活解耦。
- **返回值用具体类型**：方便日后增加字段/方法而不破坏现有调用——具体类型属于向后兼容的修改，给接口加方法则会破坏所有实现。

### 9.2 例外

- `error` 是接口：因为不同实现差异大，必须用接口承载。
- `database/sql/driver`：要求向后兼容又要演进 API，只好继续返回接口（用类型断言挑可选能力）。

### 9.3 性能权衡

- 返回 struct 走栈分配；返回 interface 通常触发堆分配。
- 优先可读性与可维护性，性能问题需先 profile 再权衡。

---

## 10. 接口与 nil

### 10.1 接口的内部实现

接口在 runtime 表现为一个二字段结构：

```
| type pointer | value pointer |
```

**只有 type 和 value 都为 nil，接口才为 nil**。

### 10.2 经典坑

```go
var pointerCounter *Counter           // pointerCounter == nil → true
var incrementer Incrementer           // incrementer == nil    → true
incrementer = pointerCounter          // 现在 type=*Counter, value=nil
fmt.Println(incrementer == nil)       // false ⚠️
```

- 接口非 `nil` 时调方法不会自动 panic；但若值为 `nil` 且方法没处理 `nil` 接收者，仍会 panic。
- 想判断接口"内部值是否 nil"必须用 reflect。

---

## 11. 接口的可比性陷阱

接口可用 `==` / `!=` 比较：当且仅当 type 与 value 都相等时才相等。但**若底层 value 类型不可比较（如 slice、map、含此类字段的 struct），运行时会 panic**：

```go
type DoubleIntSlice []int
func (d DoubleIntSlice) Double() { /* ... */ }

var dis  DoubleIntSlice = []int{1, 2, 3}
var dis2 DoubleIntSlice = []int{1, 2, 3}

func DoublerCompare(d1, d2 Doubler) { fmt.Println(d1 == d2) }

DoublerCompare(dis, dis2)   // panic: comparing uncomparable type
```

> map 以接口为 key 时同理；不可比较的 key 会触发 panic。需要绝对安全可借 `reflect.Value.Comparable` 预先检查。

---

## 12. 空接口与 `any`

### 12.1 概念

`interface{}` 不要求任何方法，因此**任何类型都满足**。Go 1.18 起引入别名 `any`，新代码统一用 `any`。

### 12.2 典型用途

存放结构未知的数据（如反序列化 JSON）：

```go
data := map[string]any{}
contents, err := os.ReadFile("testdata/sample.json")
json.Unmarshal(contents, &data)
```

### 12.3 局限

- 空接口本身没法直接做事；要取回原值得用类型断言或反射。
- Go 鼓励强类型，应**尽量避免**到处用 `any`。

---

## 13. 类型断言与 type switch

### 13.1 类型断言（type assertion）

```go
var i any = MyInt(20)

i2 := i.(MyInt)         // 断言成功
i3 := i.(string)        // panic: interface conversion ...
```

- 断言**揭示**接口内部存的具体类型，与"类型转换"语义不同：转换改变值，断言只是查询。
- 即使两类型有相同底层类型，断言也必须**精确匹配**。
- 断言运行期检查，可能 panic；类型转换编译期检查，不通过则编译失败。

### 13.2 comma-ok 安全写法

```go
i2, ok := i.(int)
if !ok {
    return fmt.Errorf("unexpected type for %v", i)
}
fmt.Println(i2 + 1)
```

> **即使确信类型正确也应使用 comma-ok**——代码会被复用、被改动，不带保护的断言迟早出事。

### 13.3 type switch

需要在多种可能的具体类型间分派时：

```go
func doThings(i any) {
    switch j := i.(type) {
    case nil:
        // i 是 nil，j 类型为 any
    case int:
        // j 是 int
    case MyInt:
        // j 是 MyInt
    case io.Reader:
        // j 是 io.Reader
    case string:
        // j 是 string
    case bool, rune:
        // 多 case 合并 → j 类型为 any
    default:
        // 未知类型 → j 类型为 any
    }
}
```

惯例：把被检查的变量"shadow"成同名（`i := i.(type)`），让代码更紧凑。`default` 分支务必保留以兜底未来新增的实现类型。

### 13.4 慎用

- 优先把入参/返回值当作"声明的类型"对待，不要去刺探别的可能类型；否则 API 含义就模糊了。
- 合理用例：
  - **可选接口**（optional interface）：检查具体类型是否还实现了一个增强接口。
  - **API 演进**：如 `database/sql` 在 Go 1.8 增加 `StmtExecContext`，老代码继续走旧路径。

```go
// io.Copy 用类型断言探测优化路径
if wt, ok := src.(WriterTo); ok { return wt.WriteTo(dst) }
if rt, ok := dst.(ReaderFrom); ok { return rt.ReadFrom(src) }
```

> ⚠️ 装饰器/包装器会让"可选接口"被掩盖（包装层可能没实现）。包装 error 同理——所以才有 `errors.Is` / `errors.As`。

---

## 14. 函数类型实现接口

允许给**任何**自定义类型挂方法，包括底层为函数的类型。`net/http` 是经典例子：

```go
type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}

type HandlerFunc func(http.ResponseWriter, *http.Request)
func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f(w, r)
}
```

任何符合 `func(ResponseWriter, *Request)` 签名的函数，转成 `HandlerFunc` 后立即满足 `Handler`。

**何时用函数类型，何时用接口？**

- 函数仅依赖入参 → 用函数类型参数。
- 函数依赖大量其他状态/方法 → 用接口；可顺便定义函数类型作"桥接"，把单函数也接进同一接口。

---

## 15. 隐式接口让依赖注入更轻松

### 15.1 思路

- **依赖注入（DI）**：让代码显式声明所需能力，注入而非自取。
- Go 隐式接口的好处：业务逻辑只声明需要什么接口，**提供方根本不需要 import 这个接口**——天然解耦。
- Java 显式接口下，client 与 provider 都要 import 同一接口；Go 不需要。

### 15.2 完整示例（精简版）

```go
// 工具函数
func LogOutput(message string) { fmt.Println(message) }

// 数据存储
type SimpleDataStore struct{ userData map[string]string }
func (sds SimpleDataStore) UserNameForID(id string) (string, bool) {
    n, ok := sds.userData[id]; return n, ok
}
func NewSimpleDataStore() SimpleDataStore { /* ... */ }

// 业务逻辑只声明依赖
type DataStore interface { UserNameForID(string) (string, bool) }
type Logger    interface { Log(string) }

// 用函数类型让 LogOutput 自动满足 Logger
type LoggerAdapter func(message string)
func (lg LoggerAdapter) Log(m string) { lg(m) }

type SimpleLogic struct {
    l  Logger
    ds DataStore
}
func (sl SimpleLogic) SayHello(id string) (string, error) {
    sl.l.Log("in SayHello for " + id)
    name, ok := sl.ds.UserNameForID(id)
    if !ok { return "", errors.New("unknown user") }
    return "Hello, " + name, nil
}
func NewSimpleLogic(l Logger, ds DataStore) SimpleLogic {
    return SimpleLogic{l: l, ds: ds}
}

// Controller 又用自己定义的 Logic 接口
type Logic interface { SayHello(string) (string, error) }
type Controller struct{ l Logger; logic Logic }

func main() {
    l     := LoggerAdapter(LogOutput)
    ds    := NewSimpleDataStore()
    logic := NewSimpleLogic(l, ds)
    c     := NewController(l, logic)
    http.HandleFunc("/hello", c.SayHello)
    http.ListenAndServe(":8080", nil)
}
```

要点回顾：

- `main` 是**唯一**知道全部具体类型的位置；替换实现只需改 `main`。
- `SimpleLogic` 不感知它满足 `Logic` 接口——接口被 client（Controller）定义。
- 函数类型 `LoggerAdapter` 让普通函数也能"实现接口"。
- `c.SayHello` 被 `http.HandleFunc` 用类型转换变成 `http.HandlerFunc`——既是把方法当函数用、又是函数类型实现接口的例子。

### 15.3 Wire

不想手写 wiring，可用 Google [Wire](https://github.com/google/wire) 通过代码生成自动产出 `main` 中那段拼装代码。

---

## 16. Go 不算面向对象，也不强求

Go 没有继承、方法重写、动态分派，但有方法、闭包、函数值。它既不是纯面向对象，也不是纯过程或函数式。**最贴切的标签是 "实用 (practical)"**：从各种范式中借鉴，目标是写出能让大团队长期维护的简单、可读代码。强行套用某种范式只会写出非惯用的 Go 代码。
