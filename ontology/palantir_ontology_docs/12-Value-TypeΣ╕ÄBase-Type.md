# Value Type 与 Base Type

## 1. 为什么单独整理

Palantir 官方明确指出 Value Type 与 Object Type、Property、Link Type 等“构建 Ontology 的类型”不同：Value Type 是围绕 field type 的语义包装和可复用约束，与 Space 关联。

因此它是 Ontology 建模的重要配套类型系统，但不应误写成与 Object Type 同级的核心 Ontology resource。

## 2. Base Type

Base Type 定义 Property 能存储什么种类的数据，并影响应用中可用的操作。

除常见原始类型外，还包括高级类型，如：

- Vector
- Geopoint
- Geoshape
- Attachment
- Time series
- Geotemporal series

## 3. Value Type

Value Type 是对 primitive / field type 的语义封装，可以增加：

- Domain-specific meaning
- Reusable constraints
- Validation
- Type safety

例如可定义 `EmailAddress` Value Type，并通过正则约束格式。这个 Value Type 可复用于多个 Object Type Property、Shared Property、Pipeline Property 或 Action Parameter。

## 4. 版本化

Palantir 对 Value Type 提供版本机制；当约束发生变化时会产生新的版本，并区分 breaking / non-breaking changes 的风险。

## 5. 官方页面

- Base types: https://www.palantir.com/docs/foundry/object-link-types/base-types
- Value types overview: https://www.palantir.com/docs/foundry/object-link-types/value-types-overview
- Value type versions: https://www.palantir.com/docs/foundry/object-link-types/value-types-versions
- Use value types: https://www.palantir.com/docs/foundry/object-link-types/use-value-type

## 6. 自研建议

强烈建议区分：

- primitive/base type：string、integer、timestamp 等。
- semantic value type：EmailAddress、CurrencyCode、EmployeeId 等。

这样才能把数据类型和业务语义同时固化到 Ontology 体系中。
