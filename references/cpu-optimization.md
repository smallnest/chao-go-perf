# CPU 优化详解

## BCE — 边界检查消除

Go 是内存安全语言，每次 slice/array 索引访问都插入边界检查。编译器在可证明安全的情况下会消除这些检查。

### 查看 BCE

```bash
go build -gcflags="-d=ssa/check_bce" . 2>&1
# 或者针对特定函数
go build -gcflags="-d=ssa/check_bce/debug=1" . 2>&1
```

输出示例：
```
./main.go:10:7: Found IsInBounds
```

`Found IsInBounds` 表示边界检查被保留（未消除）。无输出 = 已消除。

### BCE 能消除的情况

```go
// 1. 常规正序遍历
func sum(s []int) int {
    total := 0
    for i := 0; i < len(s); i++ {
        total += s[i]  // BCE ✓
    }
    return total
}

// 2. range 遍历 — 总是 BCE
func sumRange(s []int) int {
    total := 0
    for _, v := range s {
        total += v  // BCE ✓
    }
    return total
}

// 3. 常量索引
func firstThree(s []int) (int, int, int) {
    return s[0], s[1], s[2]  // BCE ✓ (如果 len(s) 被证明 ≥ 3)
}

// 4. 显式检查后的访问
func safeIndex(s []int, i int) int {
    if i >= 0 && i < len(s) {
        return s[i]  // BCE ✓
    }
    return 0
}

// 5. 掩码限制索引
func maskedIndex(s []int, i int) int {
    return s[i & (len(s)-1)]  // 仅当 len(s) 是 2 的幂, 编译器有时可以 BCE
}
```

### BCE 无法消除的情况

```go
// 1. 从 len-1 向 0 遍历
func reverseSum(s []int) int {
    total := 0
    for i := len(s) - 1; i >= 0; i-- {
        total += s[i]  // 可能保留 Bounds Check
    }
    return total
}
// 修复: 使用 range
func reverseSumFixed(s []int) int {
    total := 0
    for _, v := range s {
        total += v
    }
    return total
}

// 2. 步长 > 1
func stepSum(s []int) int {
    total := 0
    for i := 2; i < len(s); i += 3 {
        total += s[i]  // 可能保留 Bounds Check
    }
    return total
}
// 修复: 提前截断
func stepSumFixed(s []int) int {
    total := 0
    sub := s[2:]  // 一次边界检查
    for i := 0; i < len(sub); i += 3 {
        total += sub[i]  // BCE
    }
    return total
}

// 3. 多个 slice 交叉访问
func crossSum(a, b []int) int {
    total := 0
    for i := 0; i < len(a); i++ {
        total += a[i] + b[i]  // b[i] 可能保留 Bounds Check
    }
    return total
}
// 修复: 提前统一检查
func crossSumFixed(a, b []int) int {
    n := len(a)
    if len(b) < n {
        n = len(b)
    }
    total := 0
    for i := 0; i < n; i++ {
        total += a[i] + b[i]  // BCE ✓
    }
    return total
}
```

## 内联 (Inlining)

### 查看内联决策

```bash
go build -gcflags="-m -m" . 2>&1 | grep -E "inlin(e|ing)"
```

输出示例：
```
./main.go:5:6: can inline add
./main.go:12:6: cannot inline multiply: function too complex
```

### 内联条件（Go 1.21+）

- 函数体不超过内联预算（约 80 条 SSA 语句）
- 不含 `select` (部分 Go 版本)
- 不含 `defer` 和 `go` (Go 1.14+ 部分放宽)
- 不含 `recover`
- 不含闭包赋值
- mid-stack inlining: Go 1.12+ 允许包含循环的中间层函数内联

### 内联友好的代码

```go
// Good: 小方法，编译器自动内联
type Point struct { X, Y int }

func (p Point) Add(q Point) Point {
    return Point{p.X + q.X, p.Y + q.Y}
}

func (p Point) Scale(factor int) Point {
    return Point{p.X * factor, p.Y * factor}
}

// Avoid: 大函数，无法内联
func (c *Complex) ProcessAll(items []Item) {
    // 200+ 行代码...
    // 拆分：把小部分提取成独立方法
}

// Better: 拆分后小方法自动内联
func (c *Complex) processOne(item Item) {
    // 只处理一个 item，代码足够小
}
```

### 何时手动内联

绝大多数情况不需要手动内联。但热路径中的微小函数调用（profile 证明）可以手动内联：

```go
// 热路径中反复调用
func (h *Header) isFlagSet(flag int) bool {
    return h.flags & flag != 0
}

// 如果 profile 显示调用开销显著，手动内联
for _, v := range data {
    if h.flags & flag != 0 {  // 手动内联
        process(v)
    }
}
```

## 编译器 Flag 完整速查

```bash
# 逃逸分析
-gcflags="-m"            # 第一级详情
-gcflags="-m -m"         # 第二级详情

# 内联
-gcflags="-m -m" | grep inlin

# 边界检查
-gcflags="-d=ssa/check_bce"
-gcflags="-d=ssa/check_bce/debug=1"  # 更详细

# 禁用优化（调试用）
-gcflags="-l"            # 禁用内联
-gcflags="-N"            # 禁用所有优化
-gcflags="-l -N"         # 禁用内联和优化

# 汇编输出
go tool compile -S main.go
go tool compile -S main.go | grep -A5 'funcName'

# 查看优化 pass
GOSSAFUNC=myFunc go build .
# 生成 ssa.html，浏览器打开

# 查看 SSA 中间表示
GOSSAFUNC=myFunc go tool compile main.go
# 每个编译阶段的 SSA 输出
```

### GOSSAFUNC 使用

```bash
GOSSAFUNC=Fib go build fib.go
open ssa.html
```

这是理解编译器优化的最佳方式。显示了从源码到机器码的每个编译阶段。

## 编译时优化

### 死代码消除

```go
const debug = false

func process() {
    if debug {  // 编译器消除整个 if 块
        log.Println("debug info")
    }
    // ...
}
```

### 常量传播和折叠

```go
const (
    bufSize = 4096
    align   = 64
)
// bufSize * align → 262144，编译时计算

// 字符串拼接常量
const s = "Hello " + "World"  // 编译时完成
```

### 函数体优化

```go
// 无需手动 memcpy — 编译器优化
func copyStruct(dst, src *BigStruct) {
    *dst = *src  // 编译器用 memcpy
}

// string([]byte) 在某些场景下被编译器优化为 stack allocation
func toString(b []byte) string {
    return string(b)  // 如果 b 不逃逸，可能栈上分配
}
```