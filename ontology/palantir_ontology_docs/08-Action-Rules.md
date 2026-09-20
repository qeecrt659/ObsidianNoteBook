# Action Rules

## 1. 定义

Rules 定义 Action Type 的逻辑：如何把 Parameters 转换为 Ontology edits 或其他效果。

Palantir 把 Rules 大致分为：

- 改变 Ontology 的规则。
- 触发 Foundry 内外其他效果的规则。

## 2. Ontology Rules

官方文档列出的典型操作包括：

- Create object
- Modify object(s)
- Delete object
- Create link
- Delete link

此外还可以结合 function-backed actions 表达更复杂的编辑逻辑。

## 3. 其他 Rules / Effects

Action 还可以配置：

- Notification
- Webhook
- Schedule / build trigger

Webhook 可以作为 side effect，也可以作为 writeback；后者会在 Ontology edits 前执行，失败时阻止后续变更。

## 4. Rule 顺序

Palantir 文档强调 Rule 顺序会影响最终 Object edit，因此 Runtime 不能把 Rules 当作无序集合。

## 5. 官方页面

- Rules: https://www.palantir.com/docs/foundry/action-types/rules
- Explore action types: https://www.palantir.com/docs/foundry/action-types/explore-action-types
- Side effects: https://www.palantir.com/docs/foundry/action-types/side-effects-overview
- Webhooks: https://www.palantir.com/docs/foundry/action-types/webhooks

## 6. 自研建议

Action Rule 最好设计成有类型、有顺序、可验证、可版本化的执行图；不要把所有规则退化成一段不透明脚本，否则很难做静态检查、权限分析、影响分析和低代码 UI。
