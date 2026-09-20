# Object Type 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义与定位

Object Type 是现实世界实体或事件的 **schema definition**。Object 是该类型的单个实例；Object Set 是同一类型的一组对象。

例如 `Employee` 是 Object Type，某个具体员工是 Object；`Flight` 也可以是 Object Type，某次具体航班是 Object。Palantir 明确允许实体与事件都成为 Object Type，因此设计时不应局限于“主数据实体”。

## 2. 核心职责

Object Type 同时承担四类职责：

1. **语义定义**：说明这个业务对象是什么。
2. **属性契约**：定义该对象拥有的 Properties。
3. **数据映射**：把 datasource 中的数据映射成对象实例。
4. **应用/API 契约**：为 Object Explorer、Workshop、Functions、Ontology SDK 等提供稳定类型接口。

## 3. 主要元数据

官方 Metadata reference 中的核心字段包括：

- **ID**：类型级唯一标识，用于平台内部及部分配置引用；创建后不应随意变化。
- **RID**：Foundry 自动生成的全局资源标识，常用于错误与资源追踪。
- **Display name / Plural display name**：面向最终用户的单数、复数名称。
- **Description**：给使用者解释类型业务含义。
- **Icon**：应用中的视觉标识。
- **Groups**：用于发现和分类。
- **API name**：代码/API 使用的稳定名称。
- **Visibility**：normal / prominent / hidden 等展示信号。
- **Status**：experimental、active、deprecated；Object Type 还可能使用 promoted。
- **Index status**：反映最近一次索引状态。
- **Writeback**：表示是否启用用户编辑/写回能力。

## 4. Properties、Primary Key 与 Title Key

一个 Object Type 由多个 Property 构成。

### Primary Key

用于唯一标识对象实例。底层 datasource 中映射到主键的值必须能够唯一确定对象。主键选择会直接影响链接、API 访问、Action 编辑和索引，因此属于高风险 schema 决策。

### Title Key

用于最终用户界面显示对象名称。例如 Employee 可以把 `fullName` 作为 title key。Title Key 与 Primary Key 的目的不同：前者面向人，后者面向稳定唯一标识。

## 5. Datasource Mapping

创建 Object Type 时，Palantir 推荐通过引导式流程选择 backing datasource。选择数据源后，可自动将列映射为 Properties，再由建模者删除不需要的映射或修正元数据。

Object Type 也可以在没有现成数据源时创建；在支持的存储模式中，可以先创建权限所需的数据集，再逐步补充数据。

产品实现时建议把以下对象分开：

- Object Type metadata
- Datasource reference
- Datasource field → Property mapping
- Primary/title key mapping
- Index/materialization state

## 6. 创建流程

官方创建向导大致覆盖：

1. 选择或创建 datasource；
2. 设置 Object Type metadata；
3. 创建/映射 Properties；
4. 配置 Primary Key / Title Key；
5. 可选生成 Actions；
6. 选择保存位置；
7. 保存到 Ontology。

这说明 Object Type 的产品创建体验不应只是一张“类型表单”，而应把数据接入、属性、键、行为一起串联。

## 7. 编辑与 Breaking Change

Palantir 特别警告 Object Type 和 Property 的修改可能破坏下游应用。高风险变化包括：

- 替换 backing datasource；
- 修改 primary key；
- 删除 Object Type；
- 删除被应用引用的 Property；
- 修改 active 资源的 API 契约。

部分变更会触发重新索引，旧 Object Storage 模式下甚至可能导致对象短时不可用。自研平台需要有依赖分析和变更预览，而不能直接保存。

## 8. 编辑与 Writeback

Object Type 可以支持用户通过 Action 编辑。OSv2 中可以直接启用编辑能力；旧 OSv1 依赖 writeback dataset。无论底层实现怎样，产品层应区分：

- source-of-truth 数据；
- 用户/Agent 产生的 edits；
- 合并后的当前对象状态；
- edit history / provenance。

## 9. 安全

Object Type 的可见性与数据读取权限不是同一个概念。运行时还可能通过 Object Security Policy 对对象实例做 row-level filtering，并通过 Property Security Policy 做 column/cell-level 控制。

因此需要至少区分：

- Resource permission：谁能看/编辑 Object Type 定义；
- Object data permission：谁能看哪些对象；
- Property data permission：谁能看哪些字段；
- Action permission：谁能修改对象。

## 10. 与其他元素关系

- Property：描述对象特征。
- Link Type：把对象连接到其他对象。
- Action Type：创建、修改、删除对象或改变与其他对象的关系。
- Interface：Object Type 可以实现一个或多个 Interface。
- Object Type Group：用于分类、发现。
- Functions：可读取、搜索、聚合、遍历和编辑对象。

## 11. 设计建议

- 一个 Object Type 应对应一个稳定的业务概念，而不是某张物理表。
- Primary Key 应稳定、不可变、避免时间戳/浮点数等脆弱主键。
- Display Name 可业务化，API Name 必须稳定且适合代码使用。
- active/promoted 后应提高 breaking change 门槛。
- 先设计“对象如何被消费”，再决定哪些底层字段暴露为 Property。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/object-types-overview
- https://www.palantir.com/docs/foundry/object-link-types/create-object-type/
- https://www.palantir.com/docs/foundry/object-link-types/edit-object-type
- https://www.palantir.com/docs/foundry/object-link-types/object-type-metadata
- https://www.palantir.com/docs/foundry/object-link-types/allow-editing
