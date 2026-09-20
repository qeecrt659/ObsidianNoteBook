# Ontology 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 产品定义

Ontology 是 Palantir 平台中用于承载 Ontology resources 的顶层工件。它不是单纯的 schema registry，也不是单纯的知识图谱，而是把企业的真实数据、语义关系、业务逻辑、可执行动作和安全治理统一到一个可供人和 AI 使用的操作模型中。

官方定义的 Ontology resources 包括 Object Types、Link Types、Action Types、Interfaces、Shared Properties 和 Object Type Groups。Property 则作为 Object Type 的关键组成部分存在。

## 2. 产品定位

从传统架构看，Ontology 位于“原始数据/业务系统”与“分析、应用、AI、自动化”之间。其价值是：

- 把底层表、流、文件、API 等技术数据模型映射为业务世界中的实体与事件；
- 把跨系统关系转化为 Link；
- 把可执行操作转化为 Action；
- 把逻辑通过 Functions 等能力暴露给应用、用户和 Agent；
- 用统一安全模型保证同一语义资源在不同应用中保持一致的访问和操作约束。

## 3. Ontology 与 Space 的关系

Palantir 官方说明，一个 Ontology 与一个 Space 之间是 **1:1 映射**。创建 Space 时会同步创建同名 Ontology，并继承 Space 的组织标记。Ontology 可以是 private，也可以是 shared；shared ontology 用于在多个组织之间安全共享数据和工作流。

因此 Ontology 的边界同时承担：

- 模型命名空间；
- 组织/协作边界；
- 权限和资源归属边界；
- Value Type 等空间级资源的消费边界。

## 4. 决策中心模型

Palantir 在新的 Ontology 产品叙事中，把企业决策拆成四部分：

### Data

Object、Property、Link 把 ERP、MES、WMS、IoT、流数据、非结构化数据等形成企业现实的语义表示，并可以持续更新。

### Logic

业务规则、算法、预测、优化、模型和 Functions 形成决策逻辑层。Ontology 的作用不是强制所有逻辑都运行在同一个引擎中，而是提供一致的上下文和接口把异构逻辑资产连接起来。

### Action

Action Type 把现实中的“动词”建模为受控、可审计、可执行的操作。例如批准订单、重新分配资源、修改工单状态、写回 SAP 等。

### Security

安全不是外围 ACL，而是贯穿 Object、Property、Action、Function、写回和 Agent 工具调用的运行时约束。

## 5. 生命周期与治理

Ontology 不是一次性生成的静态模型。Palantir 通过 Ontology Manager、branching/change management、status metadata、依赖视图和 Marketplace 等能力支持持续演进。

在产品层面，应至少考虑：

- draft/experimental/active/deprecated 等生命周期信号；
- breaking change 检测；
- API name 稳定性；
- 上游 datasource 变更与 reindex 影响；
- 下游应用、Function、Action 的依赖；
- 分支、评审、发布、恢复；
- 共享 Ontology 的组织边界。

## 6. 运行时视角

Ontology 元数据由 Ontology Metadata Service 等后台组件维护；对象的查询、搜索、聚合由对象存储和 Object Set Service 等能力处理；Actions 负责受控写入；Functions on Objects 处理自定义逻辑。

这意味着自研平台需要把 **设计态 metadata** 与 **运行态 object data** 分离建模，但在用户体验上保持统一。

## 7. 设计建议

1. Ontology 必须成为业务 API 和应用的稳定契约，而不是底层表的镜像。
2. 类型、行为、安全、版本必须在同一治理框架下演进。
3. 不要把业务表一表一 Object Type 机械映射；应围绕真实业务实体、事件、决策对象建模。
4. Action 应与 Object/Link 同等重要，否则 Ontology 会退化为只读语义层。
5. API Name 和稳定标识符应与 Display Name 分离。

## 官方原始资料

- https://www.palantir.com/docs/foundry/ontologies/ontologies-overview
- https://www.palantir.com/docs/foundry/ontology/overview/
- https://www.palantir.com/docs/foundry/ontology/why-ontology
- https://www.palantir.com/docs/foundry/object-backend/overview
