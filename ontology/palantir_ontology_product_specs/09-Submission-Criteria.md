# Submission Criteria 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Submission Criteria 是决定某次 Action 是否允许提交的条件。它原名 validations，但现在的定位更明确：把业务规则嵌入数据编辑权限与流程治理，保证 Ontology 数据质量和操作合规。

它和“谁能编辑 Action Type 配置”不是一回事；每个 Action Type 可以拥有独立的执行条件。

## 2. 条件模型

Submission Criteria 由 **Conditions + Logical Operators** 组成。

Condition 是两个值之间的一次比较；Logical Operator 用于把多个条件组合、嵌套成复杂逻辑。

## 3. 三类 Context Template

### Current User

可以依据当前用户：

- User ID；
- Group IDs；
- Organization；
- 其他 Multipass attributes；

来决定是否允许执行。

官方特别提醒不要对 group/marking/organization membership 随意使用 NOT 条件，因为 scoped token 可能没有携带完整属性，错误的否定逻辑可能产生越权风险。

### Parameter

根据 Action Parameters 以及 Object Parameter 的 Properties 判断业务条件。例如只有 Ticket.status = Open 才允许修改 Priority。

### Execution Context

可以判断 Action 是否在 Ontology Scenario 等执行上下文中运行，从而为规划/仿真和生产操作设置不同规则。

## 4. Operators

不同 Parameter 类型支持不同比较运算，如：

- is / is not
- matches
- less than / greater than
- list/集合相关运算
- all / any / none 等逻辑组合

产品设计器应根据左右类型动态限制合法 operator，避免运行时才发现类型错误。

## 5. Failure Message

根级条件/逻辑块可以配置面向终端用户的失败消息。当 criteria 不满足时，Object Explorer、Workshop、Quiver 等可以展示具体原因。

这使 Submission Criteria 不只是访问控制，还承担“可解释业务约束”的职责。

## 6. 与 Permissions 的区别

建议严格区分：

- **Resource Permission**：谁可以查看/编辑 Action Type 定义；
- **Object/Property Security**：用户能读取哪些输入数据；
- **Submission Criteria**：在当前业务上下文下能不能提交；
- **Write Authorization**：允许 Action 写入何种安全级别的数据。

Submission Criteria 不能替代底层数据权限。

## 7. 支持与限制

官方说明 attachment 与 object set 参数不能直接用于 submission criteria。复杂判断要通过可支持的 Parameter / Function /预计算方式表达。

## 8. Test Run

Ontology Manager 支持用给定 Parameter 值验证 Submission Criteria；Action Test Run 也会按真实权限执行同一套 criteria。这是产品中非常重要的测试与治理能力。

## 9. 自研建议

- Criteria 使用类型化表达式树。
- 条件中允许引用 current user、parameter、object property、execution context。
- 所有拒绝必须返回可解释失败原因。
- 对 NOT + 权限属性做静态风险检测。
- 支持按用户/参数做策略模拟。
- Agent 执行 Action 时必须走同一 criteria evaluator，不能绕过。

## 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/submission-criteria
- https://www.palantir.com/docs/foundry/action-types/test-run
- https://www.palantir.com/docs/foundry/action-types/permissions
