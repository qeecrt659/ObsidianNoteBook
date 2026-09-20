# Ontology 核心概念

## 1. 官方定位

Palantir 把 Ontology 描述为组织的 operational layer（操作层）和数字孪生的一种实现方式。它位于 Foundry 已集成的 datasets、virtual tables、models 之上，把这些数字资产映射到现实世界的实体、事件、关系和业务操作。

从 Palantir 的表达看，Ontology 既不是单纯的数据字典，也不是只做 ER/Schema 建模，而是把：

- 语义：Objects、Properties、Links
- 动态/业务行为：Actions、Functions
- 安全与治理：Dynamic security、permissions

连接到同一套业务模型上。

## 2. 数据世界与 Ontology 的映射

Palantir 的 Core concepts 给出了很直观的类比：

| 数据结构 | Ontology |
|---|---|
| Dataset | Object Type |
| Row | Object |
| Column | Property |
| Field / Cell | Property Value |
| Join | Link Type |

这个类比非常重要：Ontology 并不是把数据重新复制成“图”，而是给数据增加业务语义、关系、行为和治理层。

## 3. 元数据与实例数据要分开

Palantir 明确区分：

- **Ontology resource / type-level metadata**：Object Type、Link Type、Action Type 等定义。
- **Ontology data**：Object、Link 以及实际 property values。

例如 `Employee` 是 Object Type；“Melissa Chang”是一个 Employee Object；`role` 是 Property；`software engineer` 是 property value。

## 4. 核心官方页面

- https://www.palantir.com/docs/foundry/ontology/overview
- https://www.palantir.com/docs/foundry/ontology/core-concepts
- https://www.palantir.com/docs/foundry/ontologies/ontologies-overview
- https://www.palantir.com/docs/foundry/object-link-types/type-reference

## 5. 自研平台落地含义

自研平台的元模型最好也把“类型定义”和“运行态实例”分层：

- Design / Metadata plane：Object Type、Property、Link Type、Action Type、Interface 等。
- Runtime / Data plane：Object instances、Link instances、property values、action executions。

这样后续做版本化、权限、SDK、索引、缓存、图查询和动作执行时会更清晰。
