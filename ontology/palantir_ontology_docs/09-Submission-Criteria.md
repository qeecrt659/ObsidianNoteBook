# Submission Criteria

## 1. 定义

Submission Criteria 是决定某个 Action 是否允许提交的条件。Palantir 以前称其为 validations。

其目标是把业务规则编码到数据编辑治理中，保证数据质量和操作合规性。

## 2. 结构

Submission Criteria 由：

- Conditions
- Operators

组合而成。条件可以基于：

- Current user
- Parameter
- Execution context
- Object / relation information
- Static values

所有 Submission Criteria 都满足后，Action 才能提交。

## 3. 与权限的区别

Submission Criteria 不等同于“谁能编辑 Action Type 元数据”的权限。

可以把它理解为：

- Permission：是否拥有访问/执行某项能力的基础授权。
- Submission Criteria：在当前业务上下文和当前输入值下，这次提交是否符合业务条件。

例如用户可能有权限执行“变更飞机”Action，但只有飞机仍处于 operational 状态时才允许提交。

## 4. 官方页面

- Submission criteria: https://www.palantir.com/docs/foundry/action-types/submission-criteria
- Action permissions: https://www.palantir.com/docs/foundry/action-types/permissions

## 5. 自研建议

Submission Criteria 应当成为 Action Type 元模型的一等子结构，而不是散落在前端校验代码中。否则无法保证同一 Action 在不同应用、SDK、自动化入口中使用同一套业务约束。
