# AIWeave 索引

## 0. AIWeave 采纳进度

> 落地工程把本表复制到 `docs/INDEX.md §0`，按工程实际启用情况标记每个规范的状态。
>
> AIWeave 骨架本身的所有 27 篇 docs-spec + 18 个 Skill 都已就位；具体哪些规范在某个工程内强制生效，由该工程自行决定。本表是工程级开关。

### 0.1 规范启用状态

| 规范 | 启用状态 | 启用日期 | 备注 |
| --- | --- | --- | --- |
| `docs-spec/00-19` 基础规范 | 🟢 启用 | {YYYY-MM-DD} | — |
| `docs-spec/20_concurrency_safety` 并发安全 | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/21_distributed_transaction` 分布式事务 | 🟢 / 🟡 / ⬜ / 🚫 | — | 仅有跨数据源时启用 |
| `docs-spec/22_performance_contract` 性能合约 | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/23_observability` 可观测性（Metrics） | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/24_cross_service_contract` 跨服务合约 | 🟢 / 🟡 / ⬜ / 🚫 | — | 单体服务可标 🚫 |
| `docs-spec/25_io_aggregation` IO 极致与聚合并行 | 🟢 / 🟡 / ⬜ / 🚫 | — | 强烈推荐（两条 IO 铁律） |
| `docs-spec/26_config_center` 配置中心与凭据加密 | 🟢 / 🟡 / ⬜ / 🚫 | — | 配置上云时启用 |
| Hooks 机制（L0 自动化防御，skills-spec/02 §4） | 🟢 / 🟡 / ⬜ | — | 强烈推荐（团队共享 settings.json 强制项） |
| `templates/docs/architecture/runtime_profile.md` 运行时基线 | 🟢 / ⬜ | — | 涉及人工运行时数据 |
| `OPERATIONS.md` §11 演进效果度量季度复盘 | 🟢 / ⬜ | — | 需工程实际运行 2 季度建立基线后启用 |

### 0.2 采纳建议（三档）

| 工程复杂度 | 推荐启用集 |
| --- | --- |
| **简单 CRUD / 内部工具服务** | 基础规范 + 20 §2 共享状态注册表 + 09 §7 领域不变量 + CLAUDE.md 危险模式清单 |
| **生产服务常态（推荐起点）** | 基础规范 + 20 + 21 + 22 §2 热路径 + 23 + 25 IO 铁律 + Hooks（L0）+ 18 §11 安全重构 + 审计 Skill `concurrency-review` / `io-review` / `domain-invariant-check` |
| **高并发 / 分布式核心** | 全部启用（含 24 + 25 + 26 + runtime_profile + Hooks + 全部审计 Skill + §11 演进效果度量） |

### 0.3 状态语义

- 🟢 启用：规范本体已就位，CLAUDE.md / OPERATIONS / 范围判定表已联动，AI 写代码时强制遵守
- 🟡 部分：规范本体就位但部分子节未联动（如 grep 锚已定义但审计 Skill 未实现）
- ⬜ 未启用：规范本体未引入，AI 不应触发对应约束
- 🚫 不启用：本工程明确不需要

---

## 顶层入口

| 文档 | 说明 |
|------|------|
| [README.md](README.md) | 规范的定位、适用范围、目录全景、启动新项目的 4 阶段路径 |
| [PRINCIPLES.md](PRINCIPLES.md) | 核心原则：双向同步、文档优先、签名冲突裁决、BUILD_STATUS 先查、暂不实现模块、测试纪律 |
| [OPERATIONS.md](OPERATIONS.md) | 操作手册：8 类工作流（设计先行 / 新增 API / 新增表 / 新增任务 / 新增 MQ / 重构 / 增量同步 / 定期审计）+ 3 类自检清单（交付前 / 代码变更 / 文档变更）+ 增量同步专属清单 |

## docs-spec/ —— `docs/` 目录的规范

编号 00-26，覆盖一个 Go 后端工程所需的全部设计视角，并支持"建设模式 + 增量同步模式"两种使用方式。

| 序号 | 规范 | 对应 docs/ 路径 | 说明 |
|------|------|---------------|------|
| 00 | [docs-spec/00_directory_layout.md](docs-spec/00_directory_layout.md) | docs/* | 标准目录结构（哪些子目录、命名规则、新增目录的判定） |
| 01 | [docs-spec/01_index_md_spec.md](docs-spec/01_index_md_spec.md) | docs/INDEX.md | 索引文件写法（每篇 md 的登记格式、描述精度、维护规则） |
| 02 | [docs-spec/02_build_status_md_spec.md](docs-spec/02_build_status_md_spec.md) | docs/BUILD_STATUS.md | 实现进度文件 + 暂不实现模块（§0）模板 + 状态图例 + 维护流程 |
| 03 | [docs-spec/03_claude_md_spec.md](docs-spec/03_claude_md_spec.md) | CLAUDE.md（项目根） | AI 总入口：项目概述 + 分层架构 + 开发规范 + 文档同步纪律 + 范围判定表 |
| 04 | [docs-spec/04_architecture_overview_spec.md](docs-spec/04_architecture_overview_spec.md) | docs/architecture/overview.md 等 | 架构总纲及衍生文档（config / infrastructure / routing / middleware / auth_flow / audit_log / render_functions / go_module / ai_dev_guide） |
| 05 | [docs-spec/05_schema_design_spec.md](docs-spec/05_schema_design_spec.md) | docs/schema/database_design.md | 数据库 DDL + 分库设计 + Redis Key 注册表 + GORM Model 规范 |
| 06 | [docs-spec/06_cache_design_spec.md](docs-spec/06_cache_design_spec.md) | docs/cache/cache_design.md + cache_helpers.md | KV 缓存设计：Redis-first / 三级缓存 / Key 模式 / Hash 字段映射 / Flush 机制 / 限流 |
| 07 | [docs-spec/07_api_interfaces_spec.md](docs-spec/07_api_interfaces_spec.md) | docs/api/*.md | API 接口详细设计与实现：请求/响应 JSON / Go struct / 校验规则 / Controller 伪代码 |
| 08 | [docs-spec/08_api_flow_spec.md](docs-spec/08_api_flow_spec.md) | docs/architecture/{auth,billing,...}_flow.md | API 逻辑描述与流程串联：跨接口/跨模块的业务链路 |
| 09 | [docs-spec/09_service_design_spec.md](docs-spec/09_service_design_spec.md) | docs/service/*.md | Service 层方法签名 + 步骤级伪代码 + 数据访问 + 错误类型选择 |
| 10 | [docs-spec/10_scheduled_tasks_spec.md](docs-spec/10_scheduled_tasks_spec.md) | docs/service/scheduled_tasks_design.md | 定时任务（cobra 外部调度 / cycle 内部协程 / crontab 定点）+ 并行策略 + 异常容错 |
| 11 | [docs-spec/11_mq_consumer_spec.md](docs-spec/11_mq_consumer_spec.md) | docs/service/scheduled_tasks_design.md §MQ | MQ 消费者设计：攒批 / 落库 / 死信队列 / offset 管理 |
| 12 | [docs-spec/12_circuit_breaker_spec.md](docs-spec/12_circuit_breaker_spec.md) | docs/circuit_breaker/circuit_breaker_design.md | 熔断 / 限流 / 降级：算法选型 / 差异化参数 / GORM Plugin / 接入方式 |
| 13 | [docs-spec/13_logging_spec.md](docs-spec/13_logging_spec.md) | docs/architecture/logging.md | 结构化日志：API 风格 / 级别判定 / 字段约定 / 敏感数据 |
| 14 | [docs-spec/14_status_codes_spec.md](docs-spec/14_status_codes_spec.md) | docs/architecture/status_codes.md | 错误码 / 状态码体系：双错误类型 / 范围划分 / 业务状态枚举 |
| 15 | [docs-spec/15_helpers_pkg_api_spec.md](docs-spec/15_helpers_pkg_api_spec.md) | docs/architecture/helpers_api.md + pkg_api.md | 工具函数 + 框架 API 契约（一切签名的真相源） |
| 16 | [docs-spec/16_constants_spec.md](docs-spec/16_constants_spec.md) | docs/architecture/constants.md | 业务常量 / 魔法数字唯一真相源 |
| 17 | [docs-spec/17_testing_design_spec.md](docs-spec/17_testing_design_spec.md) | docs/testing/testing_design.md | 测试框架 + 测试纪律 + 4 类用例 + 与 Stage 协同 |
| 18 | [docs-spec/18_mvp_rebuild_path_spec.md](docs-spec/18_mvp_rebuild_path_spec.md) | docs/architecture/mvp_rebuild_path.md | 分阶段构建路径（通用方法论）：覆盖 forward / rebuild / refactor 三种场景 |
| 19 | [docs-spec/19_incremental_sync_spec.md](docs-spec/19_incremental_sync_spec.md) | （流程文档，不直接对应一篇工程 md，但塑造工程演进的 SOP） | **增量需求同步规范**：开发完需求后如何把代码改动反向同步到全链路 docs/，B1 场景的核心规则。v1.2 范围判定表扩展 5 行约束类映射 |
| 20 | [docs-spec/20_concurrency_safety_spec.md](docs-spec/20_concurrency_safety_spec.md) | docs/architecture/concurrency_safety.md | **并发安全与运行时约束规范**：共享状态注册表 / 锁策略 / Channel / 资源生命周期 / 危险操作清单 / 并发测试策略 + B1 反向同步规则 |
| 21 | [docs-spec/21_distributed_transaction_spec.md](docs-spec/21_distributed_transaction_spec.md) | docs/service/transaction_design.md | **分布式事务与补偿设计规范**：事务模型选型 / 事务边界清单 / 补偿逻辑 / 幂等性 / 一致性窗口 / 失败路径全景图 / 伪码标记语法 + B1 反向同步规则 |
| 22 | [docs-spec/22_performance_contract_spec.md](docs-spec/22_performance_contract_spec.md) | docs/architecture/performance_contract.md | **性能合约与热路径规范**：全局 SLA / 热路径标注 / 内存预算 / 数据访问约束 / 背压策略 / 性能回归测试 / 与 12 边界划分 + B1 反向同步规则 |
| 23 | [docs-spec/23_observability_spec.md](docs-spec/23_observability_spec.md) | docs/architecture/observability.md | **可观测性规范（Metrics 限服务级）**：仅服务级基础指标 / Cardinality 控制 / Label 禁止清单 / 告警规则 / 与 13 边界划分 + B1 反向同步规则 |
| 24 | [docs-spec/24_cross_service_contract_spec.md](docs-spec/24_cross_service_contract_spec.md) | docs/architecture/cross_service_contract.md | **跨服务合约规范**：上下游依赖图 / 上游合约 / 下游合约 / 故障传播矩阵 / 接口版本管理 + B1 反向同步规则 |
| 25 | [docs-spec/25_io_aggregation_spec.md](docs-spec/25_io_aggregation_spec.md) | docs/architecture/io_contract.md | **IO 极致与聚合并行规范**：两条 IO 铁律（禁止 N+1 / 禁止独立串行编排）/ 聚合器模式（收集→批量读→并行回源→单飞→异步写回）/ 并行编排原语 / 数据访问聚合约束 / IO 往返预算 / IO 回归测试 + B1 反向同步规则 |
| 26 | [docs-spec/26_config_center_spec.md](docs-spec/26_config_center_spec.md) | docs/architecture/config.md §6-§9 | **配置中心与凭据加密规范**：配置上云权威源模型 / 配置中心 Client 抽象与拉取落盘编排 / 凭据对称加密（密钥应用层持有不入库）/ 启动期 fail-fast 加载时序（治理 config.md §6-§9，与 04 §2 边界划分） |

## skills-spec/ —— `.claude/skills/` 的规范

| 文档 | 说明 |
|------|------|
| [skills-spec/00_skill_overview.md](skills-spec/00_skill_overview.md) | Skill 体系总览：4 类分组（创建 9 / 维护 2 / 审计 6 / 终极 1）、命名约定、调用顺序、与 Stage 的协同 |
| [skills-spec/01_skill_authoring_guide.md](skills-spec/01_skill_authoring_guide.md) | Skill 编写指南：frontmatter / 步骤化结构 / 文档同步 / 测试同步 / 验证 |
| [skills-spec/02_settings_local_json_spec.md](skills-spec/02_settings_local_json_spec.md) | settings.local.json 规范：permissions / hooks / 推荐配置 |

`templates/skills/{name}/SKILL.md` 是 **18 个** Skill 的可执行骨架（每个 SKILL.md 同时充当规范与可复制的执行手册；公共章节真相源在 [skills-spec/01_skill_authoring_guide.md](skills-spec/01_skill_authoring_guide.md) §A-§E）：

| Skill | 骨架文件 |
|-------|----------|
| new-model | [templates/skills/new-model/SKILL.md](templates/skills/new-model/SKILL.md) |
| new-service | [templates/skills/new-service/SKILL.md](templates/skills/new-service/SKILL.md) |
| new-controller | [templates/skills/new-controller/SKILL.md](templates/skills/new-controller/SKILL.md) |
| new-middleware | [templates/skills/new-middleware/SKILL.md](templates/skills/new-middleware/SKILL.md) |
| new-router | [templates/skills/new-router/SKILL.md](templates/skills/new-router/SKILL.md) |
| new-scheduled-task | [templates/skills/new-scheduled-task/SKILL.md](templates/skills/new-scheduled-task/SKILL.md) |
| new-mq-consumer | [templates/skills/new-mq-consumer/SKILL.md](templates/skills/new-mq-consumer/SKILL.md) |
| new-test | [templates/skills/new-test/SKILL.md](templates/skills/new-test/SKILL.md) |
| doc-sync-check | [templates/skills/doc-sync-check/SKILL.md](templates/skills/doc-sync-check/SKILL.md) |
| update-index | [templates/skills/update-index/SKILL.md](templates/skills/update-index/SKILL.md) |
| rebuild-from-docs | [templates/skills/rebuild-from-docs/SKILL.md](templates/skills/rebuild-from-docs/SKILL.md) |
| sync-feature-to-docs | [templates/skills/sync-feature-to-docs/SKILL.md](templates/skills/sync-feature-to-docs/SKILL.md) — **B1 增量同步核心 Skill** |
| concurrency-review | [templates/skills/concurrency-review/SKILL.md](templates/skills/concurrency-review/SKILL.md) — 审计代码 ↔ 并发安全约束一致性 |
| performance-review | [templates/skills/performance-review/SKILL.md](templates/skills/performance-review/SKILL.md) — 审计代码 ↔ 性能合约约束一致性 |
| io-review | [templates/skills/io-review/SKILL.md](templates/skills/io-review/SKILL.md) — 审计代码 ↔ IO 铁律（N+1 / 串行编排 / 聚合并行）一致性 |
| new-saga-step | [templates/skills/new-saga-step/SKILL.md](templates/skills/new-saga-step/SKILL.md) — 生成 Saga 步骤代码 + 补偿 + 幂等 Key |
| domain-invariant-check | [templates/skills/domain-invariant-check/SKILL.md](templates/skills/domain-invariant-check/SKILL.md) — 审计代码 ↔ 领域不变量约束一致性 |
| failure-path-review | [templates/skills/failure-path-review/SKILL.md](templates/skills/failure-path-review/SKILL.md) — 审计失败路径文档 / 测试覆盖完整性 |

## templates/ —— 可直接复制的骨架

| 路径 | 说明 |
|------|------|
| [templates/CLAUDE.md](templates/CLAUDE.md) | 项目根 CLAUDE.md 骨架，复制到新项目根目录后填空即可 |
| [templates/docs/](templates/docs/) | 完整 docs/ 骨架（每篇 md 含目录结构 + 待填字段） |
| [templates/skills/](templates/skills/) | 18 个 .claude/skills/{name}/SKILL.md 骨架（4 类分组：创建 9 / 维护 2 / 审计 6 / 终极 1） |

---

## 变更日志

记录本规范的演进，每条变更注明日期 + 触发的实践事件。

| 日期 | 变更 | 触发事件 |
|------|------|---------|
| 2026-06-05 | v1.3 演进：IO 极致 + Hooks 机制 + 配置上云 | 落地"IO 极致"诉求与 AI 工程化标准强化：<br>**新增规范**：docs-spec/25 IO 极致与聚合并行（两条 IO 铁律：禁止 N+1 / 禁止独立串行编排；聚合器模式：收集→批量读→并行回源→单飞→异步写回；并行编排原语；IO 往返预算；IO 回归测试）/ docs-spec/26 配置中心与凭据加密（配置上云权威源模型 / Client 抽象拉取落盘编排 / 对称加密密钥应用层持有不入库 / 启动 fail-fast 时序）<br>**新增 Skill**：io-review（审计 代码 ↔ IO 铁律，审计类 5→6）<br>**新增模板**：templates/docs/architecture/io_contract.md；config.md 扩展 §6-§9<br>**Hooks 升级为强制规范**：skills-spec/02 §4 两类强制 Hook 族（IO 铁律检查 + 代码↔文档双向同步）放团队共享 settings.json；新增 L0 自动化防御层<br>**机制完善**：PRINCIPLES §13 IO 铁律 + §14 Hooks 机制；CLAUDE.md 规则 17 IO 聚合 + 规则 18 配置凭据 + 危险模式 R-IO-* / R-CONF-* + 范围判定表 3 行 + Hooks 子节；OPERATIONS 5 层防御→6 层（+L0）+ B1 清单 18→21 项；04 §2 ↔ 26 边界划分<br>**体系升级**：docs-spec 25→27 篇（00-26）；Skill 17→18 个 |
| 2026-05-15 | 初版发布（v1.0） | 从真实工程实践抽象沉淀；19 篇 docs-spec + 12 个 Skill |
| 2026-05-25 | v1.2 演进完整落地 | 按 `tc-md/aiweave-evolution-proposal.md` v1.2 推进，完整覆盖 15 项后端复杂系统痛点：<br>**新增规范**：docs-spec/20 并发安全 / 21 分布式事务 / 22 性能合约 / 23 可观测性 / 24 跨服务合约<br>**新增 Skill**：concurrency-review / performance-review / new-saga-step / domain-invariant-check / failure-path-review<br>**结构增强**：09 §7 领域不变量 + §4.N.8 事务一致性 + §10.5 伪码标记统一语法；02 §11 约束清单状态轨道 + §12 运行时基线区域；05 §8 字段演进；17 §4.7-§4.9 故障注入/并发/性能测试 framework + §5.5 高级用例；18 §11 安全重构方法论 + §11.5.4 灰度切流量；19 范围判定表扩展<br>**机制完善**：CLAUDE.md 16 条规则 + 危险模式清单（含 grep 锚）；ai_dev_guide.md 约束总清单 + 约束突破登记表 + grep 锚 rule-id 索引；OPERATIONS 第三部分 18 项 + 附录"5 层防御 × W 工作流对照表"；INDEX.md §0 采纳进度<br>**边界划分**：12 ↔ 22 / 13 ↔ 23<br>**体系升级**：Skill 体系 12 → 17，3 类分组 → 4 类分组（创建 / 维护 / 审计 / 终极） |
