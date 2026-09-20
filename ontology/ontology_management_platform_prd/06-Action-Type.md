# Action Type — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Action Type |
| 元素分类 | 一级 Ontology Resource / Kinetic Element |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Action Type 定义用户/应用/Agent 可以一次执行的一组 Ontology 变更，以及提交时可能触发的副作用。Action 是某次具体提交。

Palantir 把 Action 看成企业 Ontology 的“动词”。Object、Property、Link 描述现实状态，Action 则描述如何在受控条件下改变现实状态。

### 2. Action Type 能做什么

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

### 3. 核心组成

一个完整 Action Type 至少由以下部分构成：

#### Metadata

API Name、Display Name、Description、Status 等，形成稳定行为契约。

#### Parameters

Action 的输入接口。可以是 primitive、object reference、list 等，并支持默认值、过滤、隐藏、必填和条件覆盖。

#### Rules

把 Parameters 转换为 Ontology edits 或其他 effects。

#### Submission Criteria

决定某次 Action 是否允许提交，承载用户、参数、执行上下文等业务约束。

#### Side Effects

通知、Webhook 等外部副作用。

#### Permissions / Authorizations

决定谁能读取 Action 所需数据、执行 Action、创建/修改受安全控制的数据。

#### Observability

Action metrics、run history、action log、monitoring 等。

### 4. 创建流程

官方 Getting Started 中，典型流程是：

1. 在 Ontology Manager 新建 Action Type；
2. 选择操作的 Object Type 和操作类别；
3. 映射需要修改的 Properties；
4. 填写 Metadata；
5. 自动生成或进一步编辑 Parameters；
6. 配置 Submission Criteria；
7. Test Run；
8. 保存并在 Object Explorer、Object Views、Workshop 等应用消费。

### 5. 事务语义

官方把 Action 描述为一次 transaction，用于一次性完成定义好的对象/属性/链接变化。但与外部系统集成时要区分：

- Ontology edits 本身；
- writeback webhook；
- side-effect webhook。

Writeback webhook 在 Ontology edits 之前执行，失败时可以阻止后续变更；Side-effect webhook 在对象修改之后执行，更接近 best-effort。跨系统并非严格分布式事务，产品说明应清楚呈现失败边界。

### 6. Scale Limits

官方对 Action 有明确规模保护，例如：

- primitive list parameter 最多 10,000 个元素；
- object reference list parameter 通常最多 1,000；
- 单次 Action 最多编辑 50 个 Object Types；
- 单次 Action 最多编辑 10,000 个 Objects；
- batch call 也存在上限；
- function-backed actions 有更严格的调用限制。

自研产品应把这些作为可配置 runtime guardrail，而不是仅靠后端超时。

### 7. 测试

Test Run 会在当前 Ontology branch 上以当前用户权限评估 Action，执行同样的 object security 与 submission criteria，并展示 Proposed Changes。它非常适合作为 Action 发布前的产品能力：让建模者看到将创建/修改/删除哪些对象和 Link。

### 8. 可观测性

Action Metrics 提供成功/失败、P95 duration、运行历史等。失败可分类为：无效参数、规模限制、认证/权限、side effect、function、冲突等。

Action Log 可以记录 Action RID、Action Type RID/version、时间、用户、编辑对象、webhook/notification、scenario、revert 状态以及可选参数值等，从而形成决策审计链。

### 9. 应用消费

Action 可以在 Object Views、Object Explorer、Workshop 等处被复用。同一个 Action 的逻辑和校验在多个应用中保持一致，因此应用层不应重复实现业务写入规则。

### 10. 安全模型

Action 的安全不能只靠“谁能看到按钮”。需要同时评估：

- Action resource permission；
- 参数对象/字段的读取权限；
- Submission Criteria；
- Read/Write Authorizations；
- 被编辑 Object/Link 的权限；
- 外部 webhook 权限与凭证。

### 11. 自研建议

- Action Type 必须是元模型一等公民，不要等同于 REST endpoint。
- Parameters / Rules / Criteria 分离建模，便于复用和解释。
- 每次执行都生成唯一 execution/action ID 和审计记录。
- 支持 dry-run/proposed changes。
- 对外部写回明确 pre-commit / post-commit 语义。
- 为 Agent 暴露 Action 时复用同一权限和 submission criteria，不另建绕过通道。

### 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/overview
- https://www.palantir.com/docs/foundry/action-types/getting-started
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/test-run
- https://www.palantir.com/docs/foundry/action-types/permissions
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits
- https://www.palantir.com/docs/foundry/action-types/action-metrics
- https://www.palantir.com/docs/foundry/action-types/action-log

## 2. 自研平台产品定位
**元素类别：一级 Ontology Resource / Kinetic Element。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 把可执行业务变化建模成受控、可审计的 Ontology 操作
- 统一 Parameters、Rules、Submission Criteria、Side Effects、Permissions 和日志
- 支持简单声明式编辑与 Function-backed 复杂逻辑
- 提供稳定 Action API 给 Workshop/应用/Agent/SDK 使用

### 2.2 非目标 / 边界
- 不允许客户端直接绕过 Action 修改受控对象
- 不把前端表单校验等同于后端 Submission Criteria
- 不在 Action Type 中隐藏不可审计的外部副作用

## 3. 用户角色与权限职责
| 角色 | 主要职责 | 默认能力 |
| --- | --- | --- |
| Ontology Owner | 拥有本 Ontology 的最高治理权限；负责资源发布、删除、权限与版本策略。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Ontology Architect | 负责领域建模、跨类型一致性、命名、依赖和 breaking change 评审。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Ontology Builder | 负责日常创建/编辑资源、数据映射、规则与参数配置。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Data Engineer | 负责 Datasource、字段映射、索引、数据质量和刷新链路。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Application Developer | 通过 API/SDK/Functions/应用消费 Ontology，关注稳定 API Name 和兼容性。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Security Administrator | 负责资源权限、对象/属性策略、组织边界和访问测试。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Business Steward | 负责业务定义、描述、口径、状态和是否可正式推广。 | 按最小权限原则分配；写入/发布/授权需单独权限 |
| Viewer/Discoverer | 只读浏览 Ontology 资源、依赖关系和文档。 | 按最小权限原则分配；写入/发布/授权需单独权限 |

> **权限原则**：Discover/View、Edit Draft、Review、Publish、Manage Permission、Delete/Archive 必须拆成不同 capability；生产发布不得仅依赖“能编辑”。

## 4. 核心用户场景
| 场景 | 用户目标 |
| --- | --- |
| 创建 | 建模人员从资源目录创建新资源，完成最小合法配置并保存草稿。 |
| 编辑 | 对已有资源修改 metadata/结构；系统实时识别低/中/高风险变更。 |
| 验证 | 发布前运行 schema、引用、权限、数据兼容性和依赖检查。 |
| 评审与发布 | 将多个修改组织成 change set，经 Reviewer 批准后发布。 |
| 依赖分析 | 查看资源被哪些上游/下游资产使用，评估 breaking change。 |
| 弃用与迁移 | 将旧资源标记 Deprecated，指定 replacement 与迁移说明。 |
| 审计 | 按人、资源、时间、版本查询所有变更及执行历史。 |


## 5. 信息架构与页面设计
1. **Action Types 列表**
2. **Action Overview**
3. **Parameters/Form**
4. **Rules**
5. **Submission Criteria**
6. **Side Effects**
7. **Permissions & Authorizations**
8. **Test Run**
9. **Metrics & Logs**
10. **Dependencies**
11. **Versions**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| internalId | ID | 是 | 不可变 | 唯一 | 资源ID |
| displayName | String | 是 | 可编辑 | 1..120 | 动作名称 |
| apiName | String | 是 | 受限编辑 | camelCase/Pascal?按平台标准唯一 | 程序契约 |
| description | Markdown | 否 | 可编辑 | 清晰表达业务动词 | 说明 |
| status | Enum | 是 | 可编辑 | experimental/active/deprecated | 生命周期 |
| parameters | ParameterRef[] | 是 | 可编辑 | 顺序/依赖合法 | 输入契约 |
| rules | Rule[] | 是 | 可编辑 | 至少1条有效效果或Function | 执行逻辑 |
| submissionCriteria | Expression | 否 | 可编辑 | 表达式可编译 | 提交条件 |
| sideEffects | SideEffect[] | 否 | 可编辑 | 每个 effect 可观测 | 通知/webhook等 |
| authorizationPolicy | Object | 是 | 受控编辑 | 服务端可执行 | 读写授权 |
| functionRef | FunctionRef | 否 | 高风险编辑 | 输入输出/编辑范围兼容 | 复杂逻辑 |
| formConfig | Object | 否 | 可编辑 | 参数引用有效 | 默认表单体验 |
| timeout/limits | Object | 是 | 受控编辑 | 平台限制内 | 执行约束 |
| observability | Object | 是 | 可编辑 | 日志保留策略合法 | 审计/metrics |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 06-FR-001 | P0 | 创建 Action 时可选择对象修改、对象创建、删除、Link 编辑或 Function-backed 模板 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-002 | P0 | 参数编辑器支持类型、默认值、过滤、hidden/readOnly、override、Value Type | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-003 | P0 | Rules 编辑器支持 create/modify/delete object/link 以及平台支持的其他 effects | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-004 | P0 | Submission Criteria 使用可视表达式构建器并支持 current user、parameter、execution context | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-005 | P0 | 权限页分离“谁能编辑 Action 定义”和“谁能执行 Action” | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-006 | P0 | 支持 Read/Write Authorization，并在 test-run 中显示每一步授权结果 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-007 | P0 | Side Effects 必须展示触发时机、重试策略、幂等键、失败处理 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-008 | P0 | Test Run 支持输入模拟、对象快照、预计 edits、criteria、auth、side-effect dry-run | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-009 | P0 | 发布前静态分析参数未使用、循环 override、无效 property/link 引用、不可达 rule | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-010 | P0 | 执行日志可按 user/action/object/result/time 查询 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-011 | P1 | Metrics 展示执行量、成功率、P50/P95 时延、criteria拒绝率、side-effect失败率 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-012 | P1 | 支持 Action API schema/SDK 示例生成 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-013 | P1 | active Action 修改参数类型、删除参数、API Name、rule semantics 时标记 breaking | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-014 | P1 | 支持 Action version pinning/应用依赖版本策略 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 06-FR-015 | P2 | 对大规模 edits 明确限制并建议批量/异步 Function 模式 | 服务端必须可验证；UI 需提供对应状态/反馈 |


## 8. 创建流程
1. 用户从当前 Ontology 的 `New` 菜单进入创建向导。
2. 系统先校验用户是否具有 `Create`/`Edit Draft` 权限。
3. 用户填写最小必填 metadata；API Name 在输入时即时校验。
4. 用户完成该元素特有结构配置；所有引用资源使用可搜索选择器而不是手填 ID。
5. 页面运行本地即时校验；服务端保存时再次执行完整校验。
6. 保存为 Draft，生成 immutable internalId 和 version=1 草稿。
7. 用户可继续补充依赖、安全、描述、状态等配置。
8. 点击 `Validate` 获取 blocking/warning/info 三类检查结果。
9. 通过后加入 Change Set；若涉及高风险变更，必须填写 change reason / migration plan。
10. Reviewer 批准后发布；系统发出领域事件并刷新依赖/搜索索引。

## 9. 编辑、版本与发布流程
### 9.1 草稿编辑
- 所有修改先进入 Draft Revision；生产 active 版本保持只读。
- 每次保存递增 revision，不等于正式 release version。
- 支持“撤销本次修改”“与已发布版本比较”“恢复某个字段”。
### 9.2 Change Set
- 一个 Change Set 可以包含多个相关 Ontology resource 变更，避免只发布一半导致 schema 不一致。
- Change Set 至少包含：标题、原因、Owner、变更清单、风险等级、迁移说明、验证结果、Reviewer。
### 9.3 发布策略
- Low risk：可按团队策略自动发布。
- Medium risk：至少 1 名 Reviewer。
- High/Breaking：必须 Ontology Owner/Architect 审批，附影响分析和迁移计划。
- 发布失败必须保持旧版本继续服务，不允许半发布。

## 10. 业务校验规则
| 规则ID | 级别 | 规则 |
| --- | --- | --- |
| 06-VAL-001 | Blocking | 每个 rule 引用的 parameter/property/link 必须存在 |
| 06-VAL-002 | Blocking | required 参数必须能由用户输入/default/override 中至少一种路径赋值 |
| 06-VAL-003 | Blocking | override 依赖图不得形成循环 |
| 06-VAL-004 | Blocking | submission criteria 必须在服务端执行且不可被 UI 绕过 |
| 06-VAL-005 | Blocking | function-backed action 的声明 edit scope 必须覆盖实际 edits |
| 06-VAL-006 | Blocking | side effect 必须配置超时/重试/幂等策略或显式选择不重试 |
| 06-VAL-007 | Blocking | 删除/修改 active 参数前必须检查消费方 |
| 06-VAL-008 | Blocking | Action 至少产生一个可定义的 Ontology edit 或 side effect |
| 06-VAL-009 | Blocking | 涉及删除对象的 Action 默认要求更高风险等级和显式权限 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Draft -> Experimental -> Active -> Deprecated -> Disabled -> Archived
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Object Type
- Link Type
- Property
- Action Parameter
- Rules
- Submission Criteria
- Side Effects
- Functions
- IAM
- Audit/Metrics
### 12.2 变更影响分析必须回答
- 哪些 Ontology resources 直接引用本资源？
- 哪些 Action / Function / Interface / Link / Property 会失效？
- 哪些应用、SDK 或 API contract 可能受到影响？
- 是否需要 reindex / rematerialize / runtime reload？
- 是否存在无法自动迁移的 breaking consumer？
- 变更发布顺序是否有前后依赖？

## 13. 权限与安全要求
| 能力 | 建议权限 |
| --- | --- |
| Discover/View | Viewer/Discoverer |
| Create/Edit Draft | Builder |
| Change Status | Steward/Architect |
| Review | Reviewer/Architect |
| Publish | Ontology Owner/Release Manager |
| Manage Permission | Ontology Owner/Security Admin |
| Delete/Archive | Ontology Owner，且通过依赖检查 |
| Test/Simulate | Builder；但不得绕过真实数据读取权限 |

- 所有 API 在服务端校验当前 Ontology 组织边界。
- 用户没有读取权限的依赖资源，Impact 页面可显示“存在受限依赖”，但不得泄露名称/内容（策略可配置）。

## 14. 搜索、列表、过滤与批量操作
- 搜索字段：Display Name、API Name、Description、Owner、ID/RID。
- 过滤：Status、Owner、Group/Tag、更新时间、依赖风险、是否有 Validation Error。
- 列表默认列：Name、API Name、Status、Owner、Updated At、Used By、Validation。
- 批量操作仅允许低风险 metadata（如 Group/Owner/Status 的受限变更）；Key/Type/API Name 等禁止批量盲改。
- 导出 CSV/JSON 时尊重当前用户权限。

## 15. API 需求
- `GET /action-types`
- `POST /action-types`
- `PATCH /action-types/{id}`
- `POST /action-types/{id}:validate`
- `POST /action-types/{id}:test-run`
- `POST /actions/{apiName}:apply`
- `GET /action-types/{id}/logs`
- `GET /action-types/{id}/metrics`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| actionType.created | 对应资源状态或定义发生变化时 |
| actionType.definition.changed | 对应资源状态或定义发生变化时 |
| actionType.published | 对应资源状态或定义发生变化时 |
| action.execution.started | 对应资源状态或定义发生变化时 |
| action.execution.rejected | 对应资源状态或定义发生变化时 |
| action.execution.succeeded | 对应资源状态或定义发生变化时 |
| action.execution.failed | 对应资源状态或定义发生变化时 |
| action.sideEffect.failed | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 06_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 06_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 06_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 06_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 06_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 06_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


## 18. 非功能需求（NFR）
1. 所有资源必须使用不可变内部 ID；Display Name 可修改，API Name 受兼容性策略保护。
2. 所有写操作必须产生审计记录，至少包含 actor、时间、资源、前后差异、变更原因、关联发布版本。
3. 列表页常规查询在 10 万级元数据资源下 P95 < 2 秒；详情页元数据 P95 < 1.5 秒（不含外部数据源探测）。
4. 保存采用幂等 API；对并发编辑使用 optimistic locking/version 字段，冲突时禁止静默覆盖。
5. 删除、重命名 API Name、改变类型、改变主键等高风险操作必须先完成依赖分析并二次确认。
6. 权限必须在服务端强制执行；UI 隐藏不能替代授权。
7. 支持草稿与已发布版本隔离；未发布变更不得影响生产消费方。
8. 所有错误必须返回稳定 errorCode、用户可理解 message、resourceId、traceId；禁止只返回堆栈。
9. 所有列表视图支持搜索、过滤、排序、分页和 URL 可复现筛选状态。
10. 所有关键配置应可通过 API 导出为机器可读 JSON/YAML，便于 GitOps、审计和环境迁移。

## 19. 可观测性与运营指标
- 资源总数、Active/Deprecated 数量、7/30 天变更量。
- Validation failure rate、publish success rate、rollback rate。
- breaking change 被阻断次数、平均迁移时长。
- 详情页/列表页 P95 latency、依赖分析 P95 latency。
- 每个资源的 consumer count 和 owner completeness。

## 20. 验收标准（Acceptance Criteria）
| 用例ID | 类型 | Given | When | Then |
| --- | --- | --- | --- | --- |
| 06-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 创建 Action 时可选择对象修改、对象创建、删除、Link 编辑或 Function-backed 模板 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | 参数编辑器支持类型、默认值、过滤、hidden/readOnly、override、Value Type | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | Rules 编辑器支持 create/modify/delete object/link 以及平台支持的其他 effects | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | Submission Criteria 使用可视表达式构建器并支持 current user、parameter、execution context | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | 权限页分离“谁能编辑 Action 定义”和“谁能执行 Action” | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 Read/Write Authorization，并在 test-run 中显示每一步授权结果 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | Side Effects 必须展示触发时机、重试策略、幂等键、失败处理 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | Test Run 支持输入模拟、对象快照、预计 edits、criteria、auth、side-effect dry-run | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 06-AC-009 | Validation | 用户提交违反规则：每个 rule 引用的 parameter/property/link 必须存在 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 06-AC-010 | Validation | 用户提交违反规则：required 参数必须能由用户输入/default/override 中至少一种路径赋值 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 06-AC-011 | Validation | 用户提交违反规则：override 依赖图不得形成循环 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 06-AC-012 | Validation | 用户提交违反规则：submission criteria 必须在服务端执行且不可被 UI 绕过 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 06-AC-013 | Validation | 用户提交违反规则：function-backed action 的声明 edit scope 必须覆盖实际 edits | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 06-AC-014 | Validation | 用户提交违反规则：side effect 必须配置超时/重试/幂等策略或显式选择不重试 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


## 21. 测试要求
- **单元测试**：字段规则、命名规则、状态机、compatibility matrix、表达式校验。
- **契约测试**：API schema、错误码、乐观锁、幂等。
- **集成测试**：跨 Object/Property/Link/Action/Interface 依赖变更。
- **权限测试**：Owner/Builder/Viewer/Security Admin 的正向和越权用例。
- **迁移测试**：active 资源 breaking change、deprecate/replacement、回滚。
- **性能测试**：大量 metadata、复杂 dependency graph、并发编辑。

## 22. 研发拆分建议
| 阶段 | 范围 |
| --- | --- |
| P0 | 列表/详情/创建/编辑、字段校验、Draft、基础权限、审计、依赖引用、发布。 |
| P1 | 完整 Impact Analysis、Change Set Review、批量操作、策略测试、版本 diff、可观测指标。 |
| P2 | 跨环境 package、Marketplace/GitOps、自动迁移建议、AI 辅助建模与质量检查。 |


## 23. 与其他模块的集成契约
- **Metadata Service**：资源定义、版本、状态、API Name。
- **Dependency Graph Service**：引用边和消费方。
- **Change Management Service**：draft/change set/review/publish/rollback。
- **IAM/Policy Service**：资源权限与运行时安全。
- **Audit Service**：不可篡改变更记录。
- **Search Service**：跨资源检索。
- **Runtime/Index Service**：仅在需要对象数据、索引或 Action 执行的模块接入。

## 24. 官方资料来源
- https://www.palantir.com/docs/foundry/action-types/action-log
- https://www.palantir.com/docs/foundry/action-types/action-metrics
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/getting-started
- https://www.palantir.com/docs/foundry/action-types/overview
- https://www.palantir.com/docs/foundry/action-types/permissions
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits
- https://www.palantir.com/docs/foundry/action-types/test-run

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
