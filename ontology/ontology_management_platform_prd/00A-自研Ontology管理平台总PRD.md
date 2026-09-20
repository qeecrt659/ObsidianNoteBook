# 自研 Ontology 管理平台总 PRD

## 1. 产品愿景

建立一个以 Palantir Ontology 语义为基准的企业级 Ontology 管理平台，使业务、数据、应用和 AI 围绕统一的 Object / Property / Link / Action / Interface 模型协作。首阶段聚焦 **Ontology 设计态与治理态**，运行态对象查询、Action 执行、Function Runtime 可以通过独立服务逐步建设。

## 2. P0 产品范围

- Ontology 管理与切换
- Object Type / Property
- Link Type
- Action Type + Parameter + Rules + Submission Criteria
- Common Metadata（ID/API Name/Status/Visibility）
- Resource Permission
- Draft / Validation / Change Set / Review / Publish
- Dependency Graph / Breaking Change 基础检测
- Audit Log

## 3. P1 产品范围

- Shared Property
- Interface
- Value Type
- Action Side Effects
- Object/Property Security Policy
- Functions binding
- 高级 Impact Analysis / Migration Guide

## 4. P2 产品范围

- 跨环境 Package / Marketplace / GitOps
- AI 辅助 Ontology 建模与重复/冲突检测
- 自动迁移建议
- Runtime 深度集成与容量治理

## 5. 核心后台服务建议

```text
Web UI / Ontology Manager
        |
API Gateway / BFF
        |
+------------------------------+
| Ontology Metadata Service    |
| Type System Service          |
| Dependency Graph Service     |
| Change Management Service    |
| Validation Service           |
| IAM / Policy Service         |
| Audit Service                |
| Search Service               |
+------------------------------+
        |
Runtime adapters (optional first phase)
Object Index / Query / Action / Function / Datasource
```

## 6. 核心发布原则

- 任何生产 Ontology resource 变更先产生 Draft revision。
- 多资源相关变更必须在同一个 Change Set 里原子发布。
- Active 资源的 API Name、Primary Key、Base Type、Cardinality、Action Parameter Contract 等属于高风险/Breaking change。
- 发布前必须运行 Validation + Dependency Impact + Permission Check。
- 发布失败不能污染已发布版本。

## 7. 建议数据库实体

- ontology
- ontology_resource
- resource_revision
- object_type_definition
- property_definition
- shared_property_definition
- link_type_definition
- action_type_definition
- action_parameter_definition
- action_rule_definition
- submission_criteria_definition
- side_effect_definition
- interface_definition
- object_type_group_definition
- value_type_definition / value_type_version
- resource_dependency
- change_set / change_set_item / review
- permission_binding / policy_definition
- audit_event

元数据关系可以用关系数据库作为事实源；dependency graph 可通过关系表/图索引加速，不要求把所有运行态对象实例存进图数据库。
