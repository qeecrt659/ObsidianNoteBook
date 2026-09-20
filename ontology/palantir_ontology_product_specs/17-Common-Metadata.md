# Ontology 通用 Metadata 产品说明书：Status、API Name、Visibility、Type Class、Render Hint

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 为什么单独整理 Metadata

Palantir Ontology 中大量资源都不只有“名称 + ID”。Status、API Name、Visibility、Type Class、Render Hint 等共同决定资源的生命周期、程序契约、应用呈现和索引行为。

如果自研平台忽略这层，最终会出现“模型能建，但无法治理、无法稳定提供 API”的问题。

## 2. Status

官方对 Object Type、Property、Link Type、Action、Interface 等提供生命周期状态。核心包括：

- **experimental**：仍在开发/验证；
- **active**：已被用户应用正式依赖，应避免 breaking changes；
- **deprecated**：不再推荐使用，等待迁移；
- **example**：示例资源；
- **promoted**：Object Type 特有，表示核心可信资源。

Status 不只是标签。Palantir 会对 active 等资源加强保护，例如限制某些 API Name 变更和删除操作。

## 3. API Name

API Name 是程序化访问的稳定名称，与 Display Name 分离。

典型约束：

- Object Type：PascalCase，跨 Object Types 唯一；
- Property：camelCase，在所属 Object Type 中唯一；
- Link side：camelCase，在相关 Object Type 的 Links 中唯一；
- 长度与 reserved keywords 受限制。

一旦 API Name 被 SDK、Functions、应用代码使用，修改就可能产生 breaking change。

## 4. ID / RID / API Name 的区别

建议自研平台保留三个概念：

- **Internal immutable ID/RID**：平台内部稳定标识；
- **API Name**：程序员友好且稳定的外部契约；
- **Display Name**：业务用户友好的可变名称。

把三者合一会导致后续重命名困难。

## 5. Visibility

Visibility 是给用户应用的展示提示，例如：

- prominent
- normal
- hidden

它影响应用如何优先展示资源，但不应被当作安全策略。hidden 不等于没有读取权限。

## 6. Type Classes

Type Classes 可用于 Property、Link Type、Action Type，为应用提供额外 metadata。部分历史 type class 已逐步迁移到 Capabilities 配置。

自研时可以借鉴这种“扩展 metadata”机制，但应避免依赖无 schema 的任意 key-value。更好的方式是 versioned capability extension。

## 7. Render Hints

Render Hint 会影响 Property 的应用/索引行为，例如 searchable、sortable。关闭不需要的能力可以降低索引成本。

这意味着 Ontology metadata 与 runtime physical design 存在联系。

## 8. Change Management

Metadata 不应都允许任意修改。建议定义风险级别：

### 低风险

- Description
- Icon
- 非契约型 Display formatting

### 中风险

- Visibility
- Group
- Render Hint

### 高风险 / Breaking

- API Name
- Primary Key
- Base Type
- Link cardinality/key mapping
- 删除 active resource
- Interface required member

## 9. 自研建议

- 所有资源统一实现 Metadata contract。
- Status 驱动不同的编辑保护策略。
- API Name 有发布后稳定性保证。
- 保存前执行 dependency/breaking-change analysis。
- 为每个版本记录 who/when/why/change-set。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/metadata-statuses
- https://www.palantir.com/docs/foundry/object-link-types/metadata-typeclasses
- https://www.palantir.com/docs/foundry/object-link-types/create-object-type/
- https://www.palantir.com/docs/foundry/object-link-types/link-type-metadata
- https://www.palantir.com/docs/foundry/object-link-types/property-metadata
