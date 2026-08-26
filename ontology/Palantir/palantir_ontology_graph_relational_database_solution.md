# Palantir 风格 Ontology：关系型数据库 + 图数据库技术方案

## 1. 核心设计原则

如果目标是构建一个类似 Palantir Foundry 的 Ontology Runtime，并进一步支持业务推理，建议明确采用：

> **关系型数据库保存“事实与状态”，图数据库保存“关系网络与推理投影”。**

不要把所有 Ontology Object 全量复制成重属性图，更不要让图数据库成为业务主库。

可以将两类存储的职责理解为：

- **PostgreSQL**：回答“这个对象是什么？当前状态是什么？”
- **Graph Database**：回答“这个对象和谁有关？通过什么路径有关？这种关系意味着什么？”

Ontology 才是真正的逻辑模型，Graph Database 应当只是 Ontology 的一种 Runtime Projection。

---

## 2. 哪些数据放关系型数据库

建议 PostgreSQL 保存完整的 Ontology Runtime Object。

例如企业对象：

```json
{
  "objectType": "Company",
  "objectId": "COMPANY-001",
  "name": "上海XX科技有限公司",
  "registeredCapital": 50000000,
  "status": "ACTIVE",
  "industry": "Software",
  "registeredAddress": "...",
  "establishedDate": "2020-01-01",
  "creditCode": "...",
  "employeeCount": 356,
  "revenue": 182000000,
  "riskLevel": "MEDIUM"
}
```

这些属于对象本身的完整属性，应保存在 PostgreSQL，而不是主要存入图数据库。

---

## 3. 哪些数据需要进入图数据库

判断标准：

> **如果业务问题需要“顺着关系继续找关系”，就应该考虑进入图数据库。**

例如：

```text
A公司
 ↓ 持股
B公司
 ↓ 控股
C公司
 ↓ 投资
D公司
 ↓ 法人
张三
 ↓ 同时担任董事
E公司
```

典型图推理问题包括：

- A 公司最终控制哪些企业？
- 张三通过哪些企业与 A 集团有关？
- 供应商 X 和集团内部员工之间是否存在隐藏关系？
- 某风险公司通过几层股权关系影响了哪些子公司？
- 某设备故障会沿供应链影响哪些订单？
- 某账户与哪些异常账户形成资金闭环？

---

# 4. 图数据库建议存储的五类数据

## 4.1 核心实体节点

例如：

```text
Company
Person
Organization
Account
Asset
Project
Contract
Supplier
Customer
Equipment
Location
Product
```

但图数据库中不要保存对象的所有属性。

### PostgreSQL 中的完整对象

```text
Company
─────────────────────
object_id
name
registered_capital
address
industry
employee_count
revenue
profit
risk_level
...
```

### Graph 中的轻量节点

```text
(:Company {
    objectId: "COMP-001",
    name: "XX科技",
    status: "ACTIVE",
    riskLevel: "HIGH"
})
```

即：

> **Graph Node 是 Object 的轻量投影。**

完整属性仍然从 Object Runtime 获取。

---

## 4.2 有推理价值的 Link

这是图数据库最核心的数据。

### 企业关系

```text
Company
   │
   ├── OWNS ──────────► Company
   ├── CONTROLS ──────► Company
   ├── INVESTS_IN ────► Company
   ├── SUPPLIES ──────► Company
   ├── SIGNED ────────► Contract
   └── OPERATES ──────► Project
```

### 人员关系

```text
Person
   │
   ├── LEGAL_REP_OF ──► Company
   ├── DIRECTOR_OF ───► Company
   ├── EMPLOYEE_OF ───► Company
   ├── BENEFICIARY_OF ► Company
   └── OWNS ──────────► Company
```

### 资金关系

```text
Account
   │
   ├── TRANSFERRED_TO ─► Account
   └── OWNED_BY ───────► Company
```

### 供应链关系

```text
Supplier
   ↓ SUPPLIES
Product
   ↓ USED_BY
Factory
   ↓ PRODUCES
Product
   ↓ USED_IN
Order
```

这些 Link Type 都很适合物化进图数据库。

---

## 4.3 关系本身的重要属性

关系不能只保存 A → B，还需要保存其业务属性。

例如：

```json
{
  "shareholding": 0.36,
  "votingRights": 0.51,
  "startDate": "2022-01-01",
  "endDate": null,
  "source": "工商登记",
  "confidence": 0.98
}
```

图模型：

```text
(A:Company)
     │
     │ OWNS
     │ share = 36%
     │ votingRights = 51%
     │ validFrom = 2022-01-01
     ▼
(B:Company)
```

这些属性是后续推理：

- 实际控制
- 最终受益人
- 风险传播
- 供应链控制
- 关联交易

的基础。

---

## 4.4 时间有效性

企业关系、任职关系、股权关系通常不是永久关系。

例如：

```text
张三
 ──DIRECTOR_OF──>
 A公司
```

必须知道：

```text
2021-01-01 ～ 2024-06-30
```

建议所有边统一具备：

```text
valid_from
valid_to
event_time
ingested_at
source
confidence
version
```

例如：

```text
(:Person {id:"P001"})
-[r:DIRECTOR_OF {
    validFrom:"2021-01-01",
    validTo:"2024-06-30"
}]->
(:Company {id:"C001"})
```

这样才能查询：

> 2023 年 6 月 30 日，张三和 A 公司是什么关系？

---

## 4.5 推理产生的新关系

原始事实：

```text
A公司
  │ 80%
  ▼
B公司
  │ 70%
  ▼
C公司
```

原始数据只有：

```text
A OWNS B
B OWNS C
```

经过推理后可以生成：

```text
A
 │
 │ INDIRECTLY_CONTROLS
 ▼
C
```

例如：

```json
{
  "source": "A",
  "target": "C",
  "rule": "CONTROL_RULE_V3",
  "confidence": 1.0,
  "evidencePath": [
    "A->B",
    "B->C"
  ]
}
```

必须严格区分：

```text
FACT
```

与：

```text
INFERENCE
```

推理边建议保存：

```text
relation_type
relation_origin
rule_id
rule_version
confidence
calculated_at
evidence_path
```

这样系统才能解释：

> “为什么系统认为 A 控制 C？”

---

# 5. 哪些数据不要直接放进图数据库

## 5.1 海量交易明细

例如：

```text
10亿订单
50亿支付流水
100亿IoT事件
```

不要全部直接变成图节点。

这些数据应主要放在：

```text
Iceberg
Doris
PostgreSQL
```

图中只保留推理真正需要的关系投影。

例如：

```text
Company A
  │
  │ TRANSFER_SUMMARY
  │ amount=2.8亿
  │ count=3821
  │ last90days=true
  ▼
Company B
```

也可以只将异常交易物化成图。

---

# 6. 十亿级资金流水如何与图数据库协同

原始流水：

```text
Payment
─────────────────────
payer
payee
amount
time
currency
account
bank
...
```

保存在：

```text
Iceberg / Doris
```

通过 Flink / Spark / Doris 等计算出一段时间窗口内的关系聚合。

例如 A → B 最近 90 天：

```text
转账次数 = 918
总金额 = 2.3亿
最大单笔 = 1800万
异常交易 = 12笔
```

然后生成图边：

```text
(A)
 │
 │ FUNDS_FLOW_TO
 │ count = 918
 │ totalAmount = 230000000
 │ abnormalCount = 12
 ▼
(B)
```

这样：

```text
50亿流水
```

最终可能只需要：

```text
几千万条资金关系
```

进入图数据库。

这相当于把大规模事实明细压缩成可推理的“关系特征”。

---

# 7. 关系型数据库设计

建议 PostgreSQL 至少包含三类核心数据。

## 7.1 Object Current

```text
ontology_object
────────────────────────
tenant_id
ontology_id
object_type
object_id
version
properties JSONB
status
created_at
updated_at
```

完整 Object 属性放这里。

建议采用：

> 固定核心字段 + JSONB 动态 Ontology 属性

这样可以兼顾动态 Ontology Schema 与关系数据库的事务能力。

---

## 7.2 Link Fact

即使采用 Graph，也建议 PostgreSQL 保留一份权威 Link Fact：

```text
ontology_link
────────────────────────
link_id
tenant_id
ontology_id

link_type

source_type
source_id

target_type
target_id

properties JSONB

valid_from
valid_to

source_system
confidence

version
created_at
updated_at
```

示例：

```text
link_id = L001
link_type = OWNS

source_id = COMPANY-A
target_id = COMPANY-B

properties:
{
    "shareholding": 0.36
}
```

这是权威关系事实。

---

## 7.3 Action / Edit

业务操作仍然通过 Action / Edit Event 进行治理：

```text
action_instance
object_edit_event
link_edit_event
outbox_event
```

它们承担：

- 事务
- 权限
- 校验
- 审计
- 版本控制
- 写回
- 事件发布

---

# 8. 为什么 Link Fact 仍然需要存在 PostgreSQL

不建议：

```text
Graph DB
=
Link唯一事实源
```

推荐：

```text
PostgreSQL
=
事实真相

Graph DB
=
推理投影
```

因为 Graph Database 未来可能需要：

- 更换产品
- 重建
- 增加新关系
- 改变推理模型
- 调整索引
- 重新分区

因此：

```text
Ontology Object
+
Ontology Link Fact
+
Ontology Release
+
Reasoning Rule
      ↓
Graph Materializer
      ↓
重新生成 Graph Runtime
```

图数据库应当可以随时重建。

---

# 9. 推荐总体架构

```text
                  ┌──────────────────────┐
                  │      Ontology        │
                  │ Object / Link / Rule │
                  └──────────┬───────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │      PostgreSQL       │
                 │                       │
                 │ Object Current        │
                 │ Link Fact             │
                 │ Action / Edit         │
                 │ Ontology Metadata     │
                 └──────────┬────────────┘
                            │
                         Outbox
                            │
                            ▼
                         Kafka
                            │
               ┌────────────┴──────────────┐
               │                           │
               ▼                           ▼
       Object Materializer          Graph Materializer
               │                           │
               ▼                           ▼
         OpenSearch                  Graph Database
                                      │
                                      │
                                Graph Reasoner
                                      │
                                      ▼
                               Inferred Relations
```

大数据侧：

```text
ERP / CRM / MES / IoT
          │
          ▼
      Iceberg
          │
          ├──────────────► Doris
          │
          ▼
  Feature / Aggregate
          │
          ▼
Graph Materializer
          │
          ▼
      Graph DB
```

---

# 10. 查询时两个数据库如何协同

不要让业务开发人员直接决定使用 SQL 还是 Cypher。

建议构建统一：

```text
Ontology Query Service
```

由 Query Planner 自动路由。

## 查询示例

### 查询 Object 详情

> 查询 COMPANY-001 的详细信息。

走：

```text
PostgreSQL
```

### 属性筛选

> 上海地区风险等级 HIGH 的企业有哪些？

走：

```text
OpenSearch / PostgreSQL
```

### 一跳关系

> A 公司直接投资了哪些公司？

走：

```text
PostgreSQL Link Fact
```

### 多跳股权关系

> A 公司通过五层股权最终控制哪些企业？

走：

```text
Graph DB
```

### 隐性关系分析

> 张三和 A 公司之间有什么关系？

可能找到：

```text
张三
 ↓ 法人
X公司
 ↓ 持股
Y公司
 ↓ 供应商
A公司
```

走：

```text
Graph DB
```

### 员工关联供应商分析

```text
Employee
   ↓ FAMILY_MEMBER
Person
   ↓ OWNS
Company
   ↓ SUPPLIES
OurCompany
```

走：

```text
Graph DB
```

### 大规模统计

> 去年 A 集团全部供应商采购金额。

走：

```text
Doris
```

或：

```text
Trino + Iceberg
```

---

# 11. 查询路由建议

| 查询类型 | 推荐数据库 |
|---|---|
| Object ID 查询 | PostgreSQL |
| Object 修改 | PostgreSQL / Action |
| 当前状态 | PostgreSQL |
| 简单属性过滤 | PostgreSQL / OpenSearch |
| 大规模搜索 | OpenSearch |
| 1 跳关系 | PostgreSQL |
| 2 跳简单关系 | PostgreSQL 或 Graph |
| 3 跳及以上复杂关系 | Graph |
| 路径查询 | Graph |
| 环路识别 | Graph |
| 最短路径 | Graph |
| 控制关系推理 | Graph |
| 最终受益人 | Graph |
| 关系网络分析 | Graph |
| 社群发现 | Graph |
| 风险传播 | Graph |
| PageRank / 中心性 | Graph |
| 大规模 SUM / GROUP BY | Doris |
| 十亿级事实明细 | Iceberg / Doris |
| 历史数据回溯 | Iceberg |

---

# 12. 图数据库技术选型

## 12.1 P0：PostgreSQL + Apache AGE

第一阶段比较推荐：

```text
PostgreSQL
│
├── ontology_object
├── ontology_link
├── action_event
│
└── Apache AGE
     └── Runtime Graph
```

优点：

- 少一个大型基础设施
- 运维简单
- 关系模型与图模型共存
- Java 接入方便
- 开发成本低
- 事务处理简单
- 适合快速完成 P0/P1 图推理能力

第一阶段可实现：

```text
Object
Link
Graph Projection
1~5跳关系查询
路径查询
规则推理
Reasoning API
```

但必须通过抽象层访问 Graph，不要让业务代码直接依赖某个图数据库。

---

# 13. GraphRepository 抽象层

建议从第一天就提供：

```text
GraphRepository

findNeighbors()
findPaths()
findShortestPath()
traverse()
findCycles()
executePattern()
```

业务层只依赖 GraphRepository。

不要让业务代码直接散落大量 Cypher。

这样后期可以：

```text
Apache AGE
      ↓
JanusGraph
```

或者切换其他 Graph Engine，而不影响 Ontology Runtime。

---

# 14. 超大规模：PostgreSQL + JanusGraph

如果未来出现：

```text
数十亿节点
数百亿边
大量3~10跳关系查询
复杂路径分析
风险传播
大规模关系网络
```

建议 Graph Runtime 独立出来：

```text
PostgreSQL
   │
   │ Graph Materialization
   ▼
Kafka
   │
   ▼
JanusGraph
   │
   ├── Cassandra
   └── Search Index
```

核心优势：

```text
Ontology Runtime
和
Graph Runtime
```

可以完全解耦、独立扩容。

---

# 15. 为什么第一阶段不建议依赖 Neo4j Enterprise

Neo4j 的图模型、Cypher 和图算法生态非常成熟。

但是，如果目标明确要求：

> **基础设施全部开源、自建，并且未来可以大规模分布式扩展**

则建议优先考虑：

```text
P0：
PostgreSQL + Apache AGE

↓

P1：
PostgreSQL + 独立 Graph Service

↓

超大规模：
PostgreSQL + Kafka + JanusGraph + Cassandra
```

这样核心架构不会依赖商业版集群能力。

---

# 16. Graph Schema 应由 Ontology 生成

不要单独再建立一套独立 Graph Schema Manager。

应该：

```text
Ontology Object Type
       ↓
Graph Node Mapping

Ontology Link Type
       ↓
Graph Edge Mapping
```

例如 Ontology：

```text
Object Type:
Company

Properties:
name
revenue
riskLevel
```

设计台配置：

```text
Graph Projection = ENABLED

Graph Properties:
- name
- riskLevel
```

生成：

```text
(:Company {
    objectId,
    name,
    riskLevel
})
```

---

## Link Type 示例

Ontology：

```text
Company --OWNS--> Company
```

配置：

```text
Graph Projection = ENABLED
Traversal = ENABLED
Inference = ENABLED

Properties:
- shareholding
- votingRights
- validFrom
- validTo
```

自动生成 Graph Edge。

因此：

> **Ontology 才是真正的逻辑 Graph Model；图数据库只是 Ontology 的一种 Runtime Projection。**

---

# 17. 在 Ontology 中增加 Graph Configuration

建议在 Object Type 中增加：

```json
{
  "graphEnabled": true,

  "nodeProperties": [
    "name",
    "riskLevel"
  ],

  "reasoningEnabled": true,

  "graphIndex": [
    "objectId",
    "name"
  ]
}
```

Link Type：

```json
{
  "graphEnabled": true,

  "traversalEnabled": true,

  "reasoningEnabled": true,

  "properties": [
    "shareholding",
    "validFrom",
    "validTo"
  ]
}
```

不是所有 Object / Link 都要进入 Graph。

---

# 18. 哪些 Link 值得进入 Graph

建议至少满足一个条件：

## 18.1 经常被多跳查询

例如：

```text
OWNS
CONTROLS
SUPPLIES
MANAGES
GUARANTEES
RELATED_TO
```

## 18.2 参与业务推理

例如：

```text
OWNS
+
OWNS
→ INDIRECTLY_OWNS
```

## 18.3 参与风险传播

```text
Company HIGH_RISK
       ↓ GUARANTEES
Company ?
```

## 18.4 参与路径分析

```text
Company
→ Person
→ Company
→ Supplier
```

## 18.5 参与图算法

例如：

```text
中心性
社群
关联网络
最短路径
```

否则不必进入 Graph。

---

# 19. 推荐最终技术架构

```text
                     Ontology
        ┌──────────────────────────────┐
        │ Object Type                 │
        │ Property                    │
        │ Link Type                   │
        │ Action Type                 │
        │ Interface                   │
        │ Graph Projection Definition │
        │ Reasoning Rule              │
        └──────────────┬───────────────┘
                       │
                       ▼
                PostgreSQL
        ┌──────────────────────────────┐
        │ Ontology Metadata            │
        │ Object Current               │
        │ Link Fact                    │
        │ Action / Edit                │
        │ Reasoning Metadata           │
        └──────┬───────────────┬───────┘
               │               │
            Kafka             API
               │               │
               ▼               ▼
      Graph Materializer    Object API
               │
               ▼
        ┌──────────────┐
        │ Graph Runtime│
        │ Apache AGE   │ ← P0
        └──────┬───────┘
               │
          Graph Reasoner
               │
               ▼
       Inferred Relations
               │
               ▼
       PostgreSQL记录
       + Graph Projection


大规模事实：
Ceph / S3
   ↓
Iceberg
   ↓
Doris / Trino
   ↓
Feature Aggregation
   ↓
Graph Materializer
```

---

# 20. 技术演进路线

## 第一阶段

```text
PostgreSQL + Apache AGE
```

完成：

- Object
- Link Fact
- Graph Projection
- 1~5 跳关系查询
- 路径查询
- 规则推理
- Reasoning API
- FACT / INFERENCE 区分
- Evidence Path

## 第二阶段

建立独立：

```text
Graph Service
GraphRepository
Graph Materializer
Graph Reasoner
```

把 Graph Runtime 从业务代码中完全解耦。

## 第三阶段

当图数据达到超大规模：

```text
PostgreSQL
=
Object / Action / Link Fact

Iceberg
=
大规模事实历史

Doris
=
大规模聚合分析

JanusGraph + Cassandra
=
大规模 Graph Projection

OpenSearch
=
Object Search

Kafka
=
所有 Projection 同步

Ontology Runtime
=
统一查询层
```

---

# 21. 一句话判断数据应该放哪里

以后设计 Object Type 时，可以问三个问题：

> **“我要知道它是什么？” → PostgreSQL。**

> **“我要统计它发生了多少？” → Doris / Iceberg。**

> **“我要知道它和谁通过什么路径有关，并据此推导出什么？” → Graph。**

---

# 22. 最终建议

建议坚持：

> **PostgreSQL 的 Object + Link Fact 是权威事实；Graph Database 是根据 Ontology Link Type 构建出来的可重建“关系推理运行时”。**

这样既可以获得图数据库对：

- 多跳关系
- 路径查询
- 风险传播
- 控制关系
- 最终受益人
- 关联网络
- 社群分析

等业务推理场景的优势，又不会把整个业务数据架构绑定在 Graph Database 上。

推荐路线：

```text
P0：
PostgreSQL + Apache AGE

P1：
PostgreSQL + Kafka + 独立 Graph Service

P2 / 超大规模：
PostgreSQL + Iceberg + Doris
+ Kafka
+ JanusGraph + Cassandra
+ OpenSearch
```

最终形成：

```text
关系型数据库 = 权威事实层
图数据库     = 关系推理层
Iceberg      = 海量历史事实层
Doris        = 大规模分析层
OpenSearch   = 对象搜索层
Kafka        = 投影同步总线
Ontology     = 统一语义层
```

---

## 参考资料

- Palantir Foundry Ontology / Object / Link Type 文档
- PostgreSQL JSON / JSONB 文档
- Apache AGE 官方文档
- JanusGraph Storage Backend 官方文档
- Neo4j Graph Database / Graph Data Science 文档
