# Link Type

## 1. 定义

Link Type 是两个 Object Type 之间关系的 schema definition；Link 是两个具体 Object 之间关系的实例。

例如：

```text
Employee --employedBy--> Company
Flight   --assignedAircraft--> Aircraft
```

## 2. 关键概念

### 两端 Object Type
Link Type 会明确关联哪两个 Object Type。

### Cardinality
Palantir 会记录关系的基数信息，用于告诉应用一端对应一个还是多个对象。

常见建模上可理解为：

- one-to-one
- one-to-many
- many-to-one
- many-to-many

### Direction / side API names
Palantir 的 Link Type 在两侧需要可区分的 API 语义。对于同一对 Object Type 可以定义多个不同的 Link Type，只要它们代表不同的现实关系。

例如 Flight ↔ Aircraft 可以同时存在：

- assigned aircraft
- maintenance record relation

不能因为对象类型相同就认为是同一个关系。

## 3. 数据落地

Palantir 的 Link Type 同样会映射到底层真实数据。对于 many-to-many 关系，Link Type 自身通常需要 backing datasource；对于某些 one-to-many / one-to-one 关系，可以通过对象上的外键属性形成关系。

## 4. 元数据

官方文档列出的关键字段包括：

- ID
- RID
- Status
- Related object types
- Cardinality
- 两侧名称 / API 标识等

## 5. 官方页面

- Link types overview: https://www.palantir.com/docs/foundry/object-link-types/link-types-overview
- Link type metadata: https://www.palantir.com/docs/foundry/object-link-types/link-type-metadata

## 6. 自研建议

Link Type 至少应显式记录：两端 Object Type、两端语义名称、cardinality、directional API semantics、status、backing strategy。不要只存一个“sourceTypeId + targetTypeId”。
