# Palantir 中 Project 只用于 Ontology 管理时的使用方式

## 1. 核心定位

如果当前阶段 **Project 只用于 Ontology 管理**，暂时不承载 Dataset、代码仓库、Pipeline、Application 等其他 Foundry 资源，那么可以把 Project 收窄为：

> **Project = 一组 Ontology Resources 的归属、权限和协作管理单元。**

此时 Project 主要解决四个问题：

1. 哪些 Ontology Resources 放在一起管理；
2. 这些资源由哪个团队负责；
3. 哪些用户或用户组可以查看、编辑和管理；
4. 不同业务域之间如何形成相对独立的治理边界。

在这个模型中：

```text
Ontology
    = 企业统一语义空间

Project
    = Ontology Resources 的治理与 Ownership 边界
```

Project **不是新的 Ontology，也不是新的 Ontology Namespace**。

---

## 2. 一个完整例子：企业级 Ontology

假设企业建立：

```text
Space: Enterprise

└── Ontology: Enterprise Ontology
```

企业有三个业务域：

```text
HR
Procurement
Finance
```

如果 Project 只负责 Ontology 管理，可以设计成：

```text
Enterprise Space
│
└── Enterprise Ontology
     │
     ├── Project: HR Ontology
     ├── Project: Procurement Ontology
     ├── Project: Finance Ontology
     └── Project: Enterprise Shared Ontology
```

注意：

```text
HR Ontology Project
Procurement Ontology Project
Finance Ontology Project
```

虽然名称中带有 `Ontology`，但它们都不是独立 Ontology。

真正的 Ontology 仍然只有：

```text
Enterprise Ontology
```

Project 只是对 Enterprise Ontology 中的不同资源进行治理和权限划分。

---

## 3. HR Ontology Project 如何使用

例如建立：

```text
Project: HR Ontology
```

其中只放 HR 领域负责的 Ontology Resources：

```text
HR Ontology Project
│
├── Object Types
│   ├── Employee
│   ├── Department
│   ├── Position
│   ├── Employment
│   └── JobPosting
│
├── Link Types
│   ├── Employee → Department
│   ├── Employee → Position
│   ├── Employee → Manager
│   └── Position → Department
│
├── Action Types
│   ├── Hire Employee
│   ├── Transfer Employee
│   └── Terminate Employment
│
├── Interfaces
│
└── Shared Properties
```

### Project 权限

例如：

```text
HR Ontology Project

HR Ontology Architects
        → Owner

HR Ontology Developers
        → Editor

HR Business Analysts
        → Viewer

Procurement Team
        → Viewer / No Access
```

这样：

### HR Ontology Developer

可以修改：

```text
Employee
Department
Position
Employment
```

包括：

```text
Properties
Descriptions
Links
Metadata
...
```

### Procurement Developer

如果没有 HR Project 的 Editor 权限，则不能随意：

```text
修改 Employee
删除 Department
修改 Position
```

所以在这个阶段，Project 最核心的价值就是：

> **决定谁负责这一组 Ontology Resources，以及谁可以修改它们。**

---

## 4. Procurement Ontology Project

采购业务域同样可以建立：

```text
Project: Procurement Ontology
```

其中包含：

```text
Procurement Ontology Project
│
├── Object Types
│   ├── Supplier
│   ├── PurchaseRequisition
│   ├── PurchaseOrder
│   ├── PurchaseOrderItem
│   └── Contract
│
├── Link Types
│   ├── Supplier → PurchaseOrder
│   ├── PurchaseOrder → PurchaseOrderItem
│   ├── PurchaseOrder → Contract
│   └── PurchaseRequisition → PurchaseOrder
│
└── Action Types
    ├── Create Purchase Order
    ├── Approve Purchase Order
    └── Cancel Purchase Order
```

权限可以设置成：

```text
Procurement Ontology Architects
        → Owner

Procurement Ontology Developers
        → Editor

Procurement Analysts
        → Viewer
```

于是：

```text
Supplier
PurchaseOrder
Contract
PurchaseRequisition
```

由采购团队负责。

---

## 5. 所有 Object Types 仍然属于同一个 Ontology

虽然这些 Object Types 分别归属于不同 Project，但从 Ontology 角度看，它们仍然处于一个统一的语义空间：

```text
Enterprise Ontology
│
├── Employee
├── Department
├── Position
├── Supplier
├── PurchaseOrder
├── Contract
├── Invoice
└── Payment
```

资源的治理归属可以是：

```text
Employee
    → HR Ontology Project

PurchaseOrder
    → Procurement Ontology Project

Invoice
    → Finance Ontology Project
```

但其真正的语义标识仍然是：

```text
Enterprise Ontology.Employee
Enterprise Ontology.PurchaseOrder
Enterprise Ontology.Invoice
```

而不是：

```text
HRProject.Employee
ProcurementProject.PurchaseOrder
FinanceProject.Invoice
```

因此：

> **Project 不形成新的 Ontology Namespace。**

---

## 6. Project 的本质：Ontology Ownership

当 Project 只管理 Ontology 时，它最核心的含义其实是：

> **“这一组 Ontology Resources 是由谁负责的？”**

例如：

| Ontology Resource | 所属 Project | Owner |
|---|---|---|
| Employee | HR Ontology | HR Team |
| Department | HR Ontology | HR Team |
| Position | HR Ontology | HR Team |
| Supplier | Procurement Ontology | Procurement Team |
| PurchaseOrder | Procurement Ontology | Procurement Team |
| Invoice | Finance Ontology | Finance Team |
| Currency | Enterprise Shared Ontology | Enterprise Ontology Team |
| PurchaseOrder → Invoice | Enterprise Shared Ontology | Enterprise Ontology Team |

因此 Project 可以承担：

```text
Project
│
├── Ontology Resource Ownership
├── Ontology Resource Permission
├── Team Collaboration
├── Domain Governance
└── Resource Organization
```

---

## 7. 跨业务域 Link Type 如何管理

例如企业中需要建立：

```text
Employee
    │
    │ submits
    ▼
PurchaseRequisition
```

其中：

```text
Employee
    → HR Ontology Project

PurchaseRequisition
    → Procurement Ontology Project
```

因为它们仍然属于同一个：

```text
Enterprise Ontology
```

所以语义上可以直接建立 Link Type。

真正需要解决的是：

> 谁负责这个跨域 Link Type？

一个推荐方案是建立：

```text
Project: Enterprise Shared Ontology
```

把跨业务域的关系统一放进去治理：

```text
Enterprise Shared Ontology Project
│
├── Cross-domain Link Types
│
├── Employee
│      │ submits
│      ▼
│   PurchaseRequisition
│
├── Department
│      │ owns
│      ▼
│   CostCenter
│
├── PurchaseOrder
│      │ generates
│      ▼
│   Invoice
│
└── Supplier
       │ party to
       ▼
    Contract
```

这个 Project 可以由：

```text
Enterprise Ontology Governance Team
```

负责。

---

## 8. 为什么跨域 Link 适合单独治理

例如：

```text
Employee ── submits ──> PurchaseRequisition
```

这个关系同时涉及：

```text
HR Domain
+
Procurement Domain
```

所以它不应该只由 HR 或采购某一方单独决定。

更合理的治理模式是：

```text
HR Team
    ↓
确认 Employee 端语义

Procurement Team
    ↓
确认 PurchaseRequisition 端语义

Enterprise Ontology Team
    ↓
管理跨域 Link Type
```

于是形成：

```text
HR Project
└── Employee ─────────┐
                      │
                      │ source
                      ▼
Shared Project     submits
                      │
                      │ target
                      ▼
Procurement Project
└── PurchaseRequisition
```

注意：

`Employee` 本身仍然只存在于 HR Project 中。

不会因为跨域 Link 而复制到 Shared Project。

---

## 9. Shared Object Type 应该放在哪里

企业级 Ontology 中还会存在真正的共享对象，例如：

```text
Organization
LegalEntity
BusinessUnit
Location
Country
Currency
Person
```

这些 Object Types 不适合归属于：

```text
HR Project
```

也不适合归属于：

```text
Procurement Project
```

因为多个业务域都会依赖它们。

因此可以放在：

```text
Enterprise Shared Ontology Project
```

例如：

```text
Enterprise Shared Ontology Project
│
├── Organization
├── LegalEntity
├── BusinessUnit
├── Location
├── Country
├── Currency
└── Shared Properties
```

其他业务域直接复用这些公共 Object Types。

例如：

```text
             Organization
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
      Department        Supplier
       HR Project      Procurement
```

这样可以避免：

```text
HR.Organization
Procurement.Organization
Finance.Organization
```

这种重复语义模型。

---

## 10. Project 内部还可以使用 Folder

如果某个业务域很大，不应该因为 Object Type 数量多就不断拆 Project。

例如 HR Project 可以内部继续使用 Folder：

```text
HR Ontology Project
│
├── Workforce/
│   ├── Employee
│   ├── Employment
│   └── Position
│
├── Organization/
│   ├── Department
│   ├── OrganizationUnit
│   └── ReportingLine
│
├── Recruiting/
│   ├── JobPosting
│   ├── Candidate
│   └── Application
│
└── Compensation/
    ├── CompensationPlan
    └── PayGrade
```

可以理解为：

```text
Project
    = Ownership / Permission 边界

Folder
    = Project 内部资源分类和组织
```

---

## 11. 什么时候应该拆 Project

### 不应该因为 Object Type 数量多而拆

例如：

```text
Employee
Department
Position
Employment
Job
Candidate
JobPosting
```

即使有几十个 Object Types，如果它们：

```text
Owner 相同
权限相同
团队相同
```

完全可以放在同一个 Project 中。

### 应该因为 Ownership 或权限不同而拆

例如：

```text
Employee
Department
Position
```

由：

```text
HR Core Ontology Team
```

负责，则可以放入：

```text
HR Core Ontology Project
```

但：

```text
Compensation
SalaryGrade
BonusPlan
```

可能具有：

```text
不同 Owner
更高敏感度
更严格修改权限
```

则可以拆成：

```text
HR Compensation Ontology Project
```

最终：

```text
HR Domain

├── HR Core Ontology Project
│   ├── Employee
│   ├── Department
│   └── Position
│
└── HR Compensation Ontology Project
    ├── Compensation
    ├── PayGrade
    └── BonusPlan
```

因此：

> **Project 的拆分依据应该优先是 Ownership 和 Permission，而不是 Object Type 数量。**

---

## 12. 推荐的企业级结构

如果当前阶段只做 Ontology Management，可以采用：

```text
Enterprise Space
│
└── Enterprise Ontology
     │
     ├── HR Ontology Project
     │    ├── Employee
     │    ├── Department
     │    └── Position
     │
     ├── Procurement Ontology Project
     │    ├── Supplier
     │    ├── PurchaseOrder
     │    └── Contract
     │
     ├── Finance Ontology Project
     │    ├── Invoice
     │    ├── Payment
     │    └── CostCenter
     │
     └── Enterprise Shared Ontology Project
          ├── Organization
          ├── LegalEntity
          ├── Currency
          └── Cross-domain Link Types
```

这样可以同时满足：

```text
统一 Ontology Namespace
+
业务域自治
+
资源归属明确
+
权限独立
+
团队独立
+
支持跨域 Link Type
+
支持共享 Object Type
```

---

## 13. 最小元模型设计

如果平台当前只做 Ontology Management，不做 Runtime、Dataset、Pipeline 和 Application，可以先保持非常简单：

```text
Space
│
│ 1 : 1
▼
Ontology
│
│ 1 : N
▼
Project
│
│ 1 : N
▼
Ontology Resource
```

其中：

```text
Ontology Resource
│
├── Object Type
├── Link Type
├── Action Type
├── Interface
└── Shared Property
```

数据库可以概念化为：

```text
space
----------------
id
name

ontology
----------------
id
space_id
name

project
----------------
id
space_id
name
description

ontology_resource
----------------
id
ontology_id
project_id
resource_type
api_name
display_name
...
```

其中两个字段职责必须分清：

### ontology_id

决定：

> 这个 Resource 属于哪个统一语义空间。

### project_id

决定：

> 这个 Resource 由哪个 Project / 团队治理。

---

## 14. Object Type 的唯一性约束

Project 不是 Ontology Namespace。

因此 Object Type 的 API Name 应该在 Ontology 范围内唯一。

推荐：

```text
UNIQUE(ontology_id, api_name)
```

而不是：

```text
UNIQUE(project_id, api_name)
```

例如：

```text
Employee

Ontology:
Enterprise Ontology

Project:
HR Ontology

Owner:
HR Ontology Team
```

以及：

```text
PurchaseOrder

Ontology:
Enterprise Ontology

Project:
Procurement Ontology

Owner:
Procurement Ontology Team
```

这说明：

```text
Ontology
    → 决定语义归属与唯一命名空间

Project
    → 决定治理归属和权限
```

---

## 15. 第一版 Project 建议只承担四项职责

如果当前只建设 Ontology Management Platform，建议第一版 Project 只承担：

1. **Ontology Resource 归属**
2. **User / Group 权限**
3. **领域 Owner**
4. **Ontology Resource 的组织**

暂时不要一开始把 Project 扩展成：

```text
Jira Project
Code Repository
Dataset Workspace
Pipeline Workspace
Application Workspace
```

等综合概念。

等后续真正扩展 Foundry Runtime 时，再逐步把：

```text
Dataset
Pipeline
Repository
Application
Workflow
```

等资源纳入 Project。

这样第一版模型简单，而且未来也不需要推翻重做。

---

## 16. 最终总结

如果 Project 当前只用于 Ontology 管理，可以把它定义为：

> **Project 是 Ontology 中一组 Ontology Resources 的 Ownership、Permission、Collaboration 和 Governance 边界。**

而 Ontology 本身负责：

> **定义企业统一的语义模型、Object Types、Link Types、Action Types 以及它们之间的关系。**

两者职责可以总结为：

```text
Ontology
    = What the enterprise means

Project
    = Who owns and manages that meaning
```

即：

> **Ontology 解决“企业世界是什么以及对象之间如何关联”；Project 解决“这些 Ontology Resources 由谁负责、谁能看、谁能改、如何组织治理”。**

---

## Palantir 官方参考

- Ontology Permissions  
  https://www.palantir.com/docs/foundry/object-permissioning/ontology-permissions

- Projects and Roles  
  https://www.palantir.com/docs/foundry/security/projects-and-roles

- Ontology project-based permissions announcement  
  https://www.palantir.com/docs/foundry/announcements/2026-01
