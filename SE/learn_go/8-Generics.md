# 第 8 章 泛型（Generics）

> "Don't repeat yourself" 是常见的工程箴言。但在 Go 这种静态类型语言里，重用同一份逻辑去服务"不同类型"长期是个难题：函数参数、struct 字段的类型必须在编译期确定。Go 1.18 引入 **type parameters**（俗称 generics）后，自定义类型与函数也终于能像内置的 `map`/`slice`/`channel`/`len` 一样，对多种具体类型保持类型安全。本章围绕"为什么需要泛型 → 怎么写 → 边界与未实现的特性"层层展开。

---

## 1. 为什么需要泛型

### 1.1 编译期类型检查 vs 代码复用的矛盾

Go 是静态类型语言，每个变量、参数、字段的类型都必须在编译期已知。这种严格性让编译器能在编译期捕获大量错误，但也带来代价：**同一份逻辑要服务多种类型时，要么写多份重复代码，要么放弃编译期类型检查**。

举一个最朴素的例子。普通函数 `divAndRemainder` 只能服务 `int`：

```go
func divAndRemainder(num, denom int) (int, int, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}
```

`Node` struct 的 `val` 字段也只能存 `int`：

```go
type Node struct {
    val  int
    next *Node
}
```

如果想要一棵 `string` 或 `float64` 的二叉树，1.18 之前只剩两条路。

### 1.2 没有泛型时的两种妥协方案

#### 方案 A：每种类型复制一遍代码

为 `int`、`float64`、`string` 各写一棵树。逻辑完全一样，仅类型不同——重复且容易漏改。

#### 方案 B：用 interface 抽象顺序，牺牲类型安全

定义一个 `Orderable` 接口表达"如何比较两个值"：

```go
type Orderable interface {
    // 返回值 < 0 表示当前值更小
    // 返回值 > 0 表示当前值更大
    // 返回值 == 0 表示相等
    Order(any) int
}

type Tree struct {
    val         Orderable
    left, right *Tree
}

func (t *Tree) Insert(val Orderable) *Tree {
    if t == nil {
        return &Tree{val: val}
    }
    switch comp := val.Order(t.val); {
    case comp < 0:
        t.left = t.left.Insert(val)
    case comp > 0:
        t.right = t.right.Insert(val)
    }
    return t
}
```

让 `int` 实现这个接口：

```go
type OrderableInt int

func (oi OrderableInt) Order(val any) int {
    return int(oi - val.(OrderableInt))
}
```

问题：`Order` 的入参是 `any`，编译器**无法阻止往同一棵树里插入 `OrderableInt` 和 `OrderableString`**。下面这段代码能通过编译，但运行时会 panic：

```go
var it *Tree
it = it.Insert(OrderableInt(5))
it = it.Insert(OrderableString("nope"))
// panic: interface conversion: interface {} is main.OrderableInt, not string
```

类型安全已经被 `any` 绕过了。

### 1.3 泛型解决的不止数据结构

数据结构没有泛型还能凑合，**真正受限的是函数**。Go 标准库为此做了不少妥协：

- `math.Max` / `math.Min` / `math.Mod` 全部用 `float64`，因为 `float64` 能"几乎"覆盖所有数值类型（除了大于 `2⁵³ - 1` 的整型，会丢精度）。
- `sort.Slice` 用反射处理任意 slice，牺牲了运行性能与编译期类型检查。
- `map`、`reduce`、`filter` 这种 slice 工具函数，每种元素类型都得重写一份。

此外有些场景**完全无法实现**：不能根据接口创建新实例；不能要求"两个参数必须是相同的具体类型"；不能写"任意类型 slice 的处理函数"而不动用反射。

---

## 2. 泛型的语法骨架

### 2.1 泛型类型声明

```go
type 类型名[类型参数名 类型约束] 底层类型
```

- **类型参数（type parameter）**：方括号里的 `T`，可以叫任何名字，惯例用大写字母。
- **类型约束（type constraint）**：用 interface 描述"哪些类型能填进来"。`any` 是 universe block 里的内置约束，等价于"任何类型都行"。

最小例子：泛型栈（stack 是 LIFO 数据结构，像一摞洗碗水槽里的盘子，先放的在最底，先取出来的是最后放的）。

```go
type Stack[T any] struct {
    vals []T
}

func (s *Stack[T]) Push(val T) {
    s.vals = append(s.vals, val)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.vals) == 0 {
        var zero T
        return zero, false
    }
    top := s.vals[len(s.vals)-1]
    s.vals = s.vals[:len(s.vals)-1]
    return top, true
}
```

要点：

1. **类型声明上**：`Stack[T any]` 表示有一个名为 `T` 的类型参数，约束是 `any`。
2. **方法接收者**：写 `(s *Stack[T])`，不是 `(s *Stack)`——必须把类型参数带上。
3. **零值技巧**：`var zero T` 总是返回 `T` 的零值。这是 Pop 在空栈时返回类型安全零值的标准写法（不能直接 `return nil, false`，因为 `T` 可能是 `int` 这种没 `nil` 的类型）。

### 2.2 使用泛型类型

声明变量时把具体类型填进方括号：

```go
var intStack Stack[int]
intStack.Push(10)
intStack.Push(20)
v, ok := intStack.Pop()
fmt.Println(v, ok)
```

如果尝试 `intStack.Push("nope")`，编译器立刻报错：

```
cannot use "nope" (untyped string constant) as int value
  in argument to intStack.Push
```

**这正是泛型相对方案 B（用 interface + any）的核心收益：错误在编译期就被拦下，而不是运行时 panic。**

---

## 3. comparable 约束

### 3.1 为什么 any 不够

给 `Stack` 加一个 `Contains` 方法：

```go
func (s Stack[T]) Contains(val T) bool {
    for _, v := range s.vals {
        if v == val { // 编译错误!
            return true
        }
    }
    return false
}
```

编译器报错：

```
invalid operation: v == val (type parameter T is not comparable with ==)
```

**原理**：`any` 等价于空接口，对里面装的具体类型一无所知，自然不能保证 `==` 可用。Go 中虽然几乎所有类型都能用 `==` / `!=` 比较，但 slice、map、func 这些类型不行——所以仅约束为 `any` 不够。

### 3.2 内置约束 comparable

universe block（参见 [[4-Blocks-Shadows-Control_Structures]]）里预定义了一个 `comparable` 接口，表示"能用 `==` 和 `!=` 比较的类型"。把约束改一下：

```go
type Stack[T comparable] struct {
    vals []T
}
```

现在 `Contains` 通过编译：

```go
func main() {
    var s Stack[int]
    s.Push(10)
    s.Push(20)
    s.Push(30)
    fmt.Println(s.Contains(10)) // true
    fmt.Println(s.Contains(5))  // false
}
```

约束选择的原则：**选最弱够用的约束**。能用 `any` 就别用 `comparable`，能用 `comparable` 就别再加更具体的方法约束。

---

## 4. 泛型函数

### 4.1 语法骨架

```go
func 函数名[类型参数 约束](入参) 返回值 { ... }
```

类型参数放在函数名之后、入参之前。下面是 type parameters proposal 给出的 `Map` / `Reduce` / `Filter` 实现：

```go
// Map 把 []T1 映射为 []T2
// 两个类型参数 T1、T2 都用 any 约束
func Map[T1, T2 any](s []T1, f func(T1) T2) []T2 {
    r := make([]T2, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}

// Reduce 把 []T1 折叠成单个 T2
func Reduce[T1, T2 any](s []T1, initializer T2, f func(T2, T1) T2) T2 {
    r := initializer
    for _, v := range s {
        r = f(r, v)
    }
    return r
}

// Filter 保留满足 f 的元素
func Filter[T any](s []T, f func(T) bool) []T {
    var r []T
    for _, v := range s {
        if f(v) {
            r = append(r, v)
        }
    }
    return r
}
```

### 4.2 调用示例

```go
words := []string{"One", "Potato", "Two", "Potato"}
filtered := Filter(words, func(s string) bool {
    return s != "Potato"
})
fmt.Println(filtered) // [One Two]

lengths := Map(filtered, func(s string) int {
    return len(s)
})
fmt.Println(lengths) // [3 3]

sum := Reduce(lengths, 0, func(acc int, val int) int {
    return acc + val
})
fmt.Println(sum) // 6
```

注意调用时**没有显式写类型参数** `Filter[string](...)`——这是类型推断的功劳，详见 [[#7. 类型推断]]。

---

## 5. 泛型与接口

### 5.1 任意接口都能当约束

不止 `any` 和 `comparable`，**任何接口都能作为类型约束**。例如要求 `Pair` 的两个字段类型相同，且都实现 `fmt.Stringer`：

```go
type Pair[T fmt.Stringer] struct {
    Val1 T
    Val2 T
}
```

### 5.2 接口本身可以带类型参数

接口也能带类型参数，下面 `Differ` 嵌入了 `fmt.Stringer`，并要求实现一个 `Diff(T) float64`：

```go
type Differ[T any] interface {
    fmt.Stringer
    Diff(T) float64
}
```

### 5.3 应用：FindCloser

`FindCloser` 接收两个 `Pair`，要求其字段类型必须实现 `Differ`，返回差距更近的那个：

```go
func FindCloser[T Differ[T]](pair1, pair2 Pair[T]) Pair[T] {
    d1 := pair1.Val1.Diff(pair1.Val2)
    d2 := pair2.Val1.Diff(pair2.Val2)
    if d1 < d2 {
        return pair1
    }
    return pair2
}
```

这里 `T Differ[T]` 是个微妙的写法：T 必须实现 `Differ[T]` 接口，意味着 T 的 `Diff` 方法必须接受 T 类型本身。这种自指约束让"字段两两可比"在编译期就能保证。

定义两个具体类型：

```go
type Point2D struct {
    X, Y int
}

func (p2 Point2D) String() string {
    return fmt.Sprintf("{%d,%d}", p2.X, p2.Y)
}

func (p2 Point2D) Diff(from Point2D) float64 {
    x := p2.X - from.X
    y := p2.Y - from.Y
    return math.Sqrt(float64(x*x) + float64(y*y))
}

type Point3D struct {
    X, Y, Z int
}

func (p3 Point3D) String() string {
    return fmt.Sprintf("{%d,%d,%d}", p3.X, p3.Y, p3.Z)
}

func (p3 Point3D) Diff(from Point3D) float64 {
    x := p3.X - from.X
    y := p3.Y - from.Y
    z := p3.Z - from.Z
    return math.Sqrt(float64(x*x) + float64(y*y) + float64(z*z))
}
```

调用：

```go
pair2Da := Pair[Point2D]{Point2D{1, 1}, Point2D{5, 5}}
pair2Db := Pair[Point2D]{Point2D{10, 10}, Point2D{15, 5}}
closer := FindCloser(pair2Da, pair2Db)
```

---

## 6. type terms：让操作符进入约束

### 6.1 问题：泛型函数怎么用 `/` 和 `%`

要让 `divAndRemainder` 适配 `int` / `uint` / `int64` 等所有整型，光用方法约束办不到——`/` 和 `%` 是操作符，不是方法。**泛型靠 type element 解决**：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 |
        uint | uint8 | uint16 | uint32 | uint64 | uintptr
}
```

- **type element**：interface 内部由若干 **type term** 用 `|` 连接的元素。
- **type term**：列出来的具体类型（`int`、`uint8` 等）。
- **允许的操作符**：所有 type term 上**共同**支持的操作符。`%` 只对整型有效，所以 `Integer` 只列整型。`byte` / `rune` 不用列，因为它们是 `uint8` / `int32` 的别名。

### 6.2 type element 只能当约束用

```go
// 编译错误：interface 含 type element 时，不能作为变量、字段、参数、返回值的类型。
var x Integer
```

只有作为 type parameter 的约束时才合法。

### 6.3 泛型版 divAndRemainder

```go
func divAndRemainder[T Integer](num, denom T) (T, T, error) {
    if denom == 0 {
        return 0, 0, errors.New("cannot divide by zero")
    }
    return num / denom, num % denom, nil
}

func main() {
    var a uint = 18_446_744_073_709_551_615
    var b uint = 9_223_372_036_854_775_808
    fmt.Println(divAndRemainder(a, b))
}
```

### 6.4 用 `~` 让自定义底层类型也能匹配

默认情况下，type term **精确匹配**。下面这段会失败：

```go
type MyInt int
var myA MyInt = 10
var myB MyInt = 20
fmt.Println(divAndRemainder(myA, myB))
// MyInt does not satisfy Integer (possibly missing ~ for int in Integer)
```

`MyInt` 的底层类型是 `int`，但本身不是 `int`。在 type term 前加 `~` 表示"底层类型为该类型也接受"：

```go
type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}
```

### 6.5 type term 与方法可以共存

一个用作约束的 interface 可以**同时**列 type term 和方法，要求"底层类型是 int 且实现 String()"：

```go
type PrintableInt interface {
    ~int
    String() string
}
```

#### 不可能实例化的约束

下面这个约束**永远没有类型能满足**，因为 `int` 没有方法：

```go
type ImpossiblePrintableInt interface {
    int             // 注意没有 ~
    String() string
}

type ImpossibleStruct[T ImpossiblePrintableInt] struct {
    val T
}
```

声明本身合法，但**任何使用它的尝试都会报错**：

```go
s := ImpossibleStruct[int]{10}
// int does not implement ImpossiblePrintableInt (missing String method)
```

type term 也可以是 slice、map、array、channel、struct、function——常用于"要求底层类型 X 且带某些方法"。

### 6.6 标准库的 cmp.Ordered（1.21+）

Go 1.21 在 [cmp 包](https://pkg.go.dev/cmp) 加入了 `Ordered` 约束，表达"支持 `<`、`<=`、`>`、`>=`、`==`、`!=` 的所有类型"：

```go
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
        ~float32 | ~float64 |
        ~string
}
```

`cmp.Compare` / `cmp.Less` 是基于它的两个泛型比较函数。后续二叉树例子会用它。

---

## 7. 类型推断

### 7.1 何时能推断、何时不能

调用泛型函数时，多数情况下编译器可以从入参反推类型参数（类似 `:=` 的自动推断），无需显式写 `Filter[string](...)`。

**但有一种情况推断不出来**：类型参数仅出现在**返回值**位置。例：

```go
type Integer interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64
}

func Convert[T1, T2 Integer](in T1) T2 {
    return T2(in)
}

func main() {
    var a int = 10
    b := Convert[int, int64](a) // 必须写全两个类型参数
    fmt.Println(b)
}
```

`T2` 只出现在返回值，调用方必须显式提供所有类型实参。

### 7.2 type element 限制常量

操作符可用与否是基于"所有 type term 都支持"的并集；常量也一样——只有在**每一个 type term 上都合法**的常量才能赋值给该泛型变量。

```go
// 不合法：1000 装不进 int8
func PlusOneThousand[T Integer](in T) T {
    return in + 1_000
}

// 合法：100 在所有 Integer 类型里都装得下
func PlusOneHundred[T Integer](in T) T {
    return in + 100
}
```

---

## 8. 串起来：泛型函数 + 泛型数据结构（进阶）

把"泛型二叉树"的故事重新讲一遍，但这次彻底解决方案 B 的痛点：用一个外部的"比较函数"承担排序责任，让树本身只关心结构。

### 8.1 思路：把比较抽离为函数类型

```go
type OrderableFunc[T any] func(t1, t2 T) int
```

`OrderableFunc[T]` 是一个把"两个 T 比较"变成 `int` 的函数类型。

### 8.2 Tree 与 Node 拆开

```go
type Tree[T any] struct {
    f    OrderableFunc[T]
    root *Node[T]
}

type Node[T any] struct {
    val         T
    left, right *Node[T]
}

func NewTree[T any](f OrderableFunc[T]) *Tree[T] {
    return &Tree[T]{f: f}
}
```

### 8.3 方法只是把比较函数透传给 Node

```go
func (t *Tree[T]) Add(v T) {
    t.root = t.root.Add(t.f, v)
}

func (t *Tree[T]) Contains(v T) bool {
    return t.root.Contains(t.f, v)
}

func (n *Node[T]) Add(f OrderableFunc[T], v T) *Node[T] {
    if n == nil {
        return &Node[T]{val: v}
    }
    switch r := f(v, n.val); {
    case r <= -1:
        n.left = n.left.Add(f, v)
    case r >= 1:
        n.right = n.right.Add(f, v)
    }
    return n
}

func (n *Node[T]) Contains(f OrderableFunc[T], v T) bool {
    if n == nil {
        return false
    }
    switch r := f(v, n.val); {
    case r <= -1:
        return n.left.Contains(f, v)
    case r >= 1:
        return n.right.Contains(f, v)
    }
    return true
}
```

### 8.4 三种构造方式

#### (a) 内置类型用 cmp.Compare

```go
t1 := NewTree(cmp.Compare[int])
t1.Add(10); t1.Add(30); t1.Add(15)
fmt.Println(t1.Contains(15)) // true
fmt.Println(t1.Contains(40)) // false
```

#### (b) 自定义类型用普通函数

```go
type Person struct {
    Name string
    Age  int
}

func OrderPeople(p1, p2 Person) int {
    out := cmp.Compare(p1.Name, p2.Name)
    if out == 0 {
        out = cmp.Compare(p1.Age, p2.Age)
    }
    return out
}

t2 := NewTree(OrderPeople)
t2.Add(Person{"Bob", 30})
```

#### (c) 用方法表达式当函数

[[5-Functions]] 提到方法可以转成函数表达式（`Type.Method`）。先给类型加方法：

```go
func (p Person) Order(other Person) int {
    out := cmp.Compare(p.Name, other.Name)
    if out == 0 {
        out = cmp.Compare(p.Age, other.Age)
    }
    return out
}
```

然后传 `Person.Order`（方法表达式，签名等价于 `func(Person, Person) int`）：

```go
t3 := NewTree(Person.Order)
t3.Add(Person{"Bob", 30})
```

**这套方案完全在编译期保证类型安全，并且不要求用户实现特定接口**——把"如何比较"变成参数化能力，比"必须实现 Order 接口"更宽松。

---

## 9. comparable 的暗坑（深入）

`comparable` 约束有个反直觉的地方：**接口类型本身满足 `comparable`，但接口里装的具体类型可能不能比较**，导致运行时 panic。

例：

```go
type Thinger interface {
    Thing()
}

type ThingerInt int
func (t ThingerInt) Thing() { fmt.Println("ThingInt:", t) }

type ThingerSlice []int
func (t ThingerSlice) Thing() { fmt.Println("ThingSlice:", t) }

func Comparer[T comparable](t1, t2 T) {
    if t1 == t2 {
        fmt.Println("equal!")
    }
}
```

行为分四档：

```go
// 1. int / ThingerInt 直接传，OK
var a, b int = 10, 10
Comparer(a, b) // equal!

// 2. ThingerSlice 直接传，编译失败 —— slice 不是 comparable
var a3, b3 ThingerSlice = []int{1,2,3}, []int{1,2,3}
Comparer(a3, b3) // ThingerSlice does not satisfy comparable

// 3. 装进 Thinger 接口（值是 ThingerInt），运行 OK
var a4 Thinger = ThingerInt(20)
var b4 Thinger = ThingerInt(20)
Comparer(a4, b4) // equal!

// 4. 装进 Thinger 接口（值是 ThingerSlice），编译通过但运行 panic
a4 = ThingerSlice{1,2,3}
b4 = ThingerSlice{1,2,3}
Comparer(a4, b4)
// panic: runtime error: comparing uncomparable type main.ThingerSlice
```

**原理**：编译器只检查类型参数本身的可比性。`Thinger` 接口作为类型在编译期被认作"可比"——然而真正比较的是接口里装的动态值，如果运行时装的是 slice/map/func，就会 panic。Robert Griesemer 的博客 [All Your Comparable Types](https://go.dev/blog/comparable) 详细解释了这个设计权衡。

实务建议：**慎用 `comparable` 约束接口类型**；如果一定要用，确保运行时不会塞进不可比类型。

---

## 10. Go 泛型未实现的特性（深入）

Go 团队选择"小而克制"的泛型，许多其他语言里的能力没有引入。

| 缺失特性 | 含义 | Go 的态度 |
|---|---|---|
| 操作符重载 | 自定义类型可重载 `==`、`<<`、`[]` 等 | 不会加。`range`/`[]` 不能用于自定义容器；理由是 `<<` 在 C++ 不同类型里含义不同，可读性差 |
| 方法上的额外类型参数 | `func (fs functionalSlice[T]) Map[E any](f func(T) E) functionalSlice[E]` | 不支持。链式 `.Map().Reduce()` 写不出来，得拆开赋值 |
| Variadic type parameters | 变长泛型类型参数（如交替 string/int） | 不支持。变长入参必须是同一种已声明类型（可以是泛型） |
| Specialization | 同名函数针对特定类型重载 | 不支持。Go 本身就没有重载 |
| Currying | 部分实例化泛型函数 | 不支持 |
| Metaprogramming | 编译期生成代码 | 不支持 |

---

## 11. 习惯用法的迁移（进阶）

泛型对 Go 风格的影响：

- **`float64` 不再是"通用数值类型"** — 写数学函数应该用泛型 + `Integer` / `Ordered` 约束。
- **`interface{}` / `any` 仅表示"真的任意"** — 表达"未指定但要保持类型安全"应该用类型参数。
- **不要为了用泛型而改老代码** — 老代码继续工作；先在新场景里探索更好的设计模式。

### 性能：泛型不一定更快

Go 1.20 起编译器没显著变慢；运行时性能则要看场景：

- **不要为了"性能"把 interface 参数改成泛型参数**。一个反例（Go 1.20 实测约慢 30%）：

  ```go
  // 原来
  type Ager interface { age() int }
  func doubleAge(a Ager) int { return a.age() * 2 }

  // 改成泛型，反而更慢
  func doubleAgeGeneric[T Ager](a T) int { return a.age() * 2 }
  ```

  原因：Go 编译器为"不同 underlying type"生成不同函数，但**所有指针类型共享同一个生成的函数**，运行时通过额外查表分发——这一步反而拖慢了原本一次接口调用就完成的工作。

- 与 C++ 的差异：C++ 给每个具体类型单独实例化（编译变慢、二进制变大但运行快）；Go 选择共享生成函数（编译快、二进制小但有运行时开销）。

- **正确做法**：写易维护的代码，用基准测试（参见 [Using Benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks)）测量再决定。

---

## 12. 标准库的泛型化进程（进阶）

| 版本 | 变化 |
|---|---|
| 1.18 | 引入 `any`、`comparable`；标准库未做 API 改动；把 `interface{}` 文风替换成 `any` |
| 1.21 | `slices.Equal` / `EqualFunc` / `Insert` / `Delete` / `DeleteFunc`；`maps.Clone`；`cmp.Ordered` / `cmp.Compare` / `cmp.Less`；`sync.OnceValue` / `OnceValues` |

实务建议：优先用标准库的泛型工具，而不是自己重新实现。

### 未来可能的特性：sum types

type element 已经能在约束里枚举类型；同样的写法如果搬到普通接口里，就能表达"这个变量只可能是这几种类型之一"——也就是 sum types。一个常见用例是 JSON 字段可能是单值或数组：当前只能用 `any`，将来或许能直接声明 `string | []string`。Rust / Swift 用这种特性表达枚举，Go 的 enum 一向较弱，sum types 是一个值得期待的方向。

---

## 13. 一句话回顾

泛型的本质是**把"类型"也变成可参数化的入参**：保留编译期类型检查的同时换回代码复用。Go 的实现刻意保守——`any` / `comparable` / 任意接口都能当约束，type term 让操作符进入约束，`~` 兼容自定义底层类型——但拒绝操作符重载、方法上的额外类型参数等容易降低可读性的特性。**用泛型替代 `interface{}` + 类型断言的反射式写法；不要为追求性能盲目泛型化**。
