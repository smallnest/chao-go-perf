# 内存优化详解

## 逃逸分析 (Escape Analysis)

逃逸分析决定变量分配在栈上还是堆上。栈分配在函数返回时自动释放，零 GC 开销；堆分配需要 GC 追踪。

### 查看逃逸分析结果

```bash
go build -gcflags="-m" .           # 一级详情
go build -gcflags="-m -m" .        # 二级详情（更详细）
go build -gcflags="-m -m" . 2>&1 | grep "escapes to heap"
```

输出示例：
```
./main.go:10:2: moved to heap: x         # 变量逃逸
./main.go:15:6: can inline f             # 函数可内联
./main.go:20:6: foo does not escape      # 未逃逸
```

### 导致逃逸的 8 种常见模式

```go
// 1. 返回局部变量指针
func newFoo() *Foo {
    f := Foo{}    // 栈上创建
    return &f     // 逃逸！编译器无法保证调用者不存储指针
}

// 修复: 返回值而非指针（小对象）
func newFoo() Foo { return Foo{} }

// 2. interface 装箱
func print(v interface{}) { fmt.Println(v) }
x := 42
print(x)  // x 可能逃逸到堆（interface 动态分发）

// 3. 闭包捕获
func counter() func() int {
    count := 0
    return func() int { count++; return count }  // count 逃逸
}

// 4. 发送指针到 channel
ch := make(chan *Foo, 1)
ch <- &Foo{}  // 逃逸

// 5. 大对象分配
_ = make([]byte, 1<<20)  // 超过编译器阈值（通常 64KB），逃逸

// 6. 未知大小的 slice
func process(n int) []byte {
    return make([]byte, n)  // n 编译时未知 → 逃逸
}

// 7. defer 中引用变量
func f() {
    buf := make([]byte, 1024)
    defer func() { save(buf) }()  // buf 被闭包引用 → 逃逸
}

// 8. 反射
func getField(v interface{}, name string) reflect.Value {
    return reflect.ValueOf(v).FieldByName(name)  // v 逃逸
}
```

### 减少逃逸的策略

```go
// 策略 1: 调用者分配 (allocation up)
// Bad
func readAll(r io.Reader) ([]byte, error) {
    var buf bytes.Buffer
    io.Copy(&buf, r)
    return buf.Bytes(), nil
}
// Good: 调用者传入 buffer
func readInto(buf *bytes.Buffer, r io.Reader) error {
    _, err := io.Copy(buf, r)
    return err
}

// 策略 2: slice 代替 pointer
// Bad: 每个元素都是指针，每个都逃逸
type Store struct {
    items []*Item
}
// Good: 值切片，连续内存
type Store struct {
    items []Item
}

// 策略 3: 避免不必要的 interface
// Bad
func add(a, b interface{}) interface{} { ... }
// Good
func add(a, b int) int { ... }
```

## 分配减少技术

### 1. Slice 预分配

```go
// 每次 append 可能触发 growslice → 分配 + 拷贝
// Bad
var s []string
for _, v := range data {
    s = append(s, transform(v))
}

// Good: 预分配
s := make([]string, 0, len(data))
for _, v := range data {
    s = append(s, transform(v))
}

// 收益: allocs/op 从 ~10 降到 1
```

**预估容量的技巧：**
- 已知精确数量：`make([]T, 0, n)`
- 大概数量：`make([]T, 0, n*11/10)` (留 10% 余量)
- 完全未知：先用 `len(data)`，或用 `cap(s)` 观察增长

### 2. Map 预分配

```go
// Bad
m := make(map[string]int)
for _, v := range data { m[v.Key] = v.Value }

// Good: map 扩容非常昂贵（rehash + 迁移）
m := make(map[string]int, len(data))
for _, v := range data { m[v.Key] = v.Value }
```

### 3. strings.Builder 替代 + 拼接

```go
// Bad: O(n^2) 分配
var s string
for _, part := range parts {
    s += part   // 每次 + 都分配新字符串 + 拷贝
}

// Good: 零分配增长
var b strings.Builder
b.Grow(estimatedSize)  // 可选但推荐
for _, part := range parts {
    b.WriteString(part)
}
s := b.String()  // 只分配一次
```

### 4. []byte 代替 string 操作

```go
// string 是不可变的，每次修改都必需分配
// Bad
s := "hello"
for i := 0; i < 1000; i++ {
    s += "a"  // 1000 次分配
}

// Good: []byte 原地修改
buf := make([]byte, 0, 1000)
buf = append(buf, "hello"...)
for i := 0; i < 1000; i++ {
    buf = append(buf, 'a')
}
s := string(buf)  // 1 次分配
```

### 5. 避免 []byte ↔ string 热路径转换

```go
// 每次转换都分配内存
// string([]byte) 和 []byte(string) 都是 O(n) 复制

// 在热路径中保持 []byte
func process(r io.Reader) {
    buf := make([]byte, 4096)
    for {
        n, _ := r.Read(buf)
        // 直接在 buf[:n] 上操作，不转 string
        handle(buf[:n])
    }
}

// 只读场景的 unsafe 零拷贝（慎用，仅在确定 string 不会被修改时）
// s := unsafe.String(unsafe.SliceData(b), len(b))
```

## sync.Pool 最佳实践

### 何时使用

| 适合 | 不适合 |
|------|--------|
| 高频创建/销毁的临时对象 | 有状态的对象 |
| 对象创建成本高 | 需要精确控制生命周期的对象 |
| 对象可以复用 | 长生命周期对象 |
| `bytes.Buffer`、编解码器 | 数据库连接、文件句柄 |

### 标准用法

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()  // 关键: 清空复用，否则残留数据
    
    buf.Write(data)
    // ... 处理
    return buf.String()
}
```

### Pool 常见错误

```go
// 错误 1: 忘记 Reset
buf := bufPool.Get().(*bytes.Buffer)
buf.WriteString("hello")
bufPool.Put(buf)      // buf 内有 "hello"
buf2 := bufPool.Get() // buf2 内部还有 "hello"！

// 错误 2: Put 后继续使用
buf := bufPool.Get()
bufPool.Put(buf)
buf.WriteString("oops") // 数据竞争！

// 错误 3: 把 Pool 当缓存
// Pool 会在 GC 时清空对象，不适合需要持久化的场景

// 错误 4: 未处理 Get 返回 nil 的情况
// New 字段为 nil 时，Get 可能返回 nil
```

### 自定义 Pool 实现

```go
// 有容量限制的 Pool
type BoundedPool struct {
    pool chan *bytes.Buffer
}

func NewBoundedPool(size int) *BoundedPool {
    return &BoundedPool{pool: make(chan *bytes.Buffer, size)}
}

func (p *BoundedPool) Get() *bytes.Buffer {
    select {
    case buf := <-p.pool:
        buf.Reset()
        return buf
    default:
        return new(bytes.Buffer)
    }
}

func (p *BoundedPool) Put(buf *bytes.Buffer) {
    select {
    case p.pool <- buf:
    default:
        // 池满了，丢弃
    }
}
```

## GC 调优

### GODEBUG=gctrace

```bash
GODEBUG=gctrace=1 ./app

# 输出格式:
# gc 1 @0.012s 2%: 0.17+0.52+0.011 ms clock, 0.34+0.32/0.54/0.77+0.023 ms cpu, 4->4->2 MB, 5 MB goal, 4 P
```

解读：
- `gc 1`: 第 1 次 GC
- `@0.012s`: 程序启动后 0.012 秒
- `2%`: CPU 时间用于 GC 的比例
- `0.17+0.52+0.011 ms`: STW 标记开始 + 并发标记 + STW 标记结束
- `4->4->2 MB`: GC 开始堆大小 → GC 结束堆大小 → 存活堆大小
- `5 MB goal`: 下一次 GC 触发阈值

### GOGC 调优

```
GOGC=100（默认）: 堆增长 100% 后触发 GC
GOGC=200:       堆增长 200% 后触发 GC（GC 频率减半，内存占用增加）
GOGC=off:       关闭自动 GC（慎用）
GOGC=50:        GC 更频繁，内存更紧凑
```

### GOMEMLIMIT (Go 1.19+)

```
GOMEMLIMIT=4GiB: 堆内存软上限，接近时增加 GC 频率
GOMEMLIMIT=8GiB,GOGC=50: 组合使用
```

### 手动控制 GC

```go
// 强制 GC（通常不需要）
runtime.GC()

// 暂时禁用 GC
gcpercent := debug.SetGCPercent(-1)
defer debug.SetGCPercent(gcpercent)

// 释放空闲内存给 OS
debug.FreeOSMemory()
```

### 减少 GC 压力的完整清单

1. 减少堆分配（逃逸分析 + 预分配）
2. 用 sync.Pool 复用临时对象
3. 用 `[]byte` 代替 `string` 做中间处理
4. 避免不必要的 boxing（interface{}）
5. 小对象返回值而非指针
6. 用 `map[int]T` 代替 `map[string]T`（int key 更快）
7. 用值类型的 slice 代替指针 slice
8. 调大 GOGC 减少 GC 频率（trade-off 内存）
9. 设置 GOMEMLIMIT 避免 OOM