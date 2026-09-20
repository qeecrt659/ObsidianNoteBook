# Functions 与 Ontology

## 1. 定位

Functions 是与 Ontology 原生集成的代码逻辑能力，可以接收 Object / Object Set，读取 Properties、遍历 Links，并执行复杂业务逻辑。

Palantir 的 Ontology overview 把 Functions 与 Action Types 一起描述为组织“kinetics”的重要组成，但 Ontologies overview 的一级 Ontology resources 列表并没有把 Function 列为 Object Type / Link Type 那样的 Ontology resource。

因此更准确的分类是：**Functions 是 Ontology 原生集成的业务逻辑能力，而不是核心一级 Ontology 资源。**

## 2. 常见用途

官方文档给出的典型用途包括：

- 返回 Object Set 或变量值给应用使用。
- 计算聚合、指标或派生值。
- 编写复杂的 Function-backed Action。
- 查询外部系统丰富 Ontology Object。
- 为应用提供后端业务逻辑。

## 3. 与 Action Type 的关系

- Action Type：声明业务动作、输入、规则、验证和 effects。
- Function：用代码实现复杂计算或复杂 Ontology edits。

复杂 Action 可以由 Function 支撑，但不意味着所有 Action 都必须写 Function。

## 4. 官方页面

- Functions overview: https://www.palantir.com/docs/foundry/functions/overview
- Use functions: https://www.palantir.com/docs/foundry/functions/use-functions
- Ontology core concepts: https://www.palantir.com/docs/foundry/ontology/core-concepts

## 5. 自研建议

在自研平台中可以把 Function 作为“可被 Ontology 元模型引用的计算资源”，不要和 Object Type 等混成一个表；应通过稳定引用、版本和输入输出 schema 与 Action、Property 或应用层连接。
