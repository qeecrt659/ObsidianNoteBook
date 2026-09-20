# Action Parameter 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Parameter 是 Action Type 的输入，也是 Action Rules 与 Workshop、Object Views、Slate 等消费应用之间的接口。它类似带类型的变量，由外部用户或应用在执行时赋值。

## 2. 主要作用

Parameter 的值可以被用于：

- 规则中设置 Property 值；
- 创建/删除 Link；
- Side Effect / Webhook 输入；
- Submission Criteria 判断；
- 获取对象修改前的当前值；
- 驱动后续参数的 default、options、visibility、requiredness 等条件配置。

## 3. 类型系统

Parameter 可以使用 primitive 类型，也可以引用 Object / Object list 等 Ontology 类型。Value Type 可以施加在 Parameter 上，从而复用同一套校验规则。

产品层应记录：

- parameter id/name；
- display name / description；
- data type / object type reference；
- single or multiple values；
- required；
- visible/hidden；
- user editable；
- default；
- constraints/options；
- overrides；
- order/section。

## 4. 默认值

Palantir 支持全局 Parameter Default，用于在所有消费应用中统一预填。默认值可来自：

- 静态值；
- 前面某个 Object Parameter 的 Property；
- 特殊上下文值，例如 UUID、current user 等 type-class prefill。

应用本地传入的值通常优先于 Action Type 全局默认值。

## 5. Dropdown / Allowed Values

非对象参数可以配置 multiple-choice；Object Reference 参数可以基于 Object Set 过滤可选对象。过滤支持：

- Property 条件；
- Parameter 值；
- Object parameter property；
- Search Around；
- 自定义 starting Object Set。

这使 Action form 能表达“上下文敏感的可选值”，而不是静态表单。

## 6. Overrides

Parameter Overrides 是 Action form 动态逻辑的重要能力。每个 override block 包含：

- IF：根据之前的 Parameters 构造条件；
- THEN：改变当前 Parameter 的 constraints、visibility、requiredness、default value 等。

多个 block 同时为真时，只有第一个 block 生效，因此排序属于行为语义的一部分。

## 7. Form Sections

Palantir 支持把 Parameters 组织到 Sections 中，并配置一/两列布局、描述、折叠、隐藏和条件 override。这说明 Action Type 不仅定义后端操作，也定义一部分可复用交互契约。

## 8. 参数依赖与性能

官方特别提醒：Parameter default、Object Set options、override 等可能形成依赖链。依赖层级过深会导致 Action form 必须串行加载多个数据请求。

建议：

- 尽量让 Parameter 依赖扁平；
- 能直接依赖上游对象就不要间接依赖另一个计算出来的参数；
- 对大 Object Set 做限制和分页；
- 把复杂计算转移到 Functions，而不是构造深层前端依赖链。

## 9. Security

Object Parameter 下拉结果会遵守用户对象/属性读取权限，但静态过滤值等配置可能对能查看 Action Type 的用户可见。设计时不能把敏感值硬编码到可查看配置中。

## 10. 规模限制

Parameter list 有明确规模限制，应在设计器中提前提示，而不是在执行后失败。

## 11. 自研建议

- Parameter Schema 应独立于 Form Schema；同一参数可以有不同应用层呈现，但业务约束必须统一。
- 对 default/options/override 建依赖 DAG 并做循环检测。
- 支持 Parameter 的“来源”追踪，方便解释某值来自用户、默认、对象属性还是上下文。
- Value Type 约束应在参数层和 Property 层共享实现。

## 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/parameter-overview
- https://www.palantir.com/docs/foundry/action-types/parameters-default-value
- https://www.palantir.com/docs/foundry/action-types/parameters-filter
- https://www.palantir.com/docs/foundry/action-types/parameters-override
- https://www.palantir.com/docs/foundry/action-types/configure-sections
- https://www.palantir.com/docs/foundry/action-types/parameter-performance-considerations
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits
