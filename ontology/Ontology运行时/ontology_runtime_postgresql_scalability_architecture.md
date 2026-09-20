# Ontology Runtime 大数据量下的 PostgreSQL 存储与分层架构建议

## 1. 结论

PostgreSQL 仍然适合存储 Ontology，但不应该把所有海量运行数据都放进同一个 PostgreSQL 实例。

关键是区分两类数据：

1. **Ontology 元数据**：Object Type、Property、Link Type、Action Type、Interface、权限、版本和发布记录等。
2. **Ontology Runtime 实例数据**：对象实例、对象关系、事件、属性历史、Action 执行结果和审计记录等。

对于第一类，PostgreSQL 非常合适；对于第二类，当数据规模很大时，应采用 **PostgreSQL + 分布式事务存储 + Iceberg/OLAP/搜索引擎** 的分层架构。

---

## 2. 哪些数据应该继续放在 PostgreSQL

### 2.1 Ontology 设计态元数据

适合存储：

- Object Type 定义
- Property 定义
- Link Type 定义
- Action Type 定义
- Interface 定义
- 字段约束
- 数据源映射
- 权限策略
- Ontology 版本
- 草稿、发布和回滚记录

这些数据即使平台规模很大，通常也远小于业务对象实例数据。

例如：

- 1 万个 Object Type
- 50 万个 Property
- 10 万个 Link Type
- 数百万条版本和变更记录

这类规模仍然适合 PostgreSQL。

设计态元数据还具有明显的关系约束和事务要求，例如：

- 删除 Property 前检查是否被 Action 引用；
- 发布版本时同时更新多个定义；
- Object Type、Link Type、Interface 之间保持引用一致性；
- 一个版本要么整体发布成功，要么整体失败。

这些场景正是 PostgreSQL 擅长的。

建议：

```text
ontology-control-plane
        ↓
PostgreSQL
```

### 2.2 当前运行态对象状态

例如：

- 设备当前状态
- 订单当前状态
- 客户当前信息
- 当前库存
- 当前合同状态
- 当前告警状态

这些数据通常需要：

- 单对象快速读取；
- Action 实时更新；
- 事务一致性；
- 唯一约束；
- 并发控制；
- 毫秒级点查询。

因此，当前状态仍然可以由 PostgreSQL 或分布式 PostgreSQL 保存。

```text
Action
  ↓
PostgreSQL
  ↓
返回最新对象状态
```

---

## 3. 什么时候 PostgreSQL 会开始不合适

不能只看“有多少 TB”，还要综合看以下因素：

| 因素 | 对 PostgreSQL 的影响 |
|---|---|
| 对象总量 | 影响表、索引、备份和恢复体积 |
| 当前活跃对象量 | 影响缓存命中率和查询延迟 |
| 每秒写入量 | 影响 WAL、锁、索引和磁盘吞吐 |
| 更新、删除比例 | 影响 MVCC、VACUUM 和表膨胀 |
| 历史保留时间 | 导致表持续增长 |
| 聚合查询 | 可能扫描大量对象和关系 |
| 多租户数量 | 影响隔离和分片策略 |
| 多跳关系查询 | 影响 Join 和递归查询成本 |

PostgreSQL 分区可以改善大表管理和查询裁剪，但分区本身不会自动把单机数据库变成水平分布式数据库。

PostgreSQL 使用 MVCC。执行 `UPDATE` 或 `DELETE` 后，旧版本不会立即从磁盘清除，需要依赖 VACUUM 回收。对于高频更新、长期保存历史的大表，这会带来表膨胀和维护压力。

以下情况不适合继续全部依赖单一 PostgreSQL：

- 对象实例达到数十亿级，并持续快速增长；
- 属性和关系每天产生大量新增、更新和删除；
- 要保存对象的全部历史版本；
- 经常进行全量聚合、趋势分析和复杂扫描；
- 日志、事件、传感器数据持续高吞吐写入；
- 数据规模达到几十 TB、几百 TB，甚至 PB；
- 同一个 PostgreSQL 同时承担事务、历史、搜索和分析查询。

---

## 4. 不要把 Ontology 元数据和业务对象数据混在一起

建议至少拆分为两个逻辑平面。

### 4.1 Ontology Control Plane

保存：

- Object Type
- Property
- Link Type
- Action Type
- Interface
- 数据映射
- 权限策略
- 版本和发布记录

推荐使用：

```text
PostgreSQL
```

### 4.2 Ontology Runtime Plane

保存：

- 对象实例
- 对象关系
- Action 执行结果
- 当前业务状态
- 历史变更
- 大规模事件

运行态数据应根据用途继续拆分。

即使业务实例达到几百亿条，也不意味着 Ontology 定义本身不能继续使用 PostgreSQL。

---

## 5. 推荐的海量数据架构

```text
                    Ontology Manager
                           │
                           ▼
               PostgreSQL：Ontology 元数据
                           │
                      编译 / 发布
                           ▼
应用、API、Agent ──────► Action Service
                           │
                           ▼
                PostgreSQL / 分布式 PostgreSQL
                  当前对象状态、事务数据
                           │
                     Outbox / CDC
                           ▼
                         Kafka
                ┌──────────┼──────────┐
                ▼          ▼          ▼
             Iceberg    ClickHouse   OpenSearch
           全量历史       实时分析      搜索索引
           对象快照       聚合查询      全文检索
           审计数据       指标计算      条件搜索
```

PostgreSQL 继续承担在线事务和当前状态；Kafka/CDC 把数据变化同步到分析、历史和搜索存储。

---

## 6. 各类存储分别负责什么

### 6.1 PostgreSQL：当前状态和事务

适合保存：

- `object_current`
- `link_current`
- `action_execution`
- `transaction_outbox`
- `runtime_constraint`

适合处理：

- 根据 Object ID 查询；
- Action 修改对象；
- 创建和删除关系；
- 权限校验；
- 唯一性约束；
- 乐观锁；
- 多表事务；
- 当前状态查询。

示例：

```sql
SELECT *
FROM equipment_current
WHERE tenant_id = 'tenant-001'
  AND object_id = 'equipment-1001';
```

### 6.2 Iceberg：完整历史和冷数据

适合保存：

- `object_change_history`
- `object_snapshots`
- `link_history`
- `action_audit_history`
- `source_dataset_history`
- `large_event_dataset`

示例：

```text
PostgreSQL：
设备 D1001 当前状态 = RUNNING

Iceberg：
2026-08-01  STOPPED
2026-08-02  MAINTENANCE
2026-08-03  RUNNING
2026-08-04  FAILED
2026-08-05  RUNNING
```

Iceberg 更适合保存海量历史、快照、审计和长期冷数据，不适合作为高频在线事务数据库。

### 6.3 ClickHouse：实时聚合和分析查询

适合保存或同步：

- 对象事实表
- 事件明细
- 指标数据
- 时序数据
- 预聚合结果
- 关系统计

适合查询：

- 过去 30 天每种设备的故障次数；
- 每个区域的订单总额；
- 不同对象状态的分布；
- 对象关系数量和变化趋势。

ClickHouse 不应替代 PostgreSQL 执行余额扣减、库存更新等强事务操作。

### 6.4 OpenSearch：对象搜索

适合保存可重建的搜索索引，例如：

- 对象名称
- 对象描述
- 标签
- 全文字段
- 搜索关键词
- 部分可筛选属性

例如：

```text
查找名称包含“上海”，
状态为运行中，
类型为生产设备，
最近 7 天发生过故障的对象。
```

OpenSearch 不应成为对象数据的权威数据源。索引应能根据 PostgreSQL、Kafka 或 Iceberg 中的数据重新构建。

---

## 7. PostgreSQL 本身可以如何扩展

### 7.1 第一阶段：单 PostgreSQL 优化

可以采用：

- 表分区；
- 复合索引；
- 覆盖索引；
- 读写分离；
- 只读副本；
- 连接池；
- 批量写入；
- 冷热数据分离；
- 定期归档历史；
- 合理配置 Autovacuum。

对象历史表可按时间分区：

```sql
CREATE TABLE object_change_event (
    tenant_id   UUID,
    object_type UUID,
    object_id   UUID,
    event_time  TIMESTAMPTZ,
    payload     JSONB
) PARTITION BY RANGE (event_time);
```

只有查询条件能够命中分区键时，分区裁剪才能明显减少扫描范围。

### 7.2 第二阶段：按租户或领域分库

例如：

```text
runtime_cluster_01：制造行业租户
runtime_cluster_02：金融行业租户
runtime_cluster_03：零售行业租户
```

或者：

```text
tenant_0001～tenant_1000 → shard 01
tenant_1001～tenant_2000 → shard 02
```

Ontology Control Plane 维护路由：

```text
tenant_id → runtime_cluster
workspace_id → shard
object_type → physical_table
```

这种方式比让所有租户共享一个无限扩张的 PostgreSQL 更容易进行容量和故障隔离。

### 7.3 第三阶段：Citus 分布式 PostgreSQL

当希望保留 PostgreSQL 协议和生态，同时需要水平扩展时，可以考虑 Citus。

典型分片键：

- `tenant_id`
- `workspace_id`
- `organization_id`

示例：

```sql
SELECT create_distributed_table(
    'object_current',
    'tenant_id'
);
```

以 `tenant_id` 作为分片键，可以让同一租户的对象、关系和 Action 数据尽量共置在相同分片中。

需要注意：

- 分片键设计；
- 跨分片 Join；
- 全局唯一约束；
- 热点租户；
- 分片迁移；
- 跨租户查询；
- 分布式事务成本。

---

## 8. Ontology 对象表不要全部设计为 EAV

下面这种通用属性模型在大数据量下容易出现问题：

```text
object_property_value

object_id
property_id
string_value
number_value
date_value
boolean_value
```

假设一个对象有 100 个属性：

```text
1 亿个对象 × 100 个属性
= 100 亿行属性记录
```

查询一个对象需要聚合多行；复杂条件查询还需要多次自连接。

### 推荐模型

#### 高频核心属性：类型化物理列

```sql
CREATE TABLE equipment_current (
    tenant_id       UUID,
    object_id       UUID,
    equipment_name  TEXT,
    equipment_type  TEXT,
    status          TEXT,
    factory_id      UUID,
    install_time    TIMESTAMPTZ,
    rated_power     NUMERIC,
    version         BIGINT,
    PRIMARY KEY (tenant_id, object_id)
);
```

#### 少量扩展属性：JSONB

```sql
extra_properties JSONB
```

#### 超稀疏、动态属性：单独扩展表

```text
object_sparse_property
```

推荐原则：

```text
所有属性放 EAV              不推荐
所有属性放 JSONB            不推荐
核心字段类型化 + 扩展 JSONB  推荐
```

Ontology Manager 维护逻辑模型；发布时，将逻辑 Object Type 编译成具体的物理存储模型。

---

## 9. 关系数据应该怎么存

### 9.1 当前有效关系

放在 PostgreSQL：

```sql
CREATE TABLE object_link_current (
    tenant_id           UUID,
    link_type_id        UUID,
    source_object_id    UUID,
    target_object_id    UUID,
    created_at          TIMESTAMPTZ,
    version             BIGINT,
    PRIMARY KEY (
        tenant_id,
        link_type_id,
        source_object_id,
        target_object_id
    )
);
```

索引：

```sql
CREATE INDEX idx_link_source
ON object_link_current (
    tenant_id,
    source_object_id,
    link_type_id
);

CREATE INDEX idx_link_target
ON object_link_current (
    tenant_id,
    target_object_id,
    link_type_id
);
```

适合查询：

- 设备属于哪个工厂；
- 客户有哪些订单；
- 订单包含哪些产品。

### 9.2 历史关系

放在 Iceberg：

```text
source_object_id
target_object_id
link_type
valid_from
valid_to
operation
action_execution_id
```

### 9.3 复杂多跳图查询

例如：

- 查询 10 层供应链传播路径；
- 计算全局最短路径；
- 分析社区和中心性；
- 遍历数十亿条关系。

这类场景可以单独增加图查询引擎，但不建议一开始就把所有对象数据放进图数据库。多数业务 Ontology 查询主要是 1～2 跳关系，PostgreSQL 或分析引擎通常更易维护。

---

## 10. Action 应该如何写入海量数据架构

无论底层存在多少种存储，业务都不应该直接修改 Iceberg、ClickHouse、OpenSearch 或对象物理表。

统一经过 Action Service：

```text
用户执行 Action
      ↓
权限校验
      ↓
Ontology 规则校验
      ↓
事务写 PostgreSQL 当前状态
      ↓
同一事务写 Outbox
      ↓
提交成功
      ↓
CDC / Kafka
      ├── 更新 Iceberg 历史
      ├── 更新 ClickHouse 分析数据
      ├── 更新 OpenSearch 索引
      └── 触发下游 Workflow
```

例如“关闭设备”Action：

```sql
BEGIN;

UPDATE equipment_current
SET status = 'CLOSED',
    version = version + 1
WHERE tenant_id = :tenant_id
  AND object_id = :object_id
  AND version = :expected_version;

INSERT INTO action_execution (...);

INSERT INTO transaction_outbox (...);

COMMIT;
```

事务提交后异步同步：

```text
Iceberg：永久变更历史
ClickHouse：状态统计和实时指标
OpenSearch：更新搜索状态
```

这样可以保证 PostgreSQL 是在线事务的权威来源，同时避免让多个异构数据库参与同一个同步分布式事务。

---

## 11. 针对 Palantir 风格 Ontology 平台的具体建议

建议形成四层存储架构：

| 层级 | 推荐存储 | 保存内容 |
|---|---|---|
| Ontology Control Plane | PostgreSQL | Object Type、Property、Action、Link、版本、权限 |
| Runtime Transaction Store | PostgreSQL / Citus | 当前对象、当前关系、Action 事务 |
| Runtime Analytical Store | ClickHouse | 实时统计、聚合查询、时序和事件分析 |
| Data Lake History Store | Iceberg + MinIO/S3 | 全量历史、快照、审计、冷数据 |

查询路由建议：

```text
查对象当前状态
    → PostgreSQL

执行 Action
    → PostgreSQL

查过去某个时间的对象状态
    → Iceberg

查过去一年所有设备故障率
    → ClickHouse 或 Iceberg

全文搜索对象
    → OpenSearch

查 Object Type 和 Action 定义
    → Ontology PostgreSQL
```

---

## 12. 分阶段实施建议

### 第一阶段

```text
PostgreSQL + Transactional Outbox
```

目标：

- 完成 Ontology 元数据管理；
- 完成当前对象和关系存储；
- Action 统一写入；
- 预留 CDC 和事件扩展接口。

### 第二阶段

```text
PostgreSQL + Kafka + Iceberg
```

目标：

- 将历史记录从 PostgreSQL 分离；
- 建立对象和关系变更历史；
- 保存长期审计和对象快照。

### 第三阶段

```text
PostgreSQL / Citus + Kafka + Iceberg + ClickHouse + OpenSearch
```

目标：

- 支持水平扩展；
- 支持大规模实时分析；
- 支持全文搜索和复杂筛选；
- 支持海量历史和冷数据。

---

## 13. 最终判断

PostgreSQL 并不是因为数据量变大就必须被替换，而是需要缩小并明确其职责范围。

不推荐：

```text
PostgreSQL 数据大了
→ 全部迁移到 Iceberg
```

推荐：

```text
PostgreSQL
负责当前状态、事务、Action 和约束

Iceberg
负责海量历史、版本和冷数据

ClickHouse
负责实时分析和大规模聚合

OpenSearch
负责全文搜索和复杂检索

Kafka / CDC
负责把一次事务变化同步到不同存储
```

对于 Ontology 平台，应该从第一版就明确这些边界，但不必在第一阶段部署全部组件。这样即使对象规模从千万增长到数十亿，也不需要推翻 Ontology 元模型和 Action API，只需要逐步扩展运行态存储层。

---

## 14. 参考资料

- PostgreSQL 分区：https://www.postgresql.org/docs/current/ddl-partitioning.html
- PostgreSQL VACUUM：https://www.postgresql.org/docs/current/routine-vacuuming.html
- PostgreSQL 逻辑复制：https://www.postgresql.org/docs/current/logical-replication.html
- Apache Iceberg：https://iceberg.apache.org/docs/latest/
- Citus 文档：https://docs.citusdata.com/
- ClickHouse 使用场景：https://clickhouse.com/use-cases

## 相关笔记

- [[ontology_object_storage_design|对象存储设计]]
- [[tidb_only_ontology_reasoning_complete_solution|TiDB-only 推理方案]]
- [[tidb_ontology_sharding_and_partitioning_design|TiDB 分片设计]]
- [[tidb_iceberg_data_tiering_design|TiDB+Iceberg 数据分层]]
