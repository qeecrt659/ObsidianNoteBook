# Property 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Property 是 Object Type 中某个现实世界特征的 schema definition；Property Value 是某个具体 Object 上该特征的实际值。官方把 Property 与数据集列类比：Property 类似列的语义定义，Property Value 类似具体行中的字段值。

## 2. 产品作用

Property 不只是“字段”。它同时决定：

- 对象可携带什么数据；
- 该数据的 Base Type / Value Type；
- 搜索、排序、聚合、渲染等应用能力；
- 是否承担 Primary Key / Title Key；
- 如何映射底层 datasource；
- 是否可编辑；
- 是否受独立安全策略保护。

## 3. 主要元数据

官方 Metadata reference 的关键内容包括：

- ID
- RID
- Display Name
- Description
- Status
- API Name
- Base Type
- Keys：Primary Key / Title Key
- Visibility
- Value formatting
- Render hints
- Type classes
- 数据映射信息

在产品模型中，建议把“语义元数据”“数据类型”“展示配置”“索引能力”“数据映射”“安全配置”分成独立子结构，而不要堆在一个 JSON 中。

## 4. Base Type

Property Base Type 定义 Property 可以存储什么数据，以及用户应用能够进行什么操作。官方支持常见的 String、Integer、Short、Date、Timestamp、Boolean、Byte、Long、Float、Double、Decimal，也支持 Array、Struct、Vector、Geopoint、Geoshape、Attachment、Time Series、Geotemporal Series、Media Reference、Marking、Cipher 等扩展类型。

一些类型不适合作为 Primary Key；Vector、Struct、附件、时间序列等也不能作为 Title Key。设计器应基于 Base Type 动态限制合法配置。

## 5. Primary Key 与 Title Key

### Primary Key

Property 可被指定为对象主键，用于唯一确定对象。该字段的稳定性和唯一性直接影响 Link 与 Action。

### Title Key

用于在用户应用中显示对象的人类可读名称。通常应选择可读性强且相对稳定的字段。

自研平台应把两者明确分开，不能把“显示名称”默认当作主键。

## 6. Datasource Mapping

常规 Property 由 Object Type 的 backing datasource 中某一列提供值。映射变化会影响对象索引和下游应用。

Palantir 还支持 **Edit-only Property**：不直接映射到 backing dataset 的列，主要承载通过 Ontology 编辑产生的数据。在 OSv2 中，这允许先定义业务字段，再通过 Action 持久化用户输入，而不要求先改底层源表。

## 7. Value Formatting 与 Display

Property 可以配置数值、日期时间、User ID、Resource ID 等格式，使原始值在用户应用中更可读。Visibility 可提示应用优先或隐藏展示。

这类配置属于 Ontology 的“语义 UI 契约”，意味着应用不需要重复定义每一个字段的基本展示规则。

## 8. Render Hints 与性能

Render hints 会影响应用能力和索引，例如 searchable、sortable 等。官方文档明确指出，关闭不需要的搜索/排序能力可以改善 reindex 性能。

所以在自研平台里，不能假设所有字段都应建立全文索引/排序索引。Property 应携带“查询能力声明”。

## 9. Required / Edit-only / Derived 等扩展能力

官方 Property 文档还覆盖：

- Required Properties：用于表达对象字段的必需性；
- Edit-only Properties：只由 Ontology edits 提供值；
- Derived Properties：运行时或查询层派生；
- Mandatory Control Properties：用于把 markings、organization、classification 等安全控制关联到对象数据。

这些能力说明 Property 是运行时数据治理单元，而不是简单 schema column。

## 10. Property Security

Property Security Policy 可以在 Object Security Policy 的基础上进一步限制字段级可见性。用户通过 Object policy 但未通过 Property policy 时，可以看到对象但对应字段值会被隐藏/置空。

主键 Property 不能加入 Property Security Policy，因为对象标识需要支持对象访问和权限评估。

## 11. 变更风险

删除 Property 可能破坏 Object Views、Workshop、Functions、SDK 代码和 Action。active Property 不能直接删除。API Name 一旦作为程序契约使用，应保持稳定。

## 12. 设计建议

- Property 名称表达业务语义，不要照搬数据库缩写。
- API Name 与 Display Name 分离。
- 对可枚举、范围、正则等业务含义强的字段优先引入 Value Type。
- searchable/sortable 按真实用例开启。
- 对敏感字段单独配置安全策略，而不是只依赖 UI 隐藏。
- 把来源映射与 Property 定义解耦，为多数据源和 schema evolution 留空间。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/property-metadata
- https://www.palantir.com/docs/foundry/object-link-types/edit-properties
- https://www.palantir.com/docs/foundry/object-link-types/edit-only-properties
- https://www.palantir.com/docs/foundry/object-link-types/base-types
- https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies
