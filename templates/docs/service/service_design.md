# Service 层设计 — 总览

> 规范来源：`aiweave/docs-spec/09_service_design_spec.md`

## 1. 设计原则

### 1.1 总则
- Service 层是唯一业务逻辑层
- Service 之间可互相调用，但避免循环依赖
- {核心 service}（如 auth）直接读 Redis
- 统一错误处理（双类型）

### 1.2 数据访问规则

| 数据源 | 使用场景 | 访问方式 |
|--------|---------|---------|
| Redis | 业务校验 / 状态变更 / 限流 | `helpers.{Project}CacheClient` |
| MySQL core | ... | `helpers.MysqlClientCore` |
| MySQL trade | ... | `helpers.MysqlClientTrade` |
| MySQL log | ... | `helpers.MysqlClientLog` |
| MySQL {example_log} | ... | `helpers.MysqlClientCallLog` |
| Kafka | ... | `helpers.{Topic}PubClient` |
| 本地缓存 | ... | `helpers.{LocalCache1}` 等 |

## 2. 统一错误处理

### 2.1 两套错误类型
| Go 类型 | 定义文件 | 使用接口 | 返回格式 |
|---------|---------|---------|----------|
| `*ServiceError` | ... | ... | ... |
| `base.Error` | ... | ... | ... |

**核心规则**：
- {模块 X} 全部方法 → `*ServiceError`
- 其他 → `base.Error`

### 2.2 ServiceError 定义
### 2.3 base.Error 定义
### 2.4 Controller 层处理模式

## 3. Service 模块总览

```
service/
├── {module_1}/
├── {module_2}/
└── ...
```

### 依赖关系图

```
{ASCII DAG}
```

**调用规则**：
- {module_1} 不调用任何 service
- ...

## 4. 模块详细设计导航

| 模块 | 文件 | 方法数 | 说明 |
|------|------|--------|------|
| {module_1} | [{module_1}_service.md](...) | N | ... |

## 5. 审计日志辅助函数（如有）

### 5.1 insertOperationLog
### 5.2 全部 Admin POST 端点的审计日志调用

## 6. 定时任务 Service 方法

| 任务 | Service 方法 | 并行策略 | 说明 |
|------|-------------|---------|------|
| ... | ... | ... | ... |

## 7. Service 间调用矩阵

| 调用方 → 被调用方 | 场景 |
|-------------------|------|
| ... | ... |

**禁止的调用方向**：
- 循环依赖
- service 调 controller
