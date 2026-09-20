# TiDB + NebulaGraph：超大规模 Object / Link / Graph Projection 完整方案

## 1. 核心问题

如果一个 `Object Type` 与另一个 `Object Type` 存在 `Link`，但对象数量巨大，例如：

```text
Company
  ↓ HAS_ORDER
Order × 5000万
  ↓ HAS_SHIPMENT
Shipment × 2亿
  ↓ HAS_LOGISTICS_EVENT
LogisticsEvent × 200亿
```

不能把 Ontology 中所有 Link 都机械地 1:1 物化为 NebulaGraph 的 Graph Edge。

真正的风险包括：

- 图规模无意义膨胀；
- Company 等节点形成超高度节点（Supernode）；
- 单个节点拥有数百万甚至数千万邻接边；
- 多跳查询发生巨大邻接展开；
- 局部 Partition/Storage 热点；
- 图推理被大量结构性明细关系淹没；
- 明细数据与推理数据耦合。

因此必须明确：

> **Ontology Link ≠ Graph Edge。**

Ontology 描述业务世界中存在什么关系；Graph Projection 决定哪些关系值得进入图数据库。

---

## 2. Ontology Link 与 Graph Projection 分离

Ontology 可以定义：

```text
Company --HAS_ORDER--> Order
Order --HAS_SHIPMENT--> Shipment
Shipment --HAS_EVENT--> LogisticsEvent
```

但 Graph Runtime 不应该直接变成：

```text
Company
  ↓ 5000万条Graph Edge
Order
  ↓ 2亿条Graph Edge
Shipment
  ↓ 200亿条Graph Edge
LogisticsEvent
```

建议增加：

```text
Ontology Link
       │
       ▼
Graph Projection Policy
       │
       ├── DIRECT
       ├── INLINE
       ├── ACTIVE
       ├── AGGREGATED
       ├── BUCKETED
       ├── VIRTUAL
       └── NONE
```

| 模式 | 含义 |
|---|---|
| DIRECT | 原始关系直接进入 Graph |
| INLINE | 关系通过 TiDB 业务字段表达，不单独物化 |
| ACTIVE | 只把当前活跃对象/关系放入 Graph |
| AGGREGATED | 将大量明细关系聚合后生成 Graph Edge |
| BUCKETED | 必须保留大量单条关系时，通过 Bucket 分散超级节点 |
| VIRTUAL | Ontology 中存在 Link，但查询时从 TiDB 动态解析 |
| NONE | 完全不进入 Graph |

---

## 3. Company → Order → Shipment → LogisticsEvent 如何存储

| Object Type | 数据特点 | TiDB | Iceberg | NebulaGraph |
|---|---:|---:|---:|---:|
| Company | 核心实体 | ✅ | 可选 | ✅ 全量 |
| Customer | 核心实体 | ✅ | 可选 | ✅ 全量 |
| Supplier | 核心实体 | ✅ | 可选 | ✅ 全量 |
| Carrier | 核心实体 | ✅ | 可选 | ✅ 全量 |
| Warehouse | 核心实体 | ✅ | 可选 | ✅ 全量 |
| Order | 极大量 | ✅ 当前态 | ✅ 历史 | ⚠️ 选择性 |
| Shipment | 更大量 | ✅ 当前态 | ✅ 历史 | ⚠️ 选择性 |
| LogisticsEvent | 超大量 | 少量当前态 | ✅ 全量 | ❌ |
| GPS / Scan Event | 超大量 | ❌或仅近期 | ✅ 全量 | ❌ |

总体职责：

```text
                      Ontology
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
     TiDB           NebulaGraph         Iceberg
       │                 │                 │
完整业务对象        推理关系网络         海量历史事实
当前业务状态        轻量节点             Event
事务                聚合关系             History
Action              活跃关系             Raw Data
Link Fact           异常关系
```

---

## 4. Company → Order 不建议直接做 Graph Edge

假设 Company A 拥有 5000 万 Order。

错误设计：

```text
Company A
 ├── Order1
 ├── Order2
 ├── Order3
 ├── ...
 └── Order50000000
```

一次 `HAS_ORDER` 邻接展开就是数千万节点，这本质上已经是大规模明细列表查询，而不是关系推理。

应该交给：

```text
TiDB
TiFlash
OpenSearch
```

---

## 5. Company → Order 使用 TiDB Inline Link

对于典型 1:N 结构：

```text
Company
   ↓
Order
```

不建议额外创建海量 `ontology_link_current` 记录。

直接通过 Order 表中的 `company_pk` 表达：

```sql
CREATE TABLE order_current (
    pk BIGINT PRIMARY KEY AUTO_RANDOM,
    company_pk BIGINT NOT NULL,
    customer_pk BIGINT,
    order_no VARCHAR(128) NOT NULL,
    order_status VARCHAR(32),
    amount DECIMAL(20,2),
    order_time DATETIME(6),
    risk_level VARCHAR(20),
    properties JSON,
    KEY idx_company (company_pk, order_time)
);
```

Ontology Mapping：

```text
Link Type:
Company --HAS_ORDER--> Order

Backing:
Order.company_pk
```

---

## 6. Link Fact 分两种

### 6.1 Inline Link

适合：

```text
Company --HAS_ORDER--> Order
Order --HAS_SHIPMENT--> Shipment
Shipment --BELONGS_TO--> Carrier
```

特点：

- 1:N；
- 结构性从属关系；
- 子对象表已有引用字段；
- 通常不参与复杂多跳推理；
- 数量巨大。

可以用：

```text
order.company_pk
shipment.order_pk
shipment.carrier_pk
```

表达。

### 6.2 Relationship Fact

适合：

```text
Company --OWNS--> Company
Person --DIRECTOR_OF--> Company
Company --GUARANTEES--> Company
Company --SUPPLIES--> Company
```

特点：

- 多对多；
- 关系有属性；
- 有生命周期；
- 有来源、置信度；
- 参与推理。

这类关系进入 `ontology_link_current`。

---

## 7. Order 是否进入 NebulaGraph：Projection Policy

建议给 Object Type 增加：

```json
{
  "objectType": "Order",
  "graphProjection": {
    "mode": "ACTIVE_WINDOW",
    "retentionDays": 90,
    "conditions": ["status != CLOSED"],
    "properties": [
      "orderNo",
      "status",
      "riskLevel",
      "amount"
    ]
  }
}
```

推荐模式：

| Graph Mode | 含义 |
|---|---|
| FULL | 全部对象进入 Graph |
| ACTIVE_WINDOW | 只保存最近 N 天 |
| ACTIVE_ONLY | 只保存未结束对象 |
| EXCEPTION_ONLY | 只保存异常对象 |
| SAMPLE | 抽样 |
| NONE | 不进入 Graph |

例如：

```text
Company       → FULL
Supplier      → FULL
Carrier       → FULL
Warehouse     → FULL
Order         → ACTIVE_ONLY
Shipment      → ACTIVE_ONLY
LogisticsEvent→ NONE
```

---

## 8. LogisticsEvent 不建议全部进入 Graph

例如：

```text
1个Order
   ↓
3个Shipment

1个Shipment
   ↓
100个LogisticsEvent
```

如果有 1 亿 Order，就可能形成 300 亿 LogisticsEvent。

正确方式：

```text
LogisticsEvent
        │
        ▼
      Iceberg
```

TiDB 中的 `shipment_current` 只保存：

```text
latest_status
latest_location
last_event_time
exception_status
estimated_arrival
```

Graph 只保存 Shipment 当前推理状态。

---

## 9. Graph 中存 Shipment，而不是全部 Event

建议：

```text
Order
   │
   │ HAS_SHIPMENT
   ▼
Shipment
   │
   ├── HANDLED_BY ───► Carrier
   ├── ORIGIN ───────► Warehouse
   ├── DESTINATION ──► Warehouse
   └── CURRENT_AT ───► Location
```

Shipment Node 只保留：

```text
shipmentId
status
riskLevel
delayed
latestLocation
estimatedArrival
```

完整 Event 历史继续放 Iceberg。

---

## 10. Graph 更应该存“关系特征”

例如一个公司一年有 500 万订单。

为了回答：

> A 公司与 Supplier B 的交易关系是否紧密？

不应该把 500 万 Order 全部放进 Graph。

建议计算为：

```text
Company A
       │
       │ PURCHASES_FROM
       │ orderCount30d = 38291
       │ orderCount365d = 582193
       │ amount30d = 1.2B
       │ amount365d = 18.3B
       │ abnormalOrders = 192
       │ lastOrderTime = ...
       ▼
Supplier B
```

即：

```text
500万 Order
↓
1条 Business Relation Edge
```

---

## 11. Relationship Feature Layer

建议增加：

```text
Order × 数十亿
Shipment × 数十亿
LogisticsEvent × 数百亿
        │
        ▼
      Iceberg
        │
        ▼
Spark / Flink / TiFlash
        │
        ▼
Relationship Feature
        │
        ▼
NebulaGraph
```

生成：

```text
Company A ──PURCHASES_FROM──► Supplier B
Company A ──USES_CARRIER──► Carrier C
Supplier B ──SHIPS_TO──► Warehouse D
Carrier C ──SERVES_ROUTE──► Route SH-CD
```

Edge 可以保存：

```text
30天次数
90天次数
365天次数
交易金额
异常金额
异常比例
最近发生时间
首次发生时间
风险评分
置信度
```

---

## 12. 如果必须对每个 Order 做 Graph 遍历：Graph Bucket

如果确实需要：

```text
Company → 每一个 Order
```

不要：

```text
Company → 5000万Order
```

引入 Bucket：

```text
Company A
   │
   ├── OrderBucket 2026-08-000
   ├── OrderBucket 2026-08-001
   ├── ...
   └── OrderBucket 2026-08-255
```

Order 按：

```text
month(order_time)
+
hash(order_id) % 256
```

进入 Bucket：

```text
Company A
   │
   ├── 2026-08 / Bucket-001
   │             ├── Order 1001
   │             ├── Order 3911
   │             └── ...
   │
   ├── 2026-08 / Bucket-002
   │             ├── Order 1002
   │             └── ...
```

这样 Company 不再直接拥有几千万 Degree。

---

## 13. Bucket 不应暴露给 Ontology 用户

Ontology Logical Model：

```text
Company
   ↓ HAS_ORDER
Order
```

物理 Graph：

```text
Company
→ OrderBucket
→ Order
```

因此必须区分：

```text
Logical Graph
=
Ontology

Physical Graph
=
NebulaGraph Projection
```

---

## 14. Order → Shipment 一般可以直接 Graph 化

一个 Order 通常只对应少量 Shipment：

```text
Order
   ↓
1～10个Shipment
```

这种 Degree 较小：

```text
Order --HAS_SHIPMENT--> Shipment
```

可以直接进入 Graph。

真正危险的是：

```text
Company → 数百万Order
Shipment → 数百/数千LogisticsEvent
```

---

## 15. Link Type 的 Graph Projection Policy

例如：

```json
{
  "linkType": "HAS_ORDER",
  "sourceType": "Company",
  "targetType": "Order",
  "graphProjection": {
    "enabled": true,
    "mode": "BUCKETED",
    "bucketStrategy": "TIME_HASH",
    "timeGranularity": "MONTH",
    "bucketCount": 256,
    "retentionDays": 90,
    "reasoningEnabled": false
  }
}
```

`OWNS`：

```json
{
  "graphProjection": {
    "enabled": true,
    "mode": "DIRECT",
    "reasoningEnabled": true
  }
}
```

物流事件：

```json
{
  "linkType": "HAS_LOGISTICS_EVENT",
  "graphProjection": {
    "enabled": false,
    "mode": "NONE"
  }
}
```

---

## 16. 推荐正式支持的 Link Projection Mode

| Mode | 适用场景 | 示例 |
|---|---|---|
| DIRECT | 低/中 Degree 推理关系 | Company→OWNS→Company |
| INLINE | TiDB 字段表达关系 | Company→Order |
| ACTIVE | 只 Graph 化活跃关系 | Order→Shipment |
| AGGREGATED | 明细聚合成关系 | Company→Supplier |
| BUCKETED | 必须保留大量单条关系 | Company→Order |
| VIRTUAL | 查询时从 TiDB 解析 | Company→历史Order |
| NONE | 不参与 Graph | Shipment→Event |

---

## 17. 查询路由

### A 公司有多少订单？

```text
TiFlash / TiDB
```

### 查询 A 公司最近 100 个订单

```sql
SELECT *
FROM order_current
WHERE company_pk = ?
ORDER BY order_time DESC
LIMIT 100;
```

走 TiDB。

### 查询订单 O001 的物流

```text
TiDB shipment_current
+
Iceberg logistics_event
```

### A 公司与哪些供应商关系最紧密？

```text
NebulaGraph
Company → PURCHASES_FROM → Supplier
```

### A 公司与风险企业 X 有什么关系？

```text
NebulaGraph
```

例如：

```text
A公司
→ Supplier B
→ Director 张三
→ Owns C公司
→ Guarantees X公司
```

### A 公司多少订单使用了风险物流公司 C？

先用 Graph 找关系：

```text
A → USES_CARRIER → C
```

再用 TiDB/TiFlash 算订单明细。

原则：

> **Graph 找关系，TiDB 找明细。**

---

## 18. Query Planner

```text
Ontology Query
       │
       ▼
Query Planner
       │
 ┌─────┼──────────┬───────────┐
 │     │          │           │
 ▼     ▼          ▼           ▼
TiDB  TiFlash  NebulaGraph  Iceberg
 │     │          │           │
状态   聚合       推理        历史
明细   统计       Path        Event
事务   当前分析   Network     回溯
```

例如：

```text
getLinkedObjects(
    Company=A,
    link=HAS_ORDER
)
```

Planner 发现：

```text
HAS_ORDER
projection = INLINE
```

自动走：

```text
order.company_pk
```

而：

```text
traverse(
   Company=A,
   link=OWNS,
   depth=5
)
```

发现：

```text
OWNS
projection = DIRECT
reasoningEnabled = true
```

自动走 NebulaGraph。

---

## 19. Graph Projector 数据流

```text
                        TiDB
                         │
                       TiCDC
                         │
                         ▼
                       Kafka
                         │
              Graph Projection Service
                         │
             ┌───────────┼────────────┐
             │           │            │
          DIRECT      ACTIVE      AGGREGATED
             │           │            │
             └───────────┼────────────┘
                         ▼
                    NebulaGraph
```

历史聚合链路：

```text
Iceberg
   │
Spark / Flink
   │
Relationship Aggregator
   │
Kafka
   │
Graph Projector
   │
NebulaGraph
```

负责：

```text
30天关系
90天关系
365天关系
风险特征
历史交易特征
```

---

## 20. Graph 生命周期

对于只保留近期 Order/Shipment 的图投影，建议由 `Graph Projection Service` 显式维护：

```text
ACTIVE
→ EXPIRED
→ DELETE
```

并维护：

```text
Graph Generation
```

不要把业务生命周期完全依赖数据库 TTL。

---

## 21. Graph Space 规划

不要使用：

```text
一个Object Type = 一个Graph Space
```

建议按照：

```text
Reasoning Domain
```

划分：

```text
enterprise_relation_graph
supply_chain_graph
asset_risk_graph
```

因为 Graph 的目标是跨类型关系推理，而不是复制关系型数据库表结构。

---

## 22. 超大规模示例

假设未来：

```text
Company             1000万
Supplier            3000万
Person              2亿
Order               100亿
Shipment            200亿
LogisticsEvent      2万亿
```

不要让 NebulaGraph 保存：

```text
2万亿 Event
+
200亿 Shipment
+
100亿 Order
```

NebulaGraph 主要保存：

```text
Company
Supplier
Person
重要/活跃Order
活跃Shipment

OWNS
CONTROLS
SUPPLIES
GUARANTEES
DIRECTOR_OF
BENEFICIARY_OF

PURCHASES_FROM
USES_CARRIER
SHIPS_TO
SERVES_ROUTE

RiskRelation
InferredRelation
```

真正几十亿、几百亿、万亿级的历史：

```text
Order History
Shipment History
Logistics Event
Payment Event
IoT Event
```

交给：

```text
TiDB + Iceberg + TiFlash
```

---

## 23. 最终数据模型

```text
                       Ontology
                           │
             ┌─────────────┴─────────────┐
             │                           │
       Semantic Model              Projection Policy
             │                           │
             │              ┌────────────┼─────────────┐
             │              │            │             │
             ▼              ▼            ▼             ▼
           TiDB         NebulaGraph    TiFlash       Iceberg
             │              │            │             │
        Object Fact      Graph Fact    Analytics     History
        Link Fact        Aggregate     Aggregate     Event
        Action           Inference                  Raw Fact
        Current State    Path
```

Company → Order：

```text
Company
   │
   │ Ontology HAS_ORDER
   │
   ├───────────────────────────────┐
   │                               │
   ▼                               ▼
TiDB                           NebulaGraph
order.company_pk              不存全部Order关系
                                   │
                              PURCHASES_FROM
                                   │
                                   ▼
                               Supplier
```

订单和物流：

```text
Order
   ├──── TiDB：当前状态
   ├──── Iceberg：完整历史
   └──── Graph：活跃/异常订单

Shipment
   ├──── TiDB：当前状态
   ├──── Iceberg：完整历史
   └──── Graph：活跃物流链

LogisticsEvent
   └──── Iceberg
```

---

## 24. 每个 Object Type / Link Type 的设计检查项

创建 Object Type / Link Type 时，不应该只问“有没有 Link”，而应继续判断：

1. 这个 Link 是否需要多跳遍历？
2. 是否需要路径推理？
3. 单个 Source Object 最多可能有多少条 Link？
4. 需要的是单条明细关系还是聚合业务关系？
5. 是否可以在需要时回 TiDB 查询明细？
6. 是否只需要当前活跃数据？
7. 是否只需要异常数据？
8. 是否需要按时间窗口进入 Graph？
9. 是否可能形成 Supernode？
10. 是否需要 Bucket 化？

---

## 25. 最终原则

高基数结构型关系：

```text
Company → Order
Order → LogisticsEvent
Customer → Transaction
Account → Transaction
Equipment → SensorEvent
```

默认采用：

```text
TiDB / Iceberg
+
Inline / Virtual Link
```

关系推理型 Link：

```text
Company → OWNS → Company
Company → CONTROLS → Company
Company → SUPPLIES → Company
Person → DIRECTOR_OF → Company
Company → GUARANTEES → Company
Company → PURCHASES_FROM → Supplier
Supplier → USES_CARRIER → Carrier
```

才是 NebulaGraph 的核心数据。

最终数据职责：

```text
TiDB
=
权威业务事实
+
当前状态
+
事务
+
结构性明细关系

NebulaGraph
=
可重建的关系推理投影
+
聚合业务关系
+
活跃关系
+
异常关系
+
多跳路径

TiFlash
=
当前态大规模统计

Iceberg
=
海量历史事实
+
事件
+
长期明细

Kafka + TiCDC
=
运行态投影同步总线

Ontology
=
统一业务语义模型
```

最终可以概括为：

> **图数据库的规模应该由“推理复杂度”决定，而不是由“业务明细总量”决定。**

这样即使未来达到百亿 Object、万亿 Event，也不需要把全部数据 Graph 化，NebulaGraph 仍然只承担真正适合图模型的关系网络和推理任务。

## 相关笔记

- [[tidb_ontology_sharding_and_partitioning_design|TiDB 分片设计]]
- [[ontology_object_storage_design|对象存储设计]]
- [[palantir_ontology_graph_relational_database_solution|关系+图数据库方案]]
