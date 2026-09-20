# Shared Property 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Shared Property 是可以被多个 Object Type 使用的 Property。其核心目标是 **跨类型共享 Property metadata，但不共享底层对象数据**。

例如 Employee 与 Contractor 都包含 `start date`，可以共同使用一个 Shared Property。之后对 `start date` 的语义、类型、格式等元数据做统一管理，而每个 Object Type 仍保存各自的数据。

## 2. 产品价值

Shared Property 解决的是 Ontology 中最常见的语义漂移问题：不同团队为同一业务概念反复建立类似 Property，导致名称、类型、描述和约束不一致。

它提供：

- 统一语义定义；
- 元数据集中治理；
- 多 Object Type 复用；
- Interface 建模中的统一属性基础；
- Value Type、展示格式等配置的复用入口。

## 3. 元数据

Shared Property 官方元数据包括：

- Name
- Description
- RID
- Base Type
- Value Formatting
- Type Classes
- Render Hints
- Visibility
- 权限
- Usage：哪些 Object Type 正在使用

与普通 Property 相比，Shared Property 的关键差异是它自身是独立 Ontology resource，因此拥有独立资源生命周期和权限。

## 4. 创建方式

有两条主要路径：

1. 在 Ontology Manager 的 Shared properties 页面直接创建；
2. 将现有 Object Type 的普通 Property 转换为 Shared Property。

创建时至少需要配置名称、描述、类型等元数据，然后再把 Shared Property 添加到一个或多个 Object Type。

## 5. 使用方式

Shared Property 被添加到 Object Type 后，该 Object Type 仍需要决定它的数据从哪里来。换言之：

- Shared Property = 共享语义/元数据；
- Object Type 的属性映射 = 具体数据值来源。

这一点非常重要。自研实现不应让 Shared Property 变成“全局共享字段值”。

## 6. 编辑与传播

Shared Property 的元数据在一个中心位置维护，修改后会影响所有使用它的 Object Type。因此需要：

- Usage/依赖分析；
- breaking change 检查；
- 对 Base Type 等高风险修改设置限制；
- 对下游 Interface、Action、应用做影响提示。

官方允许在 Usage tab 查看使用方，并在 Permissions tab 管理权限。

## 7. 删除语义

Palantir 的一个重要产品行为是：删除 Shared Property 时，正在使用它的 Object Type 不会简单丢失字段，而会把这些字段恢复为普通 Property。这种设计避免了“删除全局语义资源导致多个对象类型同时丢字段”的灾难性影响。

## 8. 与 Interface 的关系

Interface 属性可以本地定义，也可以使用 Shared Property。对于企业级公共语义，例如 `name`、`location`、`startDate`、`status` 等，Shared Property 往往是构建可复用 Interface 的重要基础。

## 9. 与 Value Type 的关系

Shared Property 可以应用 Value Type，从而同时共享：

- 字段业务语义；
- 数据基础类型；
- 可复用校验约束。

例如企业统一的 `EmailAddress` Value Type 可以绑定到不同 Shared Property/Property。

## 10. 设计建议

- 只把跨领域真正统一的概念提升为 Shared Property，避免“所有字段都全局化”。
- Shared Property 应有明确 owner。
- 对高复用 Shared Property 建立严格的 breaking-change 流程。
- 在产品中提供 Usage、依赖、变更影响范围。
- 不要把 Shared Property 和 Interface 混为一谈：前者共享属性语义，后者共享对象形状与能力契约。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/shared-property-overview
- https://www.palantir.com/docs/foundry/object-link-types/create-shared-property
- https://www.palantir.com/docs/foundry/object-link-types/edit-shared-property
- https://www.palantir.com/docs/foundry/object-link-types/shared-property-metadata
