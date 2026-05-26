# AI 原生研发手册

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §10

## 1. 什么是 AI 原生工程

本工程**不是** "一个能用 AI 辅助的项目"，而是**为 AI 而设计**的项目。

> docs/ 是系统的完整蓝图，.claude/skills/ 是 AI 的操作手册。仅凭这两者，AI 可以从零重建整个工程。

## 2. 核心资产一览

| 资产 | 位置 | 作用 |
|------|------|------|
| CLAUDE.md | 项目根 | AI 总入口 |
| Skills | `.claude/skills/` | AI 操作手册 |
| 架构文档 | `docs/architecture/` | 系统级设计 |
| 接口规格 | `docs/api/` | N 个 API 完整定义 |
| Service 层 | `docs/service/` | 业务逻辑伪代码 |
| 数据模型 | `docs/schema/` | DDL + Redis Key |
| 缓存设计 | `docs/cache/` | 缓存策略 |
| 文档索引 | `docs/INDEX.md` | 文档目录 |
| 实现进度 | `docs/BUILD_STATUS.md` | 状态 |

## 3. Skill 体系（12 个）

详见 `.claude/skills/`。

## 4. 研发流程：设计先行

```
①  写 / 改 md
        ↓
②  AI 按 md 生成代码（Skill）
        ↓
③  go build / go test 验证 + 反向同步
```

## 5. 文档同步纪律

详见 `CLAUDE.md` 的「文档同步规则」章节。

## 6. 典型工作流示例

详见项目根 `aiweave/OPERATIONS.md`（如已并入工程，则放本文档）。

## 7. 对研发人员的建议

### 你需要做的
- 写好设计文档
- 审查 AI 输出
- 维护接口契约
- 定期审计

### 你不需要做的
- 手写 GORM model
- 手写 Controller boilerplate
- 手写路由注册
- 手写定时任务框架
- 维护 INDEX.md

### 黄金法则
> 文档写得越精确，AI 生成的代码越正确。把时间花在设计上，而不是编码上。

---

## 8. 约束总清单

> 本节是工程内所有"强制约束"的入口索引。各类约束的详细规则在对应文档；本节只列**约束名 → 文档锚点 → 当前状态**。
>
> 与 `CLAUDE.md 规则 X 范围判定表`的关系：CLAUDE.md 提供"代码路径 → 文档"的映射，本节提供"约束类型 → 文档锚点"的映射。两者互补，CLAUDE.md 是入口，本节是详情。

### 8.1 并发安全约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 共享状态注册表 | [`concurrency_safety.md §2`](concurrency_safety.md) | 🟢 |
| 锁粒度决策 / 加锁顺序 | [`concurrency_safety.md §3`](concurrency_safety.md) | 🟢 |
| Channel 与 Goroutine 池 | [`concurrency_safety.md §4`](concurrency_safety.md) | 🟢 |
| 资源生命周期管理 | [`concurrency_safety.md §5`](concurrency_safety.md) | 🟢 |
| 危险操作清单（含 rule-id） | [`concurrency_safety.md §6`](concurrency_safety.md) + 本文档 §10 | 🟢 |
| 并发测试策略（race / 压测 / leak） | [`concurrency_safety.md §7`](concurrency_safety.md) | 🟢 |

### 8.2 事务与一致性约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 事务模型选型 | [`transaction_design.md §1`](../service/transaction_design.md) | 🟢 |
| 事务边界清单 | [`transaction_design.md §2`](../service/transaction_design.md) | 🟢 |
| 补偿逻辑 / Saga 步骤 | [`transaction_design.md §3`](../service/transaction_design.md) | 🟢 |
| 幂等性设计 | [`transaction_design.md §4`](../service/transaction_design.md) | 🟢 |
| 最终一致性窗口 | [`transaction_design.md §5`](../service/transaction_design.md) | 🟢 |
| 失败路径全景图 | [`transaction_design.md §6`](../service/transaction_design.md) | 🟢 |

### 8.3 领域不变量约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 业务规则（必须始终成立） | `docs/service/{module}_service.md §7.1` | 🟢 |
| 隐式约束（代码中不明显） | `docs/service/{module}_service.md §7.2` | 🟢 |
| 业务状态机 | `docs/service/{module}_service.md §7.3` | 🟢 |

### 8.4 伪码标记语法（统一锚点）

| 标记 | 关联约束类 | 状态 |
|------|----------|------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 事务边界 | 🟢 |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿 | 🟢 |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验 | 🟢 |
| `[INVARIANT-CHECK: I-N]` | 领域不变量 | 🟢 |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE]` | 锁 | 🟢 |
| `[HOT-PATH]` | 热路径 | 🟢 |
| `[METRIC-EMIT: name{labels}]` | 指标 | 🟢 |

详细使用规则见 [`aiweave/docs-spec/09 §10.5`](../../../aiweave/docs-spec/09_service_design_spec.md)。

### 8.5 性能合约约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 全局 SLA（P99 / QPS / 内存 / CPU） | [`performance_contract.md §1`](performance_contract.md) | 🟢 |
| 热路径清单 + 禁止操作 | [`performance_contract.md §2`](performance_contract.md) | 🟢 |
| 内存分配预算 / 对象池 | [`performance_contract.md §3`](performance_contract.md) | 🟢 |
| 数据访问性能约束（Redis / MySQL / 缓存命中率） | [`performance_contract.md §4`](performance_contract.md) | 🟢 |
| 背压与容量保护 / 链路超时协调 / 降级层级 | [`performance_contract.md §6`](performance_contract.md) | 🟢 |
| 性能回归测试（基准 + P99 偏差 + allocs/op） | [`performance_contract.md §7`](performance_contract.md) + [`testing_design.md §4.9`](../testing/testing_design.md) | 🟢 |

### 8.6 可观测性约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| Metrics 采集范围（HTTP / 健康 / Runtime） | [`observability.md §2`](observability.md) | 🟢 |
| Cardinality 默认上限（≤ 500） | [`observability.md §3.1`](observability.md) | 🟢 |
| Label 禁止清单（不可登记突破） | [`observability.md §3.2`](observability.md) | 🟢 |
| Metric 命名约定 | [`observability.md §4`](observability.md) | 🟢 |
| 服务级告警规则（P0 / P1 / P2） | [`observability.md §5`](observability.md) | 🟢 |
| Metric 集中注册（`helpers/metrics.go`） | [`observability.md §6.1`](observability.md) | 🟢 |

### 8.7 Schema 演进约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 废弃字段注册表 | `docs/schema/database_design.md §8.1` | 🟢 |
| 字段语义变更记录 | `docs/schema/database_design.md §8.2` | 🟢 |
| Schema 兼容性规则（默认值 / 类型兼容 / 软删除窗口） | `docs/schema/database_design.md §8.3` | 🟢 |

### 8.8 安全重构约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 重构前准备清单（变更边界 / 测试覆盖 / 回滚方案 / 兼容期） | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| 重构 5 步模板 | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| AI 行为约束（≤ 200 行 / ≤ 5 commit / 每 commit 测试通过） | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| 灰度切流量（Shadow read → 双写 → 1%/10%/50%/100%） | `docs/architecture/mvp_rebuild_path.md §11.5.4` | 🟢 |

### 8.9 跨服务合约约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 上下游依赖图 | [`cross_service_contract.md §1`](cross_service_contract.md) | 🟢 |
| 上游合约（谁调我 / SLA / 超时 / 重试） | [`cross_service_contract.md §2`](cross_service_contract.md) | 🟢 |
| 下游合约（我调谁 / 超时 / 熔断 / 降级） | [`cross_service_contract.md §3`](cross_service_contract.md) | 🟢 |
| 故障传播矩阵 | [`cross_service_contract.md §4`](cross_service_contract.md) | 🟢 |
| 接口版本管理 | [`cross_service_contract.md §5`](cross_service_contract.md) | 🟢 |

### 8.10 运行时基线（如启用 runtime_profile.md）

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 流量模式 | [`runtime_profile.md §1`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 数据规模 | [`runtime_profile.md §2`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 资源消耗基线 | [`runtime_profile.md §3`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 关键超时链 | [`runtime_profile.md §4`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |

> 运行时基线属于"AI 不直接感知"维度，启用与否由工程负责人决定（详见 INDEX.md §0 采纳进度）。

---

## 9. 约束突破登记表（v1.2 新增）

> 部分约束（如 metrics cardinality 默认上限、Metric 总数）是**默认上限**而非硬限制。如确有业务理由必须突破，必须在本表登记。

| 日期 | 突破的约束 | 实际值 | 理由 | 替代方案评估 | 责任人 | 关联 PR |
|------|----------|--------|------|-------------|-------|---------|
| {YYYY-MM-DD} | {约束名（如 metric_cardinality_total_500）} | {实际值} | {业务理由} | {为什么替代方案不可行} | {开发者} | {PR-link} |

**登记纪律**：

- 突破未登记 → 视为隐式违规，审计 Skill 命中将阻断合并
- 同类突破累计 3 次 → 触发约束本身的回头复审（可能需要放宽默认值或引入新约束类）

> 本表预期为空；当 metrics / performance 上限引入后才有突破场景。

---

## 10. grep 锚 rule-id 索引（v1.2 新增）

> 危险模式清单的机械审计入口。每条 rule-id 对应一条 grep / AST 规则，由 `concurrency-review` / `performance-review`等审计 Skill 触发。
>
> **grep 锚定位为"信号级"非"判定级"**：命中标 🟡 待复核，最终判定权在人工 reviewer。误报通过 `// aiweave:allow=<rule-id>` 行内注解抑制。

### 10.1 并发安全类

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-CONC-LOCK-IO` | 锁内做网络 IO | `\.Lock\(\)[\s\S]{0,500}?(http\.\|Mysql\|Redis)` | RLock 中读小对象 | 🟢 |
| `R-CONC-LOCK-LOG` | 锁内打日志 | `\.Lock\(\)[\s\S]{0,300}?(tlog\.\|log\.)` | — | 🟢 |
| `R-CONC-MAP-RACE` | map 并发写 | 文件含 `var .* map\[` 且同文件存在 `go func` 且无 `sync\.` | 局部 map 不跨 goroutine | 🟢 |
| `R-CONC-DOUBLE-CLOSE` | 关闭已关闭的 channel | `close\([^)]+\)` 出现多次且无 sync.Once 保护 | — | 🟢 |
| `R-CONC-SEND-CLOSED` | 向已关闭的 channel 发送 | — | 见 §10.1 备注 | 🟢 |
| `R-RESOURCE-DEFER-LOOP` | defer 在 for 循环内 | `for [^{]*\{[\s\S]{0,300}?defer ` | 循环体内显式调用 Close | 🟢 |
| `R-CONC-GOROUTINE-LEAK` | 忽略 ctx.Done() | `go func` 块内无 `ctx\.Done\(\)` 且无 `select` | 短周期任务（< 1s）| 🟢 |

### 10.2 性能 / 资源类

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-PERF-LOOP-DB-QUERY` | for 循环内 DB 查询（N+1） | `for [^{]*\{[\s\S]{0,300}?\.(Find\|First\|Where\|Take)\(` | 显式批量 API（In/Pluck/Scan）/ 循环次数硬编码 ≤ 5 | 🟢 |
| `R-PERF-LOOP-ALLOC` | for 循环内大对象分配 | 循环体含 `make([]T, N)` 且 N > 1024 或 struct 字段数 > 20 | 循环前已 capacity 预分配 / 仅 error 分支 | 🟢 |
| `R-PERF-HOT-REFLECT` | 热路径用反射 | 热路径文件含 `reflect\.` | 文件含 `//go:build !hotpath` | 🟢 |
| `R-PERF-HOT-FMT` | 热路径用 fmt.Sprintf / fmt.Errorf | 热路径文件含 `fmt\.Sprintf\(\|fmt\.Errorf\(` | 错误路径 / 用于 wrap error | 🟢 |
| `R-PERF-FULL-COUNT` | 全表 COUNT(*) | `COUNT\(\*\)\s+FROM` 且无 WHERE / WHERE 列无索引 | 小表（< 1 万行）/ 后台批处理 | 🟢 |
| `R-RESOURCE-SLEEP-SYNC` | time.Sleep 做同步 | `time\.Sleep\(` 出现在 `go func` 或 `for` 内 | 重试退避（指数退避） / 测试代码 | 🟢 |
| `R-CACHE-LARGE-KEY` | Redis 大 Key（> 1MB） | `\.Set\(` 紧随大对象序列化 / SADD 一次 > 1000 元素 | 显式分片 | 🟢 |

### 10.3 事务 / 幂等类

| Rule-id | 含义 | grep 锚（参考） | 状态 |
|---------|------|---------------|------|
| `R-TXN-NO-IDEMPOTENT` | 写入类接口无幂等 Key | 路由含 `POST` 且 controller 无 SETNX / 唯一索引引用 | 🟢 |
| `R-TXN-CROSS-SOURCE` | 跨数据源写入未在 transaction_design.md §2 登记 | git diff 同时含 MySQL 写 + Redis 写 + 未登记 | 🟢 |

### 10.4 领域不变量类

| Rule-id | 含义 | 检查方式 | 状态 |
|---------|------|---------|------|
| `R-INVARIANT-MARK-MISMATCH` | 伪码 `[INVARIANT-CHECK: I-N]` 标记与代码实现脱节 | 由 `domain-invariant-check` 提取标记 + 静态分析对比 | 🟢 |
| `R-INVARIANT-MISSING-CHECK` | §7.1 业务规则在 "代码位置" 列指明的文件中无校验逻辑 | 由 `domain-invariant-check` 扫描 | 🟢 |
| `R-STATE-ILLEGAL-TRANSITION` | UPDATE 状态字段时未校验当前状态 / 跳转不在 §7.3 状态机 | 由 `domain-invariant-check` 扫描 | 🟢 |

### 10.5 跨服务合约类

| Rule-id | 含义 | grep 锚（参考） | 状态 |
|---------|------|---------------|------|
| `R-XSVC-NO-TIMEOUT` | 下游调用未显式设超时 | `(http\.Client\|grpc\.Dial)` 后无 `Timeout` / `WithTimeout` | 🟢 |
| `R-XSVC-TIMEOUT-CASCADE` | 本服务对下游设的超时 ≥ 上游对本服务的超时 | 由 `failure-path-review` + 22 §6.2 链路超时协调表对比 | 🟢 |
| `R-XSVC-SILENT-SWALLOW` | 下游调用错误被 `_ = ...` 静默吞 | `_ = .*\.Call\(` / `_ = .*\.Get\(` 等 | 🟢 |
| `R-XSVC-UNREGISTERED-CLIENT` | `api/` 新增 client 但未在 cross_service_contract.md §3 登记 | 由 `doc-sync-check` + `failure-path-review` 联合检查 | 🟢 |

### 10.6 失败路径类

| Rule-id | 含义 | 检查方式 | 状态 |
|---------|------|---------|------|
| `R-FAIL-PATH-UNDOC` | 代码中失败分支 (F-N) 未在 transaction_design.md §6 失败路径全景图覆盖 | 由 `failure-path-review` 静态分析 | 🟢 |
| `R-FAIL-PATH-NO-TEST` | 失败分支 (F-N) 无对应测试用例 | 由 `failure-path-review` 扫描 test/cases/ | 🟢 |
| `R-FAIL-PATH-WEAK-ASSERT` | 测试用例仅断言 `err != nil` 未断言 errNo / 副作用 | 由 `failure-path-review` 扫描断言模式 | 🟢 |

### 10.7 规则维护

- 新增 rule-id → 本节新增一行；同步在 `concurrency-review` / `performance-review` Skill 实现 grep 模式
- 误报率高的规则 → 优化 grep 锚或下调严重级别（🔴 → 🟡）
- 长期 0 命中的规则 → 评估是否仍有价值；可降级为"可选规则"
