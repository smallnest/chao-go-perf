# 性能工具完整指南

## pprof

### 生成 Profile

```bash
# Benchmark 生成
go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out

# 在线 HTTP endpoint
import _ "net/http/pprof"
go func() { http.ListenAndServe(":6060", nil) }()

# curl 获取
curl -o cpu.out http://localhost:6060/debug/pprof/profile?seconds=30
curl -o mem.out http://localhost:6060/debug/pprof/heap
curl -o allocs.out http://localhost:6060/debug/pprof/allocs
curl -o mutex.out http://localhost:6060/debug/pprof/mutex
curl -o block.out http://localhost:6060/debug/pprof/block
curl -o goroutine.out http://localhost:6060/debug/pprof/goroutine

# 程序中手动生成
f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

### 使用 pprof 工具

```bash
# 交互式
go tool pprof cpu.out
  (pprof) top           # 前 20 热点
  (pprof) top -cum      # 按累计时间排序
  (pprof) list funcName # 函数源码级分析
  (pprof) web           # 调用图 (需 graphviz)
  (pprof) peek funcName # 查看调用者和被调用者
  (pprof) weblist funcName  # 源码级网页

# Web UI (推荐)
go tool pprof -http=:8080 cpu.out
# 浏览器打开 http://localhost:8080
# View → Flame Graph (火焰图)
# View → Top
# View → Source

# 比较两个 profile
go tool pprof -http=:8080 -diff_base old_cpu.out new_cpu.out

# 直接从运行中的程序
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30
```

### pprof 类型说明

| Profile | 含义 | 用途 |
|---------|------|------|
| `profile` | CPU 时间采样 | 找到 CPU 热点函数 |
| `heap` | 当前堆内存 | 找到内存占用最多的位置 |
| `allocs` | 累计内存分配 | 找到分配次数最多的位置 |
| `goroutine` | goroutine 栈 | 找到 goroutine 泄漏 |
| `mutex` | 锁竞争 | 找到竞争最激烈的锁 |
| `block` | 阻塞时间 | 找到阻塞时间最长的操作 |
| `threadcreate` | 线程创建 | 线程创建分析 |

### 火焰图阅读技巧

- **宽度** = CPU 时间占比，越宽越热
- **高度** = 调用栈深度（不重要）
- 找到最宽的**叶子节点**（栈顶），那是真正耗费 CPU 的函数
- 如果一整列是 runtime.mallocgc → 内存在瓶颈

## Trace

### 生成和查看

```bash
# Benchmark 生成
go test -bench=. -trace=trace.out

# HTTP 获取
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5

# 程序中
f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()
```

```bash
# 查看
go tool trace trace.out
# 浏览器打开 http://127.0.0.1:XXXXX
```

### Trace 能看到的

- Goroutine 调度：何时创建、阻塞、运行
- GC 事件：每次 GC 的开始/结束，STW 时间
- 网络/系统调用阻塞
- Heap 使用变化
- Processor (P) 利用率

### Trace 分析要点

1. **Goroutine 分析**: 是否有大量 goroutine 处于 runnable 状态（调度瓶颈）
2. **GC 分析**: STW 时间是否过长，GC 频率是否过高
3. **P 利用率**: 是否有空闲的 P（负载不均衡）
4. **Syscall 阻塞**: 是否有大量系统调用阻塞时间
5. **Network 阻塞**: I/O 是否成为瓶颈

## benchstat

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

### 常用命令

```bash
# 基本比较
go test -bench=. -count=10 > old.txt
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt

# 多组比较
benchstat old.txt new1.txt new2.txt

# CSV 输出（便于画图）
benchstat -format csv old.txt new.txt > comparison.csv

# HTML 输出
benchstat -html old.txt new.txt > comparison.html

# 按测试名过滤
benchstat -filter=".filter=Read" old.txt new.txt

# 只显示有显著差异的
benchstat -alpha 0.05 old.txt new.txt
```

## fieldalignment

```bash
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# 检查一个包
fieldalignment ./...

# 检查整个项目
fieldalignment ./...

# 自动修复
fieldalignment -fix ./...

# 只修复超过 N bytes 的浪费
fieldalignment -fix -limit 16 ./...

# 输出示例:
# ./types.go:15: struct with 64 pointer bytes could be 56 (fix applied)
```

## Race Detector

```bash
go test -race ./...
go build -race -o myapp
go run -race main.go
```

**限制：**
- 运行时开销 ~5-10x CPU，~5-10x 内存
- 只检测实际发生的数据竞争（race 未触发时不报告）
- 不能和 `-cover` 同时使用

## 其他工具

### golangci-lint (性能相关 linter)

```bash
# 包含多个性能检查
golangci-lint run --enable=perfsprint,prealloc,errcheck
```

### pprof 火焰图终端版

```bash
go install github.com/google/pprof@latest
pprof -flame cpu.out
```

### 持续 Profile (生产环境)

```bash
# Parca, Pyroscope, Google Cloud Profiler 等
# 持续收集生产环境 profile，查看历史趋势
```

### 微基准测试陷阱总结

| 陷阱 | 后果 | 修复 |
|------|------|------|
| CPU 频率动态调整 | 结果不稳定 | 固定 CPU 频率或多次运行 |
| Thermal throttling | 逐步变慢 | 确保散热 |
| 相邻测试互相影响 | 缓存污染 | 隔离或随机顺序 |
| 垃圾回收干扰 | 偶发高延迟 | 基准前触发 GC: `runtime.GC()` |
| OS 调度干扰 | 噪声 | `-count=10` 取中位数 |
| Turbo Boost | 短测试偏快 | 足够长的 `-benchtime` |