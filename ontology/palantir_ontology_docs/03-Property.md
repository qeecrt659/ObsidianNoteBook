# Property

## 1. 定义

Property 是 Object Type 上某一现实世界特征的 schema definition；Property Value 是该属性在某个 Object 实例上的实际值。

例如 `Employee` 可以有：

- `employee number`
- `start date`
- `role`

## 2. 关键元数据

Palantir 的 Property metadata 包括或涉及：

- ID
- Display name
- Description
- RID
- Status
- Base type
- Value formatting
- Conditional formatting
- Type classes / capabilities
- Render hints
- Visibility

Property 不只是“字段名 + 数据类型”，而是带有较丰富的产品语义和应用层展示信息。

## 3. Base Type

Base Type 决定属性能存什么类型的数据，也决定很多用户应用中可以对该属性做哪些操作。

常见类型包括 String、Integer、Long、Boolean、Date、Timestamp，以及更高级的 Vector、Geopoint、Geoshape、Attachment、Time series 等。

## 4. Primary Key / Title Key

Object Type 通常需要 Primary Key 来唯一识别 Object；Title Key 则用于更友好地展示对象。

在自研平台里不要把这两个概念混合：

- Primary Key：身份唯一性。
- Title Key：用户可读展示。

## 5. Property 与 Shared Property

普通 Property 通常属于某个 Object Type；Shared Property 可以跨多个 Object Type 复用属性元数据和统一语义。

## 6. 官方页面

- Properties overview: https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- Property metadata: https://www.palantir.com/docs/foundry/object-link-types/property-metadata
- Base types: https://www.palantir.com/docs/foundry/object-link-types/base-types
- Value types: https://www.palantir.com/docs/foundry/object-link-types/value-types-overview

## 7. 对自研平台的建议

Property 元模型建议至少覆盖：标识、名称、描述、状态、基础类型、nullable/required 语义、是否主键、是否 title key、搜索/排序能力、格式化、可见性、可选 Value Type 约束。
