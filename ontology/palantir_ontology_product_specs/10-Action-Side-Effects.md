# Action Side Effects 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Side Effects 让 Action 在修改 Ontology 的同时与组织中的其他流程或外部系统交互。Palantir 官方主要提供 Notifications 与 Webhooks。

其目标是让 Ontology 不仅记录决策，还能把决策结果发送出去，形成 decision orchestration。

## 2. Notifications

Notifications 用于在 Action 执行后通知用户，例如邮件或平台通知。典型场景：

- 任务负责人发生变化；
- 风险状态升级；
- 审批完成；
- 新对象创建后通知相关团队。

通知属于业务动作的一部分，应在 Action Type 中集中配置，而不是散落在各个前端应用。

## 3. Webhooks

Webhook 可以调用外部 HTTP/API 系统，例如 ERP、CRM、SAP、Salesforce 或内部服务。

Palantir 对 webhook 区分两种执行模式：

### Writeback Webhook

- 在 Ontology object changes 之前执行；
- webhook 失败时不继续应用后续修改；
- 失败会反馈给终端用户；
- 一个 Action 只能配置一个 writeback webhook；
- response 可以供后续 rules 使用。

适用于外部系统是 source of truth，且希望先确认外部写入成功再修改 Ontology 的场景。

### Side-effect Webhook

- 在 Ontology edits 之后执行；
- 可以配置多个；
- 执行顺序不保证；
- 用户可能在 side effect 完成前就看到 Action 成功；
- 更适合 best-effort 通知或写多个外围系统。

## 4. 跨系统事务边界

Writeback webhook 提供一定程度的事务保护，但不是完整分布式事务：外部请求可能成功，而随后 Ontology edits 失败。

因此产品应明确支持：

- idempotency key；
- retry policy；
- compensation/reconciliation；
- execution log；
- external correlation ID；
- 手工重放或失败处理。

## 5. 输入与输出

Webhook 输入可以来自：

- Action Parameter；
- 静态值；
- Object Parameter Property；
- Function 计算结果。

Writeback response 可以被后续 Logic Rule 使用，从而把外部系统生成的 ID/状态写回 Ontology。

## 6. Authentication

对于使用 outbound application 的 REST source，Foundry 可以代用户管理 OAuth 2.0 token 获取与刷新。自研平台应把外部连接凭证作为受管连接资源，避免让 Action 配置直接保存 secret。

## 7. 失败可观测性

Action Metrics 会把 side effect failure 单独分类；Action Log 也可以记录调用过的 webhook 和 notification 信息。

## 8. 自研建议

- 明确区分 pre-commit writeback 与 post-commit side effect。
- Side effects 必须可观测、可重试、可追踪。
- 外部连接配置与 Action Type 分离，Action 只引用 versioned connection/webhook definition。
- 对外部写回设置权限和网络 allowlist。
- 对 Agent 调用使用同一 Action/Side-effect 机制，不直接开放任意 HTTP。

## 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/side-effects-overview
- https://www.palantir.com/docs/foundry/action-types/webhooks
- https://www.palantir.com/docs/foundry/action-types/set-up-webhook
- https://www.palantir.com/docs/foundry/action-types/action-metrics
- https://www.palantir.com/docs/foundry/action-types/action-log
