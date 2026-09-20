# Ontology 对象存储设计：是否所有对象放在一张表中

## 1. 结论

**不建议把所有对象都放在一张大表里。**

在 Ontology 平台中，用户看到的是统一的“对象模型”，但底层物理存储通常会根据对象类型、数据规模、查询模式和数据来源进行拆分。

```text
逻辑层：统一的 Ontology Object
               ↓
映射层：Object Type → 物理存储
               ↓
物理层：多张表、多个分区、多个数据库或 Iceberg 表
```

Ontology 应当统一对象的逻辑模型和访问接口，而不是强制所有对象使用同一张物理数据表。

---

## 2. 一个对象通常如何存储

例如一个设备对象：

```json
{
  "objectId": "equipment-10001",
  "objectType": "Equipment",
  "name": "数控机床-01",
  "status": "RUNNING",
  "factoryId": "factory-001",
  "ratedPower": 120.5,
  "installTime": "2025-01-10T08:00:00Z"
}
```

在 Ontology 中，它是一个逻辑对象；在 PostgreSQL 中，通常是一张类型化表中的一行：

```sql
CREATE TABLE equipment_current (
    tenant_id         UUID          NOT NULL,
    object_id         UUID          NOT NULL,
    name              TEXT          NOT NULL,
    status            VARCHAR(32),
    factory_id        UUID,
    rated_power       NUMERIC(12,2),
    install_time      TIMESTAMPTZ,
    extra_properties  JSONB,
    object_version    BIGINT        NOT NULL,
    updated_at        TIMESTAMPTZ   NOT NULL,
    PRIMARY KEY (tenant_id, object_id)
);
```

其中：

- 一行代表一个设备对象；
- 一列代表一个常用属性；
- `extra_properties` 保存少量动态扩展属性；
- `object_version` 用于乐观锁和并发控制。

---

## 3. 推荐方式：每个主要 Object Type 一张表

例如 Ontology 中有：

```text
Customer
Order
Product
Equipment
Factory
Supplier
```

可以映射为：

```text
customer_current
order_current
product_current
equipment_current
factory_current
supplier_current
```

例如客户对象表：

```sql
CREATE TABLE customer_current (
    tenant_id         UUID,
    object_id         UUID,
    customer_name     TEXT,
    customer_level    VARCHAR(32),
    phone             TEXT,
    region_code       VARCHAR(32),
    extra_properties  JSONB,
    object_version    BIGINT,
    PRIMARY KEY (tenant_id, object_id)
);
```

订单对象表：

```sql
CREATE TABLE order_current (
    tenant_id         UUID,
    object_id         UUID,
    order_no          TEXT,
    customer_id       UUID,
    amount            NUMERIC(18,2),
    status            VARCHAR(32),
    created_at        TIMESTAMPTZ,
    extra_properties  JSONB,
    object_version    BIGINT,
    PRIMARY KEY (tenant_id, object_id)
);
```

这种方式的优势：

- 属性有明确的数据类型；
- 查询性能更好；
- 容易建立索引；
- 可以使用数据库约束；
- 容易执行 Action 事务；
- 不同 Object Type 可以独立扩容；
- 不会因为某一种对象增长而影响全部对象。

这是 Ontology Runtime 最适合作为基础实现的方式。

---

## 4. 为什么不建议所有对象放在一张表

一种看似灵活的设计是：

```sql
CREATE TABLE ontology_object (
    tenant_id       UUID,
    object_type_id  UUID,
    object_id       UUID,
    properties      JSONB,
    PRIMARY KEY (tenant_id, object_type_id, object_id)
);
```

所有对象都放进去：

```text
Equipment → properties JSONB
Customer  → properties JSONB
Order     → properties JSONB
Product   → properties JSONB
```

这种方式适合原型系统，但大规模生产环境会逐渐出现问题。

### 4.1 索引困难

设备按状态查询：

```sql
SELECT *
FROM ontology_object
WHERE object_type_id = 'equipment'
  AND properties->>'status' = 'RUNNING';
```

订单按金额查询：

```sql
SELECT *
FROM ontology_object
WHERE object_type_id = 'order'
  AND (properties->>'amount')::numeric > 100000;
```

每一种属性都可能需要表达式索引，索引数量会快速增加。

### 4.2 数据类型约束较弱

JSONB 中的金额可能被错误写成字符串：

```json
{
  "amount": "一万元"
}
```

数据库难以像类型化列一样直接保证数据正确。

### 4.3 表膨胀严重

设备、订单、客户和日志全部更新同一张表，会造成：

- 所有 Object Type 竞争同一张大表；
- Autovacuum 压力集中；
- 索引膨胀集中；
- 一个高频对象类型可能拖慢整个系统。

### 4.4 查询计划不稳定

不同 Object Type 的数据分布完全不同，但共用同一张表和索引体系，数据库难以针对每个类型生成最优执行计划。

因此：

```text
所有对象一张表 + JSONB
```

适合：

- MVP；
- Object Type 很少；
- 对象总量较小；
- 查询要求不高；
- 快速验证 Ontology 模型。

但不适合长期作为海量对象的核心存储。

---

## 5. 也不建议纯 EAV 模型

另一种设计是将每个属性保存为一行：

```sql
CREATE TABLE object_property_value (
    tenant_id       UUID,
    object_id       UUID,
    property_id     UUID,
    string_value    TEXT,
    number_value    NUMERIC,
    boolean_value   BOOLEAN,
    timestamp_value TIMESTAMPTZ
);
```

一个设备对象有 50 个属性，就需要 50 行：

```text
equipment-001  name          数控机床
equipment-001  status        RUNNING
equipment-001  rated_power   120.5
equipment-001  factory_id    factory-001
...
```

如果有一亿个对象，每个对象平均 50 个属性：

```text
1亿 × 50 = 50亿行属性记录
```

读取一个对象需要把几十行重新聚合；多属性条件查询还需要反复 Join。

纯 EAV 通常只适合：

- 极少使用的稀疏属性；
- 无法提前定义的自定义属性；
- 配置类属性；
- 小规模补充数据。

不适合作为主要对象存储模型。

---

## 6. 最合理的是混合模式

推荐采用：

```text
核心属性：数据库类型化列
扩展属性：JSONB
极稀疏属性：扩展属性表
历史属性：Iceberg
搜索字段：OpenSearch
分析字段：ClickHouse
```

示例：

```sql
CREATE TABLE equipment_current (
    tenant_id          UUID,
    object_id          UUID,

    -- 核心属性
    equipment_name     TEXT,
    equipment_type     VARCHAR(64),
    status             VARCHAR(32),
    factory_id         UUID,
    rated_power        NUMERIC(12,2),
    install_time       TIMESTAMPTZ,

    -- 不常用的扩展属性
    extra_properties   JSONB,

    -- Runtime 控制字段
    object_version     BIGINT,
    created_at         TIMESTAMPTZ,
    updated_at         TIMESTAMPTZ,

    PRIMARY KEY (tenant_id, object_id)
);
```

Ontology Manager 中定义的 Property 可以带一个存储策略：

```text
Property: status
数据类型: STRING
存储方式: TYPED_COLUMN
物理列: status
是否索引: 是

Property: ratedPower
数据类型: DECIMAL
存储方式: TYPED_COLUMN
物理列: rated_power
是否索引: 否

Property: manufacturerRemark
数据类型: STRING
存储方式: JSONB
JSON路径: manufacturerRemark
是否索引: 否
```

这样逻辑模型和物理模型就被解耦了。

---

## 7. Object Type 与物理表不一定严格一对一

虽然“每个 Object Type 一张表”是最容易理解的默认方式，但实际可以有多种映射。

### 7.1 一个 Object Type 对应一张表

```text
Equipment → equipment_current
Customer  → customer_current
Order     → order_current
```

适合大多数核心业务对象。

### 7.2 多个相似 Object Type 共用一张表

例如：

```text
Person
Employee
CustomerContact
SupplierContact
```

如果它们字段高度一致，可以共享：

```text
party_person_current
```

并通过类型字段区分：

```sql
person_type VARCHAR(32)
```

只有字段结构和查询方式高度相似时才建议这样做。

### 7.3 一个 Object Type 对应多张表

例如 Customer 对象包含：

```text
基础资料
风险信息
营销标签
统计指标
扩展属性
```

底层可以拆成：

```text
customer_core
customer_risk
customer_marketing_profile
customer_metrics
```

Ontology Runtime 查询 Customer 时再进行逻辑合并：

```text
Customer Object
   ├── customer_core
   ├── customer_risk
   ├── customer_marketing_profile
   └── customer_metrics
```

这种方式适合：

- 不同数据由不同系统产生；
- 更新频率不同；
- 权限范围不同；
- 数据规模差异很大；
- 部分属性来自实时计算。

### 7.4 一个 Object Type 对应外部数据集

对象可能直接来源于：

```text
PostgreSQL 表
Iceberg 表
ERP 数据库
CRM 数据库
API
Kafka 流
```

Ontology 不一定复制全部数据，而是保存映射关系：

```text
Object Type: Customer
Primary Key: customer_id

Property:
  name        → crm_customer.customer_name
  level       → crm_customer.customer_level
  totalAmount → customer_metrics.total_amount
```

所以 Ontology 本身是一个语义层，不等于必须建立一套统一的单表存储。

---

## 8. 对象当前状态和对象历史要分开

例如设备当前状态：

```text
equipment_current
```

只保存最新状态：

| object_id | status | rated_power | updated_at |
|---|---|---:|---|
| E001 | RUNNING | 120.5 | 2026-08-06 |

历史变更不要持续堆在同一张当前状态表里，而是写入事件或历史存储：

```text
equipment_change_event
```

或者进入 Iceberg：

| object_id | old_status | new_status | changed_at |
|---|---|---|---|
| E001 | STOPPED | RUNNING | 2026-08-06 |
| E001 | FAILED | STOPPED | 2026-08-05 |

推荐：

```text
PostgreSQL
└── 保存对象最新状态

Iceberg
└── 保存对象完整历史和快照
```

否则当前查询表会无限增长，Action 更新和索引维护都会越来越困难。

---

## 9. 对象之间的关系应单独存储

对象本身和对象关系通常不要混在一张表中。

例如：

```text
Equipment 属于 Factory
Customer 创建 Order
Order 包含 Product
```

可以使用统一关系表：

```sql
CREATE TABLE object_link_current (
    tenant_id           UUID,
    link_type_id        UUID,
    source_object_type  UUID,
    source_object_id    UUID,
    target_object_type  UUID,
    target_object_id    UUID,
    link_properties     JSONB,
    version             BIGINT,
    created_at          TIMESTAMPTZ,
    PRIMARY KEY (
        tenant_id,
        link_type_id,
        source_object_id,
        target_object_id
    )
);
```

也可以对高频核心关系单独建立类型化关系表：

```text
equipment_factory_link
customer_order_link
order_product_link
```

推荐策略：

- 普通、低频 Link Type：共用通用关系表；
- 数据量巨大、高频访问的关系：独立关系表；
- 历史关系：Iceberg；
- 深层图计算：单独同步到图分析引擎。

---

## 10. 大规模情况下需要继续分区和分片

即使每个 Object Type 一张表，如果某种对象达到数十亿条，也不能只使用一张普通物理表。

例如订单表可以按租户进行 PostgreSQL 分区：

```sql
CREATE TABLE order_current (
    tenant_id    UUID,
    object_id    UUID,
    created_at   TIMESTAMPTZ,
    status       VARCHAR(32),
    amount       NUMERIC(18,2)
) PARTITION BY HASH (tenant_id);
```

分成多个分区：

```text
order_current_p0
order_current_p1
order_current_p2
...
order_current_p31
```

数据继续增长后，可以进一步分布式分片：

```text
tenant_id hash
     ↓
Runtime Cluster 01
Runtime Cluster 02
Runtime Cluster 03
Runtime Cluster 04
```

或者使用 Citus：

```text
tenant_id → 分布式分片键
```

同一租户的对象和关系应尽量放在同一个分片上，以减少跨节点事务和 Join。

---

## 11. Ontology 平台的存储映射设计

### 11.1 Ontology 元数据表

统一存放模型定义：

```text
ontology_object_type
ontology_property_definition
ontology_link_type
ontology_action_type
ontology_storage_mapping
ontology_version
```

对象类型存储映射表：

```sql
CREATE TABLE ontology_storage_mapping (
    object_type_id       UUID,
    storage_type         VARCHAR(32),
    datasource_id        UUID,
    physical_table       TEXT,
    primary_key_column   TEXT,
    tenant_column        TEXT,
    partition_strategy   JSONB,
    PRIMARY KEY (object_type_id)
);
```

属性映射表：

```sql
CREATE TABLE ontology_property_mapping (
    object_type_id       UUID,
    property_id          UUID,
    storage_mode         VARCHAR(32),
    physical_column      TEXT,
    json_path            TEXT,
    source_expression    TEXT,
    indexed              BOOLEAN,
    PRIMARY KEY (object_type_id, property_id)
);
```

### 11.2 Runtime 查询流程

根据 Object Type 动态映射到不同表：

```text
Equipment → equipment_current
Customer  → customer_current
Order     → order_current
```

查询流程：

```text
请求查询 Equipment
       ↓
读取 Ontology Object Type
       ↓
读取 storage_mapping
       ↓
定位 equipment_current
       ↓
根据 property_mapping 生成查询
       ↓
返回统一 Object JSON
```

用户看到的仍然是统一格式：

```json
{
  "objectType": "Equipment",
  "objectId": "E001",
  "properties": {
    "name": "数控机床-01",
    "status": "RUNNING",
    "ratedPower": 120.5
  }
}
```

但底层不要求所有对象存在同一张表。

---

## 12. 最终推荐架构

对于 Palantir 风格 Ontology 平台，建议采用：

```text
Ontology 逻辑层
    所有对象拥有统一访问方式

物理存储层
    核心 Object Type：每个类型独立表
    相似小型 Object Type：允许共享表
    大型 Object Type：独立分区或分片
    核心属性：类型化列
    扩展属性：JSONB
    对象关系：单独 Link 表
    当前状态：PostgreSQL
    历史数据：Iceberg
    分析数据：ClickHouse
    搜索索引：OpenSearch
```

最重要的一点是：

> Ontology 应当统一对象的逻辑模型和访问接口，而不是强制所有对象使用同一张物理数据表。

---

## 13. 第一版建议

第一版可以先实现：

```text
一个 Object Type → 一张 PostgreSQL 表
核心字段 → 类型化列
扩展字段 → JSONB
关系 → 通用 Link 表
历史 → Outbox，后续同步 Iceberg
```

这种设计实现简单，同时保留了未来向以下方向扩展的空间：

- PostgreSQL 分区；
- 多数据库分片；
- Citus；
- Iceberg；
- ClickHouse；
- OpenSearch；
- 多数据源映射。

## 相关笔记

- [[ontology_runtime_postgresql_scalability_architecture|Runtime PostgreSQL 架构]]
- [[tidb_ontology_sharding_and_partitioning_design|TiDB 分片设计]]
- [[tidb_nebulagraph_large_scale_object_link_projection_solution|TiDB+NebulaGraph 方案]]
- [[企业本体方法论_对外完整版_v2|企业本体方法论]]
