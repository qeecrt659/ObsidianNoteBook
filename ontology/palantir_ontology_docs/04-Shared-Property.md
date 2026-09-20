# Shared Property

## 1. 定义

Shared Property 是可以在多个 Object Type 上复用的属性。它的价值是让不同 Object Type 对同一业务概念使用一致的属性定义，并且集中管理属性元数据。

例如：

- Employee.startDate
- Contractor.startDate

可以共享同一个 `start date` Shared Property。

## 2. 关键特征

- 多个 Object Type 可以使用同一 Shared Property。
- 共享的是属性元数据和语义，不是底层 Object 数据。
- 元数据可以在一个地方更新，并影响使用它的多个 Object Type。
- 可以直接新建 Shared Property，也可以把已有属性转换为 Shared Property。

## 3. 元数据

官方文档列出的元数据包括：

- Name
- Description
- RID
- Base type
- Value formatting
- 以及与属性使用相关的其他配置

## 4. 为什么重要

Shared Property 是构建大型企业级 Ontology 时避免“同义不同名、同名不同义”的重要机制。

如果多个业务域都有“organizationId”“countryCode”“startDate”“currency”等公共概念，没有共享机制会快速产生语义漂移。

## 5. 官方页面

- Overview: https://www.palantir.com/docs/foundry/object-link-types/shared-property-overview
- Metadata: https://www.palantir.com/docs/foundry/object-link-types/shared-property-metadata

## 6. 自研建议

Shared Property 最好是独立可治理资源，而不是简单复制一个 Property JSON 到多个 Object Type。需要有唯一 identity、引用关系、兼容性检查和变更影响分析。
