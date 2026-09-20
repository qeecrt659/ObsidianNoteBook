# Palantir 中 7 个业务域、81 个 IT 系统的 Discovery 架构设计

## 1. 核心结论

对于当前的场景：

- 7 个业务域
- 81 个 IT 系统
- 每个 IT 系统已经挖掘出一套源 Ontology
- 不同 Ontology 中可能存在同名 Object Type
- 当前目标首先是保留原貌、观察、比较，再逐步做语义归并

推荐的 Discovery 架构是：

```text
1 Space
  ↓
1 Ontology
  ↓
7 Portfolios
  ↓
约 81 个 System Projects
  ↓
Folders / Resources
```

而不是：

```text
1 Space
  ↓
7 Folders
  ↓
81 Projects
```

原因是：

> Portfolio 更适合在同一个 Space 中组织多个 Projects，而 Folder 更适合组织 Project 内部的 Resources。

---

# 2. 为什么 7 个业务域更适合映射成 7 个 Portfolio

现实业务结构是：

```text
7 Business Domains
       │
       └── 81 IT Systems
               │
               └── 81 Source Ontologies
```

推荐映射到 Palantir：

```text
                           SPACE
                             │
                    ONE COMMON ONTOLOGY
                             │
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
   Portfolio D01         Portfolio D02        ... Portfolio D07
   Business Domain 1     Business Domain 2        Business Domain 7
       │                     │                     │
   ┌───┼────┐            ┌───┼────┐            ┌──┼──┐
   │   │    │            │   │    │            │  │  │
 SYS01 SYS02 SYS03      SYS14 SYS15 SYS16      ...   SYS81
   │   │    │
Project Project Project
```

因此可以把：

```text
Business Domain
```

映射为：

```text
Portfolio
```

而把：

```text
IT System / Source Ontology
```

映射为：

```text
Project
```

---

# 3. 示例结构

例如：

```text
Space:
Enterprise Ontology Discovery

Ontology:
Enterprise Ontology Discovery
```

业务域 1：

```text
Portfolio:
D01 - Customer Domain

    Project:
    D01-SYS01-SAP

    Project:
    D01-SYS02-Salesforce

    Project:
    D01-SYS03-Billing

    Project:
    D01-SYS04-MDM
```

业务域 2：

```text
Portfolio:
D02 - Manufacturing Domain

    Project:
    D02-SYS15-MES

    Project:
    D02-SYS16-PLM

    Project:
    D02-SYS17-SAP-PP
```

一直扩展到：

```text
Portfolio:
D07 - Business Domain 7

    Project:
    D07-SYS79

    Project:
    D07-SYS80

    Project:
    D07-SYS81
```

---

# 4. 81 个 Project 仍然属于同一个 Ontology

需要特别注意：

```text
81 Projects
      │
      └────────────┐
                   ▼
             同一个 Space
                   │
                   ▼
             同一个 Ontology
```

因此：

> 这 81 套源 Ontology 在 Palantir 中不会成为 81 个真正独立的 Palantir Ontology，而是成为同一个 Ontology 中的 81 个 Source Ontology Resource Groups。

例如：

```text
Project D01-SYS01-SAP
├── Customer
├── Order
└── Product

Project D01-SYS02-CRM
├── Customer
├── Account
└── Opportunity
```

这里两个 `Customer` 是两个不同 Object Types。

---

# 5. 同名 Object Type 怎么处理

例如：

```text
SAP Ontology
└── Customer

CRM Ontology
└── Customer

Billing Ontology
└── Customer
```

因为它们进入同一个 Palantir Ontology，所以技术 API Name 不能相同。

可以设计成：

```text
SAP.Customer
Display Name: Customer
API Name: SapCustomer
```

```text
CRM.Customer
Display Name: Customer
API Name: CrmCustomer
```

```text
Billing.Customer
Display Name: Customer
API Name: BillingCustomer
```

推荐 API Name 命名规范：

```text
<SystemCode><OriginalObjectTypeName>
```

例如：

```text
SAPCustomer
CRMCustomer
BillingCustomer

SAPOrder
CRMOrder
OMSOrder

SAPProduct
PIMProduct
PLMProduct
```

如果还需要体现业务域，可以使用：

```text
<DomainCode><SystemCode><ObjectType>
```

例如：

```text
D01CRMCustomer
D01SAPCustomer

D02SAPMaterial
D03MESMaterial
```

---

# 6. Project 可以很好地保留来源信息

例如看到：

```text
Customer
API Name = SalesforceCustomer
```

因为它保存于：

```text
Project:
D01-SYS02-SALESFORCE
```

所以可以立即知道：

```text
Business Domain
      ↓
IT System
      ↓
Project
      ↓
Original Ontology Resources
```

例如：

```text
Domain 1
   ↓
Salesforce
   ↓
D01-SYS02-SALESFORCE Project
   ↓
Customer
Account
Opportunity
```

因此 Project 可以很好地承载：

- Source System ownership
- Source Ontology provenance
- Resource permissions
- Team responsibility

---

# 7. Folder 应该放在 Project 内部

推荐资源层级：

```text
Space
  ↓
Portfolio
  ↓
Project
  ↓
Folder
  ↓
Resources
```

例如一个 SAP Project 内部可以这样组织：

```text
Portfolio: Domain 01
│
└── Project: D01-SYS01-SAP
       │
       ├── /ontology
       │      ├── Customer
       │      ├── Order
       │      ├── Product
       │      ├── Link Types
       │      └── Action Types
       │
       ├── /data
       │      ├── raw
       │      ├── cleaned
       │      └── ontology-ready
       │
       ├── /pipelines
       │
       ├── /analysis
       │
       └── /documentation
```

因此：

```text
Portfolio
=
组织多个 Projects
```

而：

```text
Folder
=
组织单个 Project 内部的 Resources
```

---

# 8. 权限边界仍然可以保持在 Project 层

即使多个 Project 属于同一个 Portfolio，也不意味着自动共享 Project 权限。

例如：

```text
Portfolio D01
│
├── SAP Project
│      SAP Team = Editor
│      Domain Team = Viewer
│
├── CRM Project
│      CRM Team = Editor
│      Domain Team = Viewer
│
└── Billing Project
       Billing Team = Editor
       Domain Team = Viewer
```

这样：

- SAP 团队可以编辑 SAP 对应的 Ontology Resources
- CRM 团队可以编辑 CRM 对应的 Ontology Resources
- Billing 团队可以编辑 Billing 对应的 Ontology Resources
- Domain Team 可以作为跨系统观察者

Project-based Ontology permissions 可以让同一个 Ontology 中不同资源拥有不同的 view/edit/manage 权限。

---

# 9. 建议每个业务域增加一个 Domain-Core Project

当开始做语义归并时，可以为每个业务域建立一个：

```text
Domain-Core Project
```

例如：

```text
Portfolio D01 - Customer Domain
│
├── D01-DOMAIN-CORE
│      │
│      ├── Customer
│      ├── Contract
│      ├── Account
│      └── Domain Interfaces
│
├── D01-SYS001-SAP
│      ├── Customer
│      ├── Order
│      └── Product
│
├── D01-SYS002-CRM
│      ├── Customer
│      ├── Account
│      └── Opportunity
│
└── D01-SYS003-BILLING
       ├── Customer
       ├── Account
       └── Invoice
```

这样就形成两个层次：

```text
SOURCE / AS-IS

SYS001-SAP
SYS002-CRM
SYS003-BILLING
```

以及：

```text
CANONICAL / TO-BE

D01-DOMAIN-CORE
```

例如：

```text
SAP.Customer ─────────┐
                      │
CRM.Customer ─────────┼──→ D01 Core.Customer
                      │
Billing.Customer ─────┘
```

这样可以逐步完成 Canonicalization，而不需要马上删除或修改原始 Source Object Types。

---

# 10. 后续还可以增加 Enterprise-Core Project

当 7 个业务域逐步整理完成之后，可以再增加企业级共享语义：

```text
SPACE
│
├── Portfolio D01
│      ├── Domain-Core
│      └── System Projects...
│
├── Portfolio D02
│      ├── Domain-Core
│      └── System Projects...
│
...
│
├── Portfolio D07
│      ├── Domain-Core
│      └── System Projects...
│
└── Enterprise-Core Project
       │
       ├── Shared Interfaces
       ├── Shared Properties
       ├── Party
       ├── Organization
       ├── Location
       └── Other Enterprise Concepts
```

最终形成三层：

```text
              Enterprise Canonical
                      ▲
                      │
               7 Domain Canonical
                      ▲
                      │
               81 System Models
```

---

# 11. 推荐的完整架构

```text
                         SPACE
               Enterprise Ontology Discovery
                           │
                        ONTOLOGY
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
   DOMAIN 01           DOMAIN 02            DOMAIN 07
   Portfolio           Portfolio            Portfolio
       │                   │                    │
       │                   │                    │
 ┌─────┼─────┐       ┌─────┼─────┐        ┌────┼────┐
 │     │     │       │     │     │        │    │    │
SYS01 SYS02 SYS03    SYS14 SYS15 SYS16 ... SYS79 SYS80 SYS81
 │     │     │
 ▼     ▼     ▼
Project Project Project
 │
 ├── Customer
 │     API = Sys01Customer
 │
 ├── Order
 │     API = Sys01Order
 │
 └── Product
       API = Sys01Product


                       +
                       │
                       ▼

               CANONICAL PROJECTS

        Domain01-Core
             │
             ├── Customer
             ├── Order
             └── Contract

        Domain02-Core
             │
             └── ...
```

---

# 12. 推荐最终结构

把原来的想法：

```text
1 Space
  ↓
7 Folders
  ↓
81 Projects
```

调整为：

```text
1 Space
  ↓
1 Ontology
  ↓
7 Portfolios
  ↓
≈81 System Projects
  ↓
Folders / Resources
```

进一步扩展为：

```text
SPACE
│
├── PORTFOLIO       ← 业务域
│
│      └── PROJECT  ← IT System / Source Ontology
│
│             └── FOLDER
│
│                    └── Resources
│
└── COMMON ONTOLOGY
       ↑
       │
       Object Types / Link Types / Actions
       分别保存在上面的 Projects 中
```

---

# 13. 一句话总结

> 对于 7 个业务域、81 个 IT 系统、81 套待盘点的 Source Ontology，推荐使用 **1 Space + 1 Ontology + 7 Portfolios + 81 System Projects**。Portfolio 用来表达业务域，Project 用来表达系统 / Source Ontology，Folder 用于组织 Project 内部的数据、Ontology Resources、Pipeline、文档等资源。后续再增加 7 个 Domain-Core Projects，逐步完成从 81 套 AS-IS 模型到 7 套 Domain Canonical Model，再到 Enterprise Canonical Model 的演进。
