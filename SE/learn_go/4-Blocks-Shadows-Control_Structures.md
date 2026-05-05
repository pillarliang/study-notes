# Chapter 4：代码块、shadowing 与控制结构（Blocks, Shadows, and Control Structures）

本章把 Go 的程序组织从"声明数据"推进到"组织逻辑"。先讲 **block**（决定标识符在何处可见、何时被 shadow），再讲三大控制结构：`if`、`for`、`switch`，最后是几乎不用但必须知道的 `goto`。

---

## 1. 代码块（Block）

Go 允许在多种位置声明变量：函数外、函数参数、函数内局部变量。**每一处声明所在的语法范围称为一个 block**。理解 block 的层级，是理解作用域和 shadowing 的前提。

Go 的 block 自外向内有 4 层：


| 层级                 | 包含的内容                                                              |
| ------------------ | ------------------------------------------------------------------ |
| **universe block** | Go 语言本身预置的标识符：`int`、`string`、`true`、`false`、`nil`、`make`、`close` 等 |
| **package block**  | 在所有函数之外声明的变量、常量、类型、函数                                              |
| **file block**     | `import` 进来的包名，仅对当前文件可见                                            |
| **inner block**    | 函数体、`if` / `for` / `switch` 等控制结构的 `{}`，每对花括号都是新的 block            |


> **universe block 的特殊之处**：Go 只有 25 个关键字，`true`、`false`、`nil`、`int`、`make`、`close` 等并不是关键字，而是声明在 universe block 中的"预声明标识符"（predeclared identifier）。这意味着它们**可以被 shadow**（见下一节）。

内层 block 可以访问外层 block 的标识符，反过来不行。

---

## 2. Shadowing（变量遮挡）

### 2.1 什么是 shadowing

> 当内层 block 声明了一个与外层同名的变量时，内层这个变量"遮住"了外层那个 —— 在内层 block 存活期间，无法访问外层那个同名变量。

```go
func main() {
    x := 10
    if x > 5 {
        fmt.Println(x)   // 10：仍是外层 x
        x := 5           // 在 if 的 block 里新建了一个 x，shadow 外层
        fmt.Println(x)   // 5：内层 x
    }
    fmt.Println(x)       // 10：if block 结束，shadowing 消失，外层 x 重新可见
}
```

### 2.2 包名也会被 shadow

`import` 导入的包名属于 file block，照样可以被局部变量 shadow：

```go
func main() {
    x := 10
    fmt.Println(x)
    fmt := "oops"        // 在函数内声明了 fmt 字符串，shadow 了 fmt 包
    fmt.Println(fmt)     // 编译错误：string 没有 Println 方法
}
```

报错信息：`fmt.Println undefined (type string has no field or method Println)`。问题不是变量名取得不好，而是这个名字**意外撞上了 file block 中的包名**。

### 2.3 universe block 标识符也会被 shadow

由于 `true`、`false`、`nil`、内置函数等都只是声明在 universe block，照样能被 shadow：

```go
fmt.Println(true)
true := 10            // 合法但极其危险：把 true 重定义成 int 10
fmt.Println(true)     // 输出 10
```

**绝对不要重定义 universe block 中的标识符**。一旦不小心做了，后果从编译失败到难以追踪的 bug 都有可能。

### 2.4 `go vet` 不会报告 shadowing

由于 shadowing 偶尔有合理用途，`go vet` 默认**不会**把它当作错误。书中第 267 页 "Using Code-Quality Scanners" 介绍的第三方工具（如 `shadow`）可以专门检测意外 shadowing。

---

## 3. `if` 语句

**语法骨架：**

```go
if [简单语句;] 条件 {
    // 分支语句
} else if [简单语句;] 条件 {     // else if 可重复，可省
    // 分支语句
} else {                         // else 可省
    // 兜底分支
}
```

Go 与其他语言有两点关键差异：

1. **条件两侧不加 `()`**。
2. **可在头部用一个简单语句声明变量**，作用域覆盖整个 `if` / `else if` / `else` 链。

下面分别展开。

### 3.1 不写括号

条件表达式两侧**不加 `()`**：

```go
n := rand.Intn(10)
if n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
```

### 3.2 在 `if` 头部声明作用域变量

可以在条件之前用一个 `simple statement` 声明变量，**该变量的作用域覆盖整个 `if` / `else if` / `else` 链**，但仅限于此：

```go
if n := rand.Intn(10); n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
// 这里访问 n 会编译错误：undefined: n
```

好处：把"只在判断里用一次"的临时变量限制在最小作用域，不污染外层。

> ⚠️ `if` 头部声明的变量同样遵循 shadowing 规则：如果与外层同名，外层那个会被遮住。

---

## 4. `for`：四种形式

C 派生语言里通常有 `for` / `while` / `do-while`，**Go 把它们都合并到了 `for` 一个关键字**，提供 4 种语法形式：

```go
for 初始化; 条件; 自增 { ... }     // 形式 1：完整 C 风格三段式
for 条件 { ... }                   // 形式 2：仅条件（替代 while）
for { ... }                        // 形式 3：无限循环（靠 break/return 退出）
for k, v := range 集合 { ... }     // 形式 4：for-range（遍历复合类型）
```

下面分别展开。

### 4.1 完整 C 风格 `for`

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

三段语义与 C 一致：**初始化**、**比较**、**自增**。两条 Go 特有约束：

- 初始化部分**必须用 `:=`**，不允许用 `var`。
- 任意一段（甚至全部）都可以省略；省略后**分号也要去掉**，否则 `go fmt` 会替你删。

省略示例：

```go
i := 0
for ; i < 10; i++ {           // 省略初始化
    fmt.Println(i)
}

for i := 0; i < 10; {         // 省略自增（自增逻辑挪进循环体）
    fmt.Println(i)
    if i % 2 == 0 {
        i++
    } else {
        i += 2
    }
}
```

### 4.2 仅条件 `for`（替代 `while`）

把初始化和自增都省掉，**且不写分号**：

```go
i := 1
for i < 100 {
    fmt.Println(i)
    i = i * 2
}
```

这是 Go 中等价于 `while (cond)` 的写法。

### 4.3 无限 `for`

连条件都省掉：

```go
for {
    fmt.Println("Hello")
}
```

跳出方式：`break` 或 `return`。

> Go 没有 `do-while`。如果要"至少执行一次再判断"，用无限 `for` + 末尾 `if !cond { break }`：
>
> ```go
> for {
>     // do something
>     if !condition {
>         break
>     }
> }
> ```

### 4.4 `break` 与 `continue`

- `break`：立即跳出最内层 `for`。
- `continue`：跳过当前迭代剩余部分，进入下一次循环。

```go
// 用 continue 把条件平铺
for i := 1; i <= 100; i++ {
    if i % 3 == 0 && i % 5 == 0 {
        fmt.Println("FizzBuzz")
        continue
    }
    if i % 3 == 0 {
        fmt.Println("Fizz")
        continue
    }
    if i % 5 == 0 {
        fmt.Println("Buzz")
        continue
    }
    fmt.Println(i)
}
```

### 4.5 `for-range`

第四种形式专门用于遍历 Go 的内置复合类型（[[3-composite_types|数组、切片、字符串、map、channel]]）以及基于它们的自定义类型。

**语法骨架（按变量取法分 4 种）：**

```go
for k, v := range coll { ... }   // 同时取 index/key 和 value
for k := range coll { ... }      // 只取 index/key（map 常用）
for _, v := range coll { ... }   // 用 _ 忽略 index/key
for range coll { ... }           // 都不取，只遍历次数
```

不同容器里两个变量的含义不同：

| 容器        | 第一个变量      | 第二个变量              |
| --------- | ---------- | ------------------ |
| 数组、切片、字符串 | `i`（index） | `v`（value，字符串里是 rune） |
| map       | `k`（key）   | `v`（value）         |
| channel   | `v`（value） | —（只有一个变量）          |

**基本用法：**

```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for i, v := range evenVals {
    fmt.Println(i, v)
}
```

每次迭代得到两个变量：**位置（index/key）** 和 **值（value）**。

用 `_` 忽略不需要的变量

Go 强制使用所有声明的变量，循环变量也不例外。如果不需要 index：

```go
for _, v := range evenVals {
    fmt.Println(v)
}
```

`_` 是占位符，告诉 Go "这里有个值，我故意忽略它"。

#### 4.5.1 只要 key 不要 value（map 专用语法糖）

```go
uniqueNames := map[string]bool{"Fred": true, "Raul": true, "Wilma": true}
for k := range uniqueNames {
    fmt.Println(k)
}
```

注意：**只省略 value 时连 `_` 都不用写**。最常见用途是把 map 当 set 使用。

> 数组、切片在迭代中通常需要 value，"只取 index" 的需求很少见 —— 如果遇到，往往说明数据结构选错了。

#### 4.5.2 遍历 map：顺序是随机的

```go
m := map[string]int{"a": 1, "c": 3, "b": 2}
for i := 0; i < 3; i++ {
    fmt.Println("Loop", i)
    for k, v := range m {
        fmt.Println(k, v)
    }
}
```

每轮 `Loop` 输出的 key 顺序都可能不同。**这是有意设计的安全特性**：

1. 早期 Go 在相同插入顺序下迭代顺序基本固定，开发者写出依赖该顺序的代码，后续升级时崩溃。
2. 如果哈希函数固定，攻击者可构造大量哈希冲突的 key 触发 **Hash DoS**（让 server 的 map 退化为线性结构）。

应对：每次创建 map 时混入随机种子，并让每次 `for-range` 的迭代顺序额外抖动。

> **例外**：`fmt.Println` 等格式化函数为了便于调试，输出 map 时**会按 key 升序排序**。

#### 4.5.3 遍历字符串：按 rune 而不是 byte

**先立三个前提：**

1. **Go 的 string 本质上是只读 byte 序列**。`len(s)` 数的是字节数，不是字符数。
2. **UTF-8 是变长编码**：ASCII 1 字节，希腊字母/拉丁扩展（π 等）2 字节，大多数汉字 3 字节，emoji/罕见字符 4 字节。
3. **rune 是 `int32` 的别名，存一个 Unicode 码点**——也就是人类直觉里的"一个字符"。例如 `'a'` 的 rune 值是 97，`'π'` 是 960，`'中'` 是 20013。

> ⚠️ 关键：同一个字符有**两种存储形态**，占用字节数不同。

| 形态 | 容器 | 占用 | 内容（以 `π` 为例） |
|---|---|---|---|
| **作为 rune 变量** | `int32`（定长） | 固定 4 字节 | 整数 `960`（码点） |
| **作为 UTF-8 字节存在 string 里** | byte 序列（变长） | 1~4 字节 | `0xCF 0x80`（两字节） |

数字 `960` 和字节 `CF 80` 不是两个不同的东西——`CF 80` 就是把 `960` 这个码点按 UTF-8 规则压成的字节表示。`for-range` 字符串做的事就是反向操作：**从 string 里按 UTF-8 规则读 1~4 字节，解码还原成一个 rune 装进 value 变量**。

**推论：两种遍历方式拿到的东西完全不同。**


| 写法                                  | 每次拿到                  | 多字节字符的下场                |
| ----------------------------------- | --------------------- | ----------------------- |
| `for i := 0; i < len(s); i++ { s[i] }` | **byte**（uint8，0~255） | 被切成独立字节，单独看无意义          |
| `for i, r := range s`               | **rune**（int32，完整码点）  | Go 自动按 UTF-8 解码，一次得一个完整字符 |


`for-range` 是 Go 中**唯一会做 UTF-8 解码**的循环形式。普通 for + `s[i]` 只是按字节走索引。这就是"想正确遍历字符必须用 `for-range`"的根本原因。

**用 `"apple_π!"` 验证：**

π 在 UTF-8 中是两字节 `0xCF 0x80`，字符串的字节布局：


| byte 位置  | 0  | 1  | 2  | 3  | 4  | 5  | **6**  | **7**  | 8  |
| -------- | -- | -- | -- | -- | -- | -- | ------ | ------ | -- |
| 字节值（hex） | 61 | 70 | 70 | 6C | 65 | 5F | **CF** | **80** | 21 |
| 字符       | a  | p  | p  | l  | e  | _  | π ←┐   | ┘      | !  |


```go
for i, r := range "apple_π!" {
    fmt.Println(i, r, string(r))
}
```

输出：

```
0 97 a
1 112 p
2 112 p
3 108 l
4 101 e
5 95 _
6 960 π      ← r=960 是 π 的码点，不是 byte 值（byte 上限 255）
8 33 !       ← byte 7 被 π 占用，下一个 rune 从 byte 8 开始
```

两条关键观察刚好对应回原理：

- `r=960` 远超 byte 范围（0~255）→ 印证原理 3：拿到的是 rune，不是 byte。
- index 从 6 跳到 8 → 印证原理 2 和推论：`for-range` 一次"吃掉"了 π 的两个字节。

**边界情况**：遇到非法 UTF-8 字节时，rune 得到 Unicode 替换字符 `0xfffd`（"�"），index 前进 1 byte。

参考上一章 [[3-composite_types#7. 字符串、Rune、Byte|字符串那节]]。

#### `for-range` 的 value 是副本

```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for _, v := range evenVals {
    v *= 2
}
fmt.Println(evenVals)  // [2 4 6 8 10 12]，原切片完全没变
```

修改 value 变量不会回写原集合 —— `v` 只是该位置元素的拷贝。

#### Go 1.22 的关键变化：每轮迭代创建新变量

- **Go 1.22 之前**：`for-range` 的 index 和 value 变量**只创建一次**，每轮迭代复用同一块内存。
- **Go 1.22 起**：默认**每轮创建新变量**。

这看似细枝末节，但解决了一个常见 bug：在循环里启动 goroutine 并捕获循环变量时，旧行为会让所有 goroutine 共享同一个变量、读到最后一轮的值（详见书中 "Goroutines, for Loops, and Varying Variables" 第 298 页）。

由于这是破坏性变更（即便修的是 bug），可通过 `go.mod` 中的 `go` 指令版本号控制：声明 `go 1.22` 及以上启用新行为，旧版本仍是旧行为。

---

## 5. 给 `for` 加标签（Labeling）

`break` 和 `continue` 默认作用于**最内层** `for`。要操作外层循环，需要给外层加 **label**。

**语法骨架：**

```go
labelName:
    for ... {
        for ... {
            break labelName       // 跳出 labelName 标记的循环（不只是最内层）
            continue labelName    // 跳到 labelName 标记循环的下一轮
        }
    }
```

- label 名后跟 `:`，缩进与外围函数大括号同层（由 `go fmt` 自动处理）。
- `break label` / `continue label` 可指向任意被标注的外层循环。

**实例：**

```go
func main() {
    samples := []string{"hello", "apple_π!"}
outer:
    for _, sample := range samples {
        for i, r := range sample {
            fmt.Println(i, r, string(r))
            if r == 'l' {
                continue outer    // 直接跳到外层下一轮
            }
        }
        fmt.Println()
    }
}
```

风格约定：label **缩进与外围函数大括号同层**（由 `go fmt` 自动处理），与下方代码块产生明显视觉差，便于扫读。

最常见用途是这种模式：

```go
outer:
    for _, outerVal := range outerValues {
        for _, innerVal := range outerVal {
            // 处理 innerVal
            if invalidSituation(innerVal) {
                continue outer   // 整个外层一旦发现问题就放弃当前 outer
            }
        }
        // 这里的代码只在所有 innerVal 都合法时执行
    }
```

嵌套 `for` 加 label 的场景不算多，但需要时它是最干净的写法。

---

## 6. 选哪种 `for`


| 场景                              | 推荐                                       |
| ------------------------------- | ---------------------------------------- |
| 遍历切片 / 数组 / map / 字符串 / channel | `for-range`（默认选项）                        |
| **遍历字符串**                       | **必须** `for-range`，否则切坏多字节字符             |
| 不是从头到尾遍历整个集合                    | 完整 `for`                                 |
| 基于计算条件循环（类似 `while`）            | 仅条件 `for`                                |
| 真无限循环（如事件循环）                    | 无限 `for`，但循环体内**必须有 `break` 或 `return`** |


对比："取数组下标 1 到倒数第 2"：

```go
// for-range：要靠 if + continue/break 凑边界
for i, v := range evenVals {
    if i == 0 {
        continue
    }
    if i == len(evenVals) - 1 {
        break
    }
    fmt.Println(i, v)
}

// 标准 for：边界一目了然
for i := 1; i < len(evenVals) - 1; i++ {
    fmt.Println(i, evenVals[i])
}
```

**只要不是从头扫到尾，标准 `for` 通常更清楚。**

> ⚠️ 标准 `for` 通过 `i++` 步进字节，**不能正确处理多字节字符串**。要在字符串里跳过若干 rune，必须用 `for-range`。

---

## 7. `switch`

C 系语言的 `switch` 因为「能比较的类型有限」「默认 fall-through」往往让人避而远之。**Go 的 `switch` 重新设计后变得真正好用**。

### 7.1 基本结构

**语法骨架：**

```go
switch [初始化语句;] [比较值] {
case 值1, 值2, ...:
    // 分支语句（不写 {}，不写 break）
case 值3:
    // 分支语句
default:
    // 兜底分支
}
```

三处可选位置（`[...]` 表示可省）：

- **初始化语句** —— 与 `if` 头部一致，声明的变量作用域覆盖整个 `switch`。省略时连后面的分号也要去掉。
- **比较值** —— 每个 `case` 用 `==` 与它比较。省略时退化为「空 switch」（见第 8 节），`case` 后写任意 bool 表达式。
- **`default`** —— 兜底分支，可省略；位置任意，习惯放最后。

**完整示例：**

```go
words := []string{"a", "cow", "smile", "gopher", "octopus", "anthropologist"}
for _, word := range words {
    switch size := len(word); size {
    case 1, 2, 3, 4:
        fmt.Println(word, "is a short word!")
    case 5:
        wordLen := len(word)
        fmt.Println(word, "is exactly the right length:", wordLen)
    case 6, 7, 8, 9:
    default:
        fmt.Println(word, "is a long word!")
    }
}
```

输出：

```
a is a short word!
cow is a short word!
smile is exactly the right length: 5
anthropologist is a long word!
```

需要逐条记住的特性：

1. **被比较的值不加 `()`**（与 `if`、`for` 一致）。
2. 可在 `switch` 头部声明作用域变量（`size := len(word)`），作用域覆盖全部 `case`。
3. 整个 `switch` 用 `{}` 包起来，但**单个 `case` 不再加 `{}`**。一个 `case` 内可以有多行语句，它们都属于同一个 block —— `case 5` 内部的 `wordLen` 只在该 `case` 内可见。
4. **case 默认不 fall-through**，不需要在每个 `case` 末尾写 `break`。
5. **多值匹配用逗号**：`case 1, 2, 3, 4:` 表示这四个值任一命中都执行该分支。
6. **空 `case` 什么都不做**（不会"穿透"到下一个）：`case 6, 7, 8, 9:` 后面没语句，遇到这些值就静默跳过。
7. `default` 处理所有未命中的值，可放在任意位置（习惯放最后）。

### 7.2 关于 `fallthrough`

为了完整性，Go 提供 `fallthrough` 关键字让某个 `case` 显式落入下一个。**但绝大多数情况下不要用它**，需要时优先重构逻辑去掉 case 之间的依赖。

### 7.3 可比较的类型

`switch` 的 case 值必须可以用 `==` 比较，即**除 slice、map、channel、function、含这些字段的 struct 之外的所有内置类型**都可以。

### 7.4 `switch` 内 `break` 的特殊用法

`case` 内可以用 `break` **提前退出该 case**。但若真有这种需要，往往说明该 case 逻辑过于复杂，应当重构。

更需要小心的是：`**switch` 嵌在 `for` 里时，case 内的 `break` 跳的是 `case` 而不是 `for`**。要跳出外层 `for`，必须给 `for` 加 label：

```go
func main() {
loop:                                        // ← 给 for 加 label
    for i := 0; i < 10; i++ {
        switch i {
        case 0, 2, 4, 6:
            fmt.Println(i, "is even")
        case 3:
            fmt.Println(i, "is divisible by 3 but not 2")
        case 7:
            fmt.Println("exit the loop!")
            break loop                       // ← 显式指明跳出 for
        default:
            fmt.Println(i, "is boring")
        }
    }
}
```

去掉 label 时，`break` 只跳出 `case 7`，`for` 会继续执行到 `i == 9`。

---

## 8. 空 `switch`（Blank Switch）

`switch` 也可以**不指定被比较值**，每个 `case` 写任意 bool 表达式。

**语法骨架：**

```go
switch [初始化语句;] {            // 注意：分号后面留空，没有比较值
case bool 表达式1:
    // 分支
case bool 表达式2:
    // 分支
default:
    // 兜底
}
```

与普通 `switch` 的唯一差别：**省掉比较值**，于是 `case` 后跟的不再是常量值，而是任意布尔表达式。

```go
words := []string{"hi", "salutations", "hello"}
for _, word := range words {
    switch wordLen := len(word); {           // ← 注意 ; 后面没有变量
    case wordLen < 5:
        fmt.Println(word, "is a short word!")
    case wordLen > 10:
        fmt.Println(word, "is a long word!")
    default:
        fmt.Println(word, "is exactly the right length.")
    }
}
```

普通 `switch` 只能做等值比较，空 `switch` 把 `case` 升级成"任意布尔判断"。仍可在头部用 `:=` 声明作用域变量。

**反例**：如果空 `switch` 的所有 case 都是同一个变量的等值比较，就退化成普通 `switch`：

```go
// 不要写成空 switch
switch {
case a == 2:
    fmt.Println("a is 2")
case a == 3:
    fmt.Println("a is 3")
}

// 应该写成普通 switch
switch a {
case 2:
    fmt.Println("a is 2")
case 3:
    fmt.Println("a is 3")
}
```

---

## 9. `if`/`else` 链 vs `switch`

功能上 `if/else` 链 和 blank `switch` 等价。选择标准：**这一组 case 在语义上是不是同一类问题**。

`switch`（包括空 switch）暗示了"这些分支属于同一个判定"。如果你正在做的多个比较彼此相关，用 `switch` 让"它们是一组"的关系一目了然：

```go
// FizzBuzz 用 blank switch 重写
for i := 1; i <= 100; i++ {
    switch {
    case i % 3 == 0 && i % 5 == 0:
        fmt.Println("FizzBuzz")
    case i % 3 == 0:
        fmt.Println("Fizz")
    case i % 5 == 0:
        fmt.Println("Buzz")
    default:
        fmt.Println(i)
    }
}
```

比 `continue` 链版本更直观，`default` 把"其他情况"显式化了。

**反过来**，如果各 case 在判断毫不相关的事情，硬塞进 `switch` 反而误导读者，老老实实用 `if/else`。

---

## 10. `goto`：是的，Go 也有

**语法骨架：**

```go
goto labelName     // 无条件跳转
...
labelName:         // 跳转目标（label 写法同 §5）
    // 跳到这里继续执行
```

Go 保留了 `goto`，但施加了**严格限制**，让它比传统 `goto` 安全得多：

- 不能跳过变量声明。
- 不能跳进内层 block 或并列 block。

下面这段同时违反两条，编译报错：

```go
func main() {
    a := 10
    goto skip            // 跳过了 b 的声明 → 非法
    b := 20
skip:
    c := 30
    fmt.Println(a, b, c)
    if c > a {
        goto inner       // 跳进 if 内部的 inner 标签 → 非法
    }
    if a < b {
    inner:
        fmt.Println("a is less than b")
    }
}
```

报错：

```
goto skip jumps over declaration of b at ./main.go:8:4
goto inner jumps into block starting at ./main.go:15:11
```

### 10.1 几乎不需要 `goto`

绝大多数 `goto` 用例都能用带 label 的 `break` / `continue` 替代。书中给出一个**勉强"合理"的人造例子**：

```go
func main() {
    a := rand.Intn(10)
    for a < 100 {
        if a % 5 == 0 {
            goto done            // 一旦满足条件，直接跳到收尾
        }
        a = a*2 + 1
    }
    fmt.Println("do something when the loop completes normally")
done:
    fmt.Println("do complicated stuff no matter why we left the loop")
    fmt.Println(a)
}
```

需求是：循环正常结束要执行 A 段，提前退出要跳过 A 段直奔 B 段。两个常见替代方案都不漂亮：

- **bool flag**：循环里设 flag，结束后 if 判断 → 多了一个状态变量。
- **复制 B 段代码**：在循环中"提前结束"那一支也写一遍 B 段 → 维护负担。

这两种替代都比 `goto` 更啰嗦或更难维护。

### 10.2 标准库中的真实用例

`strconv` 包 `atof.go` 中的 `floatBits` 方法尾部：

```go
overflow:
    // ±Inf
    mant = 0
    exp  = 1<<flt.expbits - 1 + flt.bias
    overflow = true

out:
    // Assemble bits.
    bits := mant & (uint64(1)<<flt.mantbits - 1)
    bits |= uint64((exp - flt.bias) & (1<<flt.expbits - 1)) << flt.mantbits
    if d.neg {
        bits |= 1 << flt.mantbits << flt.expbits
    }
    return bits, overflow
```

函数前半段有多种条件检查：有些情况下溢出后还要进入 `out` 段拼装位；有些直接跳到 `out`。用 `goto overflow` / `goto out` 比布尔 flag 矩阵清晰得多。

> **总结**：`goto` 可以用，但首先穷尽其他办法；只有当**用了 `goto` 之后代码确实更清晰**，才使用它。

---

## 11. 章末练习

1. 写一个 `for` 循环，把 100 个 0~100 之间的随机数填进 `[]int`。
2. 遍历该切片，按规则输出：
  - 能被 2 整除：`Two!`
  - 能被 3 整除：`Three!`
  - 同时能被 2 和 3 整除：`Six!`（**只输出 Six，不输出 Two/Three**）
  - 否则：`Never mind`
3. 新建程序：在 `main` 中声明 `total int`，写一个 `for` 循环让 `i` 从 0 到 9（不含 10），循环体：
  ```go
   total := total + i
   fmt.Println(total)
  ```
   循环结束后打印 `total`。问题：实际打印什么？bug 在哪？
  > 提示：`total := total + i` 中的 `:=` 在 `for` 的 block 内**新建了一个局部 `total`**，shadow 了外层的 `total`。每轮迭代外层 `total` 都从 0 开始计算，最终外层 `total` 仍是 0 —— 这正是本章第 2 节 shadowing 陷阱的现实版本。

---

## 12. 速查与最易错点回顾

### 控制结构对比


| 结构          | 关键特性                                    |
| ----------- | --------------------------------------- |
| `if`        | 无 `()`；可在头部声明作用域变量                      |
| 完整 `for`    | 三段都可省；初始化必须用 `:=`，省略后分号也要去              |
| 仅条件 `for`   | 等价于 `while`，**不写分号**                    |
| 无限 `for`    | 必须靠 `break` / `return` 退出               |
| `for-range` | 唯一能正确按 rune 遍历字符串的形式；map 顺序随机           |
| `switch`    | case 默认不 fall-through；多值用逗号；可在头部声明作用域变量 |
| 空 `switch`  | case 写任意布尔表达式；同变量等值比较应退化为普通 `switch`    |
| `goto`      | 限制：不跳过声明、不跳进内/并列 block；通常有更好的替代         |


### 最容易踩的坑

1. `:=` 在内层 block 里同名变量永远是新变量，**容易意外 shadow** 外层；包名、`true`/`nil` 也能被 shadow。
2. `for-range` 的 value 是**拷贝**，修改它不会影响原集合。
3. **Go 1.22 起** `for-range` 每轮迭代创建新变量，goroutine 捕获循环变量的旧坑被修复，但行为受 `go.mod` 中 `go` 指令版本控制。
4. 普通 `for` 用 `i++` 按字节步进，**不能正确遍历多字节字符串**。
5. `switch` 嵌在 `for` 里时，`break` 默认跳出的是 `case`，要跳出 `for` 必须 label。
6. map 的 `for-range` 顺序**每次都不同**（防 Hash DoS），不要依赖；调试时 `fmt.Println` 是按 key 排序输出的特例。
7. 空 `case` 是"什么都不做"，**不是 fall-through**。

