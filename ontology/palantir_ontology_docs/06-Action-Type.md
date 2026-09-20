# Action Type

## 1. 定义

Action Type 定义用户一次可以对 Objects、Property Values 和 Links 做的一组业务变更，以及 Action 提交时可能发生的 side effects。

Action 是 Action Type 的一次实际执行。

## 2. Action Type 的核心组成

按照 Palantir 当前文档，可以把 Action Type 理解为以下结构：

1. **Parameters**：输入。
2. **Rules**：把输入转换为 Ontology edits 或其他效果的业务逻辑。
3. **Submission Criteria**：动作能否提交的条件。
4. **Side Effects**：通知、Webhook 等外围影响。
5. **Form / UI configuration**：面向最终用户的表单组织与参数交互。
6. **Permissions / authorizations**：谁能看、谁能执行以及读写授权。

## 3. 典型例子

`Assign Employee` Action Type：

- 参数：Employee、newRole、newManager
- Rule：更新 Employee.role
- Rule：更新 Employee 与 Manager 的 Link
- Submission Criteria：只有 HR 或特定角色可执行；目标 Manager 必须满足条件
- Side effect：通知旧经理和新经理

## 4. Action 的语义价值

Palantir 不希望最终用户直接操作底层表字段，而是通过业务动作表达意图。

例如：

- “批准订单”
- “分配员工”
- “关闭工单”
- “调整优先级”

Action Type 把数据写入、业务规则、验证和外围系统协同统一成一个可治理能力。

## 5. 官方页面

- Overview: https://www.palantir.com/docs/foundry/action-types/overview
- Parameters: https://www.palantir.com/docs/foundry/action-types/parameter-overview
- Rules: https://www.palantir.com/docs/foundry/action-types/rules
- Submission criteria: https://www.palantir.com/docs/foundry/action-types/submission-criteria
- Side effects: https://www.palantir.com/docs/foundry/action-types/side-effects-overview
- Permissions: https://www.palantir.com/docs/foundry/action-types/permissions

## 6. 自研建议

Action Type 不要只建成“CRUD API 定义”。它至少应该同时有 input schema、business rules、submission criteria 和 governed effects，否则很难实现 Palantir 风格的业务语义层。
