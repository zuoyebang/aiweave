# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

> 项目启动时复制本骨架到项目根 `CLAUDE.md`，按 `aiweave/docs-spec/03_claude_md_spec.md` 规范填充。

---

## 项目概述

{ProjectName}（{ProjectAlias}），基于 {Framework}（Go {Version}），是一个{独立微服务/库/...}，提供 {核心能力清单}。架构上设计为 {N 种运行模式共一份代码库}，由 {cli framework} 子命令区分入口。

> **当前实现状态**：项目处于"{当前阶段}"阶段。完整实现状态一览见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md)。当你读到文档里某个 service 方法、接口或定时任务时，**不要假设它已在代码中实现**——先查 BUILD_STATUS，再决定是"调用现有实现"还是"按 skill 生成新实现"。

> **⛔ 暂不实现模块（如有，临时决策）**：**{module} 模块**（涉及 `{path1}`、`{path2}`）当前阶段**一律不进入代码实现**。设计文档完整保留，但 AI 在任何 skill 下都**不得**为这些模块生成代码。完整列表、检查规则、解除条件见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md) §0。

## 常用命令

### 构建与运行

```bash
# 启动 HTTP 服务（默认）
go run main.go

# 启动指定任务
go run main.go {task-name}

# 构建二进制文件
go build -o {binary-name}

# 查看帮助
go run main.go -h
```

### 测试

```bash
# 运行所有测试
go test ./...

# 运行指定包的测试
go test ./{package}/...

# 运行单个测试文件
go test -v ./{path}/{file}_test.go

# 运行带覆盖率
go test -cover ./...
```

### 代码格式化

```bash
go fmt ./...
```

## 项目架构

### 目录结构

```
project/
├── main.go              # 入口文件，使用 {cli} 管理命令行
├── conf/                # 配置目录
│   ├── app/             # 业务配置（随代码发布）
│   └── mount/           # 环境配置（动态修改）
├── controllers/         # 请求入口
│   ├── command/         # 定时任务入口
│   ├── http/            # HTTP 请求入口
│   └── mq/              # 消息队列消费入口
├── router/              # 路由注册
├── helpers/             # 资源初始化与辅助函数
├── components/          # 公共组件（状态码、常量等）
├── models/              # GORM 数据模型
├── service/             # 内部业务逻辑层
├── api/                 # 外部 API 调用封装（带熔断器）
├── data/                # 数据访问层封装
├── middleware/          # 中间件
├── log/                 # 日志文件目录
├── scripts/             # 脚本文件
├── sql/                 # SQL 脚本
├── docs/                # 技术方案文档（系统蓝图）
└── pkg/                 # 第三方/内部库封装
```

### 分层架构

```
controllers/ → service/ 或 api/ → models/ 或 data/ 或 helpers/
     ↓               ↓                       ↓
  参数校验        业务逻辑              数据访问
```

- **controllers/**：参数校验、调用 service/api、组装返回数据
- **service/**：内部业务逻辑
- **api/**：外部 API 调用，带熔断器保护
- **models/**：GORM 数据模型
- **data/**：复杂数据访问封装
- **helpers/**：资源初始化（MySQL / Redis / MQ / 熔断器等），提供全局客户端实例

### 配置体系

配置文件位于 `conf/` 目录：

- **mount/**：环境相关，可由配置中心动态修改
  - `config.yaml`：基础配置
  - `api.yaml`：外部 API 调用配置
  - `resource.yaml`：资源配置（MySQL / Redis / Kafka / MQ）
- **app/**：业务相关，随代码一起发布
  - `app.yaml`：熔断器配置等

敏感信息使用 `@@key` 格式占位，运行时由配置中心替换。

### 接口返回格式

> 如有多种响应格式（如 Internal vs UserSelf），列出每种。

```json
{ "errNo": 0, "errStr": "", "data": {} }
```

### 错误码规范

定义在 `components/error.go` / `components/service_error.go`，按业务范围划分（详见 `docs/architecture/status_codes.md`）。

### 路由注册

HTTP 路由在 `router/http.go` 注册，路由前缀为 `/{prefix}`。

MQ 消费者在 `router/mq.go` 中注册。

定时任务在 `router/command.go` 通过 cobra 命令注册。

### 支持的基础设施

> 列出本工程实际使用的基础设施。例：

- **MySQL**：N 个库
- **Redis**：N 个实例
- **Kafka**：N 个 topic
- **本地缓存**：gcache 等
- **熔断器**：Google SRE 自适应算法

### 核心设计原则

> 列出 3-5 条核心设计断言。例：

- **Redis-first**：高流量操作先写 Redis，定时任务 flush 到 MySQL
- **MQ 异步落库**：业务事件流水走 Kafka 异步持久化
- **统一熔断器**：Redis、MySQL、外部 API 全部由同一套 CircuitBreaker 接口保护

## 开发注意事项

- 控制器只负责参数校验、调用 service/api、组装返回数据，不写业务逻辑
- URI 命名使用小写字母和连字符，不使用下划线
- 外部 API 调用必须使用熔断器保护
- 新增错误码按规范定义在 `components/error.go`
- service 层处理内部业务逻辑，api 层处理外部服务调用
- 测试文件与被测文件同目录，命名为 `*_test.go`

### 日志规范（强制）

统一使用 Structured API（`tlog.*Logger`），**禁止使用** Sugared API（`tlog.*f`）。详细规范见 `docs/architecture/logging.md`。

```go
// 正确 ✅
tlog.InfoLogger(ctx, "task completed",
    tlog.String("task", taskName),
    tlog.Int("processed", count))

// 错误 ✗ — Sugared API
tlog.Infof(ctx, "task completed: %s, processed=%d", taskName, count)
```

**级别判定**：能自愈 → WARN；需人工介入 → ERROR。
**敏感数据禁止打印**：EntitySecret、Token、密码绝对不输出到日志。

### Hooks 机制（强制 / L0 自动化防御层）

本项目通过 `.claude/settings.json`（团队共享，入 git）配置**两类强制 Hook**，由 Claude Code 在工具事件机械执行，不依赖 AI 自觉：

| Hook 族 | 事件 | 职责 |
|--------|------|------|
| **IO 铁律检查** | 编辑/写 `.go` 后 | 跑 `R-IO-*` grep 锚，命中 N+1 / 串行编排 → 告警（可升级 PreToolUse 阻断） |
| **代码↔文档双向同步** | 编辑 `.go` 后 + 会话 Stop | 按范围判定表提醒同步 md；Stop 校验"文档同步：..."声明存在 |

Hook 脚本在 `scripts/hooks/`，纯标准工具实现。配置规范见 `aiweave/skills-spec/02_settings_local_json_spec.md §4`。Hooks 是 6 层防御的 L0 层，与下方文档同步规则（L1-L5）叠加生效。

## 文档同步规则（强制）

> **本规则是本项目的最高优先级纪律，优先级高于一切其他开发规范。任何不满足文档同步的交付都视为未完成，必须补齐后才能结束。**

### 愿景

`docs/` 中的 md 是整个系统的完整蓝图。目标是：**仅凭 docs/ 中的 md 就能用 AI 从零重建整个工程**。

### 研发流程：设计先行

```
1. 先写/改 md（技术方案文档）
2. 再根据 md 写/改代码
3. 如果实现中发现 md 需要调整，立即同步更新 md
```

### 核心原则：双向同步（绝对强制）

| 变更方向 | 强制要求 |
|---------|---------|
| **md → 代码** | 严格按 md 实现；实现中发现 md 需调整，必须同步更新 md |
| **代码 → md** | 找到关联 md 并必须同步更新；如无对应 md，必须新建 |
| **新增代码** | 必须新建/更新对应 md，并在 INDEX.md 登记 |
| **删除代码** | 必须同步删除/更新对应 md，并更新 INDEX.md |

### 18 条具体执行规则

#### 规则 1：修改代码前 — 查找关联文档
按下方"范围判定表"逐条检查，确认本次修改涉及哪些 md。

#### 规则 2：修改代码后 — 同步更新文档
准确反映代码变更到对应 md。

#### 规则 3：新增代码 — 必须有文档（零容忍）
新增任何 controller / service / 任务 / 表 / Redis Key / 错误码 / 中间件 → 必须有对应 md，并在 INDEX 登记。

#### 规则 4：修改 md 后 — 检查代码一致性
md 与代码冲突时按规则 7 处理。

#### 规则 5：更新 INDEX.md
任何新增 / 删除 / 重命名 md 后，必须同步 INDEX.md。

#### 规则 6：自检清单（每次交付前必须过一遍）
- [ ] 本次新增代码 → 是否有对应 md？
- [ ] 本次修改代码 → 是否更新关联 md？
- [ ] 本次新增/修改 md → 是否更新 INDEX.md？
- [ ] 本次删除 → 是否同步另一侧？

#### 规则 7：文档优先级的例外——签名冲突裁决
- 文档"未来设计"vs 代码不存在 → **以文档为准**
- 文档"既有契约"vs 代码已被引用 → **以代码为准**

#### 规则 8：BUILD_STATUS 先查
生成"已有文档"代码前，先查 BUILD_STATUS，区分 🟢 已实现 / ⬜ 待实现 / 🚫 暂不实现。

#### 规则 9：代码必须同步测试用例（硬约束）
任何 skill 生成代码后，**必须**同步新增/扩展 `test/cases/` 用例。漏写测试视为未交付。

#### 规则 10：并发安全约束（强制）

- 新增任何全局可变状态 → 必须在 `docs/architecture/concurrency_safety.md §2` 共享状态注册表登记
- 新增 `go func()` → 必须说明生命周期终止条件（context/signal/close），并在 §1.1 Goroutine 生命周期图追加节点
- 修改锁策略 → 必须更新 §3 锁粒度决策表；多锁场景同步 §3.2 加锁顺序约束
- 新增 `chan T` → 必须在 §4.1 Channel 清单登记，含 buffer / 满时策略 / 关闭条件
- 生成代码中使用 `map` / `slice` 并发写 → 必须声明保护机制（`sync.Map` 或外层 `RWMutex`）
- 命中 `concurrency_safety.md §6 危险操作清单` → 必须修正或在行内注解 `// aiweave:allow=<rule-id>` 抑制（每文件 ≤ 3 处）

#### 规则 11：事务与一致性约束（强制）

- 涉及多数据源写入的 Service 方法 → 必须在 `docs/service/transaction_design.md §2` 事务边界清单登记
- Service 方法伪码涉及事务 / Saga / 幂等 / 锁 / 不变量 → 必须使用 `[TXN-*]` / `[SAGA-STEP-N]` / `[IDEMPOTENT-CHECK]` / `[LOCK-*]` / `[INVARIANT-CHECK]` 统一标记
- 新增写入类接口未设计幂等 Key → AI 必须主动追问"是否需要幂等保证"
- 跨数据源写入未在 §2 登记 → AI 必须先补登记再生成代码

#### 规则 12：性能合约约束（强制）

- 在热路径（`performance_contract.md §2` 清单）中生成代码时：
  - 禁止使用 `reflect.*`
  - 禁止 `fmt.Sprintf` / `fmt.Errorf`（除错误 wrap 外）
  - 禁止在循环内分配大对象
  - 禁止同步调用外部 API
  - 禁止单条 MySQL 查询（必须批量或走缓存）
- 新增热路径方法 → 必须在 §2 清单登记 + 方法伪码加 `[HOT-PATH]` 标记
- 新增 channel → 容量必须在 `performance_contract.md §6.1` 登记，与 `concurrency_safety.md §4.1` 双向对齐
- 修改 SLA 数值 → 必须同步 `performance_contract.md §1` 与 `observability.md §5.2` 告警阈值

#### 规则 13：可观测性约束（强制）

- 禁止为业务数据创建 Metric（`{业务实体}_count` / `{核心数值字段}_sum` / `{状态字段}_distribution` 等 → 用日志 + 离线分析）
- 禁止在 `helpers/metrics.go` 之外注册 Metric
- 新增 Metric 前必须评估 cardinality（label 各维度枚举值相乘 ≤ 100 默认上限；突破需在 `ai_dev_guide.md §9` 登记）
- 禁止使用无界值作为 label（`{actor-id}` / IP / `request_id` / 动态路径）
- url label 必须使用路由模板，不可使用实际请求 URL
- 新增告警规则 → 同步 `observability.md §5.2`

#### 规则 14：Schema 演进约束（强制）

- 新增字段必须有默认值
- 字段类型变更必须兼容旧数据（如 INT → BIGINT 可；BIGINT → INT 不可）
- 删除字段必须先在 `database_design.md §8.1` 登记废弃，经过至少一个发版周期后再物理删除（DROP COLUMN）
- 修改字段语义 → 必须同步 `database_design.md §8.2` 字段语义变更记录

#### 规则 15：安全重构约束（强制）

- 不得一次性删除并重写超过 200 行的已有代码
- 重构必须拆分为 ≤ 5 个独立可回滚的 commit
- 每个 commit 必须通过完整测试
- 涉及"行为变更"的重构必须按 `mvp_rebuild_path.md §11.5.4` 灰度切流量（Shadow read → 双写 → 1%/10%/50%/100%）

#### 规则 16：跨服务合约约束（强制）

- 调用下游服务 / 新增 `api/` client → 必须在 `cross_service_contract.md §3` 下游合约登记（含 SLA / 超时 / 重试 / 熔断 / 降级）
- 新增对外路由被外部接入 → 必须在 `cross_service_contract.md §2` 上游合约登记
- 修改对外接口签名 → 必须按 `cross_service_contract.md §5` 评估是否需要升主版本号
- 下游调用必须显式设超时（禁止 0 或缺省）；本服务对下游设的超时必须 < 上游对本服务的超时（防级联）
- 禁止 `_ = downstream.Call()` 静默吞下游错误；每条失败分支必须在 `cross_service_contract.md §4` 故障传播矩阵覆盖

#### 规则 17：IO 聚合约束（强制 / 两条 IO 铁律）

- **铁律一（禁止 N+1）**：禁止在循环 / 递归内对同类资源逐条查询（DB / 缓存 / RPC / 外部 API）→ 必须循环外收集 key + 一次批量读 + `map` 装配
- **铁律二（禁止独立串行编排）**：多个**互不依赖**的 IO 禁止逐个串行 await → 必须聚合器或扇出并行；仅当后查询入参真实依赖前查询结果时串行才合法
- 多次跨实例 / 跨数据源读 → 必须走聚合器（按实例分组批量 + 并行回源 + 单飞合并 + 异步写回）
- 聚合器 / single-flight 的共享结果指针 **只读**，禁止就地改字段（并发写竞争）→ 需改先深拷贝
- 循环内单条写 → 必须批量 / pipeline
- 新增"集合访问资源 / 多依赖编排"方法 → 必须在 `io_contract.md §1` 登记往返预算、`§3/§4` 登记原语，伪码加 `[BATCH]` / `[PARALLEL]` 标记
- 拿不准是否构成 N+1 / 伪串行 → 主动报告并建议聚合方案，不擅自串行

#### 规则 18：配置中心与凭据加密约束（强制）

- 配置中心 / 资源连接凭据（账号 / 密码）→ 必须密文存放（对称加密，如 AES-256-GCM），**禁止明文**落盘 / 入库
- 加密密钥 → 必须由应用层 / 环境注入，**禁止硬编码、禁止入 git**；框架库不内置密钥
- 环境相关配置以**配置中心为权威源**，本地文件是启动期落盘副本；按 `config.md §5` 权威源矩阵归位，不混放业务 / 环境 / 凭据
- 新增配置中心拉取项 → 登记 `config.md §7.3` 拉取映射；拉取失败 / 空内容 → 必须 **fail-fast**（禁止静默空/旧配置启动）
- 接入新配置中心 → 实现 `config.md §7.1` Client 接口，网络调用只在实现层
- 出现明文凭据 / 硬编码密钥 → 主动报告，拒绝提交

### 危险模式清单

> AI 生成代码前/后必查。命中以下模式 → 停下来确认或修正。
>
> **本清单的 rule-id 索引 + 完整 grep 锚**详见 `docs/architecture/ai_dev_guide.md §10`；本节仅列摘要。
>
> **grep 锚定位为"信号级"非"判定级"**：命中标 🟡 待复核，最终判定权在人工 reviewer。误报通过 `// aiweave:allow=<rule-id>` 行内注解抑制，每个文件不超过 3 处。

| 危险模式 | 风险 | 正确做法 | Rule-id |
|---------|------|---------|---------|
| 锁内做网络 IO（HTTP / DB / Redis） | 锁持有时间不可控 | 先获取数据再加锁 | `R-CONC-LOCK-IO` |
| 锁内打日志 | 同上 + 写盘抖动 | 释放锁后再打日志 | `R-CONC-LOCK-LOG` |
| `map` / `slice` 并发写 | race condition | `sync.Map` 或 `RWMutex` | `R-CONC-MAP-RACE` |
| 关闭已关闭的 channel | panic | 单一所有者 close | `R-CONC-DOUBLE-CLOSE` |
| 向已关闭的 channel 发送 | panic | select + done channel | `R-CONC-SEND-CLOSED` |
| `defer` 在 for 循环内 | 资源延迟释放，可能 OOM | 抽子函数或显式 Close | `R-RESOURCE-DEFER-LOOP` |
| 忽略 `ctx.Done()` 的常驻 goroutine | goroutine 泄漏 | select case `<-ctx.Done()` | `R-CONC-GOROUTINE-LEAK` |
| 写入类接口无幂等 Key | 重试重复写 | SETNX + EXPIRE 或唯一索引 | `R-TXN-NO-IDEMPOTENT` |
| 跨数据源写入未登记事务边界 | 失败补偿无据 | 先补 `transaction_design.md §2` | `R-TXN-CROSS-SOURCE` |
| **【】for 循环内 DB 查询** | N+1 问题 | 批量查询 + map | `R-PERF-LOOP-DB-QUERY` |
| **for 循环内大对象分配** | GC 压力 | 循环前预分配 + 复用 | `R-PERF-LOOP-ALLOC` |
| **热路径用 `reflect.*`** | 反射性能开销 | 显式类型断言 / 代码生成 | `R-PERF-HOT-REFLECT` |
| **热路径用 `fmt.Sprintf`** | 反射 + 内存分配 | `strconv.*` 或 buffer | `R-PERF-HOT-FMT` |
| **全表 `COUNT(*)`** | 慢查询 | 维护计数器 / 限定 WHERE | `R-PERF-FULL-COUNT` |
| **`time.Sleep` 做同步** | 不可靠 + 浪费 CPU | channel / WaitGroup | `R-RESOURCE-SLEEP-SYNC` |
| **Redis 大 Key（> 1MB）** | 网络阻塞 + 慢查询 | 拆分 Hash / 分片 | `R-CACHE-LARGE-KEY` |
| **下游调用未显式设超时** | 上游超时被穿透 | 显式 `WithTimeout({T-ms})` | `R-XSVC-NO-TIMEOUT` |
| **本服务对下游超时 ≥ 上游对本服务超时** | 级联阻塞 | 按 `performance_contract.md §6.2` 协调 | `R-XSVC-TIMEOUT-CASCADE` |
| **`_ = downstream.Call()` 静默吞下游错误** | 故障不可见 | 显式处理 / 上报 | `R-XSVC-SILENT-SWALLOW` |
| **`api/` 新增 client 未登记 cross_service_contract §3** | 下游依赖失控 | 先补 §3 再合并 | `R-XSVC-UNREGISTERED-CLIENT` |
| **代码失败分支未在 transaction_design.md §6 失败路径全景图覆盖** | 漏路径无据可查 | 先补 §6 再合并 | `R-FAIL-PATH-UNDOC` |
| **失败分支无对应测试用例** | 静默回归 | 按 `testing_design.md §5.5` 高级用例 3 类补齐 | `R-FAIL-PATH-NO-TEST` |
| **伪码 `[INVARIANT-CHECK]` 标记与代码实现脱节** | 不变量声明虚化 | 修正标记或修正代码 | `R-INVARIANT-MARK-MISMATCH` |
| **【铁律一】循环内单条查询（DB/缓存/RPC）** | N+1，IO 放大 O(N) | 批量读 + map 装配 | `R-IO-N-PLUS-1` |
| **【铁律二】独立读串行编排（无数据依赖）** | RTT 累加，本可并行 | 聚合器 / 扇出并行 | `R-IO-SERIAL-ORCH` |
| **循环内单条写** | 写放大 | 批量写 / pipeline | `R-IO-LOOP-WRITE` |
| **多次跨实例读未走聚合器** | 往返失控 | 按实例分组批量 | `R-IO-NO-AGGREGATOR` |
| **就地修改 single-flight / Slot 共享指针字段** | 并发写竞争 | 只读；改前深拷贝 | `R-IO-SHARED-MUTATE` |
| **配置中心 / 资源凭据明文落盘或入库** | 凭据泄漏 | 对称加密密文存放 | `R-CONF-PLAINTEXT-CRED` |
| **加密密钥 / 密码硬编码在代码** | 密钥泄漏 | 应用层 / 环境注入 | `R-CONF-HARDCODE-SECRET` |
| **含密钥 / 明文凭据的文件入 git** | 仓库泄密 | `.gitignore` + 环境注入 | `R-CONF-SECRET-COMMIT` |
| **配置中心拉取失败未 fail-fast** | 空/旧配置静默启动 | 拉取失败即终止进程 | `R-CONF-NO-FAILFAST` |

### 范围判定表

| 变更的代码路径 / 迹象 | 检查的文档 |
|--------------|----------|
| `main.go`、`helpers/init.go` | `docs/architecture/overview.md` |
| `go.mod`、`go.sum` | `docs/architecture/go_module.md` |
| `helpers/`（资源初始化） | `docs/architecture/infrastructure.md` |
| `conf/` | `docs/architecture/config.md` |
| `router/` | `docs/architecture/routing.md` |
| `middleware/` | `docs/architecture/middleware.md` |
| `components/error.go` / `service_error.go` | `docs/architecture/status_codes.md` |
| `components/constants.go` | `docs/architecture/constants.md` |
| `api/` | `docs/architecture/overview.md`（外部 API 部分） |
| `models/` | `docs/schema/database_design.md` |
| `service/{module}/` | `docs/service/{module}_service.md` + `service_design.md` |
| `service/{audit-module}/` | `docs/architecture/audit_log.md` |
| `controllers/http/{audience}/` | `docs/api/{audience}_interfaces.md` |
| `controllers/command/`、`router/command.go` | `docs/service/scheduled_tasks_design.md` |
| `controllers/mq/`、`router/mq.go` | `docs/service/scheduled_tasks_design.md` §3 |
| `helpers/rediscache/` | `docs/cache/cache_design.md` + `architecture/helpers_api.md` |
| `helpers/circuit_breaker.go` | `docs/circuit_breaker/circuit_breaker_design.md` |
| `helpers/`（工具函数、全局变量） | `docs/architecture/helpers_api.md` |
| `pkg/**` | `docs/architecture/pkg_api.md` |
| 日志相关 | `docs/architecture/logging.md` |
| 实现进度 | `docs/BUILD_STATUS.md` |
| `test/framework/` | `docs/testing/testing_design.md` |
| `test/cases/`、`test/e2e/`、`test/perf/` | `docs/testing/testing_design.md` + 对应 `docs/api/*.md` |
| **【】`sync.Mutex` / `sync.RWMutex` / `sync.Map` / `atomic.*` 新增** | `docs/architecture/concurrency_safety.md` §2 共享状态注册表 |
| **`go func()` / `errgroup.Go` / `chan T` / `make(chan T, N)` 新增** | `docs/architecture/concurrency_safety.md` §1.1 / §4.1 |
| **`db.Transaction()` / `BeginTx` / `tx.Commit` / Saga 步骤标记新增** | `docs/service/transaction_design.md` §2 / §3 / §6；同步 `docs/service/{module}_service.md` §4.N.8 |
| **SETNX / 唯一索引 / 幂等校验类代码新增** | `docs/service/transaction_design.md` §4 幂等性设计 |
| **业务规则校验代码（`if {amount-field} < 0` 等）新增 / 修改** | `docs/service/{module}_service.md` §7 领域不变量 + 伪码 `[INVARIANT-CHECK]` 标记 |
| **约束总清单 / 约束突破登记 / grep 锚 rule-id 索引** | `docs/architecture/ai_dev_guide.md` §8 / §9 / §10 |
| **【】`helpers/metrics.go` 新增 `prometheus.MustRegister` / `NewCounterVec`** | `docs/architecture/observability.md` §2 Metrics 采集范围 + §3 Cardinality 评估 |
| **新增告警规则（YAML / Grafana）** | `docs/architecture/observability.md` §5.2 基础告警规则 |
| **新增热路径方法 / 修改热路径文件** | `docs/architecture/performance_contract.md` §2 热路径清单 + 伪码 `[HOT-PATH]` 标记 |
| **新增 `sync.Pool` / 显式预分配 / 新增 `*_bench_test.go`** | `docs/architecture/performance_contract.md` §3 内存预算 + §7.1 基准测试 |
| **修改 SLA 数值（路由 / 配置）** | `docs/architecture/performance_contract.md` §1 全局性能目标 |
| **GORM struct 字段加 `column:"-"` / DROP COLUMN / 字段类型变更** | `docs/schema/database_design.md` §8 字段演进历史 |
| **提交说明含 `refactor:` 前缀且改动 ≥ 100 行** | `docs/architecture/mvp_rebuild_path.md` §11 安全重构方法论 + §11.5.4 灰度切流量 |
| **`api/` 目录新增 client / `pkg/` 引入新 RPC 包** | `docs/architecture/cross_service_contract.md` §3 下游合约 |
| **新增 server 端路由被外部调用方接入 / 修改对外接口字段** | `docs/architecture/cross_service_contract.md` §2 上游合约 + §5 接口版本管理 |
| **【铁律】循环内单条查询/写/RPC、独立读串行编排、聚合器/扇出并行/批量读新增** | `docs/architecture/io_contract.md` §2 铁律豁免 + §3 聚合器 + §4 原语 + §1 往返预算 |
| **`helpers/{aggregator}/` 聚合器 / `single-flight` / worker pool 新增** | `docs/architecture/io_contract.md` §3 / §4 / §6（与 concurrency_safety.md §1.1 对齐） |
| **`conf/` 凭据字段 / `Encrypt`·`Decrypt` / 配置中心 `Client`·`Sync` / 拉取映射新增** | `docs/architecture/config.md` §6-§9（配置上云 / 凭据加密，详见 docs-spec/26） |

### 输出格式

每次回复末尾必须包含文档同步状态：

- 有变更：「文档同步：更新了 `docs/xxx.md`、新建了 `docs/yyy.md`」
- 无变更：「文档同步：本次修改不涉及文档变更」

### 文档目录结构

```
docs/
├── architecture/        # 架构（总纲、配置、基础设施、路由、中间件、状态码、helpers/pkg API、日志）
├── schema/              # 数据模型（DDL、分库、Redis Key 注册、GORM 规范）
├── cache/               # 缓存设计（Redis-first、Hash 字段映射、Flush、限流）
├── circuit_breaker/     # 熔断器设计
├── service/             # Service 层设计 + 定时任务 + MQ 消费者
├── api/                 # 接口规格（每类调用方一份）
├── testing/             # 测试框架设计
└── INDEX.md             # 文档索引
```
