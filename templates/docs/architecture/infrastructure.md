# 基础设施初始化

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §3

## 1. 资源清单

| 资源 | 实例数 | 用途 |
|------|--------|------|
| MySQL | {N} | {库划分} |
| Redis | {N} | {用途} |
| Kafka topic | {N} | {用途} |
| 本地缓存 | {N} | {用途} |

## 2. 初始化时序

```
helpers.PreInit()
   └── 配置加载、日志初始化

helpers.InitResource()
   ├── InitCircuitBreaker()    # 必须最先
   ├── InitRedis()
   ├── InitMysql()
   ├── InitGPool()
   ├── InitGCache()
   └── InitKafkaProducer()
```

## 3. 各资源初始化函数

| 函数 | 文件 | 职责 |
|------|------|------|
| `InitCircuitBreaker` | `helpers/circuit_breaker.go` | 注册 CB registry |
| `InitRedis` | `helpers/redis.go` | rediscache.Client 实例 |
| `InitMysql` | `helpers/mysql.go` | GORM Plugin 注入 CB |
| ... | ... | ... |

## 4. 全局客户端

详见 [`helpers_api.md`](helpers_api.md) §2。
