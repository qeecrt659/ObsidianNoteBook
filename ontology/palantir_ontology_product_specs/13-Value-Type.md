# Value Type 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Value Type 是对底层 field/base type 的 **语义包装**，可携带 metadata 和 validation constraints，用于增强 type safety、表达能力和上下文。

它和 Object Type 不同：Value Type 并不是一级 Ontology resource，而是与 Space 绑定的可复用类型资产。一个 Space 对应一个 Ontology，Value Type 只能在其 Space 内使用；官方说明 Default ontology 不支持 Value Types。

## 2. 为什么需要 Value Type

Base Type 只表达 `String`、`Integer`、`Date` 等技术类型，无法表达业务语义。

Value Type 可以表达：

- EmailAddress：String + regex；
- CountryCode：String + enum；
- Percentage：Decimal + range；
- EmployeeId：String + domain meaning；
- Latitude：Double + range。

这样业务含义和验证规则可以跨 Pipeline、Property、Shared Property、Action Parameter 重复使用。

## 3. 核心元数据

创建 Value Type 时配置：

- Name
- Description
- API Name
- Base Type
- Optional Constraint
- Recommended example/preview value

## 4. Constraints

官方支持的主要约束包括：

- **Enum**：静态允许值集合；
- **Range**：最小值、最大值、长度/数组大小范围；
- **Regex**：String 正则；
- **RID**：必须是合法 RID；
- **UUID**：必须是 UUID；
- **Array uniqueness**；
- **Nested constraint**：数组元素使用另一个 Value Type；
- **Struct element constraints**：给 Struct 字段绑定 Value Type。

设计器应按 Base Type 过滤可用 constraint。

## 5. 使用位置

Value Type 可用于：

- Object Type Property；
- Shared Property；
- Pipeline Builder logical type；
- Action Parameter。

这样同一业务值在“数据接入 → Ontology → Action 输入”全链路使用一致规则。

## 6. 运行时验证

如果给现有 Property 应用 Value Type，而已有数据违反约束，对象类型可能索引失败。说明 Value Type 不只是文档标签，而是会真正影响数据有效性和运行状态。

## 7. 版本

Value Type 有版本管理。

- name/description/apiName 等 metadata 可以调整；
- base type 和当前版本的 constraint 定义具有更强不可变性；
- 修改 constraint 会产生新版本；
- non-breaking 新版本可以自动传播到 Ontology 消费者；
- breaking change 且已有消费者时，官方建议 deprecate 旧 Value Type，并创建新类型。

## 8. 权限

权限通过 Space 管理：

- View 权限可以把 Value Type 分配给 Property/Shared Property；
- Editor/Owner 可以创建、编辑、删除。

## 9. 自研建议

- Value Type 用独立 registry 管理，并支持 version。
- Property 只引用 Value Type + version range/strategy，不复制约束。
- 对 breaking constraint 变更做消费者扫描。
- 校验逻辑应在 ingestion、indexing、Action parameter validation 共用。
- Base Type 与 Value Type 明确分层，避免业务类型直接退化为 String + description。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/value-types-overview
- https://www.palantir.com/docs/foundry/object-link-types/create-value-type
- https://www.palantir.com/docs/foundry/object-link-types/use-value-type
- https://www.palantir.com/docs/foundry/object-link-types/value-types-versions
- https://www.palantir.com/docs/foundry/object-link-types/value-types-permissions
- https://www.palantir.com/docs/foundry/object-link-types/value-type-constraints
