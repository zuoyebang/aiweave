# 性能合约与热路径

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/22_performance_contract_spec.md`](../../../docs-spec/22_performance_contract_spec.md)
>
> 本文档是 **AI 在生成涉及热路径代码、性能敏感操作、背压策略之前必须读的文档**。

---

## 1. 全局性能目标（SLA）

| 指标 | 目标值 | 测量方式 | 降级阈值 |
|------|--------|---------|---------|
| P99 latency | < `{P99-ms}` ms | APM span / Prometheus histogram | > `{P99-degrade-ms}` ms 告警 |
| QPS | > `{QPS-target}` | Prometheus counter rate | < `{QPS-floor}` 告警 |
| 内存 | < `{Memory-target}` MB RSS | `/proc/meminfo` | > `{Memory-degrade}` MB 告警 |
| CPU | < `{CPU-target}` core | container stats | > `{CPU-degrade}` core 告警 |

---

## 2. 热路径标注（AI 必读）

### 2.1 热路径定义

满足以下任一条件的代码路径：

- QPS > `{HOT-QPS-threshold}`
- P99 影响全局 SLA（P99 > 总 SLA 的 50%）
- 占总 CPU > `{HOT-CPU-threshold}`%

### 2.2 热路径清单

| 路径名 | 入口 | QPS | 允许操作 | 禁止操作 |
|-------|------|-----|---------|---------|
| `{核心校验链}` | `/{prefix}/internal/{action-A}` | `{QPS-A}` | Redis GET/HGET、本地缓存读 | MySQL 查询、反射、fmt |
| `{状态上报链}` | `/{prefix}/internal/{action-B}` | `{QPS-B}` | Redis INCR/HSET | 日志（除异常）、大对象分配 |

### 2.3 热路径标注

热路径方法的 service docs §4.N.3 处理步骤伪码中必须含 `[HOT-PATH]` 标记。

---

## 3. 内存分配预算

### 3.1 每请求允许的堆分配上限

| 接口类别 | 堆分配上限（kB / 请求） | 备注 |
|---------|----------------------|------|
| 热路径 | < `{Alloc-hot-kB}` | — |
| 普通业务接口 | < `{Alloc-normal-kB}` | — |
| 后台批处理任务 | < `{Alloc-batch-kB}` | — |

### 3.2 对象池（sync.Pool）使用清单

| 池名 | 类型 | 用途 | 容量提示 |
|------|------|------|---------|
| `{pool-A}` | `*{Type-A}` | 在 `{module}.{Method}` 热路径中复用 | — |

### 3.3 预分配策略

- slice / map 初始容量必须显式（`make([]T, 0, N)`）
- 已知大小的 slice 必须预分配

---

## 4. 数据访问性能约束

### 4.1 Redis 命令延迟预算

| 操作 | 单次延迟预算 | 备注 |
|------|------------|------|
| GET / HGET / SET / HSET | < `{Redis-simple-ms}` ms | — |
| HGETALL / SMEMBERS / SUNION | 不允许在热路径中调用 | 用 HGET 字段集或 SPOP |
| Pipeline（≥ 3 命令） | 优于多次 GET | — |

### 4.2 MySQL 查询复杂度限制

- 禁止全表扫描
- JOIN 限制：≤ `{JOIN-limit}`
- 单查询返回行数 ≤ `{Row-limit}`

### 4.3 缓存命中率目标

| Key 类别 | 命中率目标 |
|---------|----------|
| `{ns}:info:*` | > 95% |
| `{ns}:profile:*` | > 80% |

---

## 5. 并发性能与锁等待预算

| 维度 | 预算 | 引用 |
|------|------|------|
| 全局锁持有时间 | < `{Lock-hold-ms}` ms | [`concurrency_safety.md §3`](concurrency_safety.md) |
| Goroutine 数量峰值 | < `{Goroutine-peak}` | [`concurrency_safety.md §1.1`](concurrency_safety.md) |
| Channel 排队时长 | < `{Channel-wait-ms}` ms | [`concurrency_safety.md §4.1`](concurrency_safety.md) |

---

## 6. 背压与容量保护

### 6.1 各队列 / channel 的容量上限

| 队列 / channel | 容量 | 满时策略 | 引用 |
|--------------|------|---------|------|
| `{batch-ch}` | `{N}` | 丢弃尾部 + 计数告警 | [`concurrency_safety.md §4.1`](concurrency_safety.md) |

### 6.2 上游超时 vs 下游处理时间协调

| 链路 | 上游超时 | 本服务处理预算 | 下游超时 |
|------|---------|---------------|---------|
| {链路-A} | `{T-up-ms}` ms | `{T-self-ms}` ms | `{T-down-ms}` ms |

**关系约束**：`{T-self-ms} + {T-down-ms} < {T-up-ms} * 0.8`。

### 6.3 优雅降级层级

| 层级 | 触发条件 | 行为 |
|------|---------|------|
| L1 资源熔断 | 单资源连续失败 K 次 | 详见 [`circuit_breaker_design.md §2`](../circuit_breaker/circuit_breaker_design.md) |
| L2 服务降级 | L1 熔断扩散到多个核心资源 | 关闭非核心读路径 |
| L3 关闭非核心功能 | L2 仍不缓解 | 关闭统计 / 报表 / 异步落库 |

L1 的熔断器参数（K / 窗口 / 采样数）真相源在 `circuit_breaker_design.md §2`。

---

## 7. 性能回归测试

### 7.1 基准测试清单

| 基准测试 | 方法 | 目标 P99 |
|---------|------|---------|
| `Benchmark{Method-A}` | `{Module}.{Method-A}` | `{P99-A-ns}` ns/op |

### 7.2 P99 回归检测规则

- P99 偏差 > 20% → CI 标红
- allocs/op 增长 > 10% → CI 标红

framework API 见 [`docs/testing/testing_design.md §4.9`](../testing/testing_design.md)。

---

## 8. 维护流程

### 8.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 热路径文件新增 `reflect.*` / `fmt.Sprintf` | 拒绝合并 |
| 新增 `sync.Pool` / 显式预分配 | §3.2 / §3.3 登记 |
| 新增 channel 容量字段（N 与 §6.1 不一致） | §6.1 满时策略校对 |
| 新增 `*_bench_test.go` 文件 | §7.1 基准测试清单同步 |
| 修改 SLA 数值 | §1 全局性能目标表同步 |

### 8.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条热路径条目（§2 一行）/ 每个基准测试（§7.1 一行）对应 BUILD_STATUS.md §11 "性能合约约束"类目的"已设计 / 已启用"计数。
