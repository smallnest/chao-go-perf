# Benchmark 方法论

## 正确编写 Benchmark

### 基本结构

```go
// fib_test.go
var sink int  // 全局 sink，阻止编译器优化消除

func BenchmarkFib(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = fib(20)
    }
    sink = r
}
```

### 编译器优化陷阱

Go 编译器会消除无副作用的死代码。以下都是错误写法：

```go
// 错误 1: 结果未使用 → 整个调用被消除
func BenchmarkBad1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fib(20)
    }
}

// 错误 2: 常量结果 → 循环被常量折叠
func BenchmarkBad2(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fib(20)
        sum += 42  // sum 未读取，全部消除
    }
}

// 错误 3: b.N 参与计算 → 编译器难以优化
func BenchmarkBad3(b *testing.B) {
    for i := 0; i < b.N; i++ {
        for j := 0; j < b.N; j++ {  // b.N 在内层循环
            _ = fib(20)
        }
    }
}
```

正确做法：

```go
// 方案 A: 全局 sink 变量
var globalSink int
func BenchmarkA(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s += fib(i % 100)
    }
    globalSink = s
}

// 方案 B: runtime.KeepAlive (适合指针/复杂对象)
func BenchmarkB(b *testing.B) {
    for i := 0; i < b.N; i++ {
        buf := new(bytes.Buffer)
        process(buf)
        runtime.KeepAlive(buf)
    }
}
```

### ResetTimer 的正确使用

```go
func BenchmarkWithSetup(b *testing.B) {
    data := expensiveSetup()  // 只执行一次
    b.ResetTimer()            // 排除 setup 时间
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// StopTimer/StartTimer 用于条件性排除
func BenchmarkWithConditional(b *testing.B) {
    for i := 0; i < b.N; i++ {
        if needSetup() {
            b.StopTimer()
            setup()
            b.StartTimer()
        }
        process()
    }
}
```

### Benchmark 参数

```bash
go test -bench=.          # 运行所有 benchmark
go test -bench=Fib        # 正则匹配
go test -bench=. -benchmem          # 显示内存分配
go test -bench=. -count=10          # 运行 10 次
go test -bench=. -benchtime=5s      # 最少运行 5 秒
go test -bench=. -benchtime=100x    # 最少运行 100 次
go test -bench=. -cpu=1,2,4,8       # 测试不同 GOMAXPROCS
```

## benchstat 统计验证

```bash
# 安装
go install golang.org/x/perf/cmd/benchstat@latest

# 基础用法
go test -bench=. -count=10 > old.txt
# ... 做优化 ...
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt
```

### 输出解读

```
name        old time/op    new time/op    delta
FuncA       4.52µs ± 2%    3.21µs ± 1%   -28.98%  (p=0.000 n=10+9)
FuncB       12.3µs ± 5%    12.1µs ± 4%   -1.63%   (p=0.142 n=10+10)
```

- `p < 0.05` → 统计显著，优化有效
- `p > 0.05` → 差异可能只是噪声，未证明有效
- `n=10+9` → 旧的 10 次，新的 9 次
- `± X%` → 波动范围

### benchstat 高级用法

```bash
# 多组对比
benchstat old.txt new1.txt new2.txt

# CSV 输出
benchstat -format csv old.txt new.txt

# 只显示 delta
benchstat -delta-test=none old.txt new.txt

# 按 benchmark 名字过滤
benchstat -filter=".filter=FuncA" old.txt new.txt
```

## Benchmark 反模式清单

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 无 sink 变量 | 代码被消除，结果 ≈ 0 | `var sink T` 接收结果 |
| `b.N` 内层循环 | 编译器无法常量传播 | 提取 `n := make([]int, b.N)` |
| 不需要的 setup 在循环内 | 测量了 setup 时间 | `b.ResetTimer()` 或 `StopTimer/StartTimer` |
| 测试数据太小 | 全在 L1 cache，不反映真实 | 使用真实规模的数据集 |
| count=1 | 单次结果不可靠 | `-count=5` 起步，`-count=10` 推荐 |
| 只看 time/op 不看 allocs/op | 错过内存瓶颈 | 加 `-benchmem` |
| 比较时不用 benchstat | 误判噪声为改进 | benchstat old new |
| 在一个 goroutine 测试并发代码 | 测不到竞争 | 在 benchmark 内部用 goroutine |
| 无 GC 说明的环境 | 不同机器/Go 版本不可比 | 记录 Go 版本、OS、CPU |

## 表驱动 Benchmark

```go
var benchmarkCases = []struct {
    name string
    n    int
}{
    {"small", 10},
    {"medium", 1000},
    {"large", 100000},
}

func BenchmarkProcess(b *testing.B) {
    for _, bc := range benchmarkCases {
        b.Run(bc.name, func(b *testing.B) {
            data := generateData(bc.n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                process(data)
            }
        })
    }
}
```

## 比较型 Benchmark (BenchmarkComp)

Go 1.24+ 支持 `testing.BenchmarkComp`，用于新旧实现对比：

```go
func BenchmarkOld(b *testing.B) {
    for i := 0; i < b.N; i++ { oldImpl() }
}
func BenchmarkNew(b *testing.B) {
    for i := 0; i < b.N; i++ { newImpl() }
}
// go test -bench=. -count=10 | tee /tmp/bench.txt
// benchstat -col .name -row .impl /tmp/bench.txt
```