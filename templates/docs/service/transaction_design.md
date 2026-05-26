# 分布式事务与补偿设计

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/21_distributed_transaction_spec.md`](../../../docs-spec/21_distributed_transaction_spec.md)
>
> 本文档是 **AI 在生成涉及多数据源写入、跨服务调用、需要一致性保证的代码之前必须读的文档**。

---

## 1. 事务策略总览

### 1.1 本工程的事务模型选型

| 业务类别 | 事务模型 | 适用范围 |
|---------|---------|---------|
| {单数据源写入类} | 本地事务（GORM Transaction） | `{Module-A}` / `{Module-B}` |
| {跨数据源 / 强补偿类} | Saga（正向 + 补偿） | `{Module-C}` 的 `{state-change-method}` |
| {高吞吐异步类} | MQ 投递 + 最终一致 | `{flush-task}` / `{audit-stream}` |
| {跨服务远程调用类} | 本地消息表 + 重试 | 远程 `{external-api}` 调用 |

### 1.2 选型理由

| 维度 | 本地事务 | Saga | MQ 异步 |
|------|---------|------|--------|
| 一致性 | 强 | 最终 | 最终（窗口 {N}s） |
| 性能 | 单库写入快 | 多步骤序列化 | 异步，无写入路径阻塞 |
| 运维复杂度 | 低 | 高（需要补偿逻辑 + 监控） | 中（依赖 MQ 可用性） |

---

## 2. 事务边界清单（核心）

| 业务操作 | 事务范围 | 涉及数据源 | 一致性级别 | 失败策略 |
|---------|---------|-----------|-----------|---------|
| `{Module-A}.{create-method}` | 本地事务 | MySQL `{db-core}` | 强一致 | 回滚 |
| `{Module-B}.{state-change-method}` | Saga 3 步 | MySQL `{db-trade}` + Redis `{cluster}` | 最终一致（窗口 {N}s） | 补偿（详见 §3）|
| `{Module-C}.{flush-method}` | MQ 投递 | Kafka `{topic}` + MySQL `{db-log}` | 最终一致 | 重试 + 死信 |

> **维护规则**：每新增一个写入类 service 方法，本表必须新增一行；同步在 `docs/service/{module}_service.md §4.N.8 事务与一致性`引用本行。

---

## 3. 补偿逻辑设计

### 3.1 {Module-B}.{state-change-method} 的 Saga 步骤

| Step | 正向操作 | 补偿操作 | 触发条件 |
|------|---------|---------|---------|
| 1 | INSERT `{example_table_trade}` (状态=PENDING) | UPDATE 状态=CANCELLED | Step 2 / 3 失败 |
| 2 | DECR Redis `{ns}:{state-key}:%s` | INCR Redis `{ns}:{state-key}:%s` | Step 3 失败 |
| 3 | UPDATE `{example_table_trade}` (状态=DONE) | — | — |

### 3.2 补偿幂等性保证

- 补偿操作必须可重复执行而不产生副作用
- 通过 `{idempotent-key}` 去重；窗口 {N}min，存于 Redis `{ns}:compensation_lock:%s`
- 补偿失败 N 次 → 写入 `{audit-table}` 并告警，转人工

### 3.3 补偿超时

- Saga 启动后 {N}s 内未走到终态 → 自动触发补偿链
- 由 `{cleanup-task}` 定时扫描中间态记录

---

## 4. 幂等性设计

### 4.1 幂等 Key 生成规则

| 场景 | Key 模式 | 来源 |
|------|---------|------|
| {外部回调类} | `{ns}:idemp:{action}:%s` | 调用方在 header / body 内传 |
| {内部重试类} | `{ns}:idemp:{action}:%s` | service 内基于业务字段生成 |

### 4.2 去重窗口

| 场景 | 窗口 | 实现 |
|------|------|------|
| 短期回调去重 | {N}min | Redis SETNX + EXPIRE |
| 跨天去重 | 永久 | MySQL 唯一索引 |

### 4.3 幂等适用的接口清单

| 接口 | 幂等 Key | 窗口 |
|------|---------|------|
| `POST /{prefix}/{audience}/{module}/{action-1}` | `{ns}:idemp:{action-1}:%s` | 5min |
| `POST /{prefix}/{audience}/{module}/{action-2}` | `{ns}:idemp:{action-2}:%s` | 24h |

---

## 5. 最终一致性窗口

| 业务操作 | 一致性窗口承诺 | 窗口内读取策略 |
|---------|--------------|---------------|
| `{Module-B}.{state-change-method}` | {N}s 内 Redis 与 MySQL 一致 | 优先读 Redis（带版本号），跨窗口校对走 MySQL |
| `{Module-C}.{flush-method}` | {N}min 内 MySQL log 与 Kafka 一致 | 不对外暴露窗口内查询 |

---

## 6. 失败路径全景图

> 每个关键写入操作必须有 ASCII 失败路径图，覆盖 happy path + 所有失败分支 + 补偿路径。

```
                {Module}.{Method}
                       │
                       ▼
                 [checkParams]
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
         参数错误             参数 OK
                                 │
                                 ▼
                         [TXN-START]
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
         INSERT 失败       INSERT 成功         INSERT 唯一索引冲突
               │                 │                 │
               ▼                 ▼                 ▼
         [TXN-ROLLBACK]    [写 Redis]      [转更新或返回已存在]
         返回 ErrorDb            │
                                 ▼
                       ┌─────────┴─────────┐
                       ▼                   ▼
                   Redis 失败           Redis 成功
                       │                   │
                       ▼                   ▼
               ⚠️ 异步对账修正        [TXN-COMMIT]
               （不阻塞主路径）             │
                                           ▼
                                       返回 Resp
```

---

## 7. 与 Service 伪代码的关系

涉及一致性的步骤必须在 `docs/service/{module}_service.md §4.N.3 处理步骤` 伪码中使用统一标记（详见 [`aiweave/docs-spec/09 §10.5`](../../../docs-spec/09_service_design_spec.md)）：

| 标记 | 含义 |
|------|------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 本地事务边界 |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿步骤 |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验入口 |
| `[INVARIANT-CHECK: I-N]` | 此步必须保护领域不变量 |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE: {lock-name}]` | 加锁 / 释放 |

---

## 8. 维护流程

### 8.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 `BeginTx` / `db.Transaction(...)` | §2 新增一行；同步 `docs/service/{module}_service.md §4.N.8` |
| 新增 Saga 步骤标记 | §3 新增正向/补偿对照行；§6 失败路径全景图扩展分支 |
| 新增唯一索引 / SETNX 去重 | §4 新增 Key 生成规则与适用接口 |
| 修改一致性等级 | §1.1 变更日志 + §5 一致性窗口承诺更新 |

### 8.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条事务边界条目（§2 一行）对应 BUILD_STATUS.md §11 的"已设计 / 已启用"计数。新增事务边界 → BUILD_STATUS §11 对应类目"已设计条目数"+1。
