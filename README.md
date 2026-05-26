# chao-go-perf

Go 性能分析专家 Skill，基于 Dave Cheney 高性能 Go 工作坊、dgryski go-perfbook、Effective Go 和 Go 101 Optimizations 等权威资料。

## 安装

```bash
# 克隆到 Claude Code 全局 skills 目录
git clone git@github.com:smallnest/chao-go-perf.git ~/.claude/skills/chao-go-perf
```

或者：

```bash
npx skills add smallnest/chao-go-perf
```

或者在智能体中输入：
```
安装skill: https://github.com/smallnest/chao-go-perf
```

## 功能

### 性能瓶颈诊断
- CPU 热点分析（pprof CPU profile + 火焰图）
- 内存分配热点（pprof allocs + 逃逸分析）
- GC 压力分析（GODEBUG=gctrace + GOMEMLIMIT）
- 锁竞争检测（pprof mutex profile）
- 并发扩展性诊断（race detector + trace）

### 内存优化
- 逃逸分析（`-gcflags="-m"`）与堆分配减少
- Slice/Map 预分配策略
- `strings.Builder` 替代字符串拼接
- `sync.Pool` 对象复用最佳实践
- struct 字段布局优化（`fieldalignment`）
- GOGC / GOMEMLIMIT GC 调优

### 编译器优化
- BCE（边界检查消除）分析与验证
- 内联决策分析与优化
- `GOSSAFUNC` 编译器优化过程可视化
- 编译器 flag 完整速查

### CPU 缓存优化
- Cache line 与 false sharing 检测与消除
- 数据局部性优化（AoS vs SoA）
- CPU 分支预测友好代码

### 并发性能
- 锁选择决策矩阵（Mutex / RWMutex / atomic / sync.Map）
- 分片锁减少竞争
- Channel vs Mutex 性能对比
- Goroutine 泄漏检测与预防

### 工具链
- benchmark 正确编写与 benchstat 统计验证
- pprof / trace 完整使用指南
- fieldalignment struct 对齐检查

### 版本迁移
- Go 1.12 ~ 1.27 关键性能变更
- PGO (Profile-Guided Optimization) 完整工作流
- 版本升级迁移建议

## 使用

在 Claude Code 对话中自动触发，或手动调用：

```
/chao-go-perf
```

触发词包括：Go 性能、benchmark、pprof、逃逸分析、BCE、false sharing、sync.Pool、GC 优化、编译器优化 等。


## License

MIT
