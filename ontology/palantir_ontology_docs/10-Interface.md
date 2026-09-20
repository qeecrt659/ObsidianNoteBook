# Interface

## 1. 定义

Interface 是一种 Ontology Type，用于描述 Object Type 的共同 shape 和 capabilities，从而实现 Object Type polymorphism。

例如可以定义：

```text
Interface: Facility
Properties:
  facilityName
  location
```

然后让：

- Airport
- Manufacturing Plant
- Maintenance Hangar

都实现 Facility Interface。

## 2. Interface 与 Object Type 的区别

### Object Type
- Concrete
- 有实际 backing datasource
- 可以实例化为 Object

### Interface
- Abstract
- 不能直接实例化
- 不直接由 dataset backing
- 由具体 Object Type 实现

## 3. Interface 的组成

Palantir 当前文档说明 Interface 可以包含：

- Interface properties
- Shared properties
- Link type constraints
- Action type constraints
- Metadata
- Interface inheritance / extension

Object Type 可以实现多个 Interface；Interface 也可以扩展多个其他 Interface。

## 4. Link Type Constraints

Interface 可以定义它期望实现者拥有怎样的 Link 能力，例如 Facility 应当能链接到 Airline。

## 5. Action Type Constraints

当前 Palantir 还支持 beta 阶段的 Interface Action Type Constraints，用于描述所有实现某 Interface 的 Object Type 所应提供的共同 Action capability。

重要的是：constraint 描述能力契约，不直接定义具体执行逻辑；具体 Rules、Submission Criteria、Side Effects 仍配置在实际 Action Type 上。

## 6. 官方页面

- Interface overview: https://www.palantir.com/docs/foundry/interfaces/interface-overview
- Interface metadata: https://www.palantir.com/docs/foundry/interfaces/interface-metadata
- Interface link types: https://www.palantir.com/docs/foundry/interfaces/interface-link-types-overview
- Interface action type constraints: https://www.palantir.com/docs/foundry/interfaces/interface-action-type-constraints

## 7. 自研建议

Interface 对大型 Ontology 非常关键。它可以避免上层应用绑定几十个具体 Object Type。你的平台如果后续需要跨域、多团队、可扩展 SDK，建议把 Interface 纳入正式元模型，而不是等 Runtime 做完再补。
