# Action Parameter

## 1. 定义

Parameters 是 Action Type 的输入，是 Rules 与 Workshop、Slate、Object Views 等应用之间的接口。

每个 Parameter 都有类型，并且可以配置是否展示、是否可由用户修改、默认值或约束等行为。

## 2. Parameter 的用途

Parameter 的值可以被：

- Rules 使用，用于创建/修改/删除 Object 或 Link。
- Submission Criteria 使用，用于判断 Action 是否允许提交。
- Side Effects 使用，例如生成通知内容或 webhook payload。
- Parameter overrides 使用，动态改变后续参数配置。

## 3. 参数类型

Action parameters 可以接收基础值、Object reference 等多种输入；Value Type 还可用于对参数值复用验证约束。

在 Interface Action Type Constraints 中，参数约束还可以描述 object reference、interface reference、object set、attachment、media reference、struct 等参数形状。

## 4. 官方页面

- Parameters overview: https://www.palantir.com/docs/foundry/action-types/parameter-overview
- Value types: https://www.palantir.com/docs/foundry/object-link-types/value-types-overview
- Interface action constraints: https://www.palantir.com/docs/foundry/interfaces/interface-action-type-constraints

## 5. 自研建议

Parameter 应被视为强类型业务输入，而不是简单的前端表单字段。其 schema 需要被 Runtime、UI、Validation、SDK 和 Rules 共同消费。
