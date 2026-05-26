# Go 版本关键性能变更

## Go 1.12 (2019.02)

- **Mid-stack inlining**: 允许包含循环的中间层函数内联
- 运行时 timer 优化：减少 timer 相关的 goroutine 开销
- `fmt` 包性能优化

## Go 1.13 (2019.09)

- **sync.Pool GC 行为变更**: GC 只清空 victim cache，local cache 保留 → Pool 命中率大幅提升
- **defer 性能改进**: 栈上分配的 defer 更快
- `context` 包: `Err() error` 零分配
- `crypto/ed25519`: 新包，高性能签名

## Go 1.14 (2020.02)

- **defer 零开销** (常见路径): 直接代码内联，不如以前需要堆分配
- **goroutine 异步抢占**: 解决长时间运行的 goroutine 无法被抢占的问题
- timer 系统重构: 减少 timer 相关的 goroutine 数量
- `testing` 包: `t.Cleanup()` 支持

## Go 1.15 (2020.08)

- 链接器重写: 性能提升 20%，内存占用降低 30%
- 小对象分配优化: < 16 字节对象的分配

## Go 1.16 (2021.02)

- `//go:embed`: 编译时嵌入文件，运行时零开销读取
- `io/fs` 包: 文件系统接口标准化
- 运行时性能改进

## Go 1.17 (2021.08)

- **函数参数寄存器传递** (AMD64): 参数和返回值通过寄存器传递，性能提升 5-10%
- `//go:build` 构建约束
- `unsafe.Add`, `unsafe.Slice`: 安全的指针运算

## Go 1.18 (2022.03)

- **泛型**: 编译时单态化，运行时基本零开销
- `sync.Pool` 不再每次 GC 完全清空
- `strings.Cut`, `bytes.Cut`: 零分配字符串分割
- `net/netip`: 新的不可变 IP 地址类型，更小更快

## Go 1.19 (2022.08)

- **GOMEMLIMIT**: 软内存限制，防止 OOM
- **排序算法优化**: pdqsort (pattern-defeating quicksort)
- `sync/atomic` 新增类型: `Bool`, `Int32`, `Int64`, `Uint32`, `Uint64`, `Pointer` 的包装类型
- `fmt.Append`, `fmt.Appendf`, `fmt.Appendln`: 零分配格式化追加

## Go 1.20 (2023.02)

- **PGO 预览** (Profile-Guided Optimization)
- **arena 实验性**: 批量分配/释放，减少 GC 压力
- `crypto/ecdh`: 新包
- `sync.Map` 优化
- `errors.Join`: 合并多个 error

## Go 1.21 (2023.08)

- **PGO GA**: 正式可用
- **`clear` 内置函数**: 零开销清空 map/slice
- `sync.OnceFunc`, `sync.OnceValue`, `sync.OnceValues` — 更简洁的单例
- `slices` 包: `BinarySearch`, `Contains`, `Sort`, `Clone` 等
- `maps` 包: `Clone`, `Copy`, `DeleteFunc` 等
- `cmp` 包: `Ordered`, `Compare`, `Less`
- `context.WithTimeoutCause`, `context.WithDeadlineCause`

## Go 1.22 (2024.02)

- **`range` over int**: `for i := range 10`
- PGO 改进: 更多优化 pass
- 运行时: 减少 stop-the-world 时间
- `math/rand/v2` 包

## Go 1.23 (2024.08)

- **`sync.Map.Clear()`**: 高效清空
- **`atomic.And` / `atomic.Or`**: 位运算原子操作
- **结构化日志**: `log/slog` 稳定版
- **`unique` 包**: 字符串/值去重 (interning)
- `iter` 包: 迭代器接口

## Go 1.24 (2025.02)

- **`sync.Map` 重写**: hash-trie map 实现，对不相交 key 并发修改性能大幅提升
- `sync.Map.CompareAndSwap`, `sync.Map.CompareAndDelete`
- 更多泛型类型约束

## Go 1.25 (2025.08 预期)

- **`testing/synctest`**: 隔离时间环境中测试并发代码
- **`sync.WaitGroup.Go`**: `wg.Go(f)` 等价于 `wg.Add(1); go func() { defer wg.Done(); f() }()`

## Go 1.27

- `synctest.Sleep`: 在测试中模拟时间推进

## 迁移建议

### 如果要升级 Go 版本

1. **Go 1.17 是重要分水岭** — 寄存器传参带来 5-10% 免费性能提升
2. **Go 1.21+ PGO** — 生产环境开启 PGO 可额外提升 2-7%
3. **Go 1.14+ defer** — 升级后可放心在热路径使用 defer
4. **Go 1.13+ sync.Pool** — Pool 命中率大幅改善，减少 GC
5. **Go 1.24+ sync.Map** — 高并发场景考虑升级

### 检查清单

- [ ] 是否使用 Go 1.17+ 享受寄存器传参？
- [ ] 是否使用 Go 1.21+ 并启用 PGO？
- [ ] 是否将 `sync.Map` 迁移到 Go 1.24+ 的 hash-trie 实现？
- [ ] 是否使用 `sync.OnceFunc` 替代手写 Once+闭合？
- [ ] 是否使用 `clear()` 内置函数清空 map？