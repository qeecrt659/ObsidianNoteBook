# Object Type Group 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Object Type Group 是用于帮助用户搜索、发现和浏览 Ontology 的分类 primitive。它不是类型继承，也不是权限组，更不会改变 Object Type schema。

## 2. 核心用途

当 Ontology 中 Object Types 达到几十、几百甚至更多时，单纯按名称搜索会降低可发现性。Group 提供业务分类入口，例如：

- HR
- Supply Chain
- Customer 360
- Finance
- Manufacturing

一个 Object Type 可以属于一个或多个 Groups。

## 3. 配置方式

Group 由 Ontology Manager 管理，通常由 Ontology owner/editor 创建。也可以在 Object Type Overview 直接添加/移除 Group。

## 4. 搜索与发现

官方支持：

- Ontology Manager Search 搜索 Group；
- Object Type table 按 Group 展示/筛选；
- Object Explorer 首页展示 Groups。

因此 Group 是“信息架构”能力，不是数据模型关系。

## 5. 权限

要查看 Object Type Group，用户需要对 Group 所在 Project 具备 Viewer 权限。

官方 2024 年后已把旧 tag-based group 迁移到新的 group primitive。新模型更明确地把 Group 当作独立 Ontology resource 管理。

## 6. 与其他元素区别

### Group vs Link Type

Group 不表达对象实例之间的业务关系。

### Group vs Interface

Group 不要求成员 Object Types 有共同 Property 或能力。

### Group vs Shared Property

Group 不共享 metadata/schema，只是分类。

## 7. 设计建议

- Group 用于“用户如何发现类型”，不要用来编码业务层级关系。
- Group 名称应面向业务消费者而不是平台团队。
- 支持多 Group 归属。
- Group 变更一般不应影响 API，但会影响导航和发现体验。
- 大规模 Ontology 中可对 Group 配 owner、description、排序和推荐策略。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/type-groups
- https://www.palantir.com/docs/foundry/object-link-types/object-type-metadata
