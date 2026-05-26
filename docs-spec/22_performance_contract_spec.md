# 22 - 性能合约与热路径规范

> 规定 `docs/architecture/performance_contract.md` 的内容结构与维护规则。
>
> 本规范是 AIWeave v1.2 演进新增的 P1 规范之一，对应痛点 #6 性能模型误判、#8 背压策略（部分）。

---

## 1. 定位

`performance_contract.md` 是 **AI 在生成涉及热路径代码、性能敏感操作、背压策略之前必须读的文档**。它回答四个问题：

> 1. 工程的全局性能 SLA 是什么？
> 2. 哪些代码路径属于"热路径"，热路径里有哪些禁止的操作？
> 3. 每请求的资源预算是多少？
> 4. 背压 / 容量保护策略是什么？

没有这篇文档：
- AI 在热路径里用反射 / `fmt.Sprintf` / 大对象分配，把 P99 推高
- AI 在热路径里发起同步外部 API 调用，拖慢整体吞吐
- AI 不知道 channel 容量上限，引入背压失控

---

## 2. 与 12_circuit_breaker 的边界划分（强制 / 双源真相规避）

22 与 12 在"背压 / 降级"上有重叠，本节先划清边界：

| 内容 | 12_circuit_breaker.md | 22_performance_contract.md |
| --- | --- | --- |
| 资源差异化熔断参数（K / 窗口 / 采样数） | ✅ 真相源 | 不写 |
| GORM Plugin / `cb.Do` 接入方式 | ✅ 真相源 | 不写 |
| 熔断时回退响应矩阵 | ✅ 真相源 | 不写 |
| 队列 / channel 容量上限 | 不写 | ✅ 真相源（§6.1） |
| 上游超时 vs 下游处理时间协调 | 不写 | ✅ 真相源（§6.2） |
| 优雅降级层级（L1/L2/L3 业务降级） | 引用 22 §6.3 | ✅ 真相源 |

**22 §6 中所有涉及"熔断器参数"的描述必须显式写"详见 12_circuit_breaker §X"**，不重复。

---

## 3. 顶层结构（强制）

`performance_contract.md` 必须按以下章节顺序组织：

- `## 1.` 全局性能目标（SLA）
- `## 2.` 热路径标注（核心 / AI 必读）
- `## 3.` 内存分配预算
- `## 4.` 数据访问性能约束
- `## 5.` 并发性能与锁等待预算（引用 20_concurrency_safety）
- `## 6.` 背压与容量保护（含与 12_circuit_breaker 的引用）
- `## 7.` 性能回归测试（引用 17_testing_design §4.9）
- `## 8.` 维护流程（含 B1 反向同步规则）

> **完整章节骨架见** [`templates/docs/architecture/performance_contract.md`](../templates/docs/architecture/performance_contract.md)。

---

## 4. §1 全局性能目标（SLA）

```markdown
| 指标 | 目标值 | 测量方式 | 降级阈值 |
|------|--------|---------|---------|
| P99 latency | < `{P99-ms}` ms | APM span / Prometheus histogram | > `{P99-degrade-ms}` ms 告警 |
| QPS | > `{QPS-target}` | Prometheus counter rate | < `{QPS-floor}` 告警 |
| 内存 | < `{Memory-target}` MB RSS | `/proc/meminfo` | > `{Memory-degrade}` MB 告警 |
| CPU | < `{CPU-target}` core | container stats | > `{CPU-degrade}` core 告警 |
```

> **占位符示例**：`{P99-ms}` 通常取 50；`{QPS-target}` 通常取 10000；具体由工程业务决定。

---

## 5. §2 热路径标注（核心 / 强制）

### 5.1 热路径定义

满足以下任一条件的代码路径：

- QPS > `{HOT-QPS-threshold}`（推荐 1000）
- P99 影响全局 SLA（P99 > 总 SLA 的 50%）
- 占总 CPU > `{HOT-CPU-threshold}`%（推荐 20%）

### 5.2 热路径清单（表格模板）

```markdown
| 路径名 | 入口（路由 / 任务 / 消费者） | QPS | 允许操作 | 禁止操作 |
|-------|-------------------------|-----|---------|---------|
| `{核心校验链}` | `/{prefix}/internal/{action-A}` | `{QPS-A}` | Redis GET/HGET、本地缓存读 | MySQL 查询、反射、fmt |
| `{状态上报链}` | `/{prefix}/internal/{action-B}` | `{QPS-B}` | Redis INCR/HSET | 日志（除异常）、大对象分配 |
| `{flush-task}` | cobra command 内循环 | — | 批量 MySQL 写、Redis pipeline | 单条 MySQL 查询、context lookup |
```

### 5.3 热路径标注的对外暴露

热路径方法的 `service docs §4.N.3 处理步骤`伪码中**必须**含 `[HOT-PATH]` 标记（详见 [`docs-spec/09 §10.5`](09_service_design_spec.md)）。

---

## 6. §3 内存分配预算

```markdown
### 3.1 每请求允许的堆分配上限

| 接口类别 | 堆分配上限（kB / 请求） | 备注 |
|---------|----------------------|------|
| 热路径 `{核心校验链}` | < `{Alloc-hot-kB}` | 通常 < 10 kB |
| 普通业务接口 | < `{Alloc-normal-kB}` | 通常 < 100 kB |
| 后台批处理任务 | < `{Alloc-batch-kB}` | 视业务定 |

### 3.2 对象池（sync.Pool）使用清单

| 池名 | 类型 | 用途 | 容量提示 |
|------|------|------|---------|
| `{pool-A}` | `*{Type-A}` | 在 `{module}.{Method}` 热路径中复用 | — |

### 3.3 预分配策略

- slice / map 初始容量必须显式（`make([]T, 0, N)` 不写 `make([]T, 0)`）
- 已知大小的 slice 必须预分配（追加超过 1 次即视为已知）
```

---

## 7. §4 数据访问性能约束

```markdown
### 4.1 Redis 命令延迟预算

| 操作 | 单次延迟预算 | 备注 |
|------|------------|------|
| GET / HGET / SET / HSET | < `{Redis-simple-ms}` ms（通常 < 1） | — |
| HGETALL / SMEMBERS / SUNION | 不允许在热路径中调用 | 用 HGET 字段集或 SPOP |
| Pipeline（≥ 3 命令） | 优于多次 GET | — |

### 4.2 MySQL 查询复杂度限制

- 禁止全表扫描（EXPLAIN type=ALL）
- JOIN 限制：≤ {JOIN-limit}（推荐 2）
- 单查询返回行数 ≤ `{Row-limit}`（推荐 1000）
- 写入 `{example_table}` 必须走主键或唯一索引

### 4.3 缓存命中率目标

| Key 类别 | 命中率目标 | 测量方式 |
|---------|----------|---------|
| `{ns}:info:*`（热数据） | > 95% | Redis stats |
| `{ns}:profile:*`（次热） | > 80% | — |
```

---

## 8. §5 并发性能与锁等待预算

```markdown
| 维度 | 预算 | 引用 |
|------|------|------|
| 全局锁持有时间 | < `{Lock-hold-ms}` ms | [`concurrency_safety.md §3`](concurrency_safety.md) |
| Goroutine 数量峰值 | < `{Goroutine-peak}` | [`concurrency_safety.md §1.1`](concurrency_safety.md) |
| Channel 排队时长 | < `{Channel-wait-ms}` ms | [`concurrency_safety.md §4.1`](concurrency_safety.md) |
```

锁粒度 / 加锁顺序的真相源在 [`docs-spec/20_concurrency_safety_spec.md §3`](20_concurrency_safety_spec.md)，本节仅承载"性能预算"维度。

---

## 9. §6 背压与容量保护

### 9.1 各队列 / channel 的容量上限

```markdown
| 队列 / channel | 容量 | 满时策略 | 引用 |
|--------------|------|---------|------|
| `{batch-ch}` | `{N}` | 丢弃尾部 + 计数告警 | [`concurrency_safety.md §4.1`](concurrency_safety.md) |
| `{retry-q}` | `{M}` | 阻塞 + 上游超时回压 | — |
```

### 9.2 上游超时 vs 下游处理时间协调

```markdown
| 链路 | 上游超时 | 本服务处理预算 | 下游超时 |
|------|---------|---------------|---------|
| {链路-A} | `{T-up-ms}` ms | `{T-self-ms}` ms | `{T-down-ms}` ms |
```

**关系约束**：`{T-self-ms} + {T-down-ms} < {T-up-ms} * 0.8`（留 20% 缓冲）。

### 9.3 优雅降级层级（L1 / L2 / L3）

```markdown
| 层级 | 触发条件 | 行为 | 与 12_circuit_breaker 的关系 |
|------|---------|------|----------------------------|
| L1 资源熔断 | 单资源连续失败 K 次 | 详见 [`docs-spec/12 §2`](12_circuit_breaker_spec.md) | ✅ 12 是真相源 |
| L2 服务降级 | L1 熔断扩散到多个核心资源 | 关闭非核心读路径，仅保留写入兜底 | 22 是真相源 |
| L3 关闭非核心功能 | L2 仍不缓解 | 关闭统计 / 报表 / 异步落库非紧急部分 | 22 是真相源 |
```

L1 的熔断器参数（K / 窗口 / 采样数）**不写在本文档**，详见 [`docs-spec/12 §2`](12_circuit_breaker_spec.md)。

---

## 10. §7 性能回归测试

```markdown
### 7.1 基准测试清单（go test -bench）

| 基准测试 | 方法 | 目标 P99 |
|---------|------|---------|
| `Benchmark{Method-A}` | `{Module}.{Method-A}` | `{P99-A-ns}` ns/op |

### 7.2 P99 回归检测规则

- 与基线对比，P99 偏差 > 20% → CI 标红
- 内存分配（allocs/op）增长 > 10% → CI 标红

### 7.3 与 17_testing_design §4.9 的关系

详细的基准测试 framework API 见 [`docs-spec/17 §4.9`](17_testing_design_spec.md)。本节仅承载"目标值 + 回归阈值"。
```

---

## 11. AI 行为约束

```markdown
### 性能约束（强制）

- 在热路径（§2 清单）中生成代码时：
  - 禁止使用反射（`reflect.*`）
  - 禁止 `fmt.Sprintf`（用 `strconv` 或 buffer）
  - 禁止在循环内分配大对象
  - 禁止同步调用外部 API
  - 禁止单条 MySQL 查询（必须批量或走缓存）
- 新增数据库查询 → 必须说明索引命中情况
- 新增内存缓存 → 必须在 §3 登记预分配策略
- 新增 channel → 容量必须在 §6.1 登记
```

---

## 12. §8 维护流程（含 B1 反向同步规则）

### 12.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 热路径文件（§2 清单内）新增 `reflect.*` / `fmt.Sprintf` / 反射相关 | 拒绝合并；或在 §2 移除该文件并报告 |
| 新增 `sync.Pool` / 显式预分配 | §3.2 / §3.3 登记 |
| 新增 channel 容量字段（`make(chan T, N)`，N 与 §6.1 不一致） | §6.1 满时策略校对；超出预算需调整 |
| 新增 `*_bench_test.go` 文件 | §7.1 基准测试清单同步 |
| 修改 SLA 数值（路由 / 配置） | §1 全局性能目标表同步 |
| 新增同步外部 API 调用 | 检查是否在热路径；如是 → 拒绝合并 |

### 12.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增热路径方法 | §2 必须新增一行 + 方法伪码加 `[HOT-PATH]` 标记 |
| 修改 SLA | §1 数值更新 + 评估告警阈值 |
| 引入新数据源 | §4 性能约束评估 + §5 并发预算评估 |

---

## 13. 与其他文档的关系

| 内容 | performance_contract.md | 12_circuit_breaker.md | 20_concurrency_safety.md | 23_observability.md |
|------|-------------------------|-----------------------|--------------------------|--------------------|
| SLA 目标值 | ✅ 真相源 | 不写 | 不写 | 引用（告警阈值） |
| 热路径清单 | ✅ 真相源 | 不写 | 不写 | 不写 |
| 内存 / 分配预算 | ✅ 真相源 | 不写 | 不写 | 不写 |
| 熔断器参数 | 引用 | ✅ 真相源 | 不写 | 不写 |
| 锁等待预算 | ✅（数值） | 不写 | 引用（粒度规则）| 不写 |
| 队列容量上限 | ✅ 真相源 | 不写 | 引用 | 不写 |
| Metric 注册 | 不写 | 不写 | 不写 | ✅ 真相源 |

---

## 14. 与 Skill 的联动

- **`/performance-review`**：扫描新增代码对照 §2 热路径清单 + grep 锚 `R-PERF-*`
- **`/new-service`** / **`/new-controller`**：生成涉及热路径的方法时，提示先读 §2 / §3
- **`/doc-sync-check`**：扫描代码中的 `reflect.*` / `fmt.Sprintf` / `make(chan T, N)` 是否符合 §1-§6 约束

---

## 15. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§14）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 15.1 §1 全局 SLA（示例）

| 指标 | 目标值 | 测量方式 | 降级阈值 |
|------|--------|---------|---------|
| P99 latency | < 50 ms | APM span | > 100 ms 告警 |
| QPS | > 10000 | Prometheus counter | < 5000 告警 |
| 内存 | < 2 GB RSS | /proc/meminfo | > 1.5 GB 告警 |
| CPU | < 4 core | container stats | > 6 core 告警 |

### 15.2 §2 热路径清单（示例：鉴权 + 计费服务）

| 路径名 | 入口 | QPS | 允许操作 | 禁止操作 |
|-------|------|-----|---------|---------|
| 鉴权链 | /api/internal/auth/verify | 10K+ | Redis GET/HGET、本地 user cache | MySQL 查询、反射、fmt |
| 状态上报链 | /api/internal/billing/report | 5K+ | Redis INCR/HSET | 日志（除异常）、大对象分配 |
| flush_state_to_db | cobra command | — | 批量 MySQL 写、Redis pipeline | 单条 MySQL 查询 |

### 15.3 §6.2 链路超时协调（示例）

| 链路 | 上游超时 | 本服务处理预算 | 下游超时 |
|------|---------|---------------|---------|
| 鉴权链（client → gateway → 本服务 → Redis） | 200 ms | 30 ms | 5 ms |

关系：30 + 5 = 35 < 200 × 0.8 = 160 ✅
