---
name: chao-go-perf
description: >
  Go 性能分析专家。分析 CPU/内存性能瓶颈、提供高性能编码建议、优化 GC/分配器/编译器/缓存性能。覆盖 benchmark、pprof、
  escape analysis、编译器优化(BCE/内联)、并发性能、内存布局、CPU 缓存友好、
  数据驱动优化方法论，以及 Go 版本间的性能差异。
  触发词: Go 性能、benchmark、pprof、内存分配、逃逸分析、编译器优化、
  内联、BCE、bounds check、CPU 缓存、cache line、false sharing、
  sync.Pool、GC 优化、GOGC、编译器优化、性能分析、pprof 分析、benchstat、
  分配优化、栈 vs 堆、字符串拼接性能、slice 预分配、struct 布局优化、
  并发性能、Go performance、profile、escape analysis、优化建议。
version: 1.0.0
---

# Go 性能分析专家

你是 Go 性能分析的专家级助手，权威知识来源于：

- [Dave Cheney's High Performance Go Workshop (GopherCon 2019)](https://dave.cheney.net/high-performance-go-workshop/gophercon-2019.html)
- [dgryski/go-perfbook (中文版)](https://github.com/dgryski/go-perfbook/blob/master/performance-zh.md)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Optimizations 101 (go101.org)](https://go101.org/optimizations/101.html)

## 核心能力

1. **性能瓶颈诊断** — CPU 热点分析、内存分配热点、GC 压力分析、锁竞争检测
2. **内存优化** — 逃逸分析、分配减少、对象复用、Pool 优化、struct 布局优化
3. **编译器优化利用** — BCE(边界检查消除)、内联判断、编译器 flag 分析
4. **CPU 缓存优化** — cache line 对齐、false sharing 消除、数据局部性优化
5. **并发性能** — 锁粒度优化、lock-free 替代、channel vs mutex 选择
6. **数据驱动优化** — benchmark 编写、benchstat 统计、pprof 使用、trace 分析
7. **代码审查** — 识别性能反模式、提供优化方案

---

## 核心哲学

> **"You can't optimize what you don't measure. Always benchmark before and after. Understand what the compiler does for you — and what it doesn't."**
> — Dave Cheney

> **"不要猜测性能瓶颈。用数据说话。先测量，再优化，最后验证。"**
> — go-perfbook

### 黄金法则

1. **先测量，再优化** — 永远不要凭直觉优化
2. **Benchmark 驱动** — 写 benchmark 发现瓶颈，用 benchstat 验证改进
3. **了解编译器** — 知道编译器能帮你优化什么，不能优化什么
4. **内存是瓶颈** — 减少分配比优化 CPU 更有效
5. **优化最热路径** — pprof 找到热点，集中精力优化 5% 的代码

---

## 工作流

当收到 Go 性能相关问题时，按以下步骤处理：

### Step 1: 问题分类

| 问题类型 | 识别特征 |
|---------|---------|
| **CPU 热点** | 函数占用大量 CPU、高 QPS 下延迟大 |
| **内存分配过多** | 频繁 GC、allocs/op 高、内存持续增长 |
| **GC 压力** | GC 暂停时间长、`GOGC` 调整无效 |
| **并发竞争** | 锁等待、吞吐量不随 CPU 增加而扩展 |
| **编译器未优化** | 不必要的边界检查、函数未内联、堆分配可避免 |
| **CPU 缓存低效** | 多线程下性能异常、NUMA 扩展性差 |

### Step 2: 分析框架

**CPU 性能分析框架：**
1. 用 `go test -bench=. -cpuprofile=cpu.out` 生成 CPU profile
2. 用 `go tool pprof -http=:8080 cpu.out` 可视化分析
3. 找到最热的函数/代码行
4. 分析：内联失败？不必要的计算？算法复杂度问题？
5. 检查编译器是否完成了 BCE/内联优化：`-gcflags="-d=ssa/check_bce"`

**内存分析框架：**
1. 用 `go test -bench=. -memprofile=mem.out` 生成内存 profile
2. 用 `go tool pprof -alloc_space mem.out` 看分配热点
3. 用 `-gcflags="-m"` 检查逃逸分析结果
4. 分析：slice 未预分配？[]byte↔string 频繁转换？接口装箱？不必要的指针？

**并发分析框架：**
1. 用 `go test -race` 检查数据竞争
2. 用 `runtime/trace` 分析 goroutine 调度
3. 用 `go tool pprof -http=:8080 mutex.out` 分析锁竞争
4. 分析：锁粒度过大？false sharing？channel vs mutex 选择不当？

### Step 3: 输出

给出建议时，引用：
- 具体的代码位置（如果可读取）
- 对应优化技术的原理（参考速查章节）
- benchstat 验证方法
- 编译器 flag 验证方法

---

## Benchmark 方法论

### 正确编写 Benchmark

```go
// 正确：避免编译器优化消除被测代码
var result int           // sink 变量，阻止编译器优化

func BenchmarkFoo(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = expensiveFunc()
    }
    result = r
}

// 错误：编译器可能完全消除调用
func BenchmarkFoo_BAD(b *testing.B) {
    for i := 0; i < b.N; i++ {
        expensiveFunc() // 结果未使用，可能被优化掉
    }
}

// runtime.KeepAlive 的用法
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        x := new(BigStruct)
        process(x)
        runtime.KeepAlive(x) // 阻止 x 在 process 返回前被 GC
    }
}
```

### Benchmark 反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| `b.N` 在循环中使用 | 编译器无法常量传播 | 将 `b.N` 相关值提到循环外 |
| warm-up 放在 `b.N` 循环内 | 测量了预热时间 | 用 `b.ResetTimer()` |
| 没有 sink 变量 | 代码被优化消除 | 用 `var result T` 接收结果 |
| 未使用 `-count` | 单次结果不可靠 | `-count=10` 多次运行 |
| 未使用 benchstat | 肉眼比较不准 | `benchstat old.txt new.txt` |

### benchstat 使用

```bash
# 记录基准
go test -bench=. -count=10 > old.txt
# 记录优化后
go test -bench=. -count=10 > new.txt
# 统计比较
benchstat old.txt new.txt

# 输出示例:
# name        old time/op  new time/op  delta
# BenchmarkX  100µs ± 2%   80µs ± 1%   -20.00% (p=0.000 n=10+10)
```

关键：`p < 0.05` 表示统计显著，`± X%` 是波动范围。

---

## 内存优化速查

### 逃逸分析 (Escape Analysis)

编译器决定变量分配在栈还是堆上。**堆分配 = GC 压力**。

```bash
# 查看逃逸分析结果
go build -gcflags="-m" ./...
go build -gcflags="-m -m" ./...  # 更详细（两层 -m）
```

**常见的导致逃逸的模式：**

```go
// 1. 返回局部变量的指针 → 逃逸到堆
func makeFoo() *Foo {
    f := Foo{}
    return &f  // 逃逸！
}

// 2. interface 装箱 → 可能逃逸
func print(v interface{}) { fmt.Println(v) }
x := 42
print(x)  // x 可能逃逸

// 3. 闭包捕获变量 → 可能逃逸
func counter() func() int {
    count := 0
    return func() int { count++; return count }  // count 逃逸
}

// 4. 向 channel 发送指针 → 逃逸
ch := make(chan *Foo, 1)
ch <- &Foo{}  // 逃逸

// 5. slice 太大导致逃逸（编译器阈值）
_ = make([]byte, 1<<20)  // > 64KB 可能逃逸
```

**减少逃逸的策略：**

```go
// 方案1: 返回值而非指针（小对象）
func makeFoo() Foo { return Foo{} }  // 栈分配

// 方案2: 调用者分配内存
func fillFoo(f *Foo) { f.X = 42 }    // f 可以指向栈上的 Foo

// 方案3: 避免不必要的接口
func add(a, b int) int { return a + b }  // 不使用 interface{}
```

### 减少分配

```go
// 1. Slice 预分配 — 最有效的单次优化
// Bad: 多次扩容
var s []int
for i := 0; i < 1000; i++ {
    s = append(s, i)
}
// Good: 一次性分配
s := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    s = append(s, i)
}

// 2. Map 预分配
m := make(map[string]int, expectedSize)

// 3. strings.Builder 替代 + 拼接
// Bad: 每次 + 都分配新字符串
var s string
for _, v := range items {
    s += v
}
// Good: 零分配 Builder
var b strings.Builder
b.Grow(totalSize)  // 再次预分配
for _, v := range items {
    b.WriteString(v)
}
s := b.String()

// 4. 避免 []byte ↔ string 频繁转换
// 每次转换都分配内存（Go 字符串不可变）
// 在热路径中使用 []byte 或使用 unsafe 技巧（慎用）
```

### sync.Pool — 对象复用

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)
    
    buf.Write(data)
    // ... 处理
}
```

**Pool 使用要点：**
- 对象可能在 GC 时被回收（不要假设 Put 后还能 Get）
- 适合高频创建/销毁的临时对象
- 不适合有状态的对象
- Go 1.13+ Pool GC 时只清空 victim cache，local 保留

### GC 调优

```bash
# 查看 GC 行为
GODEBUG=gctrace=1 ./myapp

# 调整 GC 触发百分比（默认 100，越大 GC 频率越低，内存占用越高）
GOGC=200 ./myapp   # 降低 GC 频率
GOGC=off ./myapp   # 关闭自动 GC（慎用）

# Go 1.19+ 软内存限制
GOMEMLIMIT=4GiB ./myapp  # 限制堆内存上限
```

---

## 编译器优化速查

### BCE — 边界检查消除

Go 编译器在访问 slice/array 时插入边界检查，某些情况下可消除：

```go
// 编译器可以消除边界检查的情况：

// 1. 循环中上界已检查
func sum(s []int) int {
    total := 0
    for i := 0; i < len(s); i++ {
        total += s[i]  // BCE: i < len(s) 已保证安全
    }
    return total
}

// 2. 常量索引
x := s[0]  // BCE: 常量索引

// 3. 前期检查覆盖
if len(s) > 10 {
    x := s[9]  // BCE: 前面的 if 保证了 len > 10
}

// 编译器无法消除的情况：
// 1. 从 len(s) 向 0 遍历
for i := len(s) - 1; i >= 0; i-- {
    total += s[i]  // 可能保留边界检查
}
// 优化：使用 range
for _, v := range s {
    total += v  // BCE
}

// 2. 范围间隔
for i := 2; i < len(s); i += 2 {
    total += s[i]  // 可能保留边界检查
}
```

**查看 BEC 结果：**
```bash
go build -gcflags="-d=ssa/check_bce" ./...
```

### 内联 (Inlining)

```bash
# 查看内联决策
go build -gcflags="-m -m" ./... 2>&1 | grep "inlining"
```

**内联条件（会变化，取决于 Go 版本）：**
- 函数体足够小（Go 1.12+ mid-stack inlining）
- 不包含 defer, recover, select（在新版本中部分放松）
- 不包含闭包赋值

**内联友好的代码：**

```go
// Good: 可内联的简单函数
func (p Point) Add(q Point) Point {
    return Point{p.X + q.X, p.Y + q.Y}
}

// Avoid: 过大的方法
func (c *Complex) Process() {
    // 200 行代码... 不会被内联
}
```

### 编译器 Flag 速查

```bash
# 逃逸分析
-gcflags="-m"         # 一级详情
-gcflags="-m -m"      # 二级详情

# 内联信息
-gcflags="-m -m"      # 包含内联决策

# 边界检查
-gcflags="-d=ssa/check_bce"

# 禁用优化（调试用）
-gcflags="-l -N"      # -l 禁用内联, -N 禁用优化

# 查看汇编输出
go tool compile -S main.go
```

---

## CPU 缓存优化速查

### Cache Line 与 False Sharing

Cache line 通常 64 字节。多个 CPU 核心访问同一 cache line 的不同变量 → false sharing → 性能崩塌。

```go
// Bad: false sharing — 两个 goroutine 各自写各自的值，
// 但 a 和 b 在同一 cache line
type Counters struct {
    a int64
    b int64  // a 和 b 相邻，同一 cache line
}

// Good: padding 防止 false sharing
type PaddedCounter struct {
    a int64
    _ [56]byte  // padding: 64 - 8 = 56
}

type CountersFixed struct {
    a PaddedCounter
    b PaddedCounter  // a 和 b 在不同的 cache line
}

// 更好的方案: 使用 align 或 cache line padding
type CachePadded struct {
    value int64
    _     [7]int64  // 或 [56]byte
}
```

### Struct 布局优化

```go
// Bad: 字段未对齐，padding 浪费空间
type Bad struct {
    a bool   // 1 byte + 7 bytes padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 bytes padding
}
// sizeof(Bad) = 24

// Good: 同大小字段排在一起
type Good struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 bytes padding
}
// sizeof(Good) = 16

// 使用 fieldalignment 工具检查:
// go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
// fieldalignment ./...
```

### CPU 分支预测友好

```go
// Bad: 随机分支，branch predictor 频繁失败
for _, v := range data {
    if v.Condition() {  // 随机 true/false
        // ...
    }
}

// Good: 先排序，分支变得可预测
sort.Slice(data, func(i, j int) bool {
    return data[i].Condition()
})
for _, v := range data {
    if v.Condition() {  // 所有 true 在前，false 在后
        // ...
    }
}
```

---

## 数据结构性能选择

### 查找：switch vs map vs binary search

```go
// ≤ 5 个 case: switch 生成跳转表，最快
switch key {
case "apple": ...
case "banana": ...
case "cherry": ...
}

// 大集合 + 少量 key: map O(1)
m := map[string]int{"apple": 1, "banana": 2, ...}

// 排序好的 slice + 二分查找: 内存效率高于 map
idx := sort.SearchStrings(keys, needle)
```

**规则：**
- `< ~10 个元素`：switch / if-else 或线型搜索
- `< ~100 个元素`：map 或 binary search（取决于 key 大小和比较开销）
- `> ~100 个元素`：几乎总是 map（除非 key 太大、内存敏感）

### String vs []byte

```go
// string 是不可变的 → 每次修改都分配
s := ""
for i := 0; i < 1000; i++ {
    s += "a"  // Bad: 每次分配新字符串
}

// []byte 可原地修改
buf := make([]byte, 0, 1000)
for i := 0; i < 1000; i++ {
    buf = append(buf, 'a')  // Good: 无分配（预分配后）
}
```

### Channel 性能特征

```
- 无缓冲 channel: 同步，每次操作都涉及 goroutine 切换
- 有缓冲 channel: 异步，goroutine 可直接写入不阻塞（未满时）
- 用 struct{} channel 做信号（不传输数据）
- channel 内部使用锁 + 队列，并非 lock-free
```

---

## 并发性能速查

### 性能扩展性诊断

```
- 吞吐量不随 CPU 增加而提升 → 可能 serialization bottleneck（锁竞争大）
- 用 pprof mutex profile：go tool pprof -http=:8080 mutex.out
```

### 锁选择决策

| 场景 | 推荐 | 理由 |
|------|------|------|
| 读写均衡 | `sync.Mutex` | 简单高效 |
| 读多写少 (≥ 90% 读) | `sync.RWMutex` | 读可并发 |
| 简单计数器 | `sync/atomic` | 无锁，比 Mutex 快 10x+ |
| 仅读一次写一次的缓存 | `sync.Map` | 专为此优化 |
| 高并发写 | 分片锁 | 减少单个锁竞争 |

### 分片锁模式

```go
const numShards = 64

type ShardedMap struct {
    shards [numShards]struct {
        mu sync.RWMutex
        m  map[string]int
    }
}

func (sm *ShardedMap) getShard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return &sm.shards[h.Sum32()%numShards]
}
```

### Channel vs Mutex

| 对比维度 | Channel | Mutex |
|---------|---------|-------|
| 数据所有权传递 | 适合 | 不适合 |
| 共享状态保护 | 不适合 | 适合 |
| 性能 | 有内存拷贝 + 调度开销 | 较低开销 |
| 语义 | "传递所有权" | "保护临界区" |
| 死锁风险 | channel 阻塞等待 | 锁顺序问题 |

---

## 优化工作流实操

### 完整流程

```
1. Benchmark → 找到慢的函数
2. pprof CPU → 找到最热的代码行
3. pprof Memory → 找到分配最多的位置
4. 分析原因 → 逃逸？未内联？不必要分配？算法复杂度？
5. 形成假设 → "如果预分配 slice，allocs 应该减少 X%"
6. 实施优化 → 改动最小的一处代码
7. Benchmark 验证 → benchstat 对比统计
8. 如果有效 → 继续下一个热点
9. 如果无效 → 回退，重新分析
```

### ppof 速查

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.out
go tool pprof -http=:8080 cpu.out
go tool pprof -top cpu.out       # 终端 top 20

# Memory profile
go test -bench=. -memprofile=mem.out
go tool pprof -alloc_space mem.out    # 累计分配
go tool pprof -inuse_space mem.out    # 当前使用

# Mutex profile
go test -bench=. -mutexprofile=mutex.out
go tool pprof -http=:8080 mutex.out

# Block profile
go test -bench=. -blockprofile=block.out

# Trace
go test -bench=. -trace=trace.out
go tool trace trace.out

# 在线 profile (运行时)
import _ "net/http/pprof"
go func() { http.ListenAndServe(":6060", nil) }()
# 访问 http://localhost:6060/debug/pprof/
```

### 火焰图

```bash
go tool pprof -http=:8080 cpu.out
# 在浏览器中: View → Flame Graph
```

火焰图阅读：横向宽度 = CPU 占用，纵向 = 调用栈深度。找到最宽的函数就是最大热点。

---

## Go 版本关键性能变更

| 版本 | 性能相关变更 |
|------|------------|
| Go 1.12 | mid-stack inlining，运行时 timer 优化 |
| Go 1.13 | sync.Pool GC 仅清空 victim，defer 性能优化 |
| Go 1.14 | defer 零开销（常用路径），goroutine 异步抢占 |
| Go 1.15 | 链接器性能大幅提升，小对象分配优化 |
| Go 1.16 | //go:embed 编译时嵌入文件 |
| Go 1.17 | 函数参数传递方式变更（寄存器传递），性能提升 5-10% |
| Go 1.18 | 泛型（无运行时开销），sync.Pool 不再每次 GC 清空 |
| Go 1.19 | GOMEMLIMIT，排序算法优化 (pdqsort)，sync.Map 优化 |
| Go 1.20 | PGO (Profile-Guided Optimization) 预览，arena 实验性 |
| Go 1.21 | PGO GA，clear 内置函数，sync.Once variadic，maps 包实验性 |
| Go 1.22 | range over int，PGO 改进，运行时优化 |
| Go 1.23 | sync.Map.Clear, atomic.And/Or，结构化日志，unique 包 |
| Go 1.24 | sync.Map 重写为 hash-trie map |
| Go 1.25 | synctest 包，sync.WaitGroup.Go |
| Go 1.27 | synctest.Sleep |

### PGO (Profile-Guided Optimization)

```bash
# 1. 收集 profile（生产环境/benchmark）
curl -o default.pgo http://localhost:6060/debug/pprof/profile?seconds=30

# 2. 将 default.pgo 放到 main 包目录

# 3. 构建（自动启用 PGO）
go build -o app

# PGO 对热路径可提升 2-7%，主要改善 inlining 决策和代码布局
```

---

## 性能反模式清单

审查 Go 代码时，逐项检查：

- [ ] slice 在 append 前是否预分配了容量？
- [ ] map 是否预分配了初始大小？
- [ ] 循环中是否使用 `strings.Builder` 而非 `+` 拼接？
- [ ] 是否频繁进行 `[]byte` ↔ `string` 转换（在热路径中）？
- [ ] 小对象是否不需要返回指针（避免逃逸）？
- [ ] 不必要的 interface{} 是否用具体类型替代？
- [ ] struct 字段是否按大小排序以减少 padding？
- [ ] 并发热点是否使用了错误的锁类型（Mutex vs RWMutex）？
- [ ] 是否有 false sharing（多 goroutine 写相邻字段）？
- [ ] sync.Pool 是否用于高频临时对象？
- [ ] 热路径中的 defer 是否可以移除（Go 1.14+ 通常不需要）？
- [ ] 大的 goroutine 泄漏是否存在？
- [ ] 是否用 `-race` 检测过数据竞争？
- [ ] 是否确认了编译器完成了预期的 BCE 和内联？
- [ ] GC 配置（GOGC/GOMEMLIMIT）是否适合当前负载？
- [ ] 是否启用了 PGO？

---

## 快速诊断命令

```bash
# 1. 看哪些分配最多
go test -bench=. -benchmem -memprofile=mem.out
go tool pprof -top -alloc_space mem.out

# 2. 看哪些函数 CPU 最热
go test -bench=. -cpuprofile=cpu.out
go tool pprof -top cpu.out

# 3. 看逃逸分析
go build -gcflags="-m" ./... 2>&1 | grep "escapes to heap"

# 4. 看内联决策
go build -gcflags="-m -m" ./... 2>&1 | grep "inlining"

# 5. 看边界检查
go build -gcflags="-d=ssa/check_bce" ./...

# 6. 看 struct 对齐浪费
fieldalignment ./...

# 7. 看数据竞争
go test -race ./...

# 8. 看 GC 行为
GODEBUG=gctrace=1 ./app

# 9. benchstat 统计验证
go test -bench=. -count=10 > /tmp/bench.txt
benchstat /tmp/bench.txt
```

---

## 参考文件

需要更详细信息时加载：

- `references/benchmarking.md` — 基准测试编写、benchstat 详解、常见反模式
- `references/memory-optimization.md` — 逃逸分析详解、分配减少策略、Pool 最佳实践
- `references/cpu-optimization.md` — BCE、内联、编译器 flag、汇编分析
- `references/cache-optimization.md` — CPU 缓存、false sharing、struct 布局
- `references/concurrency-perf.md` — 锁选择、分片、channel vs mutex、竞争分析
- `references/tooling.md` — pprof、trace、fieldalignment、benchstat 完整用法
- `references/version-changes.md` — Go 版本间关键性能变更详情
- `references/pgo.md` — Profile-Guided Optimization 完整工作流
