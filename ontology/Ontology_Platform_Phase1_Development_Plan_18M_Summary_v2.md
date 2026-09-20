# Ontology Platform 第一阶段开发内容与开发计划

## 1. 文档目的

本文基于当前项目边界，对第一阶段 Ontology Platform 的开发内容和开发顺序进行收敛。

当前明确的架构前提：

- 不自研 Pipeline / ETL / ELT / Data Integration Platform。
- 第三方 Pipeline 负责数据采集、清洗、转换、CDC、调度与数据质量，并最终将标准化数据写入 TiDB。
- 不自研 Automation / Workflow Engine，也不自研审批平台或审批流程引擎。
- 外部 Automation Engine 负责任务调度、Trigger、Workflow、Retry 和流程编排；外部 Approval Platform 负责审批流程定义、审批任务、会签/条件审批、审批人规则和审批状态流转。
- 暂不开发 Application Builder、完整 Developer Platform 产品化能力、独立 Operations Platform 产品、AIP / AI Platform。
- 当前目标是优先完成企业级 Ontology Runtime Platform。

第一阶段平台的核心目标：

> 建立从 Platform Kernel / Security Foundation → Ontology Modeling → TiDB Mapping & Data Access → Object / Link / Query Runtime → Action / Edit → Function / Event / External Automation / Approval Integration 的完整闭环。

本计划使用两种不同的拆分维度：

- **15 个领域模块**：用于定义长期能力边界、代码边界和责任边界，不代表 15 个串行开发阶段。
- **6 个交付阶段**：用于组织 10 人团队的并行研发和阶段验收，阶段之间允许有计划地重叠。

第一阶段不追求完整复制 Foundry 全家桶，而是以 1～2 个真实业务域为牵引，按可运行的 Vertical Slice 持续交付。

---

# 2. 项目总体边界

## 2.1 外部 Data Pipeline 负责

```text
业务系统
   │
   ▼
Third-party Pipeline
   │
   ├── 数据采集
   ├── ETL / ELT
   ├── CDC
   ├── Transform
   ├── Join
   ├── Clean
   ├── Data Quality
   ├── Schedule
   └── Pipeline Monitoring
   │
   ▼
TiDB
```

本平台不开发上述 Pipeline 能力。

---

## 2.2 外部 Automation / Approval Platform 负责

```text
Schedule
Trigger
Workflow
Condition
Retry
Process Orchestration
Workflow State

Approval Process Definition
Approver Resolution
Countersign / Sequential / Conditional Approval
Human Task
Approval State
```

本平台不负责流程调度、Workflow Runtime 或审批 Runtime。

外部 Automation / Approval Platform 通过：

```text
Event
+
ObjectSet API
+
Function API
+
Action API
+
Approval Request API / Approval Callback
```

使用 Ontology Platform。

---

# 3. 第一阶段自研范围

第一阶段共规划 15 个领域模块。模块编号只用于能力归类，不代表实际开发顺序。

| 编号  | 模块                                              | 主要开发内容                                                                                                                                                             | 优先级   |
| --- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----- |
| 01  | Resource & Identity / Control Plane             | UID/RID、Enrollment、Organization、Space、Portfolio、Project、Folder、Resource、Role、Permission、Marking、Revision、Workspace、ChangeSet、Branch、Proposal、Approval Request/Status、Audit、Outbox | 当前阶段  |
| 02  | Object Type & Property                          | Object Type、Property、Primary Key、API Name、类型系统、约束、状态、Schema Revision、Property Lifecycle                                                                            | 最高    |
| 03  | TiDB Datasource & Ontology Mapping              | Datasource、Database/Table/Column Metadata、Object Type ↔ Table、Property ↔ Column、Primary Key Mapping                                                                | 最高    |
| 04  | Object Runtime                                  | Object Instance、Object Identity、单对象/批量读取、Property Loading、Materialization、Pagination、Serialization、Cache、Runtime Backend Abstraction                                  | 最高    |
| 05  | Link Type & Relationship Runtime                | Link Type、Source/Target、Cardinality、Foreign Key/Join Mapping、正向/反向关系、Link Traversal、可选 Graph Projection                                                           | 核心    |
| 06  | ObjectSet / Query Engine                        | Filter、Sort、Search、Pagination、Projection、Aggregation、Group By、Link Traversal、Query IR、SQL Translation                                                              | 最高    |
| 07  | Edit Runtime                                    | 内部 Create/Update/Delete、Edit-only Property、批量编辑、Validation、Optimistic Concurrency、事务与写入策略                                                                     | 核心    |
| 08  | Action Engine                                   | 对外统一写入口、Action Type、Parameter、Constraint、Validation、Submission Criteria、Side Effect、Action Execution、Audit                                                     | 核心    |
| 09  | Function Platform                               | Function Registry、Function Contract、Version、Deployment、Execution、Timeout、Permission、Action 调 Function                                                              | 重要    |
| 10  | Interface & Ontology Abstraction                | Interface、Interface Property、Interface Action、Constraint、Object Type Implements、Property/Action Mapping                                                            | 重要    |
| 11  | Value Type / Shared Model                       | Base Type、Value Type、Value Type Version、Constraint、Shared Property、Object Type Group                                                                                | 最高    |
| 12  | TiDB Data Access Runtime                        | Connection Pool、Credential、Schema Discovery、Mapping Runtime、SQL Compiler、安全 SQL、Timeout、Query Execution                                                            | 最高    |
| 13  | Ontology Dependency & Impact Analysis           | 从首个 Revision 开始记录依赖边，逐步实现 TiDB Column → Property → Object/Link → Function/Action 的上游/下游与变更影响分析                                                            | 重要    |
| 14  | Data Security Runtime                           | 从首个 Runtime 开始落实 Object/Row/Property 级权限、Organization、Marking、Security Filter、Query Rewrite、Security Pushdown                                                     | 企业级必需 |
| 15  | External Automation / Approval Integration & Event Gateway | 从早期稳定 API/Event/Approval Contract 开始，逐步实现 Object Event、Subscription、Webhook/Event Bus、外部审批请求与回调、Service Principal、Idempotency、Callback | 重要    |

---

# 4. 四个核心技术域

## 4.1 Control & Governance

主要模块：

```text
01 Resource & Identity / Control Plane
14 Data Security Runtime
```

负责：

```text
平台有什么资源
资源放在哪里
资源属于哪个 Space / Project
谁可以访问
谁可以修改
谁可以执行 Action
修改如何版本化
如何提交外部审批并受审批结果约束
如何审计
数据级访问如何控制
```

---

## 4.2 Ontology Modeling

主要模块：

```text
02 Object Type & Property
05 Link Type & Relationship Runtime
08 Action Engine
10 Interface & Ontology Abstraction
11 Value Type / Shared Model
13 Dependency & Impact Analysis
```

负责表达企业业务世界：

```text
Object Type
Property
Link
Action
Interface
Value Type
Shared Property
Dependency
```

例如：

```text
Customer
   │
   │ places
   ▼
ServerOrder
   │
   │ contains
   ▼
ServerModel
   │
   │ requires
   ▼
Component
```

---

## 4.3 Ontology Data Runtime

主要模块：

```text
03 TiDB Mapping
04 Object Runtime
06 ObjectSet / Query Engine
07 Edit Runtime
12 TiDB Data Access Runtime
```

核心链路：

```text
Ontology
   ↓
Object Type / Property
   ↓
Mapping
   ↓
Object Runtime
   ↓
ObjectSet / Query
   ↓
Security Rewrite
   ↓
SQL Translation
   ↓
TiDB
```

上述顺序仅表示请求处理链。Data Security 的 Policy Model、身份上下文和强制执行点必须与 Object Runtime、Query IR、Cache Key 和 SQL Compiler 同期设计，不能在 Runtime 完成后再补。

这一层让用户通过 Ontology 访问业务对象，而不是直接使用数据库表和列。

---

## 4.4 Business Capability Runtime

主要模块：

```text
08 Action Engine
09 Function Platform
15 External Automation / Approval Integration
```

负责向外部系统提供真正可执行的业务能力：

```text
Query Object
Execute Action
Internal Edit Transaction
Call Function
Publish Object Event
```

对外修改必须优先通过 Action API；Edit Runtime 是 Action Engine 使用的内部事务执行能力，不单独暴露可绕过权限、Validation 和 Audit 的通用写接口。

外部 Automation / Approval Platform 负责：

```text
什么时候执行
按照什么流程执行
失败是否重试
是否需要人工审批
步骤如何编排
```

---

# 5. 十五个领域模块概括说明

本阶段认可并保留以下 15 个领域模块。后续项目文档中，本节只描述各模块的职责与业务价值，不展开到详细数据模型、接口、存储结构或内部实现设计。

## 5.1 01 — Resource & Identity / Control Plane

负责整个平台最底层的资源、身份、目录、权限、版本、分支、Proposal、外部审批状态映射和审计治理，为后续所有 Ontology 和 Runtime 能力提供统一的 Control Plane 基础。本模块不实现审批流程、审批任务或审批人规则。

## 5.2 02 — Object Type & Property

负责定义企业业务对象及其属性模型，包括对象类型、属性、主键、类型和生命周期等，是 Ontology 业务语义建模的核心入口。

## 5.3 03 — TiDB Datasource & Ontology Mapping

负责把 TiDB 中已经由第三方 Pipeline 整合完成的表和字段映射成 Object Type 与 Property，使物理数据能够进入 Ontology 语义层。

## 5.4 04 — Object Runtime

负责把 Object Type 转换为真正可读取和使用的 Object Instance，并提供对象身份、属性读取、批量访问、分页、序列化、缓存及 Runtime Backend Abstraction。

第一阶段默认由 TiDB 提供业务事实和主要对象查询能力：

```text
Object Type / Property / Mapping
→ Query Plan / Safe SQL
→ TiDB / TiFlash
→ Object Instance
```

Object Runtime 不直接绑定具体图数据库。若业务场景和性能基准证明需要独立图引擎，则通过 Runtime Backend / GraphRepository 抽象接入可重建的 Graph Projection。

## 5.5 05 — Link Type & Relationship Runtime

负责定义和运行 Object 之间的业务关系，并提供正向、反向及受控多跳关系遍历能力。

第一阶段优先支持：

```text
Foreign Key / Join Table Mapping
→ Link Fact
→ SQL Join / Recursive CTE
→ Link Traversal
```

当出现以下场景，并且 TiDB 基准测试无法达到目标 SLO 时，再启用 Graph Projection：

```text
3 跳及以上高频遍历
复杂路径查询
环路 / 最短路径
关系网络分析
风险传播或控制关系推理
```

Graph Projection 必须可由 TiDB 中的 Object / Link Fact 重新构建，不成为唯一事实源。

## 5.6 06 — ObjectSet / Query Engine

负责通过 Ontology 语义查询 Object，而不是让业务用户直接操作 SQL；支持对象过滤、排序、分页、聚合和关系遍历等核心查询能力。

## 5.7 07 — Edit Runtime

作为内部事务执行层，负责 Object 的创建、修改、删除及批量编辑，并处理 Ontology 管理数据与第三方 Pipeline 管理数据之间的写入边界。核心能力包括 Validation、Optimistic Concurrency、Idempotency、Transaction、Outbox 和 Audit。

## 5.8 08 — Action Engine

负责定义并执行面向业务对象的业务动作，使 Ontology 从“描述业务”扩展为“执行和改变业务状态”。Action API 是平台对用户和外部系统暴露的主要写入口，内部调用 Edit Runtime 完成事务修改。

需要人工审批的 Action 不在 Action Engine 内嵌审批节点或审批状态机。Action Engine 只生成绑定 Action Request、参数摘要和目标对象版本的 External Approval Request；收到模块 15 验证通过的 Approved 终态后，重新执行权限、Submission Criteria 和版本检查，再以同一 Idempotency Key 提交 Edit Transaction。Rejected、Cancelled、Expired、伪造或旧版本回调不得执行 Action。

## 5.9 09 — Function Platform

负责承载可复用的业务计算和复杂业务逻辑，为 Action、外部 Automation 以及其他 Runtime 能力提供统一函数执行能力。

## 5.10 10 — Interface & Ontology Abstraction

负责为不同 Object Type 提供统一的业务抽象和契约，使多个不同对象类型可以通过相同的属性或 Action 语义被统一使用。

## 5.11 11 — Value Type / Shared Model

负责定义企业级可复用的数据语义和共享模型，例如金额、国家代码、邮箱地址以及 Shared Property 等统一类型。Base Type、Value Type 和 Property Constraint 属于 Ontology Foundation，必须与 Object Type / Property 同期建设；高级 Shared Model 和 Object Type Group 可以后续增强。

## 5.12 12 — TiDB Data Access Runtime

负责 Ontology Runtime 与 TiDB 之间的统一数据访问，包括连接、Schema 发现、查询执行和 Ontology 查询到 TiDB 查询之间的转换。

## 5.13 13 — Ontology Dependency & Impact Analysis

负责维护 TiDB、Property、Object、Link、Function、Action 等资源之间的依赖关系，并在模型变更时分析上游来源和下游影响。

本模块不负责第三方 Pipeline 内部的完整 Data Lineage，平台的依赖分析边界从 TiDB 开始。

依赖边必须从第一个 Schema Revision 和 Mapping 发布流程开始记录；高级可视化、迁移建议和跨资源 Impact Analysis 可以在后续阶段实现。

## 5.14 14 — Data Security Runtime

负责把 Control Plane 中的权限和强制安全控制落实到 Object、Row、Property 和 Query 运行时，确保业务数据访问符合企业安全要求。

本模块是贯穿所有交付阶段的横切能力，不作为第 14～17 个月才开始的独立尾部阶段。

## 5.15 15 — External Automation / Approval Integration & Event Gateway

负责把 Ontology Platform 与外部 Automation / Workflow Engine、Approval Platform 连接起来，通过事件、ObjectSet、Function、Action、Approval Request 和 Callback 接口对外提供可编排、可审批的业务能力。

外部 Automation Engine 负责调度、流程和业务重试；外部 Approval Platform 负责审批模板、审批人、会签/条件审批、Human Task 和审批状态机。本平台仅负责为 Ontology Proposal 或 Action Request 发起/撤销审批请求、保存外部流程实例引用、接收验签回调、幂等更新审批镜像状态、执行 Publish / Action Execute Gate、对账和审计，不开发审批平台。

API Contract、Event Envelope、Approval Contract、Service Principal 和 Idempotency 需要在 Stage 1 与 Action Runtime 阶段提前稳定；最后阶段只完成适配、联调和生产硬化。

---

# 6. 当前暂不开发内容

当前阶段明确不开发以下产品：

```text
Application Builder

完整 Developer Platform 产品化能力

独立 Catalog 产品扩展

独立 Operations Platform 产品

审批平台 / 审批流程引擎

AIP / AI Platform

LLM Gateway

Agent

AIP Logic

Ontology Tool Calling
```

注意：

虽然不建设独立 Developer Platform 和 Operations Platform，但核心 Runtime 仍必须具备必要的：

```text
REST API
Authentication
Service Account
Logs
Metrics
Tracing
Health Check
Runtime Monitoring
```

这些属于生产运行基础设施，而不是独立产品模块。

---

# 7. 推荐研发计划

第一阶段总开发周期规划为 **18 个月**，以 10 名开发人员为基准。

计划采用 **6 个交付阶段、3 条长期研发线、横切安全与质量保障** 的方式推进。阶段之间允许重叠，但每个阶段必须形成可演示、可测试、可验收的产品增量。

| 交付阶段 | 主要内容 | 建议周期 | 核心目标 |
|---|---|---|---|
| Stage 1 | Complete Control Plane V1 & Modeling Foundation | 第 1～4 个月 | 完整交付 Control Plane V1，包括 Portfolio、复杂 Folder、完整 Workspace、Branch/Proposal，以及外部审批请求、回调和发布门禁 |
| Stage 2 | Secure Read-only Vertical Slice | 第 3～8 个月 | 完成 TiDB Mapping、Object/Link/Query Runtime 和读取安全，实现首个端到端只读业务域 |
| Stage 3 | Operational Ontology | 第 7～11 个月 | 合并交付 Edit Runtime 与 Action Engine，实现事务写入、审批门禁、并发控制、审计和事件 Outbox |
| Stage 4 | Programmability & Integration | 第 10～14 个月 | 建立精简 Function Runtime、Interface、稳定 API/Event Contract、外部 Automation 集成，并对 Approval Adapter 做生产硬化 |
| Stage 5 | Enterprise Governance & Hardening | 第 12～16 个月 | 强化 Control Plane 的规模、冲突合并、外部审批集成、权限/Marking、审计合规和运维能力，并完成 Dependency/Impact、HA 与按需 Graph Projection |
| Stage 6 | Pilot & General Availability | 第 16～18 个月 | 以 1～2 个真实业务域完成生产试点、迁移、压测、安全测试、故障演练和正式发布 |

## 7.1 Stage 1 — Complete Control Plane V1 & Modeling Foundation

主要范围：

```text
01 Resource & Identity / Control Plane V1 完整功能集合
02 Object Type & Property
11 Base Type / Value Type / Property Constraint
12 TiDB Connection / Credential / Schema Discovery 骨架
14 Identity Context / Policy Model
15 External Approval Request / Callback Contract 与首个 Adapter 闭环
```

Stage 1 不采用“只实现最小资源模型、其余能力后移”的方式。Control Plane 是后续所有 Ontology Resource、协作、权限、版本和发布流程的共同底座，因此 01 模块必须在 Stage 1 形成完整可用的 V1 闭环。

### 7.1.1 Resource & Identity

```text
UID / RID 生成、解析与全局唯一约束
Enrollment / Organization
User / Group / Service Principal Identity Reference
Resource Type / Resource Registry / Resource Lifecycle
Ownership / Creator / Maintainer / Status / Visibility
```

### 7.1.2 完整资源层级

```text
Organization
└── Space
    └── Portfolio
        └── Project
            └── Folder（支持多级嵌套、移动、排序和路径解析）
                └── Resource
```

Stage 1 必须完成 Portfolio、Project、复杂 Folder 和 Resource 的统一树模型，包括：

- 任意层级 Folder 嵌套与循环引用防护；
- Folder / Resource 的移动、重命名、排序、归档和恢复；
- 完整路径、父子关系、唯一性和可见性规则；
- 跨 Portfolio / Project 移动的权限、引用和审计校验；
- 大目录分页、搜索和按权限过滤；
- 资源删除保护及被依赖资源的删除阻断。

### 7.1.3 完整 Workspace 与变更生命周期

```text
Workspace
├── Workspace Member / Role
├── Base Revision / Working Revision
├── ChangeSet / ChangeSet Item
├── Branch
├── Validation Result
├── Proposal / Review / Comment
├── External Approval Request / Status / Callback
└── Merge / Publish / Archive
```

Stage 1 的 Workspace 不是简单 Draft 表，而是完整的隔离协作空间，必须支持：

- Workspace 创建、成员、角色、所有者、状态和生命周期；
- 基于已发布 Revision 创建工作版本；
- 一个 ChangeSet 原子包含多个 Ontology Resource 变更；
- Branch 创建、保存、比较、提交 Proposal 和关闭；
- Proposal Review、Comment、提交/撤销外部审批请求、审批状态同步；
- 本地 Review / Comment 仅用于协作和问题修订，不产生 Approved / Rejected 审批结论；
- 发布前 Validation、Permission Check、Dependency Check、External Approval Result Check；
- 外部审批回调验签、时间戳/Nonce 防重放、幂等、乱序处理、流程实例与 Revision 绑定；
- 只有当前 Proposal 对应的有效外部审批结果为 Approved 时才允许 Publish；Rejected、Cancelled、Expired 或旧 Revision 的回调不得放行；
- 发布成功后的不可变 Revision、版本引用和变更摘要；
- 发布失败不污染 Main / Published Revision；
- 并发修改检测和基础冲突提示。

### 7.1.4 Permission、Marking、Audit 与 Outbox

```text
Role / Permission / Resource Policy
Organization / Space / Portfolio / Project / Folder 权限继承
Resource Protection
基础 Marking 与访问要求
External Approval Mapping / Publish Gate
完整 Audit Event
Transactional Outbox
```

Stage 1 必须确保所有资源创建、移动、修改、审批请求、审批回调、状态变化、发布、归档和权限变更都产生统一 Audit Event；需要对外传播的变更通过 Transactional Outbox 可靠发布。

Stage 1 与 Stage 5 的边界是：

- **Stage 1 完成功能闭环**：上述资源、Workspace、Branch、Proposal、External Approval Integration、Permission、Audit 和 Outbox 均可被真实使用；审批流程本身由外部平台运行。
- **Stage 5 完成企业级强化**：针对大规模资源树、外部审批对账、复杂组织策略、冲突合并、合规留存、批量治理和灾备进行扩展与硬化，而不是补做 Stage 1 缺失功能或自研审批引擎。

阶段退出条件：

> 用户可以在 Organization → Space → Portfolio → Project → 多级 Folder 中管理资源，在完整 Workspace 中通过 Branch → ChangeSet → Proposal → External Approval Request → Approval Callback → Publish 流程发布 Object Type / Property / Value Type；审批由外部平台完成，本平台仅在验证有效 Approved 结果后放行发布。全过程受权限和 Marking 控制，具备 Audit、Outbox、失败回滚和已发布 Revision 隔离，并能够安全配置一个 TiDB 数据源。

## 7.2 Stage 2 — Secure Read-only Vertical Slice

主要范围：

```text
03 TiDB Datasource & Ontology Mapping
04 Object Runtime
05 Link Type 与基础 Link Traversal
06 ObjectSet / Query Engine
12 TiDB Data Access Runtime
13 基础 Dependency Capture
14 Object / Row / Property Read Enforcement
```

查询优先通过 Query IR → Safe SQL → TiDB / TiFlash 执行。第一版支持主键查询、批量读取、Filter、Sort、Pagination、Projection、基础 Aggregation 和 1～2 跳关系查询。

阶段退出条件：

> 至少一个真实业务域可以从 TiDB 映射 3 个以上 Object Type 和 2 个以上 Link Type，并通过带 Row / Property 权限的 ObjectSet API 查询。

## 7.3 Stage 3 — Operational Ontology

主要范围：

```text
07 Edit Runtime
08 Action Engine
Link Create / Delete
Optimistic Concurrency
Idempotency
Transaction / Outbox / Audit
External Approval Gate for Action Request
```

Action API 是外部统一写入口；Edit Runtime 是内部事务执行层。Action 必须统一执行参数校验、Submission Criteria、权限校验、版本一致性检查、Side Effect 和 Audit。需要审批的 Action 通过 External Approval Request / Callback 完成门禁，本平台不运行审批流程。

阶段退出条件：

> 外部调用方能够通过幂等 Action 安全修改 Object / Link；需要审批的 Action 只有在收到与当前 Action Request 和对象版本绑定的有效外部 Approved 结果后才执行。重复提交、重复/乱序审批回调、并发冲突、部分失败和 Side Effect 失败均有明确处理与审计记录。

## 7.4 Stage 4 — Programmability & Integration

主要范围：

```text
09 Function Registry / Contract / Version / Execution
10 Interface & Ontology Abstraction
15 Service Principal / API / Event / Webhook / Approval Adapter Hardening
Function-backed Action
```

Function Platform 首期优先复用现有容器或 Kubernetes 执行底座，不在本阶段自研完整多语言 FaaS、IDE、构建平台和包市场。

阶段退出条件：

> 外部 Automation Engine 可以使用 Service Principal 调用 ObjectSet、Action 和 Function，并通过稳定 Event Contract 接收可重试、可去重的对象事件；外部 Approval Platform 可以通过稳定的请求/回调契约驱动 Ontology Publish Gate 和 Action Execute Gate。

## 7.5 Stage 5 — Enterprise Governance & Hardening

主要范围：

```text
01 Control Plane Enterprise Hardening
13 高级 Dependency / Impact Analysis
14 完整 Data Security Runtime
Query Cost / Timeout / Rate Limit
Cache Security Isolation
HA / Backup / Restore / Upgrade / Rollback
可选 Graph Projection
```

### 7.5.1 01 Control Plane Enterprise Hardening

Stage 5 不再首次建设 Portfolio、Folder、Workspace、Branch、Proposal 或外部审批对接。这些能力已经在 Stage 1 完成功能闭环。本阶段针对真实业务运行后的规模、合规和复杂协作要求进行强化；审批流程、审批任务和审批规则仍由外部 Approval Platform 提供。

资源层级与规模强化：

```text
十万级以上 Resource / Folder 的分页、搜索和索引优化
大型 Folder Tree 的移动、重算、异步任务和失败恢复
Portfolio / Project / Folder 配额与容量限制
批量移动、批量归档、批量恢复和批量权限变更
资源孤儿、断链、循环引用和路径不一致修复工具
跨 Portfolio / Project 引用的完整性检查
```

Workspace、Branch 与合并强化：

```text
长期 Workspace 清理、冻结、归档和恢复
Branch Rebase / Compare / Merge Check
Ontology Resource 级冲突检测
字段级冲突展示和人工解决
并行 Proposal 的冲突与依赖排序
失败 Merge 的补偿、重试和一致性校验
大 ChangeSet 的分批校验和原子发布
```

外部审批集成与发布治理强化：

```text
Resource Type / Action Type / Space / Marking → 外部审批流程模板映射
Approval Request 与 Ontology Revision / Proposal 或 Action Request / Object Version 的不可变绑定
Requested / InReview / Approved / Rejected / Cancelled / Expired 镜像状态
Callback Signature / Nonce / Timestamp / Idempotency / Out-of-order Protection
外部 Process Instance / Task / Approver / Decision / Decision Time 证据归档
审批超时、丢失回调、状态不一致的主动对账、告警和人工修复
高风险 Breaking Change、紧急发布和职责分离规则交由外部审批平台执行
Publish / Execute Gate 只接受当前 Revision 或 Action Request 的有效终态，禁止管理员直接篡改审批结果
```

Permission 与 Marking 强化：

```text
多组织共享 Space 的访问隔离
Portfolio / Project / Folder / Resource 权限继承与覆盖
Permission Explain / Check Access
大规模 Group Membership 变化后的权限重算
Marking 传播、组合、冲突，以及需要审批时的外部门禁
Service Principal 生命周期、密钥轮换和最小权限
缓存中的权限版本隔离与即时失效
```

Audit、合规与运维强化：

```text
Audit Retention / Export / Search
不可篡改或可验证的审计链
管理员操作和 Break-glass 审计
Outbox 积压监控、重放、去重和死信处理
Control Plane Backup / Restore / Point-in-time Recovery
Schema Migration / Online Upgrade / Rollback
治理 Dashboard、SLO、Alert 和 Runbook
```

Stage 5 的 Control Plane 退出条件：

> Stage 1 的全部 Control Plane 功能在目标资源规模、并发和组织复杂度下稳定运行；权限解释、冲突解决、外部审批对账与 Publish / Execute Gate、批量治理、审计留存、备份恢复与升级回滚均通过验收，平台内部不存在自研审批流程引擎。

Graph Database 是否进入本阶段，由 Stage 2～3 的真实查询基准决定。只有高频复杂多跳、路径、环路或图算法场景无法达到目标 SLO 时，才引入独立 Graph Runtime。

阶段退出条件：

> 权限绕过测试、依赖影响检查、容量压测、故障切换、备份恢复和升级回滚均通过，并达到预先定义的 SLO、RPO 和 RTO。

## 7.6 Stage 6 — Pilot & General Availability

本阶段原则上不再增加大型模块，重点完成：

```text
1～2 个真实业务域生产试点
数据与 Ontology Migration
API / SDK / 管理 UI 收口
生产监控与告警
安全测试与权限审计
容量压测与故障演练
运维手册与用户文档
正式发布与回滚预案
```

阶段退出条件：

> 至少两个端到端业务流程在生产或准生产环境稳定运行，核心 SLO 达标，且不存在 P0 / P1 级未关闭缺陷。

---

# 8. 推荐开发依赖关系

总体采用三条稳定研发线，而不是九个阶段串行交接：

```text
Lane A — Metadata & Governance
Complete Control Plane V1
    → Object / Property / Value Type
    → Revision / ChangeSet / Publish
    → Interface / Dependency / Enterprise Hardening

Lane B — Data & Runtime
TiDB Access / Schema Discovery
    → Mapping / Object Runtime / Query IR
    → Link Traversal / Aggregation / Performance
    → Optional Graph Projection

Lane C — Operations & Integration
API / Event Contract Skeleton
    → Internal Edit Runtime / Action Engine
    → Function Runtime / Event Gateway
    → External Automation / Approval Integration

Cross-cutting
Identity / Security / Audit / Observability / Test Automation
```

关键依赖关系：

```text
Object / Property / Value Type
        ↓
Mapping + Object Identity
        ↓
Query IR + Runtime Security
        ↓
Link Traversal
        ↓
Internal Edit Transaction
        ↓
Action
        ↓
Function-backed Action / External Automation
```

其中 Security、Dependency Capture、Audit 和 Event Contract 不处于链路尾部，而是在相应资源首次出现时同步建设。

## 8.1 10 人团队建议分工

由于 Stage 1 要完整交付 Control Plane V1，前四个月需要向 Metadata & Governance 倾斜；从 Stage 2 主体开发开始，再把一名开发人员转入 Data & Runtime。

| 时段 | Metadata & Governance | Data & Runtime | Operations & Integration | Platform Engineering |
|---|---:|---:|---:|---:|
| M1～M4 | 4 | 3 | 2 | 1 |
| M5～M18 | 3 | 4 | 2 | 1 |

| 研发线 | 人数 | 主要责任 |
|---|---:|---|
| Metadata & Governance | 4 → 3 | Control Plane、复杂资源树、Workspace/Branch/Proposal、外部审批状态映射与发布门禁、Ontology Modeling、Revision、Dependency；至少 1 人负责管理 UI / Full-stack |
| Data & Runtime | 3 → 4 | TiDB Access、Mapping、Object/Link/Query Runtime、性能和可选 Graph Projection |
| Operations & Integration | 2 | Edit、Action、Function、Event、External Automation / Approval Platform Adapter |
| Platform Engineering | 1 | CI/CD、测试基础设施、部署、可观测性、压测、备份恢复和故障演练 |

Data Security 由各研发线共同负责：Metadata 负责 Policy Model，Runtime 负责读取强制执行，Operations 负责 Action / Function 执行权限，Platform Engineering 负责安全测试与审计证据。

如果 10 人中还包含专职产品、测试或运维人员，则优先缩小 Function、Graph、Interface 高级能力和非核心数据源范围；Stage 1 的完整 Control Plane V1 不作为默认裁剪项。

---

# 9. 第一关键里程碑：安全只读 Vertical Slice（目标 M8）

第一个关键里程碑是形成完整的只读 Ontology Vertical Slice：

```text
Control Plane
    ↓
Object Type / Property / Value Type
    ↓
TiDB Mapping
    ↓
Object Runtime / Link Runtime
    ↓
Security Rewrite
    ↓
ObjectSet / Query / TiDB
```

目标：

> 可以定义并发布一个业务 Ontology，通过安全的 Ontology API 查询对应 Object Instance 和基础关系，而不是直接访问 TiDB 表。

最低验收规模：

```text
1 个真实业务域
≥ 3 个 Object Type
≥ 2 个 Link Type
对象级、行级和属性级读取权限
主键、过滤、排序、分页、投影和基础聚合
```

---

# 10. 第二关键里程碑：Operational Ontology（目标 M11）

第二个关键里程碑是形成 Operational Ontology：

```text
Object
    ↓
Link
    ↓
Query
    ↓
Action
    ↓
Internal Edit Transaction
    ↓
Audit / Outbox / Event
```

目标：

> 平台不仅能够描述和查询业务，还可以通过受权限控制、可审计、可幂等的 Action 修改业务状态。

---

# 11. 第三关键里程碑：External Automation / Approval Integration（目标 M14）

第三个关键里程碑是完成与外部 Automation Engine 和 Approval Platform 的生产级集成：

```text
External Automation
        ↓
Service Principal / Stable API Contract
        ↓
ObjectSet / Function / Action
        ↓
Event / Webhook / Callback

External Approval Platform
        ↕
Approval Request / Cancel / Status Query / Signed Callback
        ↕
Proposal / Revision / Publish Gate
Action Request / Object Version / Execute Gate
Audit / Reconciliation
```

目标：

> 外部调度与流程平台可以安全地使用 Ontology Platform 提供的业务能力；外部审批平台可以完成审批并通过受信回调控制 Ontology 发布，二者都无需直接访问 TiDB 或绕过 Ontology。

第四个发布门槛为 M18 General Availability：至少两个真实业务流程完成生产或准生产验证，并通过容量、安全、灾备、升级和回滚验收。

---

# 12. 最终第一阶段产品形态

第一阶段最终架构：

```text
                    Third-party Pipeline
                            │
                            ▼
                           TiDB
              Physical Business Data Source of Truth
                            │
══════════════════════════════════════════════════════════
                   Ontology Platform
══════════════════════════════════════════════════════════
                            │
            ┌───────────────┼────────────────┐
            ▼               ▼                ▼
    Control Plane     Ontology Metadata   Security Policy
     PostgreSQL          PostgreSQL          Runtime
            │               │                │
            └───────────────┼────────────────┘
                            ▼
               TiDB Mapping / Data Access
                            ▼
                Query IR / Safe SQL Compiler
                            ▼
              Object / Link / ObjectSet Runtime
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
         Action Engine             Function Runtime
               │                         │
               ▼                         │
       Internal Edit Runtime             │
               │                         │
               └────────────┬────────────┘
                            ▼
                    Audit / Outbox / Event
                            │
══════════════════════════════════════════════════════════
             API / SDK / Event / Approval Gateway
══════════════════════════════════════════════════════════
                  ┌─────────┴─────────┐
                  ▼                   ▼
      External Automation     External Approval
             Engine               Platform

Optional Projection（仅在基准证明需要时）：

TiDB Object / Link Fact
        → Materializer / CDC
        → Graph Database
        → Complex Traversal / Graph Algorithm
```

---

# 12.1 Graph Database Runtime 定位

第一阶段默认采用以下职责划分：

```text
PostgreSQL
=
Control Plane Metadata Source of Truth

TiDB
=
Physical Business Data Source of Truth
+ Default Object / Link Query Executor

Graph Database
=
Optional Rebuildable Runtime Projection
```

Graph Database 不作为第一阶段的默认前置依赖。只有满足以下条件时才引入：

```text
存在明确的复杂多跳、路径、环路或图算法业务需求
TiDB SQL / Recursive CTE 无法达到约定 SLO
数据同步延迟和最终一致性可以被业务接受
Materializer、重放、重建和一致性校验机制已经具备
```

引入后，Graph Database 只保存从 TiDB Object / Link Fact 生成的运行时投影，用于复杂关系遍历和图计算。图投影必须能够完整重建，并通过 GraphRepository / Runtime Backend 抽象访问。

本原则用于避免 10 人团队在 P0 同时承担双存储、CDC、Materialization、一致性修复、灾备和两套查询执行器的全部复杂度。

---

# 13. 第一阶段产品定位

第一阶段不以“完整复制 Foundry 全家桶”为目标。

建议正式定位为：

> **Enterprise Ontology Runtime Platform**

核心能力：

```text
Control Plane
Ontology Modeling
TiDB Mapping
Object Runtime
Link Runtime
ObjectSet / Query
Action / Internal Edit Runtime
Function Platform（精简版）
Interface
Value Type
Dependency & Impact
Data Security
External Automation / Approval Integration
Optional Graph Projection
```

暂不承担：

```text
Data Pipeline
Workflow / Automation Runtime
Approval Platform / Approval Workflow Runtime
Application Builder
AIP / AI
Agent
LLM
```

这样能够把研发资源集中在 Palantir Ontology 最核心的：

> **Semantic + Operational + Governance**

三层能力上。

## 13.1 18 个月范围约束

10 名开发人员在 18 个月内可以交付面向 1～2 个真实业务域的生产可用 V1，但不承诺 15 个模块全部达到 Foundry 同等级的成熟度。

首期必须限制为：

```text
完整 Control Plane V1：Portfolio / Complex Folder / Workspace / Branch / Proposal / External Approval Integration
1 个企业 IdP / OIDC 集成
1 种主要业务数据源：TiDB
1～2 个真实业务域
1 套统一 Action 事务模型
1 套外部 Automation 集成协议
1 套外部 Approval Platform 请求、回调、对账与发布门禁协议
Function 复用现有容器 / Kubernetes 执行底座
Graph Database 按基准测试结果决定
必要的管理 UI、REST API 和基础 SDK
```

以下能力不得在没有调整工期或人员的情况下同时承诺：

```text
自研完整多语言 FaaS / IDE / Build Platform
通用 Marketplace / Package Distribution
多图数据库运行时
无限制 Graph Algorithm Platform
完整低代码 Application Builder
跨地域 Active-Active
多种异构数据源的完整读写支持
```

若要求上述能力与 15 个模块全部达到企业级成熟度，应将计划调整为 24～30 个月，或增加 Runtime、Security、Platform Engineering 和 QA 人员。

---

# 14. 后续文档规划

建议接下来依次编写。先锁定跨模块契约和高风险架构决策，再分别展开领域 Specification：

```text
00A — Phase 1 Scope & Acceptance Baseline

00B — Runtime Storage ADR
      （TiDB Default Runtime + Optional Graph Projection）

00C — Identity / Security / Audit End-to-End Contract

00D — API / Event / Idempotency Contract

01 — Resource & Identity Specification
      （当前已有）

02 — Object Type / Property / Base Type / Value Type Specification

03 — TiDB Datasource & Ontology Mapping Specification

03A — External Data Pipeline & TiDB Integration Contract

04 — Object Runtime & Runtime Backend Abstraction Specification

05 — Link Type & Link Runtime Specification

05A — Optional Graph Projection Specification
      （仅在性能基准触发后编写）

06 — ObjectSet & Query Engine Specification

07 — Internal Edit Runtime Specification

08 — Action Engine Specification

07/08 必须联合评审，Action 是外部写入口，Edit 是内部事务执行层

09 — Function Platform Minimal Scope Specification

10 — Interface & Ontology Abstraction Specification

11 — Shared Property / Object Type Group Enhancement Specification

12 — TiDB Data Access Runtime Specification

13 — Ontology Dependency & Impact Analysis Specification

14 — Data Security Runtime Specification

15 — External Automation / Approval Integration & Event Gateway Specification
```

每份 Specification 必须包含：Scope、Non-goals、API Contract、Data Model、Security、Failure Modes、Observability、Migration、Test Plan、Performance Budget 和阶段验收标准。

---

# 15. 阶段验收与质量门槛

任何 Stage 只有同时满足功能、测试、安全、性能和运维条件后才视为完成。不得以“接口已经实现”代替阶段验收。

| 质量维度 | 最低要求 |
|---|---|
| Functional | 阶段定义的 Vertical Slice 在真实或准真实数据上端到端运行 |
| Unit / Component Test | 核心领域规则、Query IR、SQL Compiler、Policy 和 Validation 分支具备自动化测试 |
| Integration Test | PostgreSQL、TiDB、Identity Provider、External Approval Platform、Event / Webhook 等真实集成路径具备测试环境覆盖 |
| Security Test | 默认拒绝、越权读取、属性泄露、Action 越权、Service Principal 和缓存隔离测试通过 |
| Concurrency Test | Revision、Action、Optimistic Concurrency、Idempotency 和 Outbox 的竞争条件测试通过 |
| Performance Test | 明确数据规模、并发、P95/P99 延迟、吞吐、超时和资源预算，并形成可重复基准 |
| Resilience Test | TiDB 超时、部分失败、重复事件、下游不可用、节点重启和重放恢复场景通过 |
| Operability | Logs、Metrics、Tracing、Health Check、Dashboard、Alert 和 Runbook 完整 |
| Migration | Schema / Mapping / Policy 变更具备升级、兼容、回滚和数据修复方案 |
| Documentation | API、SDK、管理员、运维和故障排查文档随功能一起交付 |

## 15.1 各里程碑必须覆盖的测试

```text
M4 Complete Control Plane V1 & Modeling Foundation
├── UID / RID 全局唯一性与引用完整性
├── Portfolio / Project / 多级 Folder 创建、移动、归档与恢复
├── Folder 循环引用、越权移动和被依赖资源删除阻断
├── Workspace 成员、角色、状态与隔离
├── Branch / ChangeSet / Proposal / External Approval Request / Callback / Publish 全流程
├── 审批回调验签、幂等、防重放、乱序、旧 Revision 和伪造 Approved 拒绝测试
├── 并发修改、发布失败回滚和 Published Revision 隔离
├── Permission / Marking 默认拒绝与继承
├── Audit Event 完整性和 Transactional Outbox 重放
├── Breaking Change Validation
└── TiDB Credential 隔离

M8 Read-only Vertical Slice
├── Mapping Golden Test
├── Query IR → SQL 等价性测试
├── Row / Property Security Negative Test
├── Pagination / Aggregation / Link Traversal
└── 目标规模性能基准

M11 Operational Ontology
├── Action Validation / Permission
├── Approval-gated Action 的 Approved / Rejected / 重复回调与对象版本变化
├── Optimistic Concurrency Conflict
├── Duplicate Submission / Idempotency
├── Transaction Rollback
└── Outbox 重放与 Side Effect 失败

M14 Integration
├── Service Principal Scope
├── API Contract Compatibility
├── Event Duplicate / Out-of-order / Retry
├── Function Timeout / Isolation
├── External Approval Timeout / Reconciliation / Fail-closed
└── External Automation End-to-End

M18 General Availability
├── Production-like Load Test
├── Security Review
├── Backup / Restore Drill
├── Upgrade / Rollback Drill
├── Failure Injection
└── 两个真实业务流程验收
```

---

# 16. 关键架构决策与风险控制

## 16.1 必须在 Stage 1 锁定的决策

```text
ADR-01：TiDB 与可选 Graph Projection 的事实源边界
ADR-02：Object Identity / RID / Primary Key 规则
ADR-03：Schema Revision / ChangeSet / Publish 原子性
ADR-04：Query IR、SQL Compiler 与 Security Rewrite 顺序
ADR-05：Action / Edit Transaction / Outbox 一致性模型
ADR-06：外部 Pipeline 管理字段与 Ontology Edit 管理字段的写入边界
ADR-07：Function 执行隔离、版本和权限模型
ADR-08：Event Envelope、Idempotency Key 和 Delivery Semantics
ADR-09：外部审批请求、Revision / Action Request 绑定、回调信任、状态对账与 Publish / Execute Gate
```

## 16.2 主要风险

| 风险 | 影响 | 控制方式 |
|---|---|---|
| Control Plane V1 范围大 | Stage 1 延期会阻塞所有 Ontology Resource 和协作流程 | M1～M4 投入 4 人；按 Resource Tree、Workspace、Change Lifecycle、Governance 四个子域并行建设，以端到端发布闭环统一验收，不把 Portfolio、复杂 Folder 或完整 Workspace 后移 |
| Security 后补 | Query、Cache、Action 大面积重构 | Policy Model 从 Stage 1 开始，读取强制执行进入首个 Vertical Slice |
| TiDB 与 Graph 双运行时过早并存 | 一致性、CDC、运维和故障恢复成本失控 | TiDB 默认执行；Graph 由业务需求和性能基准触发 |
| Edit 与 Action 双写入口 | 权限绕过、语义不一致和审计缺失 | Action 对外、Edit 对内，统一事务与审计链 |
| Function Platform 过度建设 | 消耗 Runtime 主线资源 | 复用现有执行底座，只开发 Ontology Contract、权限和版本管理 |
| 最后两个月才做集成 | API 不稳定，Automation / Approval Platform 无法按期联调 | Stage 1 固化 Approval Contract，Stage 2 固化只读 API，Stage 3 固化 Action/Event Contract |
| 缺少真实业务域 | 模块完成但无法证明产品价值 | 从 Stage 1 选定试点域，所有里程碑围绕同一 Vertical Slice 演进 |
| 缺少专职质量和平台能力 | 测试、发布、灾备被持续延期 | 固定 1 人负责 Platform Engineering，各研发线共同承担自动化测试 |
| 外部 Approval Platform 不可用或状态不一致 | Proposal 无法发布、Action Request 无法执行，或错误审批结果绕过门禁 | Outbox 可靠发起、签名回调、幂等与乱序保护、定时对账、Fail-closed、告警和人工修复；不在本平台降级为自研审批 |

---

# 17. 第一阶段完成定义

18 个月计划完成时，平台必须同时满足：

```text
可以定义、版本化、评审和发布 Ontology
可以把 TiDB Table / Column 映射为 Object / Property / Link
可以通过安全的 ObjectSet API 查询、过滤、聚合和遍历对象
可以通过 Action 安全修改 Object / Link 并完整审计
可以运行受版本和权限控制的 Function
可以通过 API / Event 与外部 Automation Engine 集成
可以通过 Approval Request / Callback 与外部 Approval Platform 集成，并仅在有效审批结果下发布 Ontology 或执行需要审批的 Action
可以分析核心资源依赖和 Breaking Change 影响
可以执行对象级、行级和属性级访问控制
可以在目标规模下达到约定 SLO
可以完成备份恢复、升级回滚和故障演练
至少两个真实业务流程完成生产或准生产验收
```

只有满足上述条件，第一阶段才视为完成；“15 个模块均有代码”不等于产品已经达到可用状态。

---
