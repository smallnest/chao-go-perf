# CPU 缓存与内存布局优化

## Cache Line 基础

- CPU cache line: 64 字节 (x86, ARM)
- L1 cache: ~32KB, ~1ns 延迟
- L2 cache: ~256KB, ~4ns 延迟  
- L3 cache: ~若干 MB, ~12ns 延迟
- 主内存: ~100ns 延迟

**访问同一 cache line 的数据几乎无开销；跨越 cache line 则需额外加载。**

## False Sharing

当多个 CPU 核心访问同一 cache line 的不同变量时，即使逻辑上互不干扰，CPU 缓存一致性协议也会导致性能崩塌。

### 问题识别

```go
// 问题代码: a 和 b 在同一 cache line
type Counters struct {
    a int64  // offset 0
    b int64  // offset 8 — 与 a 同一 cache line
}

func main() {
    c := &Counters{}
    var wg sync.WaitGroup
    wg.Add(2)
    
    // goroutine 1 只写 c.a
    go func() {
        for i := 0; i < 1e8; i++ { c.a++ }
        wg.Done()
    }()
    // goroutine 2 只写 c.b
    go func() {
        for i := 0; i < 1e8; i++ { c.b++ }
        wg.Done()
    }()
    wg.Wait()
}
// 两个 goroutine 逻辑上互不干扰，但性能可能下降 50-100x
```

**诊断：** 多线程性能远低于单线程，且随核数增加不升反降。

### 解决方案

```go
// 方案 1: Padding — 填充到 cache line 大小
type PaddedInt64 struct {
    value int64
    _     [56]byte  // 64 - 8 = 56 bytes padding
}

type CountersFixed struct {
    a PaddedInt64  // cache line 0
    b PaddedInt64  // cache line 1
}

// 方案 2: array of structs → 每个独立 cache line
type CountersArray [2]struct {
    value int64
    _     [56]byte
}

// 方案 3: Go 1.19+ atomic types 也需要注意
type Counters struct {
    a atomic.Int64
    _ [56]byte
    b atomic.Int64
    _ [56]byte
}
```

### 不需要过度 Padding 的场景

- 只读数据：多核共享读取不触发 false sharing
- 频繁读同一字段：缓存一致性协议会自动共享
- 单 goroutine 访问：不存在竞争

## Struct 布局优化

### 对齐规则

Go 类型对齐要求：
- `bool`, `byte`, `int8` → 1 byte
- `int16`, `uint16` → 2 bytes
- `int32`, `uint32`, `float32`, `rune` → 4 bytes
- `int64`, `uint64`, `float64`, `complex64`, 指针 → 8 bytes
- `complex128` → 16 bytes
- struct → 内部最大字段的对齐

### 优化示例

```go
// Bad: 24 bytes, 33% 浪费
type BadLayout struct {
    a bool   // 1 byte + 7 bytes padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 bytes padding
}

// Good: 16 bytes, 紧凑排列
type GoodLayout struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 bytes padding
}

// Better: 所有字段对齐，无浪费 (如果有 2 个以上 bool/int8)
type Compact struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte
    d bool   // 1 byte
    e bool   // 1 byte + 4 bytes padding (凑够 8 对齐)
}
```

### 使用 fieldalignment 工具

```bash
# 安装
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# 检查
fieldalignment ./...

# 自动修复（带 -fix）
fieldalignment -fix ./...
```

输出示例：
```
./model.go:15: struct with 40 pointer bytes could be 32
./model.go:45: struct with 104 pointer bytes could be 96
```

### 布局原则

1. **从大到小排列字段** — int64/pointer → int32 → int16 → int8/bool
2. **相同大小的排在一起**
3. **连锁 padding** — 如果小字段后面跟大字段会浪费，把大字段放前面
4. **不要为省几个字节过度 layout** — 保持可读性，仅在热路径优化

## 数据局部性优化

### 结构体数组 (AoS) vs 数组结构体 (SoA)

```go
// AoS: 每个元素包含所有字段 — 只访问一个字段时浪费 cache
type AoS struct {
    X, Y, Z float64
}
data := make([]AoS, 10000)
// 遍历 X 时，每次加载 24 bytes，只用 8 bytes → 2/3 浪费

// SoA: 每个字段一个数组 — 遍历单字段时 cache 利用率 100%
type SoA struct {
    X, Y, Z []float64
}
data := SoA{
    X: make([]float64, 10000),
    Y: make([]float64, 10000),
    Z: make([]float64, 10000),
}
// 遍历 X 时，每次加载 cache line 全是 X → 无浪费
```

**选择规则：**
- 总是访问所有字段：AoS (如 Point 的 X,Y 总是一起用)
- 只访问部分字段：SoA (如物理引擎只更新 X 坐标)
- 混合访问模式：Hybrid (如分块处理 4x4 矩阵)

### Column-oriented Storage

```go
// 有时只需要某些列的数据
// 将热列拆分为独立 slice 可以显著提升 cache 命中
type UserStore struct {
    IDs   []int64
    Names []string
    Ages  []int
    // ... 冷数据放其他结构
}
```

### 链表 vs 数组

```go
// Bad for cache: 链表 — 随机内存访问
type Node struct {
    Value int
    Next  *Node  // cache miss
}

// Good for cache: 连续数组 — cache line 预取友好
type List struct {
    values []int
}
```

**规则：**
- 需要频繁遍历：数组/slice (cache 友好)
- 需要频繁插入/删除中间：链表（但考虑 arena allocator 改善 cache）

## CPU 分支预测优化

### 排序使分支可预测

```go
// Bad: 随机分支
func processRandom(data []Item) int {
    sum := 0
    for _, v := range data {
        if v.Value < threshold {  // 随机 true/false
            sum += v.Value
        }
    }
    return sum
}

// Good: 先排序
func processSorted(data []Item) int {
    sort.Slice(data, func(i, j int) bool {
        return data[i].Value < data[j].Value
    })
    sum := 0
    for _, v := range data {
        if v.Value < threshold {  // true 连续，false 连续
            sum += v.Value
        }
        if v.Value >= threshold {
            break  // 提前终止更好
        }
    }
    return sum
}
// 排序 + 分支预测改善 → 可达 3-5x 提升
```

### Lookup Table 替代分支

```go
// 多个分支
func getCategory(v int) string {
    switch {
    case v < 10:  return "small"
    case v < 50:  return "medium"
    case v < 100: return "large"
    default:      return "huge"
    }
}

// Lookup table (适合有限、连续的值域)
var categories = [...]string{
    // index 0-9 → "small", 10-49 → "medium", ...
}
func getCategoryFast(v int) string {
    if v < 0 || v >= len(categories) {
        return "huge"
    }
    return categories[v]
}
```

## Arena Allocator (Go 1.20+ 实验性)

```go
// 大量小对象 → 一并分配，一并释放，减少 GC 压力
// Go 1.22 中 arena 已移除，推荐使用 sync.Pool 或自定义 allocator

// 替代方案: 自定义 bump allocator
type BumpAllocator struct {
    buf    []byte
    offset int
}

func NewBumpAllocator(size int) *BumpAllocator {
    return &BumpAllocator{buf: make([]byte, size)}
}

func (a *BumpAllocator) Alloc(size int) []byte {
    if a.offset+size > len(a.buf) {
        return nil
    }
    result := a.buf[a.offset : a.offset+size]
    a.offset += size
    return result
}

func (a *BumpAllocator) Reset() {
    a.offset = 0
}
```