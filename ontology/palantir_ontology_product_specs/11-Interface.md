# Interface 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Interface 是描述 Object Type “形状与能力”的 Ontology Type，用于在多个 Object Types 之间提供多态能力。

例如 `Facility` Interface 可以定义 `Facility Name`、`Location` 等公共属性，由 Airport、Manufacturing Plant、Maintenance Hangar 等不同 Object Types 实现。

## 2. 产品价值

没有 Interface 时，应用和 Function 常常直接依赖具体 Object Type，导致新增相似类型时需要重构。Interface 引入稳定抽象层，使工作流可以面向能力编程。

主要价值：

- Object Type polymorphism；
- 跨类型统一查询/消费；
- Marketplace/package 的公共契约；
- 新 Object Type 实现 Interface 后自动兼容既有工作流；
- 降低领域之间的类型耦合。

## 3. Interface 的组成

官方说明 Interface 包含：

- Metadata；
- Interface Properties；
- Link Type Constraints；
- Action Type Constraints；
- Extension / inheritance。

## 4. Interface Properties

Properties 可以：

- 直接定义在 Interface 上（官方推荐）；
- 使用 Shared Properties。

每个 Interface Property 可以标记 required 或 optional。

实现 Interface 的 Object Type 必须把本地 Property 映射到 required Interface Property；optional 可以不映射。

## 5. Link Type Constraints

Interface 可以声明期望的 Link 能力。例如所有 Facility 都必须能链接到某个 Region。实现 Interface 的 Object Type 需要选择/创建满足约束的具体 Link Type。

这不是“Interface 自己存一条 Link”，而是对实现类型提出结构契约。

## 6. Action Type Constraints

Interface 还可以定义行为能力契约，例如所有 Ticket 类型都应支持 `Close` Action。Object Type 实现 Interface 时，需要把具体 Action Type 映射到该 constraint。

这使 Interface 从“数据结构接口”升级为“数据 + 行为能力接口”。

## 7. Interface Implementation

一个 Object Type 实现 Interface 时，需要：

1. 选择 Interface；
2. 映射 local Properties；
3. 映射 required link constraints；
4. 映射 required action constraints。

实现后，Object Set Service 可以按 Interface 搜索并返回具体实现类型的对象；SDK 消费者也可以使用 Interface API Name 操作它们。

## 8. Interface Extension

Interface 可以 extend 一个或多个其他 Interfaces，并继承：

- shared/interface properties；
- link type constraints；
- action type constraints。

因此可以构建能力组合，例如：

```text
LocatedEntity
   ↑
Facility
   ↑
Airport
```

也支持多继承式组合能力，但必须控制契约复杂度。

## 9. 当前支持边界

官方明确指出 Interface 仍处于持续扩展中，不同应用的支持程度不同。TypeScript OSDK 和部分 Object Set/Action 能力支持较好，而 Workshop、部分 Functions 版本等可能存在限制。

自研平台应给 Interface 能力设置 feature compatibility matrix，避免用户以为所有运行时都天然支持。

## 10. 与 Shared Property 的区别

- Shared Property：共享一个属性的语义和 metadata；
- Interface：共享一组属性 + link/action capabilities 的类型契约；
- Object Type Group：只是分类/发现，不要求结构一致。

## 11. 设计建议

- 用 Interface 表达稳定跨领域能力，不要把它当普通标签。
- Interface Properties 尽量小而稳定。
- Required constraint 会影响所有实现者，新增 required 成员属于高风险 breaking change。
- 能力型 Interface 可以比“大而全业务基类”更可组合。
- 平台必须展示 Interface → implementing Object Types 的 usage graph。

## 官方原始资料

- https://www.palantir.com/docs/foundry/interfaces/interface-overview
- https://www.palantir.com/docs/foundry/interfaces/create-interface
- https://www.palantir.com/docs/foundry/interfaces/implement-interface
- https://www.palantir.com/docs/foundry/interfaces/extend-interface
- https://www.palantir.com/docs/foundry/action-types/actions-on-interfaces
