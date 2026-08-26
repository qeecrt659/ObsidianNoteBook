# TiDB-only Ontology Runtime：仅使用 TiDB/TiFlash 存储与推理的完整解决方案

## 1. 目标与总体原则

本方案面向一个类似 Palantir Foundry 的 Ontology 平台，目标是：

- 不引入图数据库；
- 不引入 Kafka、Doris、OpenSearch 等额外基础设施作为 P0/P1 必选组件；
- 所有设计态元数据、运行态 Object、Link、Action、规则、推理结果均存储在 TiDB；
- 推理不依赖 Instance Graph，而是依赖：
  - Schema Graph
  - Reasoning Rule
  - Semantic Query Planner
  - SQL Generator
  - TiDB / TiFlash 执行

核心原则：

> **Ontology 决定“应该沿什么语义关系推理”，Reasoning Planner 决定“如何生成 SQL”，TiDB 决定“如何高效执行这些 SQL”。**

---

# 2. 总体架构

```text
                         ┌──────────────────────────────┐
                         │       Ontology 设计台        │
                         │                              │
                         │ Object Type                  │
                         │ Property                     │
                         │ Link Type                    │
                         │ Interface                    │
                         │ Action Type                  │
                         │ Reasoning Rule               │
                         │ Data Mapping                 │
                         │ Version / Release            │
                         └──────────────┬───────────────┘
                                        │
                                     Publish
                                        │
                                        ▼
                         ┌──────────────────────────────┐
                         │            TiDB              │
                         │                              │
                         │ Ontology Metadata            │
                         │ Object Instance              │
                         │ Link Instance                │
                         │ Action / Edit                │
                         │ Rule                         │
                         │ Inference Result             │
                         │ Audit                        │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                     ┌──────────────────────────────────┐
                     │       Ontology Runtime           │
                     │                                  │
                     │ Schema Graph                     │
                     │ Semantic Query Parser            │
                     │ Path Finder                      │
                     │ Rule Engine                      │
                     │ Semantic Query Planner           │
                     │ SQL Generator                    │
                     │ Cost / Safety Planner            │
                     └──────────────┬───────────────────┘
                                    │
                        ┌───────────┴────────────┐
                        │                        │
                        ▼                        ▼
                     TiKV                    TiFlash
                        │                        │
                   OLTP / Detail          Join / Aggregate
                   Action / Write         大规模分析推理
                   Point Lookup           MPP
```

P0/P1 可以不引入：

```text
NebulaGraph
Neo4j
JanusGraph
Kafka
OpenSearch
Doris
Redis
```

---

# 3. Ontology 的定位

不要把 Ontology 理解为：

```text
Ontology = Graph Database
```

应该理解为：

```text
Ontology
=
Semantic Model
+
Schema Graph
+
Business Rules
+
Data Mapping
```

Schema Graph 仅描述类型级关系：

```text
Company
  │
  ├── HAS_ORDER ─────► Order
  ├── OWNS ──────────► Company
  └── SUPPLIES ──────► Company

Order
  │
  └── HAS_SHIPMENT ──► Shipment

Shipment
  │
  └── HANDLED_BY ────► Carrier
```

这里没有 Object Instance。

---

# 4. 设计态元数据存储

建议建立逻辑数据库：

```text
ontology_meta
```

核心表：

```text
ontology
ontology_branch
ontology_release

object_type
property_definition
interface_definition

link_type
link_mapping

action_type
action_parameter

reasoning_rule
reasoning_rule_version

physical_mapping

index_definition
runtime_deployment
```

---

# 5. Object Type 表

```sql
CREATE TABLE ontology_object_type (

    id BIGINT PRIMARY KEY AUTO_RANDOM,

    ontology_id BIGINT NOT NULL,

    api_name VARCHAR(128) NOT NULL,

    display_name VARCHAR(256) NOT NULL,

    description TEXT,

    primary_key_property VARCHAR(128),

    physical_table VARCHAR(128),

    status VARCHAR(32),

    version INT NOT NULL,

    created_at DATETIME(6),
    updated_at DATETIME(6),

    UNIQUE KEY uk_object_type (
        ontology_id,
        api_name,
        version
    )
);
```

示例 Object Type：

```text
Company
Order
Shipment
Carrier
Person
Contract
Supplier
```

---

# 6. Property 定义

建议：

```text
ontology_property
─────────────────────────

id
object_type_id

api_name
display_name

data_type

nullable
required

searchable
filterable
aggregatable

physical_column
json_path

is_primary_key
is_title_key

version
```

例如：

```text
Company.name
→ company.company_name

Company.riskLevel
→ company.risk_level

Company.registeredCapital
→ company.registered_capital
```

---

# 7. Object Instance 的物理存储原则

不建议把所有 Object 全部放进一张：

```text
ontology_object
```

并使用：

```text
properties JSON
```

承载全部业务数据。

更合理的做法是：

> **Object Type 是逻辑 Schema；每个大型业务 Object Type 映射到 TiDB 的独立物理表。**

例如：

```text
Company → company
Order → sales_order
Shipment → shipment
Carrier → carrier
Contract → contract
```

---

# 8. Company 物理表

```sql
CREATE TABLE company (

    pk BIGINT PRIMARY KEY AUTO_RANDOM,

    company_id VARCHAR(128) NOT NULL,

    company_name VARCHAR(512),

    credit_code VARCHAR(64),

    province VARCHAR(64),

    industry VARCHAR(128),

    registered_capital DECIMAL(20,2),

    risk_level VARCHAR(32),

    status VARCHAR(32),

    extra_properties JSON,

    version BIGINT NOT NULL DEFAULT 1,

    created_at DATETIME(6),

    updated_at DATETIME(6),

    UNIQUE KEY uk_company_id(company_id),

    KEY idx_company_risk(
        risk_level,
        province
    )
);
```

Ontology Mapping：

```text
ObjectType = Company

physicalTable = company

Company.id
→ company.company_id

Company.name
→ company.company_name

Company.riskLevel
→ company.risk_level
```

---

# 9. Order 物理表

```sql
CREATE TABLE sales_order (

    pk BIGINT PRIMARY KEY AUTO_RANDOM,

    order_id VARCHAR(128) NOT NULL,

    company_pk BIGINT NOT NULL,

    customer_pk BIGINT,

    supplier_pk BIGINT,

    order_status VARCHAR(32),

    order_amount DECIMAL(20,2),

    order_time DATETIME(6),

    risk_level VARCHAR(32),

    version BIGINT NOT NULL DEFAULT 1,

    extra_properties JSON,

    UNIQUE KEY uk_order_id(order_id),

    KEY idx_company_order(
        company_pk,
        order_time
    ),

    KEY idx_supplier_order(
        supplier_pk,
        order_time
    )
);
```

建议：

- 内部 PK 使用 `BIGINT AUTO_RANDOM`
- 业务 ID 单独建立唯一索引
- 避免连续主键形成热点

---

# 10. 动态 Property 策略

推荐：

```text
80% 高频核心 Property
→ 正常 Column

20% 扩展 Property
→ JSON
```

不要：

```text
100% Property → JSON
```

高频查询字段：

```text
riskLevel
province
industry
status
amount
eventTime
```

应该成为独立列。

低频扩展字段可以进入：

```text
extra_properties JSON
```

如果某 JSON 字段后来变成高频字段，可以通过 Generated Column 抽取并建立索引。

示例：

```sql
ALTER TABLE company
ADD COLUMN customer_level VARCHAR(32)
GENERATED ALWAYS AS (
    JSON_UNQUOTE(
        JSON_EXTRACT(
            extra_properties,
            '$.customerLevel'
        )
    )
) STORED;

CREATE INDEX idx_customer_level
ON company(customer_level);
```

---

# 11. Link Type：三种物理存储策略

建议 Link Type 支持三类物理 Mapping：

```text
INLINE_FOREIGN_KEY
JOIN_TABLE
RELATION_OBJECT
```

---

# 12. INLINE_FOREIGN_KEY

适用于：

```text
Company
   ↓ HAS_ORDER
Order
```

不要创建：

```text
5000万条 Link Record
```

直接通过：

```text
sales_order.company_pk
```

表达。

Ontology：

```text
Company --HAS_ORDER--> Order
```

Mapping：

```text
source:
Company.pk

target:
Order.company_pk
```

这类结构关系非常适合：

```text
Company → Order
Order → Shipment
Shipment → Carrier
Department → Employee
```

---

# 13. JOIN_TABLE

适用于多对多关系：

```text
Company
   ↓ SUPPLIES
Company
```

例如：

```sql
CREATE TABLE company_supplier_relation (

    pk BIGINT PRIMARY KEY AUTO_RANDOM,

    buyer_company_pk BIGINT NOT NULL,

    supplier_company_pk BIGINT NOT NULL,

    valid_from DATETIME(6),

    valid_to DATETIME(6),

    relation_status VARCHAR(32),

    UNIQUE KEY uk_supplier_relation(
        buyer_company_pk,
        supplier_company_pk,
        valid_from
    ),

    KEY idx_buyer(
        buyer_company_pk
    ),

    KEY idx_supplier(
        supplier_company_pk
    )
);
```

Ontology Mapping：

```text
Company --SUPPLIES--> Company

linkTable:
company_supplier_relation
```

---

# 14. RELATION_OBJECT

当关系本身拥有重要业务属性时，不应只视为简单 Link。

例如：

```text
Company A
    │
    │ OWNS
    │
    │ 36%
    ▼
Company B
```

需要保存：

```text
shareholding
votingRights
validFrom
validTo
source
confidence
```

建立：

```sql
CREATE TABLE company_ownership (

    pk BIGINT PRIMARY KEY AUTO_RANDOM,

    shareholder_company_pk BIGINT NOT NULL,

    owned_company_pk BIGINT NOT NULL,

    shareholding DECIMAL(10,6),

    voting_rights DECIMAL(10,6),

    valid_from DATE,

    valid_to DATE,

    source_system VARCHAR(128),

    confidence DECIMAL(8,6),

    version BIGINT,

    KEY idx_shareholder(
        shareholder_company_pk
    ),

    KEY idx_owned(
        owned_company_pk
    )
);
```

适合：

```text
股权
担保
合同关系
任职关系
资金关系
```

---

# 15. Schema Graph Runtime

Ontology Release 发布后，Runtime 从 TiDB 加载：

```text
Object Type
Link Type
Link Mapping
Rule
Interface
```

构建 Java 内存中的 Schema Graph。

例如：

```text
Company
 ├── HAS_ORDER ──────► Order
 ├── OWNS ───────────► Company
 └── SUPPLIES ───────► Company

Order
 └── HAS_SHIPMENT ───► Shipment

Shipment
 └── HANDLED_BY ─────► Carrier
```

典型规模：

```text
100～1000个 Object Type
数百～数千个 Link Type
```

这个规模无需任何图数据库。

---

# 16. Reasoning Engine 模块

建议：

```text
Reasoning Engine

├── Semantic Parser
├── Schema Graph
├── Path Finder
├── Rule Engine
├── Query Planner
├── SQL AST Builder
├── Cost Guard
├── TiDB Executor
├── TiFlash Executor
└── Explain Engine
```

运行过程：

```text
用户查询
   ↓
理解业务语义
   ↓
找到 Object Type
   ↓
Schema Graph 找路径
   ↓
选择 Reasoning Rule
   ↓
解析 Link Mapping
   ↓
生成 SQL AST
   ↓
估算语义成本
   ↓
TiKV / TiFlash
   ↓
执行
   ↓
Object Set + Evidence
```

---

# 17. 第一类推理：固定路径推理

问题：

> A 公司有哪些订单使用了高风险物流公司？

Schema Graph：

```text
Company
   ↓ HAS_ORDER
Order
   ↓ HAS_SHIPMENT
Shipment
   ↓ HANDLED_BY
Carrier
```

Path Finder：

```text
Company
→ Order
→ Shipment
→ Carrier
```

Mapping：

```text
Order.company_pk
Shipment.order_pk
Shipment.carrier_pk
```

生成 SQL：

```sql
SELECT DISTINCT
    o.order_id,
    o.order_status,
    s.shipment_id,
    c.carrier_id,
    c.carrier_name,
    c.risk_level

FROM company co

JOIN sales_order o
    ON o.company_pk = co.pk

JOIN shipment s
    ON s.order_pk = o.pk

JOIN carrier c
    ON c.pk = s.carrier_pk

WHERE
    co.company_id = ?
AND
    c.risk_level = 'HIGH';
```

核心：

> **Schema Graph 发现路径，TiDB 执行实例推理。**

---

# 18. 第二类：多跳固定 Schema 推理

问题：

> 某供应商停产会影响哪些最终客户？

Ontology：

```text
Supplier
   ↓ SUPPLIES_PRODUCT
Product
   ↓ USED_BY
Company
   ↓ PRODUCES
Product
   ↓ USED_BY
Company
```

Path Finder 可以限定：

```text
depth <= 4
```

然后生成：

```text
JOIN
+
JOIN
+
JOIN
+
JOIN
```

最终由 TiDB Optimizer 决定：

```text
Join Order
Index Join
Hash Join
Merge Join
TiKV / TiFlash
```

Semantic Planner 不应该自己重造完整数据库 Optimizer。

---

# 19. 第三类：递归推理

例如：

> A 公司最终控制哪些企业？

数据：

```text
A → B 80%
B → C 70%
B → D 30%
C → E 60%
```

物理表：

```text
company_ownership
```

使用 TiDB Recursive CTE：

```sql
WITH RECURSIVE control_chain AS (

    SELECT
        shareholder_company_pk AS root_company,
        owned_company_pk,
        shareholding,
        1 AS depth

    FROM company_ownership

    WHERE shareholder_company_pk = ?
      AND shareholding > 0.5

    UNION ALL

    SELECT
        cc.root_company,
        co.owned_company_pk,
        co.shareholding,
        cc.depth + 1

    FROM control_chain cc

    JOIN company_ownership co
      ON co.shareholder_company_pk =
         cc.owned_company_pk

    WHERE
        co.shareholding > 0.5

    AND cc.depth < 8
)

SELECT DISTINCT owned_company_pk
FROM control_chain;
```

结果：

```text
A controls B
A controls C
A controls E
```

---

# 20. Recursive Reasoning 的保护机制

绝对不要允许：

```text
depth = unlimited
```

建议：

```text
默认 maxDepth = 5
```

特殊 Rule：

```text
maxDepth = 8
```

极少允许：

```text
maxDepth > 10
```

必须同时限制：

```text
Cycle Detection
Visited Object
Max Result Count
Max Intermediate Rows
Timeout
Cost Limit
```

例如：

```text
A → B
B → C
C → A
```

必须防止无限递归。

---

# 21. 第四类：规则推理

推理不一定需要关系遍历。

例如：

```text
Company.riskLevel = HIGH
AND
Company.debtRatio > 80%
AND
Company.guaranteeAmount > 5亿

→

CriticalRiskCompany
```

Rule：

```text
CRITICAL_COMPANY_RULE
```

生成：

```sql
SELECT *
FROM company

WHERE risk_level = 'HIGH'

AND debt_ratio > 0.8

AND guarantee_amount > 500000000;
```

---

# 22. 第五类：关系 + 规则联合推理

例如：

> 找出由高风险企业实际控制，同时又给其他国企提供担保的公司。

Schema：

```text
Company
  ↓ CONTROLS
Company
  ↓ GUARANTEES
Company
```

Rule：

```text
source.riskLevel = HIGH
AND
control > 50%
```

Planner：

```text
Path
+
Predicate
+
Recursive Rule
```

编译成：

```text
Recursive CTE
+
JOIN
+
WHERE
```

---

# 23. Reasoning Rule 不要直接保存 SQL

不要：

```text
reasoning_rule.sql = "SELECT..."
```

应该保存 Semantic Rule。

例如：

```json
{
  "ruleId": "actual-control",

  "inputType": "Company",

  "traverse": {
    "linkType": "OWNS",
    "direction": "OUT",
    "recursive": true,
    "maxDepth": 6
  },

  "condition": {
    "property": "shareholding",
    "operator": "GT",
    "value": 0.5
  },

  "outputRelation": "CONTROLS"
}
```

SQL 是：

```text
Compiled Artifact
```

而 Semantic Rule 才是：

```text
Source of Truth
```

这样数据库表结构变化时，只需要修改 Mapping / Compiler。

---

# 24. 推理结果策略

分两类。

## 24.1 即席推理

例如：

> A 与 B 有什么关系？

直接运行：

```text
不持久化
```

返回 Query Result。

## 24.2 高频推理

例如：

```text
最终控制关系
最终受益人
企业风险等级
供应链影响关系
```

建议 Materialize：

```sql
CREATE TABLE inferred_relation (

    pk BIGINT PRIMARY KEY AUTO_RANDOM,

    ontology_id BIGINT,

    rule_id BIGINT,

    rule_version INT,

    source_object_type_id BIGINT,

    source_object_pk BIGINT,

    target_object_type_id BIGINT,

    target_object_pk BIGINT,

    relation_type VARCHAR(128),

    confidence DECIMAL(8,6),

    evidence JSON,

    computed_at DATETIME(6),

    valid_until DATETIME(6),

    UNIQUE KEY uk_inference (
        rule_id,
        rule_version,
        source_object_pk,
        target_object_pk,
        relation_type
    )
);
```

第一次：

```text
推理
```

后续：

```text
直接查询 inferred_relation
```

---

# 25. Evidence 必须保存

尤其是监管、风控、审计场景。

不要只保存：

```text
A controls E
```

应该保存：

```json
{
  "result": "A CONTROLS E",

  "rule": "CONTROL_RULE_V3",

  "evidence": [
    {
      "from": "A",
      "to": "B",
      "shareholding": 0.8
    },
    {
      "from": "B",
      "to": "C",
      "shareholding": 0.7
    },
    {
      "from": "C",
      "to": "E",
      "shareholding": 0.6
    }
  ]
}
```

这样 Explain Engine 才能回答：

> 为什么系统认为 A 控制 E？

---

# 26. FACT 与 INFERRED 必须分离

事实：

```text
A 持股 B 60%
```

存：

```text
company_ownership
```

推理：

```text
A CONTROLS B
```

存：

```text
inferred_relation
```

原则：

```text
事实层
→ Authoritative

推理层
→ Rebuildable
```

绝对不要把推理结果写回事实表。

---

# 27. Action 全部使用 TiDB 事务

例如：

```text
ApproveOrder
ChangeRiskLevel
AddSupplier
CreateGuarantee
```

流程：

```text
Action Service
      ↓
TiDB Transaction
```

事务内：

```text
action_instance
object_edit
link_edit
object_current
audit_event
```

例如：

```text
BEGIN;

校验权限
校验状态
校验版本

写 action_instance

更新业务 Object

写 object_edit / link_edit

写 audit_event

COMMIT;
```

---

# 28. Action 并发策略

普通业务 Action：

```text
审批
修改状态
创建关系
删除关系
```

可优先采用 TiDB 悲观事务。

高并发对象修改可以使用应用层版本控制。

例如：

```text
version = 103
```

更新：

```sql
UPDATE sales_order

SET
    order_status = 'APPROVED',
    version = 104

WHERE pk = ?
AND version = 103;
```

如果：

```text
affectedRows = 0
```

返回：

```text
OBJECT_VERSION_CONFLICT
```

---

# 29. TiFlash 的角色

只用 TiDB 不代表所有查询都应该打 TiKV。

建议职责：

```text
TiKV
=
事务
点查
小范围查询
Action
小 Object Set

TiFlash
=
大规模 Join
Aggregation
大 Object Set
分析型推理
```

从应用视角仍然是一套 TiDB SQL 接口。

---

# 30. 哪些表创建 TiFlash Replica

优先：

```text
company

company_ownership

company_supplier_relation

sales_order

shipment

contract

payment_summary

inferred_relation
```

元数据表通常不需要。

例如：

```sql
ALTER TABLE company
SET TIFLASH REPLICA 1;

ALTER TABLE company_ownership
SET TIFLASH REPLICA 1;

ALTER TABLE sales_order
SET TIFLASH REPLICA 1;
```

---

# 31. 三类 Query Path

## FAST PATH

```text
Point Lookup
Small Join
Small Object Set
       ↓
TiKV
```

## ANALYTIC PATH

```text
Large Join
Group By
Aggregate
Large Object Set
       ↓
TiFlash
```

## RECURSIVE PATH

```text
Recursive Relation
Path Reasoning
       ↓
TiDB Recursive CTE
```

Semantic Query Planner 负责语义规划。

TiDB Optimizer 负责物理执行优化。

---

# 32. 不要自己重造完整数据库 Optimizer

Semantic Planner 只负责：

```text
Object Type
↓
Table

Property
↓
Column

Link Type
↓
Join Condition

Reasoning Rule
↓
Predicate / Recursive Structure
```

至于：

```text
Join Order
Hash Join
Index Join
Broadcast
Shuffle
```

尽量交给 TiDB CBO。

---

# 33. Semantic Cost Guard

Semantic Planner 应关注：

```text
depth
link cardinality
estimated object count
recursive flag
cross-domain
aggregation
```

例如：

```text
Company
→ Order
```

某公司可能：

```text
5000万 Order
```

Planner 不允许默认直接展开。

必须要求：

```text
Filter
Limit
Aggregation
```

例如：

```text
最近30天
riskLevel=HIGH
LIMIT 100
```

---

# 34. Cardinality Metadata

Link Type 建议保存：

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
```

同时增加：

```text
estimatedAvgCardinality
estimatedP95Cardinality
estimatedMaxCardinality
```

例如：

```text
Company --HAS_ORDER--> Order

avg = 50000
P95 = 1000000
max = 50000000
```

Planner 看到：

```text
HAS_ORDER
```

即可判断：

```text
EXPENSIVE_LINK
```

---

# 35. Link Query Policy

例如：

```json
{
  "linkType": "HAS_ORDER",

  "queryPolicy": {

    "maxDirectRows": 10000,

    "requireFilter": true,

    "defaultLimit": 100,

    "allowRecursive": false,

    "preferAggregation": true
  }
}
```

而：

```text
Company --OWNS--> Company
```

可以：

```json
{
  "queryPolicy": {

    "maxDirectRows": 1000,

    "allowRecursive": true,

    "maxDepth": 8
  }
}
```

---

# 36. Statistics 必须维护

生产环境必须持续维护：

```text
Auto Analyze
+
关键表定期 ANALYZE
```

尤其：

```text
company
sales_order
company_ownership
shipment
company_supplier_relation
```

Semantic Planner 决定语义路径。

TiDB Statistics 与 CBO 决定 SQL 物理执行效率。

---

# 37. Partition 策略

TiDB 本身底层已经基于 TiKV Region 分布式存储，因此不要因为表大就机械 Partition。

Partition 主要用于：

```text
生命周期
历史删除
查询裁剪
租户隔离
```

例如：

```text
Action History
Audit Event
Inference History
```

可以按月 Range Partition。

当前态表：

```text
company
sales_order
shipment_current
```

优先使用：

```text
AUTO_RANDOM
+
正确索引
+
TiKV Region
```

不要过早设计大量 Partition。

---

# 38. 资源隔离

虽然只用 TiDB，但仍应区分不同工作负载：

```text
ontology_action
HIGH

ontology_query
MEDIUM

ontology_reasoning
MEDIUM / LOW

ontology_batch_inference
LOW

ontology_admin
LOW
```

目标：

```text
复杂推理
不能拖慢
Action / Object Point Query
```

---

# 39. Online Reasoning 与 Batch Reasoning

## Online Reasoning

例如：

> A 公司实际控制哪些企业？

建议：

```text
目标响应 < 1～3 秒

depth <= 5
result <= 1000
```

超出则拒绝、缩小范围或转异步 Job。

## Batch Reasoning

例如：

> 计算全国全部企业最终控制关系。

不能用一个 HTTP 请求执行。

应使用：

```text
Reasoning Job
↓
按 Company PK 分批
↓
多个 Worker
↓
执行 SQL
↓
写 inferred_relation
```

仍然全部使用 TiDB。

---

# 40. Reasoning Job

建议：

```text
reasoning_job
──────────────────

job_id

rule_id
rule_version

scope

status

total_objects

processed_objects

success_count
failure_count

started_at
completed_at
```

再建立：

```text
reasoning_job_partition
```

例如：

```text
Company PK

0~100万
100万~200万
...
```

多个 Worker 并行处理。

---

# 41. Version 与推理结果绑定

例如：

```text
Ontology Version = 1.6

CONTROL Rule Version = 3
```

推理结果必须记录：

```text
ontology_version = 1.6
rule_version = 3
```

Rule v4 发布后重新推理。

不要覆盖 v3。

这样系统可以解释：

> 为什么昨天认为 A 控制 B，今天结果变化了？

可能原因：

```text
Rule Version 变化
Fact Version 变化
Ontology Version 变化
```

---

# 42. Explainable Reasoning

一次推理结果建议包含：

```text
Inference

├── Result
├── Rule
├── Rule Version
├── Ontology Version
├── Input Objects
├── Input Links
├── Evidence
├── SQL Plan ID
├── Data Snapshot / Time
├── Confidence
└── Generated At
```

监管、审计、风控场景应把 Explainability 作为 P0/P1 能力，而不是后补。

---

# 43. 穿透监管示例

问题：

> 查询央企 A 最终控制的所有企业，并找出其中与高风险供应商存在交易的企业。

Schema Graph：

```text
Company
  ↓ OWNS*
Company
  ↓ PURCHASES_FROM
Supplier
```

Rule：

```text
OWNS.shareholding > 50%
```

第一阶段：

```text
Recursive CTE
```

计算：

```text
A
→ B
→ C
→ D
```

第二阶段：

```text
ControlledCompany
JOIN
SupplierRelation
JOIN
Supplier
```

第三阶段：

```text
Supplier.riskLevel = HIGH
```

最终 SQL 逻辑：

```text
Recursive CTE
+
JOIN
+
WHERE
```

---

# 44. 大规模推理交给 TiFlash

例如：

```text
ControlledCompany = 100万

Supplier Relation = 10亿

Supplier = 5000万
```

此时大规模：

```text
JOIN
+
GROUP BY
```

应由 TiFlash MPP 承担。

从架构上依然：

```text
Semantic Planner
→ SQL
→ TiDB
→ TiFlash MPP
```

无需增加图数据库。

---

# 45. TiDB-only 推理能力边界

| 推理能力 | TiDB-only 适合度 |
|---|---:|
| Object 属性过滤 | ★★★★★ |
| Link 一跳 | ★★★★★ |
| 固定路径 2～5 跳 | ★★★★★ |
| 关系属性判断 | ★★★★★ |
| 聚合推理 | ★★★★★ |
| Rule Engine | ★★★★★ |
| 股权递归 | ★★★★☆ |
| 组织层级递归 | ★★★★☆ |
| 供应链固定路径 | ★★★★☆ |
| 风险规则 | ★★★★★ |
| 大规模 Join | ★★★★★（TiFlash） |
| 任意关系 5～10 跳探索 | ★★☆☆☆ |
| 所有可能路径 | ★★☆☆☆ |
| 最短路径 | ★★☆☆☆ |
| PageRank | ★☆☆☆☆ |
| Community Detection | ★☆☆☆☆ |

因此：

> **TiDB-only 非常适合“业务规则推理”和“有语义约束的路径推理”，不适合无限制通用图算法。**

---

# 46. 为什么这符合企业 Ontology 的主要需求

企业实际推理通常是：

```text
谁控制谁？
谁给谁担保？
谁给谁供货？
哪个订单用了哪个物流？
风险怎么传播？
哪个供应商影响哪些生产线？
哪些合同违反规则？
```

这类问题通常都有：

```text
Object Type
Link Type
Rule
```

因此非常适合：

```text
Semantic Planner
→ SQL
→ TiDB / TiFlash
```

---

# 47. 什么时候未来再考虑图数据库

只有当核心需求变成：

```text
不知道路径是什么

从任意 Object 出发

任意 Link Type

depth 8～15

寻找所有路径

最短路径

环路分析

社区发现

中心性计算

图算法
```

此时才重新评估专用 Graph Engine。

P0/P1 不应该为了未来可能存在的需求而增加复杂度。

---

# 48. 最终目标架构

```text
                   Ontology Platform
                          │
          ┌───────────────┴────────────────┐
          │                                │
       Design Time                      Runtime
          │                                │
          ▼                                ▼
   Ontology Metadata                Schema Graph
          │                          Reasoning Rule
          │                          Query Planner
          └───────────────┬────────────────┘
                          │
                          ▼
                        TiDB
           ┌──────────────┼───────────────┐
           │              │               │
           ▼              ▼               ▼
       Metadata       Business Data   Inference Data
           │              │               │
      Object Type      Company          Results
      Link Type        Order            Evidence
      Action Type      Shipment         Audit
      Rule             Contract
      Mapping          Link Fact
                           │
                   ┌───────┴───────┐
                   │               │
                   ▼               ▼
                 TiKV           TiFlash
                   │               │
              OLTP / Point      OLAP / MPP
              Action            Join
              Small Query       Aggregate
                                Large Reasoning
```

---

# 49. Runtime 核心模块

建议：

```text
Ontology Metadata Service
        ↓
Ontology Compiler
        ↓
Schema Graph Runtime
        ↓
Semantic Query Parser
        ↓
Path Finder
        ↓
Reasoning Rule Engine
        ↓
Semantic Query Planner
        ↓
SQL AST Generator
        ↓
Query Cost Guard
        ↓
TiDB Executor
        ↓
Inference Explain Engine
```

最关键的 P0 模块：

1. **Ontology Compiler**
   - 把设计态模型编译成 Runtime Model。

2. **Schema Graph Runtime**
   - 在内存中维护 Object Type 与 Link Type 的逻辑图。

3. **Path Finder**
   - 根据 Object Type、Link Type 找合法语义路径。

4. **Link Mapping Resolver**
   - 将 Link Type 映射到 FK、Join Table、Relation Object。

5. **Reasoning Rule Engine**
   - 处理 Predicate、递归、聚合、关系推导。

6. **Semantic Query Planner**
   - 将语义路径与规则转换成 SQL 逻辑计划。

7. **SQL AST Generator**
   - 生成可控、可审计的 SQL，而不是字符串拼接。

8. **Query Cost Guard**
   - 防止高基数 Link、深递归、超大 Object Set 直接展开。

9. **Inference Explain Engine**
   - 输出 Rule、Evidence、Version、Execution 信息。

---

# 50. 最终建议

如果目标是建设类似 Palantir 的企业级 Ontology 平台，P0/P1 建议完全不使用图数据库。

采用：

```text
TiDB
+
TiKV
+
TiFlash
+
Java Ontology Runtime
```

数据模型：

```text
Object Type
→ TiDB 物理表

Property
→ Column / JSON Path

1:N Link
→ Foreign Key Mapping

N:N Link
→ Join Table

复杂关系
→ Relationship Object

固定多跳推理
→ JOIN

递归关系推理
→ WITH RECURSIVE

大规模推理
→ TiFlash MPP

高频推理结果
→ Materialized Inference Table

设计态逻辑图
→ Runtime 内存 Schema Graph
```

最终核心原则：

> **Ontology 管语义，TiDB 管事实，Reasoning Planner 管推理计划，TiFlash 管大规模分析执行。**

这使系统在保持 Palantir 风格 Object Type / Link Type / Rule / Action 语义抽象的同时，只维护一套数据库技术栈，显著降低开发、运维和数据一致性复杂度。

---

## 参考的行业实践方向

本方案借鉴并融合以下通用实践：

- Palantir Foundry 的 Object Type / Link Type / backing datasource 语义建模思路；
- 关系模型中的 Foreign Key、Join Table、Relation Object 模式；
- Semantic Layer / Logical Model 与 Physical Storage 解耦；
- Rule Engine 将语义规则作为 Source of Truth，而 SQL 作为 Compiled Artifact；
- Query Planner 与数据库物理 Optimizer 分工；
- Online Reasoning 与 Batch Reasoning 分离；
- FACT 与 INFERRED 分层；
- 推理结果 Evidence / Version / Explainability；
- TiDB 分布式 OLTP + TiFlash MPP 的 HTAP 架构；
- 高基数关系 Query Guard 与递归深度限制。
