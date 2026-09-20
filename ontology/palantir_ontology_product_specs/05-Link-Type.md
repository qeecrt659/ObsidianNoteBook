# Link Type 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Link Type 是两个 Object Type 之间关系的 schema definition；Link 是两个具体 Object 之间该关系的实例。两个端点也可以属于同一 Object Type。

Link Type 不是“数据库外键”的简单别名。Palantir 把它作为 Ontology 的一等资源，让应用、Functions、搜索、Action 可以直接按照业务关系进行遍历和操作。

## 2. 关系方向

同一个 Link Type 的两端可以双向遍历，不需要为反向再建一个 Link Type。每一侧可以有自己的 Display/API name，使调用者从不同对象出发时获得自然的业务命名。

同一对 Object Types 可以存在多个不同 Link Types，只要它们表达不同的真实关系，例如 Flight ↔ Aircraft 既可以表达“assigned aircraft”，也可以另建“maintenance-related aircraft”。

## 3. Cardinality

官方支持的关系基数包括：

- one-to-one
- one-to-many
- many-to-one
- many-to-many

one-to-one 更多表达设计意图，并不意味着底层所有情况下都会自动做数据库级唯一约束，因此建模和数据质量仍需单独治理。

## 4. 三种主要关系实现方式

### Object type foreign keys

适合 one-to-one / many-to-one。一个 Object Type 的某个 Property 存储另一 Object Type 的 Primary Key。

### Join table dataset

用于 many-to-many。Link Type 自己由一个包含双方 Primary Key 的 datasource 支撑。Palantir 可以为新 Link Type 自动生成正确 schema 的 join table。

### Backing object type

Object-backed link 允许把关系本身用 Object Type 来承载，从而为关系附加更丰富的属性和生命周期。适合“关系本身也是业务实体”的场景。

## 5. 核心元数据

官方 Metadata reference 包括：

- ID
- RID
- Status
- 两端 Object Types
- Cardinality
- 每一侧 API Name
- 每一侧 Visibility
- Type Classes
- datasource/key mapping

API Name 是 SDK/Functions 等程序化遍历的核心契约。例如从 Flight 侧访问 `assignedAircraft`。

## 6. 数据映射

对于 foreign-key 关系，需要一侧的外键 Property 与另一侧 Primary Key 匹配。

对于 many-to-many，link datasource 中双方列必须分别映射到两个 Object Type 的 Primary Key，类型不匹配会阻止保存。

因此 Link Type 设计器应实时校验：

- 端点类型；
- cardinality；
- key 类型兼容性；
- datasource schema；
- API Name 冲突；
- 自环关系；
- 已存在的相似 Link。

## 7. 编辑与写回

用户可以通过 Action 创建/删除 Link。对于 one-to-many / one-to-one 等外键型 Link，官方规则说明常常需要通过修改对象上的 foreign key Property 实现；many-to-many 通常通过 link backing datasource/edit storage 处理。

这意味着自研 Action runtime 必须理解 Link 的具体存储策略，而不能把所有 Link 都当作独立边表。

## 8. Link 与业务语义

Link Type 应表达业务关系本身，而不是技术 join。例如：

- Employee → Employer
- Order → Customer
- Flight → Assigned Aircraft

而不是 `employee_company_fk`、`order_customer_join`。

## 9. Breaking Change

高风险修改包括：

- 替换 backing datasource；
- 修改 key mapping；
- 修改 cardinality；
- 修改 active Link 的 API Name；
- 删除被应用/Function/Action 使用的 Link。

Palantir 对 active link 的 API Name 修改有保护。自研平台也应把 API Name 视为发布后的稳定契约。

## 10. 设计建议

- Link Type 用业务动词/关系命名，而非数据库名。
- 明确每一侧的自然语言和 API name。
- many-to-many 关系量大时，要单独考虑索引和存储策略。
- 当“关系本身有属性”时，优先考虑关系对象化，而不是不断向边上附加非标准字段。
- 建模工具应展示 Link graph 和依赖关系，帮助发现重复关系与错误基数。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/link-types-overview
- https://www.palantir.com/docs/foundry/object-link-types/create-link-type
- https://www.palantir.com/docs/foundry/object-link-types/edit-link-types
- https://www.palantir.com/docs/foundry/object-link-types/link-type-metadata
- https://www.palantir.com/docs/foundry/object-link-types/allow-editing
