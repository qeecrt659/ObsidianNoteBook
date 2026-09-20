# Palantir Foundry 中 HR 与采购业务域的 Ontology 组织方式

## 1. 问题背景

假设企业需要分别对两个业务域进行 Ontology 建模：

- **HR 业务域**：建立独立的 Ontology A，包含自己的 Object Type、Link Type、Action Type 等；
- **采购业务域**：建立独立的 Ontology B，同样包含自己的 Object Type、Link Type、Action Type 等。

核心问题是：在 Palantir Foundry 中，这种多业务域、多 Ontology 的场景应该如何实现？什么时候应该真正拆成多个 Ontology，什么时候更适合放到同一个企业级 Ontology 中？

---

## 2. Palantir Foundry 原生支持多个 Ontology

Palantir Foundry 支持在一个 Foundry 环境中存在多个 Ontology。

其核心组织方式可以理解为：

```text
Foundry
│
├── HR Space
│    │
│    └── HR Ontology / Ontology A
│         ├── Object Types
│         ├── Link Types
│         ├── Action Types
│         ├── Interfaces
│         ├── Shared Properties
│         └── Object Type Groups
│
└── Procurement Space
     │
     └── Procurement Ontology / Ontology B
          ├── Object Types
          ├── Link Types
          ├── Action Types
          ├── Interfaces
          ├── Shared Properties
          └── Object Type Groups
```

一个 Space 与一个 Ontology 对应，因此，如果确实需要两个相互独立的 Ontology，可以分别建立：

```text
HR Space
    ↓
HR Ontology

Procurement Space
    ↓
Procurement Ontology
```

在 Ontology Manager 中，具备相应权限的用户可以在多个 Ontology 之间进行切换。

---

## 3. HR Ontology 的示例

例如，HR Ontology 可以包含：

```text
HR Ontology
│
├── Object Types
│    ├── Employee
│    ├── Department
│    ├── Position
│    ├── Employment
│    └── ...
│
├── Link Types
│    ├── Employee ↔ Department
│    ├── Employee ↔ Manager
│    ├── Employee ↔ Position
│    └── ...
│
├── Action Types
├── Interfaces
├── Shared Properties
└── Object Type Groups
```

HR 团队可以独立管理属于 HR Ontology 的对象类型、关系类型及其他 Ontology Resources。

---

## 4. Procurement Ontology 的示例

采购业务域可以独立建立：

```text
Procurement Ontology
│
├── Object Types
│    ├── Supplier
│    ├── PurchaseOrder
│    ├── PurchaseRequisition
│    ├── Contract
│    ├── Material
│    └── ...
│
├── Link Types
│    ├── Supplier ↔ PurchaseOrder
│    ├── PurchaseOrder ↔ Contract
│    ├── PurchaseOrder ↔ Material
│    └── ...
│
├── Action Types
├── Interfaces
├── Shared Properties
└── Object Type Groups
```

采购团队也可以在自己的 Ontology 范围内独立演进。

---

## 5. Space、Ontology、Project、Object Type Group 的职责区别

在 Palantir Foundry 中，这几个概念需要明确区分。

| 层级 | Palantir 概念 | 典型用途 |
|---|---|---|
| 一级边界 | Space | 权限、资源及 Ontology 的顶层组织边界 |
| 语义模型 | Ontology | 保存 Object Type、Link Type、Action Type 等语义模型 |
| 资源组织与权限 | Project | 对具体 Ontology Resources、数据、应用进行项目级组织和权限控制 |
| Ontology 内部分类 | Object Type Group | 用于对 Object Type 进行逻辑分类、搜索和浏览 |

例如：

```text
HR Space
│
├── HR Ontology
│
├── HR Ontology Core Project
│    ├── Employee
│    ├── Department
│    ├── Position
│    └── HR Link Types
│
├── HR Data Project
│
└── HR Applications Project


Procurement Space
│
├── Procurement Ontology
│
├── Procurement Ontology Core Project
│    ├── Supplier
│    ├── PurchaseOrder
│    ├── Contract
│    └── Procurement Link Types
│
├── Procurement Data Project
│
└── Procurement Applications Project
```

---

## 6. 最关键的限制：不同 Ontology 之间不能直接建立 Link Type

这是进行企业级 Ontology 设计时非常重要的一点。

假设：

```text
HR Ontology
└── Employee

Procurement Ontology
└── PurchaseOrder
```

那么不能直接在两个 Ontology 之间定义：

```text
Employee
    │
    │ creates
    ▼
PurchaseOrder
```

因为 `Employee` 和 `PurchaseOrder` 属于两个不同 Ontology。

也就是说，**跨 Ontology 的 Object Type 之间不能直接创建 Link Type**。

这会直接决定企业业务域是否应该真正拆成多个 Ontology。

---

## 7. 为什么企业级场景中不一定应该“一业务域一个 Ontology”

真实企业业务往往是跨域连接的。

例如：

```text
Employee
   │
   │ creates
   ▼
PurchaseRequisition
   │
   │ becomes
   ▼
PurchaseOrder
   │
   │ placed with
   ▼
Supplier
```

如果：

```text
Employee
    → HR Ontology

PurchaseRequisition
PurchaseOrder
Supplier
    → Procurement Ontology
```

那么：

```text
Employee ── creates ──> PurchaseRequisition
```

就无法直接以 Link Type 的形式建立。

类似的跨域关系还可能包括：

```text
Employee → PurchaseRequest
Department → PurchaseOrder
Department → Contract
CostCenter → PurchaseOrder
PurchaseOrder → Invoice
Invoice → Payment
```

如果各业务域被拆成完全不同的 Ontology，这些跨业务域语义关系都会受到限制。

---

## 8. 更适合企业级数字孪生的方案：统一 Enterprise Ontology

如果企业的目标是构建类似 Palantir Foundry 的企业级数字孪生，更常见且更合理的设计是建立统一的 Enterprise Ontology：

```text
Enterprise Ontology
│
├── HR Domain
│    ├── Employee
│    ├── Department
│    ├── Position
│    └── Employment
│
├── Procurement Domain
│    ├── Supplier
│    ├── PurchaseOrder
│    ├── PurchaseRequisition
│    └── Contract
│
├── Finance Domain
│    ├── Invoice
│    ├── Payment
│    └── CostCenter
│
└── Manufacturing Domain
     ├── Material
     ├── BOM
     ├── WorkOrder
     └── Equipment
```

这样就可以形成统一的跨域关系：

```text
Employee
   │
   ▼
PurchaseRequisition
   │
   ▼
PurchaseOrder
   │
   ├────────► Supplier
   │
   ▼
Invoice
   │
   ▼
Payment
```

这更符合 Ontology 作为企业“数字孪生语义层”的设计思想。

---

## 9. 如何在统一 Ontology 中保持 HR 和采购的独立性

将 HR 和采购放入同一个 Ontology，并不意味着两个业务域必须混在一起管理。

可以通过 **Project、权限和 Object Type Group** 保持领域自治。

例如：

```text
                     Enterprise Ontology
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
   HR Domain          Procurement Domain      Finance Domain
       │                     │                     │
   HR Project        Procurement Project     Finance Project
       │                     │                     │
   Employee            PurchaseOrder             Invoice
   Department          Supplier                  Payment
   Position            Contract                 CostCenter
       │                     │                     │
       └────────────── Link Types ─────────────────┘
```

还可以定义 Object Type Groups：

```text
HR
Procurement
Finance
Manufacturing
Sales
```

这样可以同时实现：

```text
逻辑独立：       HR / Procurement
团队独立：       HR Team / Procurement Team
权限独立：       Project Permissions
开发独立：       Project
资源分类：       Object Type Groups
语义层统一：     Enterprise Ontology
跨域连接：       Link Types
```

---

## 10. 两种方案的对比

### 方案 A：HR 与采购分别建立独立 Ontology

```text
HR Space
   ↓
HR Ontology A

Procurement Space
   ↓
Procurement Ontology B
```

适用场景：

- HR 与采购在语义上完全独立；
- 两个业务域几乎不存在需要直接建模的跨域关系；
- 权限、组织或监管要求必须彻底隔离；
- 两个 Ontology 独立生命周期管理；
- 不需要构建统一的企业级知识图谱。

优点：

- 边界非常清晰；
- 权限隔离简单；
- 团队可以完全独立开发；
- 一个业务域的模型不会直接影响另一个业务域。

缺点：

- 不支持跨 Ontology Link Type；
- 跨业务域关联能力受限；
- 很难形成完整的企业级数字孪生。

---

### 方案 B：一个统一 Enterprise Ontology，业务域内部自治

```text
Enterprise Ontology
│
├── HR
├── Procurement
├── Finance
├── Sales
└── Manufacturing
```

使用：

- Project
- Project Permissions
- Object Type Groups
- 命名规范
- 领域 Owner

对各业务域进行治理。

优点：

- 可以建立任意跨业务域 Link Type；
- 可以形成企业级语义网络；
- 非常适合数字孪生；
- 更适合跨部门分析、AI Agent、业务推理等场景。

缺点：

- 对 Ontology 治理能力要求更高；
- 必须明确不同 Domain 的 Owner；
- 必须建立统一的命名、版本、审批及跨域依赖规范。

---

## 11. 对企业级 Ontology 平台设计的建议

如果目标是建设一个类似 Palantir Foundry 的企业 Ontology 平台，建议不要简单采用：

```text
一个业务域 = 一个物理 Ontology
```

更合理的抽象是：

```text
Ontology
   │
   ├── Domain
   │     ├── Object Types
   │     ├── Link Types
   │     ├── Action Types
   │     └── Interfaces
   │
   ├── Domain
   │
   └── Domain
```

其中：

```text
Ontology
    = 企业级语义边界

Domain / Project
    = 业务域治理边界

Object Type Group
    = Ontology 内的分类与浏览机制
```

这样能够同时满足：

1. 业务域自治；
2. 团队独立开发；
3. 权限隔离；
4. 模型独立演进；
5. 跨域语义关联；
6. 企业级统一数字孪生。

---

## 12. 最终判断原则

如果 HR 和采购：

```text
完全独立
并且不需要：
Employee → PurchaseOrder
Employee → PurchaseRequest
Department → Contract
CostCenter → PurchaseOrder
```

那么可以采用：

```text
HR Ontology A
Procurement Ontology B
```

如果未来需要构建：

```text
Employee
    ↓
PurchaseRequest
    ↓
PurchaseOrder
    ↓
Supplier
    ↓
Contract
    ↓
Invoice
    ↓
Payment
    ↓
CostCenter
```

那么更适合采用：

```text
Enterprise Ontology
    ├── HR Domain
    ├── Procurement Domain
    ├── Finance Domain
    └── ...
```

而不是把每一个业务域物理拆分成独立 Ontology。

---

## 13. 一句话总结

> **Ontology 更适合作为企业级语义边界；Project、权限以及 Object Type Group 更适合作为业务域的组织和治理边界。**

只有当两个业务域在语义和业务关联上都需要真正隔离时，才建议拆成不同 Ontology。

---

## 14. Palantir 官方参考资料

- Ontologies overview  
  https://www.palantir.com/docs/foundry/ontologies/ontologies-overview

- Ontology core concepts  
  https://www.palantir.com/docs/foundry/ontology/core-concepts

- Ontology permissions  
  https://www.palantir.com/docs/foundry/object-permissioning/ontology-permissions

- Link types overview  
  https://www.palantir.com/docs/foundry/object-link-types/link-types-overview

- Object type groups  
  https://www.palantir.com/docs/foundry/object-link-types/type-groups
