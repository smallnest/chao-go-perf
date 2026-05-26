# 并发性能详解

## 并发扩展性分析

### 扩展性测试

```go
func BenchmarkConcurrent(b *testing.B) {
    for _, procs := range []int{1, 2, 4, 8, 16} {
        b.Run(fmt.Sprintf("GOMAXPROCS=%d", procs), func(b *testing.B) {
            b.SetParallelism(procs)
            b.RunParallel(func(pb *testing.PB) {
                for pb.Next() {
                    doWork()
                }
            })
        })
    }
}
```

理想扩展：GOMAXPROCS 翻倍 → 吞吐量翻倍。实际中需要注意识别瓶颈。

### 常见扩展性瓶颈

| 瓶颈 | 症状 | 诊断工具 |
|------|------|---------|
| 锁竞争 | 吞吐量不随 CPU 增加 | pprof mutex profile |
| False sharing | 多核反而更慢 | 检查相邻 atomic 字段 |
| 内存分配 | GC STW 随 goroutine 增加 | pprof alloc_space |
| Channel 阻塞 | goroutine 等待时间 | pprof block profile |
| Goroutine 泄漏 | 内存持续增长 | runtime.NumGoroutine() |

## 锁选择

### 决策流程图

```
是否只需要保护简单整数/布尔/指针?
  └── Yes → atomic 操作
  └── No → 读写比例?
            ├── 几乎全读 (> 90%) → sync.RWMutex
            ├── 读写均衡 → sync.Mutex
            └── 偶尔写入一次 → sync.Once / sync.Map
```

### 性能对比

```go
// 基准测试 (近似值，实际需测量):

// atomic:   1-2 ns/op  — 单个原子指令
// Mutex:    15-30 ns/op — 无竞争时
// RWMutex:  20-40 ns/op — 无竞争时读
// channel:  50-200 ns/op — 有调度开销
```

### 锁粒度优化

```go
// Bad: 粗粒度锁 — 保护整个结构
type BigCache struct {
    mu    sync.Mutex
    items map[string]Item
}

// Good: 分片锁 — 减少竞争
const numShards = 64

type ShardedCache struct {
    shards [numShards]struct {
        mu    sync.RWMutex
        items map[string]Item
    }
}

func (c *ShardedCache) getShard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return &c.shards[h.Sum32()%numShards]
}

// 注意: 分片数量通常取 2^n (位运算取模) 或质数 (分布更均匀)
```

### Lock-free 替代

```go
// 场景 1: 简单计数器
// Bad
var count int
var mu sync.Mutex
func inc() { mu.Lock(); count++; mu.Unlock() }
// Good
var count atomic.Int64
func inc() { count.Add(1) }

// 场景 2: 状态标志
// Bad
var ready bool
var mu sync.Mutex
func setReady() { mu.Lock(); ready = true; mu.Unlock() }
func isReady() bool { mu.Lock(); defer mu.Unlock(); return ready }
// Good
var ready atomic.Bool
func setReady() { ready.Store(true) }
func isReady() bool { return ready.Load() }

// 场景 3: CAS 无锁更新
// 用 atomic.CompareAndSwap 实现无锁栈、无锁队列等
func (s *LockFreeStack) Push(v interface{}) {
    for {
        old := s.head.Load()
        node := &node{value: v, next: old}
        if s.head.CompareAndSwap(old, node) {
            return
        }
        // CAS 失败→重试
    }
}
```

## sync.Map vs map + RWMutex

### sync.Map 适用场景

```
最优场景:
  (1) key 只写一次但读很多次 (如配置缓存)
  (2) 多个 goroutine 操作不相交的 key 集合

不适用场景:
  (1) 大量不同的 key 写入
  (2) 需要 Len() 方法
  (3) 需要 range 遍历 (每次都 promo dirty)
```

### 性能对比

```go
// 当 key 集合稳定(只读)时 sync.Map 远优于 map+Mutex
// 当 key 频繁变化时 sync.Map 内部 promo 机制反而变慢

// 经验: 不确定时先用 map+RWMutex，用 benchmark 验证
// 不要因为 "线程安全" 就默认用 sync.Map
```

## Channel vs Mutex

### 语义选择

```
Channel: 传递数据所有权
Mutex:   保护共享状态
```

```go
// Channel 适合: 数据所有权转移
func producer(out chan<- Item) {
    for _, item := range items {
        out <- item  // 所有权转移给消费者
    }
    close(out)
}

// Mutex 适合: 共享缓存
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func (c *Cache) Get(key string) (Item, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    return item, ok
}
```

### 性能考量

```go
// Channel 内部开销:
// 1. 内存拷贝 (发送值时)
// 2. goroutine 调度 (阻塞时)
// 3. 锁 + 等待队列

// 高频操作 (> 1M ops/s) 中用 channel 做状态机可能瓶颈
// 此时考虑分片锁或 atomic
```

## 减少锁持有时间

```go
// Bad: 锁持有时间长
func (c *Cache) Process(key string) error {
    c.mu.Lock()
    defer c.mu.Unlock()
    data := c.fetchFromRemote(key)  // I/O 在锁内！
    c.items[key] = data
    return nil
}

// Good: 只在必要时持有锁
func (c *Cache) Process(key string) error {
    data := c.fetchFromRemote(key)  // I/O 在锁外
    
    c.mu.Lock()
    c.items[key] = data
    c.mu.Unlock()
    return nil
}

// Better: 如果大部分缓存命中，用读锁 + 写锁升级
func (c *Cache) Process(key string) error {
    c.mu.RLock()
    data, ok := c.items[key]
    c.mu.RUnlock()
    if ok {
        return nil  // 缓存命中，无写锁
    }
    
    data = c.fetchFromRemote(key)
    c.mu.Lock()
    c.items[key] = data
    c.mu.Unlock()
    return nil
}
```

## Goroutine 泄漏

```go
// 常见泄漏模式

// 1. channel 永远没有接收者
func leak1() {
    ch := make(chan int)
    go func() {
        ch <- 42  // 永远阻塞 → goroutine 泄漏
    }()
    // ch 没有被接收，goroutine 永远等待
}

// 修复: 用 buffered channel 或 select + context
func noleak1(ctx context.Context) {
    ch := make(chan int, 1)
    go func() {
        select {
        case ch <- 42:
        case <-ctx.Done():
            return
        }
    }()
}

// 2. select 没有退出条件
func leak2() {
    ch1, ch2 := make(chan int), make(chan int)
    go func() {
        select {
        case <-ch1:
        case <-ch2:
        }  // 两个 channel 都没有数据 → 永远阻塞
    }()
}

// 修复: 加 default 或 context
func noleak2(ctx context.Context) {
    go func() {
        select {
        case <-ch1:
        case <-ch2:
        case <-ctx.Done():
            return
        }
    }()
}

// 3. WaitGroup 计数不匹配
// Add 了 N 次但 Done 了 N-1 次 → Wait 永远不返回
```

### 检测 Goroutine 泄漏

```go
// 测试时检测
func TestNoLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    // ... 测试代码 ...
    after := runtime.NumGoroutine()
    if after > before {
        t.Errorf("goroutine leak: %d before, %d after", before, after)
    }
}

// Go 1.26+ 运行时自动检测泄漏:
// panic: goroutine leak detected: goroutine x is waiting forever

// 更可靠的检测:
// go.uber.org/goleak
```

## Concurrency 模式性能

### Worker Pool

```go
// 避免为每个请求创建 goroutine
// 用 worker pool 控制并发

func workerPool(jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    go func() { wg.Wait(); close(results) }()
}
```

### 扇出/扇入 (Fan-out/Fan-in)

```go
// 扇出: 一个输入 channel → 多个 worker
func fanOut(in <-chan Job, workers int) []<-chan Result {
    outs := make([]<-chan Result, workers)
    for i := 0; i < workers; i++ {
        ch := make(chan Result)
        go func() {
            defer close(ch)
            for job := range in {
                ch <- process(job)
            }
        }()
        outs[i] = ch
    }
    return outs
}

// 扇入: 多个 worker → 一个输出 channel
func fanIn(channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

### Pipeline

```go
// 流水线: 每个阶段处理完后传给下一个阶段
func pipeline(in <-chan int) <-chan int {
    // 阶段 1
    ch1 := make(chan int)
    go func() {
        defer close(ch1)
        for v := range in {
            ch1 <- v * 2
        }
    }()
    
    // 阶段 2
    ch2 := make(chan int)
    go func() {
        defer close(ch2)
        for v := range ch1 {
            ch2 <- v + 1
        }
    }()
    
    return ch2
}
```