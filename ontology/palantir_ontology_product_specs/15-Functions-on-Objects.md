# Functions / Functions on Objects 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 在 Ontology 中的位置

Functions 不在 `Ontologies > Overview` 列出的六类一级 Ontology resources 中，但 Palantir 的 Ontology building 文档把 **Action types and functions** 一起描述为组织的 kinetic elements。Functions 是把任意复杂业务逻辑与 Ontology 对象结合起来的主要编程扩展机制。

## 2. Functions on Objects（FOO）

Functions on Objects 允许代码：

- 读取 Object Properties；
- 查询 Object Sets；
- 遍历 Links / Search Around；
- 聚合对象；
- 计算指标；
- 构造 Ontology Edits；
- 作为 Function-backed Action 的执行逻辑。

因此 FOO 可以理解为 Ontology 的强类型业务逻辑层。

## 3. Object Set

Object Set 是某一 Object Type 的无序对象集合，支持：

- filter；
- search around links；
- aggregate；
- retrieve objects；
- pagination/iteration。

相比把大量 objects 直接加载成数组，Object Set 更适合延迟加载和大规模处理。

## 4. Function-backed Actions

普通 Action Rules 足以处理简单 CRUD，但复杂业务可使用 Ontology Edit Function，例如：

- 修改一个 Incident，同时修改所有 linked Alerts；
- 根据多个对象计算结果再写回；
- 一次创建多个不同类型对象并建立 Links。

Function-backed Actions 同时受到 Action limits 与 Function execution limits。

## 5. Ontology Edits

Function 可以构造创建、修改对象等 edits，再由 Action 或 Functions runtime 受控执行。这让复杂算法仍能落在 Ontology 的权限和审计框架中。

## 6. 语言与版本

Palantir 目前存在 TypeScript v2、Python、TypeScript v1 等不同 Functions 能力线，各语言对 Interface、Media、Ontology edit 等支持可能不同。产品需要维护清晰的 capability matrix。

## 7. 性能原则

官方 Functions 文档强调：

- 批量加载对象；
- 使用 Object Set/分页，而不是逐对象网络调用；
- 大规模 edit 分块；
- 避免无界对象加载。

## 8. 与 Action 的边界

建议：

- 可声明式表达的简单 edit 用 Action Rules；
- 需要复杂计算、循环、跨对象关联逻辑时用 Function；
- 最终对终端用户暴露的业务操作仍优先通过 Action Type 包装，从而复用 Parameters、Submission Criteria、Permissions、日志和 UI。

## 9. 自研建议

如果实现 Palantir-style runtime，可将 Function 设计为：

- versioned function artifact；
- typed inputs/outputs；
- generated Ontology SDK；
- read permissions propagation；
- declared edit scope；
- resource/time/scale limits；
- logs/metrics/tracing；
- Action Type binding。

不要让 Function 成为绕过 Ontology 权限的后门。

## 官方原始资料

- https://www.palantir.com/docs/foundry/ontology/overview/
- https://www.palantir.com/docs/foundry/workshop/functions-overview
- https://www.palantir.com/docs/foundry/functions/api-object-sets
- https://www.palantir.com/docs/foundry/action-types/function-actions-overview
- https://www.palantir.com/docs/foundry/functions/optimize-performance
