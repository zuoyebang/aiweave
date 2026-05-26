# 配置体系

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §2

## 1. 配置文件分类

- **mount/**：环境相关，配置中心动态修改
- **app/**：业务相关，随代码发布

## 2. 配置文件清单

| 文件 | 内容范围 | 何时修改 |
|------|---------|---------|
| `conf/mount/config.yaml` | 服务端口、日志、accessLog、pprof | 环境差异 |
| `conf/mount/api.yaml` | 外部 API 调用配置 | 上线新外部依赖 |
| `conf/mount/resource.yaml` | MySQL / Redis / Kafka / MQ | 资源迁移 |
| `conf/app/app.yaml` | 熔断器参数等业务参数 | 调优 |

## 3. 加载流程

```
helpers.PreInit() → conf.InitConf() → conf.RConf / conf.AConf 全局变量
```

## 4. 占位机制

敏感信息使用 `@@key` 格式占位，运行时由配置中心替换。

## 5. Go 结构体映射

| yaml 节点 | Go struct |
|----------|----------|
| resource.mysql | `TMysql` |
| resource.redis | `TRedis` |
| resource.kafkaPub | `map[string]TKafkaPub` |
| ... | ... |
