# NebulaGraph 作为 Ontology Runtime 实例存储的架构建议

## 1. 结论

可以。

对于以 Palantir-style Ontology 为目标的 Ontology Runtime，**Object Instance 和 Link Instance 全部存入 NebulaGraph 是合理且自然的设计**。

但需要明确区分：

- **所有 Object / Link 实例都放入 NebulaGraph**
- **整个 Ontology 平台只使用 NebulaGraph 一个数据库**

前者可行且推荐；后者通常不推荐。

更合理的设计是：

- Ontology Definition / Metadata：TiDB 或 PostgreSQL
- Object Instance / Link Instance：NebulaGraph
- 大文件、文档、图片、视频：对象存储
- 全文检索：OpenSearch
- 强事务业务状态：TiDB / PostgreSQL 或独立 Transaction Runtime

---

## 2. Ontology 与 NebulaGraph 的直接映射关系

NebulaGraph 是 Property Graph 数据库，因此 Ontology 的实例模型可以自然映射到图模型。

| Ontology 概念 | NebulaGraph 映射 |
|---|---|
| Object Type | Tag |
| Object Instance | Vertex |
| Object Primary Key | VID |
| Object Property | Vertex Property |
| Link Type | Edge Type |
| Link Instance | Edge |
| Link Property | Edge Property |
| Link Source | src VID |
| Link Target | dst VID |

例如 Ontology 定义：

```text
Object Type
├── Customer
├── Order
└── Product

Link Type
├── PLACED
└── CONTAINS
```

实例关系：

```text
Customer: C001
    │
    └── PLACED
           │
           ▼
       Order: O1001
           │
           └── CONTAINS
                  │
                  ▼
            Product: P900
```

在 NebulaGraph 中可以直接表达为：

```text
(Customer:C001)
       │
       │ PLACED
       ▼
(Order:O1001)
       │
       │ CONTAINS
       ▼
(Product:P900)
```

---

## 3. 推荐的总体架构

建议把 Ontology Definition 和 Ontology Runtime 分离。

```text
                   Ontology Platform
                          │
              ┌───────────┴────────────┐
              │                        │
        Ontology Definition      Ontology Runtime
              │                        │
      Object Type                       │
      Link Type                         │
      Property                          │
      Action Type                       │
      Rules                             │
              │                         │
              ▼                         ▼
       Metadata Store             NebulaGraph
                                     │
                       ┌─────────────┴─────────────┐
                       │                           │
                    Object                      Link
                    Instance                   Instance
                       │                           │
                     Vertex                      Edge
```

### Definition 层

Definition 层主要存：

- Object Type
- Property 定义
- Link Type
- Action Type
- Rules
- Constraint
- Display Metadata
- Version
- Schema Lifecycle

推荐存储：

```text
TiDB / PostgreSQL
```

### Runtime Instance 层

Runtime 实例层主要存：

- Customer
- Order
- Product
- Supplier
- Employee
- Contract
- Device
- Site
- Organization
- 各种 Link Instance

推荐存储：

```text
NebulaGraph
```

NebulaGraph 在这里承担：

> Graph-native Ontology Instance Store

---

## 4. Object Property 可以直接存入 NebulaGraph

例如定义 Object Type：

```text
Customer

customerId: string
name: string
level: int
revenue: double
country: string
createdAt: datetime
```

在 NebulaGraph 中可以映射为：

```ngql
CREATE TAG Customer(
    name string,
    level int,
    revenue double,
    country string,
    createdAt datetime
);
```

实例：

```text
VID = customer:100001

name       = Huawei
level      = 1
revenue    = ...
country    = CN
createdAt  = ...
```

因此常规业务 Property 没有必要再额外跳到关系数据库查询。

这对于多跳图遍历尤其有价值。

---

## 5. Link Instance 更适合存入 NebulaGraph

例如：

```text
Customer C001
     │
     │ PLACED
     ▼
Order O001
```

Link 本身还可以拥有 Property：

```text
PLACED

orderTime
channel
region
operator
```

可以定义：

```ngql
CREATE EDGE PLACED(
    orderTime datetime,
    channel string,
    region string
);
```

形成：

```text
Customer
    │
    │ PLACED
    │ orderTime = ...
    │ channel = WEB
    ▼
Order
```

因此 Ontology Link Instance 与 NebulaGraph Edge 非常匹配。

---

## 6. Link ID 的设计注意事项

NebulaGraph 的 Edge 并不是依赖独立 EID 唯一标识。

一条 Edge 通常由以下组合确定：

```text
<src VID, edge type, rank, dst VID>
```

例如：

```text
C001
+
PLACED
+
100
+
O001
```

因此建议自己的 Ontology 平台仍然为每条 Link 增加业务级：

```text
link_id
```

例如：

```text
PLACED
-----------------
link_id
created_at
updated_at
source_system
```

需要注意：

> link_id 是 Ontology / 业务层 ID，不是 NebulaGraph 原生 Edge ID。

---

## 7. 如果一个 Link 本身是业务实体，应建模成 Object

这是 Ontology 建模中非常重要的一条规则。

简单关系：

```text
Customer ──BOUGHT──> Product
```

可以直接使用 Link / Edge。

但如果“购买”本身有大量业务属性：

```text
Purchase
------------------
purchaseId
quantity
price
currency
tax
discount
paymentMethod
invoiceId
timestamp
status
```

此时更合理的是：

```text
Customer
    │
    │ MADE
    ▼
Purchase
    │
    │ OF
    ▼
Product
```

即：

```text
Purchase = Object
```

而不是：

```text
Purchase = Link
```

原则：

> 当一段关系拥有独立身份、生命周期、复杂属性或需要被其他对象引用时，应优先提升为 Object Type。

---

## 8. 多值 / 复杂 Property 的处理

对于类似：

```json
{
  "customerId": "1001",
  "name": "Huawei",
  "phones": [
    "138...",
    "139..."
  ]
}
```

不建议简单把复杂集合全部塞成单一 Property。

更适合 Ontology 的表达方式是：

```text
Customer
   │
   ├── HAS_PHONE ──> Phone1
   │
   └── HAS_PHONE ──> Phone2
```

即：

```text
多值复杂属性
→
Object + Link
```

这也更符合 Ontology 的语义建模思想。

---

## 9. 大字段和文件不要直接存 NebulaGraph

例如：

- PDF
- Word
- Image
- Video
- CAD
- Source Code
- 超大 JSON
- 大型二进制对象

不建议：

```text
Vertex
  property.file = 100MB Binary
```

推荐：

```text
                       Object
                         │
                 NebulaGraph Vertex
                         │
              ┌──────────┴──────────┐
              │                     │
       normal properties       file reference
                                    │
                                    ▼
                              Object Storage
                              S3 / OSS / ...
```

例如：

```text
Contract Object

contractId = C001
title = xxx
amount = 10,000,000
fileUri = s3://ontology/contracts/C001.pdf
```

NebulaGraph 中只存：

```text
contractId
title
amount
fileUri
```

真正文件放对象存储。

---

## 10. 全文检索能力应独立设计

对于：

- 大文本 Property
- 文档内容
- 模糊搜索
- 中文全文检索
- 多字段相关性排序

建议：

```text
OpenSearch
```

架构：

```text
NebulaGraph
     │
     ├── Object / Link Graph Query
     │
     └── Structured Properties

OpenSearch
     │
     └── Full-text Search
```

Ontology Runtime 再在 Service Layer 统一提供查询能力。

---

## 11. NebulaGraph 不应承担全部事务责任

如果未来存在 Action：

```text
创建 Order
+
修改 Customer.balance
+
减少 Inventory.stock
+
创建多条 Link
+
写 Payment
```

并要求：

```text
BEGIN

A
B
C
D
E

COMMIT
```

且必须满足：

```text
全部成功
或
全部失败
```

则不能简单把 NebulaGraph 当成传统关系型事务数据库使用。

因此完整的 Ontology Platform 应把：

```text
Graph Runtime
```

和：

```text
Transactional Action Runtime
```

分开设计。

推荐：

```text
Action Runtime
     │
     ├── Validation
     ├── Rules
     ├── Permission
     ├── Transaction
     ├── Workflow
     └── Audit
             │
             ├── TiDB / PostgreSQL
             └── NebulaGraph
```

也就是说：

> NebulaGraph 管图状态和图遍历；强事务流程由独立 Runtime 或事务数据库协调。

---

## 12. 为什么更推荐“实例全部进入图”

一种常见设计是：

```text
TiDB
存 Object Instance

NebulaGraph
只存关系
```

查询流程会变成：

```text
NebulaGraph
     │
找到 ID
     │
     ▼
TiDB
     │
查询属性
```

如果需要多跳：

```text
Nebula
   ↓
100 IDs
   ↓
TiDB
   ↓
Properties
   ↓
Nebula
   ↓
Next Hop
   ↓
TiDB
```

这会显著增加 Runtime 的复杂度和数据同步问题。

如果把 Object 常用属性也放入 NebulaGraph：

```text
Customer
   ↓
Order
   ↓
Product
   ↓
Supplier
   ↓
Factory
```

则很多查询可以直接在图数据库完成：

```text
Graph Traversal
+
Property Filter
+
Property Projection
```

因此对于 Ontology Runtime：

> Graph 应是一等 Instance Store，而不只是一个“关系索引”。

---

## 13. 推荐的最终存储分层

| 数据 | 推荐存储 |
|---|---|
| Object Type 定义 | TiDB / PostgreSQL |
| Property 定义 | TiDB / PostgreSQL |
| Link Type 定义 | TiDB / PostgreSQL |
| Action Type | TiDB / PostgreSQL |
| Rules | TiDB / PostgreSQL |
| Constraint / Schema Version | TiDB / PostgreSQL |
| Object Instance | **NebulaGraph Vertex** |
| Object 常用 Property | **NebulaGraph Vertex Property** |
| Link Instance | **NebulaGraph Edge** |
| Link Property | **NebulaGraph Edge Property** |
| 多值复杂 Property | Object + Link |
| 超大文本 | Object Storage / OpenSearch |
| PDF / Image / Video | Object Storage |
| 全文索引 | OpenSearch |
| 强事务业务状态 | TiDB / PostgreSQL / Transaction Runtime |

---

## 14. 推荐的整体技术架构

```text
                         Ontology Platform
                                │
             ┌──────────────────┼───────────────────┐
             │                  │                   │
      Ontology Metadata    Ontology Instance      Blob / Search
             │                  │                   │
             ▼                  ▼                   ▼
        TiDB / PG          NebulaGraph       S3 + OpenSearch
                                │
                         ┌──────┴──────┐
                         │             │
                      Object          Link
                      Vertex          Edge
                         │             │
                    Properties    Properties
```

未来增加 Action Runtime 后：

```text
                      Ontology Platform
                             │
        ┌────────────────────┼─────────────────────┐
        │                    │                     │
  Definition Runtime    Graph Runtime        Action Runtime
        │                    │                     │
    TiDB / PG            NebulaGraph            TiDB / PG
                             │                     │
                         Object/Link          Transaction
                                             Rules
                                             Validation
                                             Audit
                                             Workflow
```

---

## 15. 最终建议

对于 Palantir-style Ontology Runtime：

### 推荐

```text
所有 Object Instance
        +
所有 Link Instance
        ↓
   NebulaGraph
```

即：

```text
Object Instance → Vertex
Link Instance   → Edge
```

NebulaGraph 作为：

> **Graph-native Ontology Instance Store**

### 不推荐

整个 Ontology 平台只依赖：

```text
NebulaGraph
```

更合理的是：

```text
            Ontology Platform
                   │
          ┌────────┼─────────┐
          │        │         │
       Metadata  Instance   Blob
          │        │         │
          ▼        ▼         ▼
       TiDB/PG  NebulaGraph  S3
                  │
              ┌───┴───┐
              │       │
            Object   Link
              │       │
            Vertex   Edge
```

最终架构原则可以概括为：

> **Definition 使用 Metadata Store，Instance 使用 NebulaGraph，Blob 使用对象存储，全文搜索使用 OpenSearch，强事务由独立 Transaction / Action Runtime 负责。**

这样既保留图数据库在多跳关系查询中的优势，又不会让 NebulaGraph 承担它不擅长的 Metadata、Blob、全文搜索和强事务职责。
