# TiDB 替换 PostgreSQL 后的分库分片与 Ontology 存储设计

## 1. 结论

使用 TiDB 替换 PostgreSQL 后，通常不需要像传统 MySQL 分库分表中间件那样，手工指定“某个租户进入哪个数据库、哪张分片表”。

TiDB 的基本机制是：

```text
应用看到：
一个逻辑数据库
一张逻辑表

TiDB 底层：
表的 Key Space
    ↓
自动拆分成多个 Region
    ↓
PD 自动调度到不同 TiKV 节点
    ↓
每个 Region 通过 Raft 保存多个副本
```

因此，TiDB 的物理分片主要由系统自动完成。应用侧通常只需要设计好：

- 逻辑数据库和表；
- 主键；
- 索引；
- 是否需要逻辑分区；
- 数据放置和容灾策略。

---

## 2. TiDB 中“分库”和“分片”的四个概念

### 2.1 Database：逻辑数据库

例如：

```sql
CREATE DATABASE ontology_meta;
CREATE DATABASE ontology_runtime;
CREATE DATABASE ontology_audit;
```

这些 Database 主要用于：

- 表分类；
- 权限管理；
- Ontology 元数据与运行态数据隔离；
- 不同业务域隔离。

它们通常不表示：

```text
ontology_meta    固定存储在 TiKV 节点 1
ontology_runtime 固定存储在 TiKV 节点 2
```

底层数据仍然由 TiDB 自动拆成 Region，并分布在 TiKV 节点上。

因此，一般不需要设计：

```text
ontology_runtime_00
ontology_runtime_01
ontology_runtime_02
```

也不需要由应用维护：

```text
tenant_id % 16 → database_07
```

---

### 2.2 Table Partition：逻辑表分区

TiDB 支持多种逻辑分区方式，包括：

- RANGE；
- RANGE COLUMNS；
- LIST；
- LIST COLUMNS；
- HASH；
- KEY。

例如，按租户做 KEY 分区：

```sql
CREATE TABLE equipment_current (
    tenant_id        BIGINT       NOT NULL,
    object_id        BINARY(16)   NOT NULL,
    equipment_name   VARCHAR(255),
    status           VARCHAR(32),
    properties       JSON,
    object_version   BIGINT       NOT NULL,
    updated_at       DATETIME(6)  NOT NULL,

    PRIMARY KEY (tenant_id, object_id) CLUSTERED,
    KEY idx_tenant_status (tenant_id, status)
)
PARTITION BY KEY (tenant_id)
PARTITIONS 32;
```

其逻辑过程是：

```text
tenant_id 经过 Hash
       ↓
映射到 32 个逻辑分区之一
       ↓
每个分区继续自动拆分成多个 Region
       ↓
Region 分布到多个 TiKV 节点
```

需要注意：

> `PARTITIONS 32` 表示 32 个逻辑表分区，不表示 32 台服务器，也不表示 32 个独立数据库实例。

---

### 2.3 Region：TiDB 真正的物理分片单元

Region 才是 TiDB 底层真正的数据分片。

例如一张逻辑表：

```text
equipment_current
```

底层可能自动拆分为：

```text
Region 1001：Key A ～ Key B
Region 1002：Key B ～ Key C
Region 1003：Key C ～ Key D
Region 1004：Key D ～ Key E
```

这些 Region 会自动分布到多个 TiKV 节点：

```text
TiKV Node 1
TiKV Node 2
TiKV Node 3
TiKV Node 4
```

新增 TiKV 节点后，PD 可以将部分 Region 迁移到新节点，从而重新平衡容量和负载。

应用不需要知道某个对象具体位于哪个 Region 或哪个 TiKV 节点，只需要执行普通 SQL：

```sql
SELECT *
FROM equipment_current
WHERE tenant_id = 10001
  AND object_id = ?;
```

TiDB 会自动完成路由。

因此：

```text
没有 PARTITION BY
≠ 没有物理分片
```

即使一张表没有主动设置逻辑分区，TiDB 仍会通过 Region 自动分片。

---

### 2.4 Placement Policy：数据副本放置策略

如果需要控制某些数据位于特定机房、区域、机架或存储节点，可以使用 Placement Policy。

例如：

```sql
CREATE PLACEMENT POLICY ontology_policy
PRIMARY_REGION = "tokyo"
REGIONS = "tokyo,osaka"
FOLLOWERS = 2;
```

应用到数据库：

```sql
ALTER DATABASE ontology_runtime
PLACEMENT POLICY = ontology_policy;
```

应用到表：

```sql
ALTER TABLE equipment_current
PLACEMENT POLICY = ontology_policy;
```

也可以应用到单个分区：

```sql
ALTER TABLE equipment_current
PARTITION p0
PLACEMENT POLICY = ontology_policy;
```

Placement Policy 主要控制：

- Leader 所在区域；
- Follower 副本位置；
- 副本数量；
- 跨可用区容灾；
- 存储介质约束。

它不是传统意义上的应用层分片路由，不是：

```text
tenant A → TiKV 1
tenant B → TiKV 2
```

而更接近：

```text
这张表的 Leader 放在东京
Follower 分布在东京和大阪
总共保存 3 个副本
```

---

## 3. Ontology 对象表是否需要主动分区

大多数情况下，第一版不需要给所有对象表主动设置逻辑分区。

推荐先建立普通 TiDB 表：

```sql
CREATE TABLE equipment_current (
    tenant_id        BIGINT       NOT NULL,
    object_id        BINARY(16)   NOT NULL,
    equipment_name   VARCHAR(255),
    equipment_type   VARCHAR(64),
    status           VARCHAR(32),
    factory_id       BINARY(16),
    rated_power      DECIMAL(18,4),
    properties       JSON,
    object_version   BIGINT       NOT NULL,
    created_at       DATETIME(6)  NOT NULL,
    updated_at       DATETIME(6)  NOT NULL,

    PRIMARY KEY (tenant_id, object_id) CLUSTERED,

    KEY idx_tenant_status (
        tenant_id,
        status
    ),

    KEY idx_tenant_factory (
        tenant_id,
        factory_id
    ),

    KEY idx_tenant_updated (
        tenant_id,
        updated_at
    )
);
```

底层仍然会自动变成：

```text
equipment_current
    ├── Region 1
    ├── Region 2
    ├── Region 3
    └── Region N
```

所以第一阶段应优先做好主键、索引和查询模式，而不是先规划大量逻辑分区。

---

## 4. 什么时候按 tenant_id 分区

满足以下情况时，可以考虑按租户做 KEY 分区：

- 租户数量很多；
- 绝大多数查询都带 `tenant_id`；
- 单张对象表写入压力非常高；
- 需要进一步打散多个租户的写入；
- 需要对分区应用单独的数据放置策略。

示例：

```sql
CREATE TABLE customer_current (
    tenant_id        BIGINT       NOT NULL,
    object_id        BINARY(16)   NOT NULL,
    customer_name    VARCHAR(255),
    customer_level   VARCHAR(32),
    risk_level       VARCHAR(32),
    properties       JSON,
    object_version   BIGINT       NOT NULL,
    updated_at       DATETIME(6)  NOT NULL,

    PRIMARY KEY (tenant_id, object_id) CLUSTERED,

    KEY idx_customer_name (
        tenant_id,
        customer_name
    ),

    KEY idx_risk_level (
        tenant_id,
        risk_level
    )
)
PARTITION BY KEY (tenant_id)
PARTITIONS 32;
```

需要注意：

```text
查询包含 tenant_id
→ 可以定位较少分区

查询不包含 tenant_id
→ 可能访问全部分区
```

分区过多会增加 RPC、执行计划和索引管理成本。

第一阶段通常可从以下数量中评估：

```text
16、32 或 64 个分区
```

最终数量应通过租户规模、写入并发和实际压测确定，而不是按照服务器数量机械设置。

---

## 5. 什么时候按时间分区

当前对象状态表通常不建议按时间分区，因为这类查询主要依据：

- `tenant_id`；
- `object_id`；
- 状态；
- 关系；
- 业务属性。

以下表更适合按时间分区：

- `action_execution`；
- `object_change_event`；
- `link_change_event`；
- `audit_log`；
- `transaction_outbox`。

例如：

```sql
CREATE TABLE action_execution (
    tenant_id          BIGINT       NOT NULL,
    execution_id       BINARY(16)   NOT NULL,
    action_type_id     BINARY(16)   NOT NULL,
    object_id          BINARY(16),
    execution_status   VARCHAR(32),
    executed_at        DATETIME(6)  NOT NULL,
    request_payload    JSON,
    result_payload     JSON,

    PRIMARY KEY (
        tenant_id,
        execution_id,
        executed_at
    )
)
PARTITION BY RANGE COLUMNS (executed_at) (
    PARTITION p202608 VALUES LESS THAN ('2026-09-01'),
    PARTITION p202609 VALUES LESS THAN ('2026-10-01'),
    PARTITION p202610 VALUES LESS THAN ('2026-11-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
```

删除过期数据时可以直接删除旧分区：

```sql
ALTER TABLE action_execution
DROP PARTITION p202608;
```

不过，对于 Ontology 平台，更建议通过 TiCDC 或 Kafka 把完整历史归档到 Iceberg，TiDB 仅保留近期记录。

---

## 6. 主键设计比分区数量更重要

TiDB 底层按照 Key Range 分片。

如果使用单调递增主键：

```sql
id BIGINT AUTO_INCREMENT
```

新数据可能持续写入最右侧 Key Range，形成写热点。

可使用 `AUTO_RANDOM` 打散递增主键：

```sql
CREATE TABLE ontology_object (
    object_id       BIGINT PRIMARY KEY AUTO_RANDOM,
    tenant_id       BIGINT       NOT NULL,
    object_type_id  BIGINT       NOT NULL,
    properties      JSON,
    object_version  BIGINT       NOT NULL,

    KEY idx_tenant_object (
        tenant_id,
        object_type_id,
        object_id
    )
);
```

预计建表后立即有大量并发写入时，可以预切分 Region：

```sql
CREATE TABLE ontology_object (
    object_id       BIGINT PRIMARY KEY AUTO_RANDOM(5),
    tenant_id       BIGINT       NOT NULL,
    object_type_id  BIGINT       NOT NULL,
    properties      JSON
)
PRE_SPLIT_REGIONS = 4;
```

`PRE_SPLIT_REGIONS = 4` 通常表示预切分为 `2^4 = 16` 个 Region。

---

## 7. 使用 UUID、ULID 或外部对象 ID

如果对象 ID 来源于外部系统，例如：

- UUID；
- ULID；
- 业务编码；
- 源系统主键。

可以使用：

```sql
object_id BINARY(16)
```

常见主键设计：

```sql
PRIMARY KEY (tenant_id, object_id) CLUSTERED
```

随机 UUID 通常比纯递增 ID 分布更分散。

但如果 `tenant_id` 是主键第一列，同一超大租户的数据仍然位于一个连续 Key 范围。随着数据增长，该范围会自动拆成多个 Region，但在初始高并发导入时仍可能出现热点。

一种进一步打散的方法是增加 Hash 前缀：

```sql
CREATE TABLE equipment_current (
    shard_key        TINYINT      NOT NULL,
    tenant_id        BIGINT       NOT NULL,
    object_id        BINARY(16)   NOT NULL,
    status           VARCHAR(32),

    PRIMARY KEY (
        shard_key,
        tenant_id,
        object_id
    ) CLUSTERED,

    UNIQUE KEY uk_tenant_object (
        tenant_id,
        object_id
    )
);
```

其中：

```text
shard_key = hash(object_id) % 32
```

这样数据可以更均匀地分布在 Key Space 中。

缺点是按对象 ID 查询时，必须计算相同的 `shard_key`。该逻辑应封装在统一的 ObjectStore 中，不应散落在业务代码中。

---

## 8. SHARD_ROW_ID_BITS

如果表没有聚簇主键，TiDB 可能使用隐式 `_tidb_rowid`。

大量递增写入时，可以使用：

```sql
CREATE TABLE object_extension (
    tenant_id       BIGINT,
    object_id       BINARY(16),
    property_data   JSON
)
SHARD_ROW_ID_BITS = 4;
```

`SHARD_ROW_ID_BITS = 4` 表示将隐式 Row ID 分散到 16 个范围中。

也可以结合预切分：

```sql
CREATE TABLE object_extension (
    tenant_id       BIGINT,
    object_id       BINARY(16),
    property_data   JSON
)
SHARD_ROW_ID_BITS = 4
PRE_SPLIT_REGIONS = 4;
```

对于 Ontology 核心对象表，仍建议显式定义主键，而不是主要依赖 `_tidb_rowid`。

---

## 9. 是否可以明确指定某张表存在哪台 TiKV

一般不能像传统分片中间件一样直接指定：

```text
equipment_current → TiKV Node 1
customer_current  → TiKV Node 2
```

TiDB 需要保留自动调度和故障恢复能力。

可以通过节点标签和 Placement Policy 指定拓扑约束，例如：

```text
region=tokyo
zone=tokyo-a
disk=ssd
```

然后创建放置策略：

```sql
CREATE PLACEMENT POLICY ssd_policy
CONSTRAINTS = '[+disk=ssd]';
```

应用到表：

```sql
ALTER TABLE equipment_current
PLACEMENT POLICY = ssd_policy;
```

可以控制：

- 副本数量；
- Leader 和 Follower 角色；
- 区域和可用区；
- 机架；
- SSD 或其他存储介质。

但具体 Region 在哪个 TiKV 节点，仍由 PD 调度。

---

## 10. Ontology 平台的 TiDB 推荐设计

### 10.1 逻辑数据库划分

```text
ontology_control
    Object Type
    Property
    Link Type
    Action Type
    Interface
    Version
    发布记录

ontology_runtime
    当前对象
    当前关系
    Action 执行状态

ontology_operation
    Outbox
    近期审计
    近期变更事件
```

这些 Database 是逻辑 Schema，不是传统意义上的物理分库。

---

### 10.2 每个核心 Object Type 一张逻辑表

建议：

```text
equipment_current
customer_current
order_current
factory_current
supplier_current
```

不建议长期将全部对象放入：

```text
ontology_object(properties JSON)
```

统一 JSON 表仅适合：

- MVP；
- 长尾 Object Type；
- 低频对象；
- Schema 极不稳定的数据。

---

### 10.3 第一阶段不主动分区

第一阶段建议：

```text
一个核心 Object Type
        ↓
一张普通 TiDB 逻辑表
        ↓
合理的复合主键与索引
        ↓
由 TiDB 自动拆分 Region
```

示例：

```sql
CREATE TABLE equipment_current (
    tenant_id        BIGINT       NOT NULL,
    object_id        BINARY(16)   NOT NULL,
    equipment_name   VARCHAR(255),
    status           VARCHAR(32),
    factory_id       BINARY(16),
    properties       JSON,
    object_version   BIGINT       NOT NULL,
    updated_at       DATETIME(6)  NOT NULL,

    PRIMARY KEY (tenant_id, object_id) CLUSTERED,
    KEY idx_status (tenant_id, status),
    KEY idx_factory (tenant_id, factory_id)
);
```

---

### 10.4 出现明确热点后再增加 KEY 分区

例如：

```sql
ALTER TABLE equipment_current
PARTITION BY KEY (tenant_id)
PARTITIONS 32;
```

执行之前需要验证：

- 查询是否基本都包含 `tenant_id`；
- 唯一索引是否满足分区表要求；
- 跨租户查询是否会扫描所有分区；
- 32 个逻辑分区是否确实优于非分区表；
- 是否真的存在 Region 写热点。

---

### 10.5 历史数据不长期保留在 TiDB

推荐职责划分：

```text
TiDB
    当前对象
    当前关系
    Action 事务
    近期事件

Iceberg
    完整对象历史
    关系历史
    对象快照
    长期审计记录
```

TiDB 的分布式能力主要解决当前在线状态数据的扩展问题，不代表全部历史、日志和分析数据都应该永久放在 TiDB。

---

## 11. 推荐演进路线

### 第一阶段

```text
普通 TiDB 表
不主动逻辑分区
合理设计主键和索引
依靠 Region 自动分片
```

### 第二阶段

发现递增主键热点后，评估：

- `AUTO_RANDOM`；
- `SHARD_ROW_ID_BITS`；
- `PRE_SPLIT_REGIONS`；
- Hash 或 KEY 分区。

### 第三阶段

对象表达到很大规模后：

```text
按 tenant_id 设置 16～64 个 KEY 分区
结合查询模式和实际压测调整
```

### 第四阶段

使用 Placement Policy 控制：

- 跨机房副本；
- Leader 所在区域；
- 副本数量；
- 存储介质。

### 历史数据

持续通过 TiCDC、Kafka 或数据同步任务归档到 Iceberg。

---

## 12. 最终建议

在 TiDB 中，不要继续采用传统 PostgreSQL 或 MySQL 的方式，预先设计：

```text
16 个数据库 × 64 张分片表
```

更推荐：

```text
一个核心 Object Type
        ↓
一张 TiDB 逻辑表
        ↓
合理的主键与索引
        ↓
TiDB 自动 Region 分片
        ↓
确有业务需求时再增加 KEY、RANGE 等逻辑分区
```

不建议一开始建立：

```text
equipment_000
equipment_001
...
equipment_063
```

核心原则是：

> TiDB 的 Region 已经承担了主要的物理分片职责。应用和 Ontology Runtime 应关注逻辑模型、主键设计、索引、热点控制和数据生命周期，而不是自行维护大量物理分片表。

## 相关笔记

- [[ontology_object_storage_design|对象存储设计]]
- [[tidb_write_hotspot_explained|TiDB 热点写入]]
- [[tidb_nebulagraph_large_scale_object_link_projection_solution|TiDB+NebulaGraph 方案]]
