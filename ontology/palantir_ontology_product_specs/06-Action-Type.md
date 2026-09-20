# Action Type 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Action Type 定义用户/应用/Agent 可以一次执行的一组 Ontology 变更，以及提交时可能触发的副作用。Action 是某次具体提交。

Palantir 把 Action 看成企业 Ontology 的“动词”。Object、Property、Link 描述现实状态，Action 则描述如何在受控条件下改变现实状态。

## 2. Action Type 能做什么

官方能力覆盖：

- 创建对象；
- 修改一个或多个对象；
- 删除对象；
- 创建/删除 Link；
- 调用 Function 执行复杂 Ontology edits；
- 调用外部 Webhook；
- 发送通知；
- 触发构建；
- 处理 Scenario 相关操作等。

一个 Action Type 可以组合多个 Rules；Function rule 属于特殊路径，不能与普通 Ontology rules 任意混用。

## 3. 核心组成

一个完整 Action Type 至少由以下部分构成：

### Metadata

API Name、Display Name、Description、Status 等，形成稳定行为契约。

### Parameters

Action 的输入接口。可以是 primitive、object reference、list 等，并支持默认值、过滤、隐藏、必填和条件覆盖。

### Rules

把 Parameters 转换为 Ontology edits 或其他 effects。

### Submission Criteria

决定某次 Action 是否允许提交，承载用户、参数、执行上下文等业务约束。

### Side Effects

通知、Webhook 等外部副作用。

### Permissions / Authorizations

决定谁能读取 Action 所需数据、执行 Action、创建/修改受安全控制的数据。

### Observability

Action metrics、run history、action log、monitoring 等。

## 4. 创建流程

官方 Getting Started 中，典型流程是：

1. 在 Ontology Manager 新建 Action Type；
2. 选择操作的 Object Type 和操作类别；
3. 映射需要修改的 Properties；
4. 填写 Metadata；
5. 自动生成或进一步编辑 Parameters；
6. 配置 Submission Criteria；
7. Test Run；
8. 保存并在 Object Explorer、Object Views、Workshop 等应用消费。

## 5. 事务语义

官方把 Action 描述为一次 transaction，用于一次性完成定义好的对象/属性/链接变化。但与外部系统集成时要区分：

- Ontology edits 本身；
- writeback webhook；
- side-effect webhook。

Writeback webhook 在 Ontology edits 之前执行，失败时可以阻止后续变更；Side-effect webhook 在对象修改之后执行，更接近 best-effort。跨系统并非严格分布式事务，产品说明应清楚呈现失败边界。

## 6. Scale Limits

官方对 Action 有明确规模保护，例如：

- primitive list parameter 最多 10,000 个元素；
- object reference list parameter 通常最多 1,000；
- 单次 Action 最多编辑 50 个 Object Types；
- 单次 Action 最多编辑 10,000 个 Objects；
- batch call 也存在上限；
- function-backed actions 有更严格的调用限制。

自研产品应把这些作为可配置 runtime guardrail，而不是仅靠后端超时。

## 7. 测试

Test Run 会在当前 Ontology branch 上以当前用户权限评估 Action，执行同样的 object security 与 submission criteria，并展示 Proposed Changes。它非常适合作为 Action 发布前的产品能力：让建模者看到将创建/修改/删除哪些对象和 Link。

## 8. 可观测性

Action Metrics 提供成功/失败、P95 duration、运行历史等。失败可分类为：无效参数、规模限制、认证/权限、side effect、function、冲突等。

Action Log 可以记录 Action RID、Action Type RID/version、时间、用户、编辑对象、webhook/notification、scenario、revert 状态以及可选参数值等，从而形成决策审计链。

## 9. 应用消费

Action 可以在 Object Views、Object Explorer、Workshop 等处被复用。同一个 Action 的逻辑和校验在多个应用中保持一致，因此应用层不应重复实现业务写入规则。

## 10. 安全模型

Action 的安全不能只靠“谁能看到按钮”。需要同时评估：

- Action resource permission；
- 参数对象/字段的读取权限；
- Submission Criteria；
- Read/Write Authorizations；
- 被编辑 Object/Link 的权限；
- 外部 webhook 权限与凭证。

## 11. 自研建议

- Action Type 必须是元模型一等公民，不要等同于 REST endpoint。
- Parameters / Rules / Criteria 分离建模，便于复用和解释。
- 每次执行都生成唯一 execution/action ID 和审计记录。
- 支持 dry-run/proposed changes。
- 对外部写回明确 pre-commit / post-commit 语义。
- 为 Agent 暴露 Action 时复用同一权限和 submission criteria，不另建绕过通道。

## 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/overview
- https://www.palantir.com/docs/foundry/action-types/getting-started
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/test-run
- https://www.palantir.com/docs/foundry/action-types/permissions
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits
- https://www.palantir.com/docs/foundry/action-types/action-metrics
- https://www.palantir.com/docs/foundry/action-types/action-log
