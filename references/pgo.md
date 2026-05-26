# PGO — Profile-Guided Optimization

## 什么是 PGO

PGO (Profile-Guided Optimization) 是 Go 1.20 引入（1.21 GA）的编译器优化技术。通过收集生产环境（或 benchmark）的 profile 数据，反馈给编译器，使编译器做出更优的内联、代码布局等决策。

## 效果

- 典型提升: **2-7%**（CPU 密集型）
- 最好的案例: **10-15%**
- 开销: 构建时间增加 ~10-20%

## 完整工作流

### 步骤 1: 收集 Profile

```bash
# 方法 A: 从生产环境 HTTP endpoint
curl -o default.pgo http://localhost:6060/debug/pprof/profile?seconds=30

# 方法 B: 从 benchmark
go test -bench=. -cpuprofile=default.pgo

# 方法 C: 程序内嵌
f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
// ... 运行代表性工作负载 ...
pprof.StopCPUProfile()
```

### 步骤 2: 放置 Profile

将 profile 文件命名为 `default.pgo`，放在 `main` 包所在目录：

```
myapp/
├── main.go
└── default.pgo    # ← 放在这里
```

### 步骤 3: 构建（自动启用）

```bash
go build -o app
# PGO 自动启用！无需额外 flag
```

### 步骤 4: 收集新的 Profile（迭代）

```bash
# 用优化后的程序收集 profile，进入下一轮 PGO 迭代
curl -o default.pgo http://localhost:6060/debug/pprof/profile?seconds=30
go build -o app  # 用新 profile 重新构建
```

## 重要细节

### Profile 代表性

```bash
# 好: 代表真实负载
curl -o default.pgo http://prod-server:6060/debug/pprof/profile?seconds=60

# 不好: 微基准测试不代表真实负载
go test -bench=SingleFunc -cpuprofile=default.pgo
```

Profile 必须**代表生产环境的真实工作负载**，否则 PGO 可能无效果甚至负优化。

### PGO 做了什么

1. **更好的内联决策**: 热函数更激进内联，冷函数保留不内联
2. **代码布局优化**: 热路径代码放在一起，提高指令缓存命中
3. **虚调用去虚拟化**: 如果 profile 显示某个接口总是同一种实现，直接调用

### 验证 PGO 效果

```bash
# 1. 无 PGO 构建
rm -f default.pgo
go build -o app-no-pgo
go test -bench=. -count=10 > no-pgo.txt

# 2. 有 PGO 构建
# (放置 default.pgo)
go build -o app-pgo
go test -bench=. -count=10 > pgo.txt

# 3. 对比
benchstat no-pgo.txt pgo.txt
```

### CI/CD 集成

```bash
# 在 CI 中
curl -o default.pgo https://artifact-repo/profiles/$(date +%Y%m%d).pgo
go build -o app
# 或者在构建前下载最新 profile
```

### 多 Profile 合并

```bash
# Go 1.21+ 支持合并多个 profile
go tool pprof -proto -output merged.pgo profile1.pgo profile2.pgo
```

## 注意事项

- **始终验证**: 用 benchmark 确认 PGO 确实带来提升
- **Profile 更新频率**: 代码变动较大时重新收集
- **冷路径可能变慢**: PGO 优化热路径，冷路径可能稍微变慢
- **构建时间**: CI 中构建时间会增加 10-20%
- **不是银弹**: 配合良好的代码优化实践，PGO 只提供额外 2-7%
- **default.pgo 应该提交到版本控制吗**: 观点有分歧，通常推荐提交（确保可重现构建），但保持定期更新

## PGO 与其他优化对比

| 优化 | 提升幅度 | 工作量 | 适用性 |
|------|---------|--------|--------|
| slice 预分配 | 10-50% | 低 | 几乎所有 Go 程序 |
| 逃逸分析优化 | 10-30% | 中 | GC 密集型程序 |
| 算法优化 | 10-1000% | 高 | 取决于具体问题 |
| 并发优化 | 10-200% | 中-高 | 并发程序 |
| **PGO** | **2-7%** | **低** | **几乎免费** |
| Go 版本升级 | 5-10% | 低 | 所有程序 |

PGO 的优势是**几乎零工作量**，通常作为其他优化完成后的额外加成。