# Palantir Ontology 元素地图与官方资料索引

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 本套说明书如何理解“Ontology 元素”

Palantir 官方的 `Ontologies > Overview` 与 `Types reference` 对 Ontology 资源边界给出了最清晰的定义。一个 Ontology 用于存放 Ontology resources，官方明确列出的一级资源包括：

1. **Object Types**
2. **Link Types**
3. **Action Types**
4. **Interfaces**
5. **Shared Properties**
6. **Object Type Groups**

同时，`Types reference` 把 **Property** 明确描述为 Object Type 的组成要素。Action Type 的官方文档又明确把 **Parameters、Rules、Submission Criteria、Side Effects** 作为其核心配置结构。因此，本套文档按三层组织：

### A. 一级 Ontology Resources

- Object Type
- Link Type
- Action Type
- Interface
- Shared Property
- Object Type Group

### B. 官方明确的关键子元素

- Property
- Action Parameter
- Action Rules
- Submission Criteria
- Action Side Effects

### C. 与 Ontology 强绑定但不属于一级 Ontology Resource 的类型/能力

- Value Type
- Base Type
- Functions / Functions on Objects
- Security & Permissions
- 通用 Metadata：Status、API Name、Visibility、Type Class、Render Hint 等

## 2. Palantir Ontology 的核心产品思想

Palantir 现在把 Ontology 描述为企业决策系统的核心，而不仅仅是“语义数据模型”。其设计把企业决策拆成四个互相关联的层面：**Data、Logic、Action、Security**。

- Object、Property、Link 把数据映射成企业现实中的实体、事件、属性和关系。
- Functions 与其他逻辑资产表达计算、推理、规则和算法。
- Action Types 把“可执行变化”作为 Ontology 中的一等能力。
- Security 贯穿读取、编辑、Action 执行、外部系统写回等环节。

因此，如果要实现 Palantir-style Ontology 平台，不应该只实现一个“图模型设计器”。真正完整的产品至少要同时覆盖：类型定义、数据映射、关系、行为、业务约束、运行时权限、版本与变更治理、应用/API 消费。

## 3. 元素之间的关系

```text
Ontology
├── Object Type
│   ├── Property
│   │   ├── Base Type
│   │   └── 可选 Value Type
│   ├── Primary Key / Title Key
│   ├── Datasource Mapping
│   ├── Interface Implementation
│   └── Object instances
├── Shared Property
├── Link Type
│   ├── Cardinality
│   ├── Foreign-key / Join-table / Object-backed storage
│   └── Link instances
├── Action Type
│   ├── Parameters
│   ├── Rules
│   ├── Submission Criteria
│   ├── Side Effects / Webhooks / Notifications
│   ├── Permissions / Authorizations
│   └── Observability / Action Log
├── Interface
│   ├── Interface Properties
│   ├── Link Type Constraints
│   ├── Action Type Constraints
│   └── Interface Extension
└── Object Type Group
```

Functions、Value Types、Security Policies 等与上述资源紧密绑定，但官方并未把它们全部列入 Ontology resources 一级清单。

## 4. 对自研 Ontology 平台的直接启示

如果希望产品模型尽可能贴近 Palantir，元模型层建议至少保留以下分层：

- **语义层**：Object Type、Property、Shared Property、Link Type、Interface、Group。
- **行为层**：Action Type、Parameter、Rule、Submission Criteria、Side Effect。
- **类型层**：Base Type、Value Type、Struct 等。
- **治理层**：Status、Visibility、API Name、Permissions、Security Policies、版本、依赖与 breaking change 检测。
- **运行层**：Object/Link 实例、索引、搜索、Object Set、Action Execution、Writeback、Functions。

产品上最容易出现的偏差，是把 Object/Link 做得很完整，却把 Action、Interface、Security、版本治理当成附属功能。Palantir 官方文档已经说明，这些能力共同组成可运行的企业 Ontology。

## 官方原始资料

- https://www.palantir.com/docs/foundry/ontologies/ontologies-overview
- https://www.palantir.com/docs/foundry/object-link-types/type-reference
- https://www.palantir.com/docs/foundry/ontology/overview/
- https://www.palantir.com/docs/foundry/ontology/why-ontology
- https://www.palantir.com/docs/foundry/ontology-manager/overview/
