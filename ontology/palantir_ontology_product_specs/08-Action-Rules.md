# Action Rules 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Rules 定义 Action Type 的执行逻辑，即如何把 Parameters 转化为 Ontology edits 或其他 Foundry effects。它是 Action Type 的“输出/执行层”。

## 2. 两大类规则

### Ontology Rules

直接改变 Ontology 数据：

- Create Object
- Modify Object(s)
- Delete Object(s)
- Create Link
- Delete Link

对于 one-to-many / one-to-one 等外键关系，创建/删除 Link 往往通过修改对象 foreign key Property 完成。

### Other Effects / Integrations

可以调用 Functions、Webhook、通知、build 等能力，实现超出简单对象编辑的流程。

## 3. 值映射

修改 Object 时，每个目标 Property 需要定义值来自哪里。官方支持的典型来源包括：

- **From parameter**：从同类型 Parameter 获取；
- **Object parameter property**：从 Object Reference Parameter 的某个 Property 获取；
- **Static value**：规则内部常量；
- **Current User / Time**：运行时上下文值；
- Webhook response / Function output 等更高级来源。

产品实现应把 value source 设计成可扩展 expression，而不是固定几种硬编码字段。

## 4. Create Object

创建对象时 Primary Key 是必填的，因为新 Object 必须有唯一标识。其他 Property 可由 Parameter、静态值、上下文或 Function 计算得到。

## 5. Modify Object(s)

目标对象一般由 Object Reference Parameter / list Parameter 提供。规则指定可以修改哪些 Property，以及每个新值如何计算。

## 6. Link Rules

Link 的创建/删除受到其 cardinality 和 storage model 影响。many-to-many 可直接编辑 Link storage；foreign-key backed links 则可能转化为 Property edit。

因此 Rule engine 必须与 Ontology metadata 联动，不能脱离 Link 定义独立执行。

## 7. Function-backed Rules

当普通 declarative rule 不足以表达：

- 跨多个关联对象修改；
- 基于复杂计算修改属性；
- 一次创建多个不同类型对象和 Link；
- 复杂循环/聚合逻辑；

可以使用 Ontology Edit Function。Function-backed Action 灵活度更高，但受到 Functions runtime 与 Action limits 的双重约束。

## 8. Rule 组合

一个 Action Type 可以组合多个普通 Rules，从而在一次业务动作中完成多个受控变化。Function rule 是特殊情况，不能简单与所有 Ontology rules 并列组合。

产品层应明确 Rule 执行阶段、依赖、失败处理和原子性边界。

## 9. 安全与权限

Rule 能做什么不等于执行用户一定有权限做什么。运行时仍必须评估 Action permissions、Submission Criteria、Object/Property Security、Read/Write Authorization 等。

## 10. 测试与解释

规则设计器应该支持：

- 给定一组 Parameter 值进行 test run；
- 展示 proposed object/link changes；
- 标注每个目标值的来源；
- 展示未满足的权限或 criteria；
- 解释 Function/Webhook 等外部依赖。

## 11. 自研建议

- Rule 使用声明式 AST / IR 表达，不要只存 UI JSON。
- value source 统一抽象。
- Rule 与 Ontology schema 做静态类型检查。
- 任何 schema 变化都应重新验证受影响 Rules。
- 支持 execution plan / proposed changes，方便 Agent 与用户审计。

## 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/rules
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/function-actions-overview
- https://www.palantir.com/docs/foundry/action-types/test-run
