# {ProjectName} — 数据库设计

> 规范来源：`aiweave/docs-spec/05_schema_design_spec.md`

## 1. 数据库概览

### 设计哲学
- {例：Redis-first / MQ 异步落库 / 分库设计}

### 表清单

| #  | 表名 | 说明 | 数据量预估 | 所属库 |
|----|------|------|-----------|--------|
| 1  | `{table_1}` | ... | 万级 | db_xxx_core |
| 2  | `{table_2}` | ... | 十万级 | db_xxx_core |
| ... | ... | ... | ... | ... |

## 2. 分库设计

### 2.1 分库策略
按业务域 + 数据量级 + 读写模式划分。

### 2.2 各库清单与特点

```
┌──────────────────────────────┐
│  MySQL 实例 1                 │
│  db_xxx_core（核心库）        │
│                              │
│  table_1   万级               │
│  table_2   十万级             │
│                              │
│  特点：读多写少 / Redis 缓存   │
└──────────────────────────────┘
```

### 2.3 容量与性能预估

## 3. 表结构 DDL

### 3.1 {table_1}

**用途**：

**所属库**：db_xxx_core

**预估数据量**：万级

**写入特征**：

```sql
CREATE TABLE `{table_1}` (
    `id` bigint unsigned NOT NULL AUTO_INCREMENT,
    `field_1` varchar(32) NOT NULL DEFAULT '' COMMENT '字段说明',
    -- ...
    `deleted` tinyint NOT NULL DEFAULT '0' COMMENT '0=正常, 1=已删除',
    `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_field_1` (`field_1`),
    KEY `idx_field_x` (`field_x`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='表注释';
```

**字段说明**：
| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| ... | ... | ... | ... | ... |

**索引**：
| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| ... | ... | ... | ... |

**关联**：
- 上游：
- 下游：

**生命周期**：

**Redis 缓存**（如有）：
- Key 模式：
- 同步方式：

### 3.2 {table_2}

（同上结构）

## 4. Redis Key 设计

### 4.1 热数据缓存
| Key 模式 | 类型 | 说明 | 写入时机 |
|---------|------|------|---------|
| `{ns}:info:{entityId}` | Hash | ... | ... |

### 4.2 状态数据缓存

#### 4.2.1 `{state-1}:{entityId}:{actionId}` Hash 完整字段映射

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| ... | ... | ... | ... | ... | ... | ... |

### 4.3 频率限制
### 4.4 Flush 脏标记
### 4.5 {受限模块}数据（如有）
### 4.6 完整 Hash 字段映射汇总

## 5. 与 MySQL 的关系

| 数据类别 | Redis 角色 | MySQL 角色 | 同步机制 |
|----------|-----------|-----------|---------|
| ... | ... | ... | ... |

## 6. GORM Model 层设计规范

### 6.1 包命名（按库分包）
| 数据库 | Go 包名 | 目录 |
|--------|---------|------|
| db_xxx_core | `db_xxx_core` | `models/db_xxx_core/` |

### 6.2 编码约定
- NOT NULL → 值类型；可空 → 指针类型
- 软删除：`Deleted int8`（不用 gorm.DeletedAt）
- 敏感字段：`json:"-"`
- date 类型：用 string

### 6.3 类型映射
（按 docs-spec/05 §8.3）

### 6.4 GORM tag 规则
（按 docs-spec/05 §8.4）

### 6.5 动态分表（如有）

## 7. 索引设计

每张表的索引清单（详见 §3.X）。

## 8. 数据生命周期（如有过期/归档）
