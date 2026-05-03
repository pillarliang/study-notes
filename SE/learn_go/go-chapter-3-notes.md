# Chapter 3：复合类型（Composite Types）

本章介绍 Go 中的内置复合类型：数组、切片、字符串、map、结构体，以及配套的内置函数和最佳实践。

---

## 1. 数组（Array）—— 太死板，不建议直接使用

### 1.1 声明语法

数组所有元素必须是同一类型，长度是类型的一部分。

```go
var x [3]int                  // 长度 3，零值初始化为 [0, 0, 0]
var x = [3]int{10, 20, 30}    // 数组字面量
var x = [12]int{1, 5: 4, 6, 10: 100, 15} // 稀疏数组：仅指定非零索引
// 结果：[1, 0, 0, 0, 0, 4, 6, 0, 0, 0, 100, 15]

var x = [...]int{10, 20, 30}  // 用 ... 让编译器自动推导长度
```

#### 稀疏初始化语法详解

`索引: 值` 表示"把**值**放到指定**索引**位置"，未指定的位置自动填零值。**不带索引**的元素接在**上一个索引之后**。该语法对**数组、切片、map** 字面量都适用。

以 `[12]int{1, 5: 4, 6, 10: 100, 15}` 为例，逐元素拆解：

| 写法      | 索引 | 值  | 说明                               |
| --------- | ---- | --- | ---------------------------------- |
| `1`       | 0    | 1   | 默认从 0 开始                      |
| `5: 4`    | 5    | 4   | 显式跳到索引 5；中间索引 1~4 补 0  |
| `6`       | 6    | 6   | 紧跟上一个，索引 = 5 + 1           |
| `10: 100` | 10   | 100 | 显式跳到索引 10；中间索引 7~9 补 0 |
| `15`      | 11   | 15  | 紧跟上一个，索引 = 10 + 1          |

最终值：`[1, 0, 0, 0, 0, 4, 6, 0, 0, 0, 100, 15]`。

等价的"啰嗦写法"：

```go
var x = [12]int{
    0: 1, 1: 0, 2: 0, 3: 0, 4: 0,
    5: 4, 6: 6,
    7: 0, 8: 0, 9: 0,
    10: 100, 11: 15,
}
```

适用场景：绝大多数位置都是零值、只有少数位置有值的初始化。例如想要长度 100、只有首尾有值的切片：

```go
config := []int{42, 99: 7}  // 长度 100，[0]=42、[99]=7，中间全是 0
```

> 切片版本不需要预先写死长度，长度由"最大索引 + 1"推导得到。
> 真正稀疏的数据更多用 map，本语法常用在测试夹具和小型查找表里。

### 1.2 多维数组

Go 只有一维数组，多维通过"数组的数组"模拟：

```go
var x [2][3]int  // 长度为 2 的数组，每个元素是长度为 3 的 int 数组
```

### 1.3 比较与读写

- 长度相同且元素全等的两个数组可用 `==` / `!=` 比较。
- 索引越界：常量索引 → 编译错误；变量索引 → 运行时 panic。
- `len(x)` 返回数组长度。

### 1.4 数组的"死板"之处

> **数组长度是类型的一部分**：`[3]int` 与 `[4]int` 是两个不同类型。

由此带来三个限制：

1. 不能用变量指定长度（必须编译期确定）。
2. 不同长度的数组**不能互相类型转换**。
3. 因此无法写一个能处理任意长度数组的函数。

**结论**：除非长度在编译期就完全确定（例如加密哈希的固定输出长度），否则不要直接用数组。**数组存在的主要意义是为切片提供底层存储**。

---

## 2. 切片（Slice）—— Go 中最常用的序列类型

### 2.1 切片字面量

```go
var x = []int{10, 20, 30}                  // 注意：[] 内不写长度
var x = []int{1, 5: 4, 6, 10: 100, 15}     // 稀疏初始化
var x [][]int                              // 二维切片
```

> 区分：`[...]` 创建数组；`[]` 创建切片。

### 2.2 nil 切片

```go
var x []int   // x 此时为 nil
fmt.Println(x == nil) // true
```

- 切片的零值是 `nil`，长度和容量都为 0。
- 切片**不可比较**：除了与 `nil` 比较外，使用 `==` / `!=` 是编译错误。

### 2.3 切片相等性比较（Go 1.21+）

```go
import "slices"

x := []int{1, 2, 3, 4, 5}
y := []int{1, 2, 3, 4, 5}
slices.Equal(x, y)     // true，要求元素可比较
slices.EqualFunc(x, y, func(a, b int) bool { ... }) // 自定义比较函数
```

> `reflect.DeepEqual` 是历史遗留方案，新代码不要再用。

---

## 3. 切片的内置函数

### 3.1 len 与 append

```go
var x []int
x = append(x, 10)         // 单值追加
x = append(x, 5, 6, 7)    // 多值追加
y := []int{20, 30, 40}
x = append(x, y...)       // 用 ... 把切片展开
```

> Go 是**值传递**语言，传给 `append` 的是切片头部副本，所以**必须把返回值赋回原变量**，否则编译报错。

### 3.2 容量（Capacity）

- **长度（length）**：当前已存元素数。
- **容量（capacity）**：底层数组预留的空间。
- 当 length == capacity 时再 append，runtime 会**分配更大的底层数组并复制数据**，原数组变成垃圾。

容量增长规则（Go 1.18 起）：

| 当前容量 | 增长方式                            |
| -------- | ----------------------------------- |
| < 256    | 翻倍                                |
| ≥ 256    | `(cap + 768) / 4`，逐渐收敛到 25%   |

```go
// Example 3-1
var x []int
fmt.Println(x, len(x), cap(x)) // [] 0 0
x = append(x, 10)
fmt.Println(x, len(x), cap(x)) // [10] 1 1
x = append(x, 20)
fmt.Println(x, len(x), cap(x)) // [10 20] 2 2
x = append(x, 30)
fmt.Println(x, len(x), cap(x)) // [10 20 30] 3 4
x = append(x, 40)
fmt.Println(x, len(x), cap(x)) // [10 20 30 40] 4 4
x = append(x, 50)
fmt.Println(x, len(x), cap(x)) // [10 20 30 40 50] 5 8
```

#### 增长公式拆解

##### cap < 256：翻倍

新 cap = 旧 cap × 2。注意 0 是特例（0×2 还是 0），从 0 开始的第一步会跳到 1。书中 Example 3-1 完整轨迹：

| 操作        | len | cap | 说明              |
| ----------- | --- | --- | ----------------- |
| 起始        | 0   | 0   | nil 切片          |
| append 1 个 | 1   | 1   | 0 → 1（特例）     |
| append 1 个 | 2   | 2   | 1 → 2（×2）       |
| append 1 个 | 3   | 4   | 2 → 4（×2）       |
| append 1 个 | 4   | 4   | 还够，不扩容      |
| append 1 个 | 5   | 8   | 4 → 8（×2）       |

继续会是 8 → 16 → 32 → 64 → 128 → 256，到 256 切换到下方公式。

##### cap ≥ 256：`(cap + 768) / 4` 是"增长量"，不是新容量

```text
new_cap = old_cap + (old_cap + 768) / 4
```

验证书中两个数据：

- `cap = 512`：增长量 = (512 + 768) / 4 = 320，新 cap = 832，增长率 ≈ **63%** ✓
- `cap = 4096`：增长量 = (4096 + 768) / 4 = 1216，新 cap = 5312，增长率 ≈ **30%** ✓

##### 为什么"收敛到 25%"

把增长率公式拆开：

```text
增长率 = (cap + 768) / 4 / cap = 1/4 + 192 / cap
```

`cap` 越大，`192 / cap` 越接近 0，整体增长率就趋近 **1/4 = 25%**。

| cap     | 192 / cap | 增长率 |
| ------- | --------- | ------ |
| 256     | 0.75      | ~100%  |
| 512     | 0.375     | ~63%   |
| 1024    | 0.1875    | ~44%   |
| 4096    | 0.047     | ~30%   |
| 65536   | 0.003     | ~25.3% |
| 1000000 | 0.0002    | ~25%   |

##### 设计直觉

- 小切片**翻倍**：append 摊销到接近 O(1)，浪费的内存也很小。
- 大切片**只涨 25%**：避免一次性吃掉几十 MB 内存，增速温和但仍能保持均摊性能。

> 已知容量上限就一次性建好，避免多次扩容拷贝。

### 3.3 make

`make` 用于显式指定类型、长度、（可选）容量：

```go
x := make([]int, 5)        // len=5, cap=5，元素全为 0
x := make([]int, 5, 10)    // len=5, cap=10
x := make([]int, 0, 10)    // len=0, cap=10，可直接 append
```

> ⚠️ **常见误区**：`make([]int, 5)` 后再 `append(x, 10)`，结果是 `[0 0 0 0 0 10]`，而不是 `[10 0 0 0 0]`。`append` 永远在末尾追加。

> ⚠️ 容量不能小于长度，否则编译/运行时报错。

### 3.4 clear（Go 1.21+）

把切片所有元素置为零值，**长度不变**：

```go
s := []string{"first", "second", "third"}
clear(s)
fmt.Println(s, len(s)) // [  ] 3
```

### 3.5 切片声明的最佳实践

| 场景 | 推荐写法 |
|------|---------|
| 切片可能始终为空 | `var data []int`（nil 切片） |
| 已有初始值且不会变化 | 切片字面量 `data := []int{2, 4, 6, 8}` |
| 作为 buffer 使用 | `make([]byte, n)` 指定非零长度 |
| 完全确定大小，索引赋值 | `make([]T, n)` |
| 大致知道大小，append 填充 | `make([]T, 0, n)` 或 `var x []T` |

> 作者偏好：用 `var x []T` 初始化为 nil，再 append；可能稍慢但不易出 bug。

---

## 4. 切片的切片（Slicing）

### 4.1 基本语法

```go
x := []string{"a", "b", "c", "d"}
y := x[:2]   // [a b]
z := x[1:]   // [b c d]
d := x[1:3]  // [b c]
e := x[:]    // [a b c d]
```

### 4.2 子切片共享底层存储 ⚠️

切片的切片**不复制数据**，多个切片共享同一段底层数组。

```go
x := []string{"a", "b", "c", "d"}
y := x[:2]  // [a b]
z := x[1:]  // [b c d]
x[1] = "y"
y[0] = "x"
z[1] = "z"
fmt.Println(x) // [x y z d]
fmt.Println(y) // [x y]
fmt.Println(z) // [y z d]
```

### 4.3 子切片 + append：更隐蔽的坑

子切片的容量 = 原切片容量 − 起始偏移。append 子切片可能**覆写父切片的数据**。

```go
x := []string{"a", "b", "c", "d"}
y := x[:2]
fmt.Println(cap(x), cap(y)) // 4 4
y = append(y, "z")
fmt.Println(x) // [a b z d]   ← x 被改了！
fmt.Println(y) // [a b z]
```

#### 子切片容量公式详解

```text
子切片 cap = 原切片 cap − 子切片起始偏移
```

子切片**指向同一个底层数组**，只是起点变了。它能"看到"的容量，是从它的起点一直到底层数组末尾的距离。

以 `x := []string{"a", "b", "c", "d"}`（cap=4）为例：

| 表达式   | 起点偏移 | len | cap           |
| -------- | -------- | --- | ------------- |
| `x[:2]`  | 0        | 2   | 4 − 0 = **4** |
| `x[1:]`  | 1        | 3   | 4 − 1 = **3** |
| `x[2:3]` | 2        | 1   | 4 − 2 = **2** |
| `x[3:4]` | 3        | 1   | 4 − 3 = **1** |

```text
索引:    0   1   2   3
值:     "a" "b" "c" "d"

x[:2]   [-------]               len=2, cap=4 (一直到末尾)
x[1:]       [-----------]       len=3, cap=3
x[2:3]          [---]           len=1, cap=2
x[3:4]              [---]       len=1, cap=1
```

**底层原理**：切片在 runtime 是个三元组：

```go
type slice struct {
    ptr unsafe.Pointer  // 指向底层数组某个位置
    len int             // 当前可见元素数
    cap int             // 从 ptr 到底层数组末尾的容量
}
```

取子切片时，`ptr` 移到子切片起点，`cap` 自然变成"从新 ptr 到底层数组末尾"还剩多少格子。这正是 append 子切片会"漏出去"覆写父切片的根本原因。

### 4.4 三段切片表达式（Full Slice Expression）

第三个参数显式限定子切片的容量，避免 append 时影响父切片：

```go
x := make([]string, 0, 5)
x = append(x, "a", "b", "c", "d")
y := x[2:2:4]   // len=0, cap=2
z := x[2:4:4]   // len=2, cap=2
// 此时 y、z 各自独立扩容，不会回写到 x
```

公式变形：

```text
三段切片 cap = max − 起始偏移
```

只要把 `max` 卡在子切片的 `high` 处，append 就一定会扩容，不会回写父切片。

#### append 扩容后是否还共享内存？

**结论**：扩容后 `y` 已经指向新的底层数组，与 `x` 完全脱钩。再修改 `y[0]` 不会影响 `x`。

```go
x := []string{"a", "b", "c", "d"}
y := x[:2:2]              // len=2, cap=2，与 x 共享底层数组
y = append(y, "z")        // cap 不够 → 分配新底层数组并复制
y[0] = "Q"
fmt.Println("x:", x)      // [a b c d]   ← 完全没变
fmt.Println("y:", y)      // [Q b z]
```

内存示意：

```text
第 1 步：y := x[:2:2]
底层数组 A: ["a", "b", "c", "d"]
              ↑
              ├── x: ptr→A[0], len=4, cap=4
              └── y: ptr→A[0], len=2, cap=2     ← 此时 y[0]=... 会改到 x

第 2 步：y = append(y, "z")  扩容
底层数组 A: ["a", "b", "c", "d"]   ← x 仍指向这里
底层数组 B: ["a", "b", "z", _]     ← y 改指向新数组
              ↑
              └── y: ptr→B[0], len=3, cap=4

第 3 步：y[0] = "Q"
底层数组 A: ["a", "b", "c", "d"]   ← x 不受影响
底层数组 B: ["Q", "b", "z", _]     ← 只有 y 看得见
```

> **关键判断**：`append` **是否触发扩容**决定共享是否断开。
>
> - **没扩容** → 父子切片仍共享底层数组，互相能改到对方。
> - **扩容了** → 子切片搬家到新数组，从此互不相干。
>
> 三段切片表达式的妙处就是把 cap 卡死，**强制让下一次 append 必须扩容**，从而保证父子切片在那一刻就彻底脱钩。
>
> **建议**：取子切片后，要么完全只读，要么始终用三段表达式。

---

## 5. copy 函数

`copy` 用于创建**完全独立**的切片副本。

```go
x := []int{1, 2, 3, 4}
y := make([]int, 4)
num := copy(y, x)         // 返回拷贝的元素数 4
fmt.Println(y, num)       // [1 2 3 4] 4
```

要点：
- 拷贝数量取 `min(len(src), len(dst))`，与容量无关。
- 支持源/目标重叠：`copy(x[:3], x[1:])` → x 变为 `[2 3 4 4]`。
- 不需要返回值时可不接收。
- 数组也能作为 src 或 dst：`copy(y, d[:])` / `copy(d[:], x)`。

---

## 6. 数组与切片的互转

### 6.1 数组 → 切片

```go
xArray := [4]int{5, 6, 7, 8}
xSlice := xArray[:]   // 整体转切片
y := xArray[:2]
z := xArray[2:]
```

> 注意：从数组取切片，**也共享内存**（与数组取切片同样的副作用）。

### 6.2 切片 → 数组

数组是**值类型**，转换会**复制数据**到新内存。修改切片不影响转换后的数组：

```go
xSlice := []int{1, 2, 3, 4}
xArray := [4]int(xSlice)        // 完整转换
smallArray := [2]int(xSlice)    // 只取前 2 个
xSlice[0] = 10
fmt.Println(xArray)     // [1 2 3 4]   未变
fmt.Println(smallArray) // [1 2]
```

约束：

- 数组长度必须**编译期确定**，不能用 `[...]`。
- 数组长度 > 切片**长度**（不是容量）会运行时 panic。

### 6.3 切片 → 数组指针（共享内存）

```go
xSlice := []int{1, 2, 3, 4}
xArrayPointer := (*[4]int)(xSlice)
xSlice[0] = 10
xArrayPointer[1] = 20
fmt.Println(xSlice)         // [10 20 3 4]
fmt.Println(xArrayPointer)  // &[10 20 3 4]
```

---

## 7. 字符串、Rune、Byte

### 7.1 字符串的本质

> Go 中的字符串是**字节序列**（不一定是 rune 序列）。源代码默认 UTF-8 编码，所以字符串字面量通常按 UTF-8 存储。

支持索引和切片表达式：

```go
var s string = "Hello there"
var b byte = s[6]   // 116（小写 t 的 ASCII）
var s2 = s[4:7]     // "o t"
var s3 = s[:5]      // "Hello"
```

### 7.2 索引/切片字符串的陷阱 ⚠️

字符串以**字节**为单位，但一个 UTF-8 码点可能占 1~4 字节：

```go
var s string = "Hello ☀"
var s4 string = s[6:]   // 只取了 ☀ 的第一个字节，得到一个无效字符 �
fmt.Println(len(s))     // 10，不是 7
```

> 仅当确定字符串只含单字节字符时，才能放心用索引/切片。

### 7.3 字符串与切片互转

```go
var a rune = 'x'
var s string = string(a)  // "x"
var b byte = 'y'
var s2 string = string(b) // "y"

// 字符串 ↔ []byte / []rune
var s string = "Hello, ☀"
var bs []byte = []byte(s) // UTF-8 字节序列
var rs []rune = []rune(s) // 码点序列
```

> ⚠️ **常见 bug**：`string(int(65))` 返回的是 `"A"`，**不是** `"65"`。从 Go 1.15 起，`go vet` 会拦截除 rune/byte 外的 int → string 转换。

### 7.4 实践建议

- 子串与码点提取：用标准库的 `strings` 和 `unicode/utf8` 包，而不是裸切片。
- 遍历字符（按码点）：下一章会用 `for-range`。

---

## 8. Map

### 8.1 声明方式

```go
var nilMap map[string]int           // nil map：可读不可写（写入会 panic）
totalWins := map[string]int{}       // 空 map 字面量：可读可写
teams := map[string][]string{
    "Orcas":   []string{"Fred", "Ralph", "Bijou"},
    "Lions":   []string{"Sarah", "Peter", "Billie"},
    "Kittens": []string{"Waldo", "Raul", "Ze"},
}
ages := make(map[int][]string, 10)  // 预分配容量
```

### 8.2 Map 的特性

- 零值为 `nil`，长度为 0。
- 不可比较（不能用 `==`），只能与 `nil` 比较。
- 键必须是**可比较**类型 → 不能用 slice 或 map 作为键。
- 容量可以超过初始指定值，会自动扩容。
- `len(m)` 返回键值对数。

### 8.3 读写

```go
totalWins := map[string]int{}
totalWins["Orcas"] = 1
totalWins["Lions"] = 2
fmt.Println(totalWins["Orcas"])    // 1
fmt.Println(totalWins["Kittens"])  // 0  ← 键不存在返回零值
totalWins["Kittens"]++             // 由零值 0 自增到 1
```

### 8.4 comma ok 惯用法

区分"键存在但值为零"与"键不存在"：

```go
m := map[string]int{"hello": 5, "world": 0}
v, ok := m["hello"]   // 5  true
v, ok = m["world"]    // 0  true
v, ok = m["goodbye"]  // 0  false
```

### 8.5 删除与清空

```go
delete(m, "hello")  // 不存在或 m 为 nil 都不会报错；无返回值
clear(m)            // 清空所有键值对，长度变为 0（与切片不同）
```

### 8.6 Map 比较（Go 1.21+）

```go
import "maps"
maps.Equal(m, n)
maps.EqualFunc(m, n, func(a, b V) bool { ... })
```

### 8.7 用 Map 模拟集合

Go 没有内置 set，可用 `map[T]bool`：

```go
intSet := map[int]bool{}
vals := []int{5, 10, 2, 5, 8, 7, 3, 9, 1, 2, 10}
for _, v := range vals {
    intSet[v] = true
}
fmt.Println(len(vals), len(intSet)) // 11 8
fmt.Println(intSet[5])              // true
fmt.Println(intSet[500])            // false
```

> 也可以用 `map[T]struct{}` 节省内存（空 struct 占 0 字节），但赋值与判断更繁琐，需要用 comma ok。

### 8.8 切片 vs Map：何时用哪个

- **切片**：顺序敏感、按位置访问的列表型数据。
- **map**：用非整数自然键（如名字）组织的数据。

---

## 9. Struct（结构体）

### 9.1 声明与字面量

```go
type person struct {
    name string
    age  int
    pet  string
}

var fred person                      // 零值结构体：所有字段为字段类型的零值
bob := person{}                      // 与上等价
julia := person{                     // 位置字面量：必须按声明顺序提供所有字段
    "Julia",
    40,
    "cat",
}
beth := person{                      // 命名字面量：可乱序、可省略
    age:  30,
    name: "Beth",
}
bob.name = "Bob"                     // 字段访问用点号
```

> 同一字面量中**不能混用**位置式与命名式。
> 推荐命名式：可读、可维护，新增字段不会破坏旧代码。

### 9.2 匿名结构体

```go
var person struct {
    name string
    age  int
    pet  string
}
person.name = "bob"
person.age = 50

pet := struct {
    name string
    kind string
}{
    name: "Fido",
    kind: "dog",
}
```

适用场景：
1. JSON / Protobuf 等数据序列化（unmarshal/marshal）。
2. 表驱动测试。

### 9.3 结构体的可比较性

- 字段全为可比较类型 → 整个结构体可比较。
- 含有 slice、map、function、channel 字段 → 不可比较。
- Go 没有运算符重载，不能像 Python 那样定义 `__eq__`，要比较只能自己写函数。

### 9.4 结构体类型转换

允许在结构体之间做类型转换的条件：**字段名、顺序、类型完全一致**。

```go
type firstPerson struct {
    name string
    age  int
}
type secondPerson struct { // 字段名/顺序/类型都相同 → 可转换
    name string
    age  int
}
type thirdPerson struct {  // 顺序不同 → 不可转换
    age  int
    name string
}
type fourthPerson struct { // 字段名不同 → 不可转换
    firstName string
    age       int
}
type fifthPerson struct {  // 多了字段 → 不可转换
    name          string
    age           int
    favoriteColor string
}
```

> **匿名结构体的特殊待遇**：当至少有一方是匿名结构体且字段一致时，可以直接 `==` 比较，也可以直接赋值，无需类型转换。

```go
type firstPerson struct {
    name string
    age  int
}
f := firstPerson{name: "Bob", age: 50}
var g struct {
    name string
    age  int
}
g = f             // 合法
fmt.Println(f == g) // 合法
```

---

## 10. 本章重点回顾

| 类型 | 是否可比较 | 长度可变 | 零值 | 备注 |
|------|-----------|---------|------|------|
| 数组 `[N]T` | ✅（元素可比较时） | ❌ | 元素全为零值 | 长度是类型的一部分，少用 |
| 切片 `[]T` | ❌（仅可与 nil 比） | ✅ | `nil` | 最常用序列容器 |
| 字符串 `string` | ✅ | ❌ | `""` | 不可变字节序列 |
| map `map[K]V` | ❌（仅可与 nil 比） | ✅ | `nil` | 键必须可比较 |
| struct | 视字段而定 | ❌ | 各字段零值 | 自定义聚合类型 |

**最容易踩的坑**：

1. `append` 一定要把返回值赋回原变量。
2. 切片的子切片**共享底层数组**，必要时用三段切片表达式或 `copy` 隔离。
3. 字符串切片以**字节**为单位，跨多字节 UTF-8 字符会切坏。
4. nil map 可读不可写，写入会 panic；nil 切片可以直接 `append`。
5. `make([]T, n)` 创建的切片已有 n 个零值元素，再 `append` 是**追加在后面**，不是覆盖。
