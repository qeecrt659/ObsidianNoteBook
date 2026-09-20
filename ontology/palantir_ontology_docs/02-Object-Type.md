# Object Type

## 1. 定义

Object Type 是现实世界实体或事件的 schema definition。单个 Object 是 Object Type 的实例，Object Set 是多个 Object 的集合。

典型例子：

- `Employee`：员工实体类型
- `Company`：公司实体类型
- `Flight`：航班事件类型
- `Order`：订单业务对象类型

## 2. 关键组成

根据 Palantir 官方文档，一个 Object Type 的核心内容通常包括：

- 唯一标识：ID / API name / RID
- Display name / Plural display name
- Description
- Icon / visual metadata
- Properties
- Primary key
- Title key
- Backing datasource
- Status（如 active / experimental / deprecated）
- Groups / interfaces 等附加建模信息

其中 Properties、Primary Key、Title Key 和 Backing Datasource 是实现层最关键的部分。

## 3. Object Type 与 Object 的区别

- Object Type：类型定义、元数据。
- Object：某个现实对象或事件的实例。

例如：

```text
Object Type: Employee
Object: employee_id = E10086
Properties:
  name = "Alice"
  department = "R&D"
```

## 4. Backing datasource

Palantir 的 Object Type 不是纯抽象模型。实际 property values 通常来自 backing datasource。也就是说，Ontology 建模和数据管道之间存在映射关系。

这也是 Palantir 与传统“只画概念模型”的 Ontology 工具的重要区别之一。

## 5. 官方页面

- Object types overview: https://www.palantir.com/docs/foundry/object-link-types/object-types-overview
- Object type metadata: https://www.palantir.com/docs/foundry/object-link-types/object-type-metadata
- Create object type: https://www.palantir.com/docs/foundry/object-link-types/create-object-type/index.html
- Types reference: https://www.palantir.com/docs/foundry/object-link-types/type-reference

## 6. 对自研平台的元模型建议

建议至少保留：

```yaml
ObjectType:
  id:
  apiName:
  rid:
  displayName:
  pluralDisplayName:
  description:
  status:
  icon:
  primaryKeyProperty:
  titleKeyProperty:
  properties: []
  implementedInterfaces: []
  groups: []
  backingDatasources: []
```

注意：上面是便于实现的抽象表达，不代表 Palantir 官方 API 的原样字段结构。
