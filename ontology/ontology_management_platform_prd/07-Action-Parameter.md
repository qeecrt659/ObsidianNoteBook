# Action Parameter — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Action Parameter |
| 元素分类 | Action Type 关键子元素 |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Parameter 是 Action Type 的输入，也是 Action Rules 与 Workshop、Object Views、Slate 等消费应用之间的接口。它类似带类型的变量，由外部用户或应用在执行时赋值。

### 2. 主要作用

Parameter 的值可以被用于：

- 规则中设置 Property 值；
- 创建/删除 Link；
- Side Effect / Webhook 输入；
- Submission Criteria 判断；
- 获取对象修改前的当前值；
- 驱动后续参数的 default、options、visibility、requiredness 等条件配置。

### 3. 类型系统

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

### 4. 默认值

Palantir 支持全局 Parameter Default，用于在所有消费应用中统一预填。默认值可来自：

- 静态值；
- 前面某个 Object Parameter 的 Property；
- 特殊上下文值，例如 UUID、current user 等 type-class prefill。

应用本地传入的值通常优先于 Action Type 全局默认值。

### 5. Dropdown / Allowed Values

非对象参数可以配置 multiple-choice；Object Reference 参数可以基于 Object Set 过滤可选对象。过滤支持：

- Property 条件；
- Parameter 值；
- Object parameter property；
- Search Around；
- 自定义 starting Object Set。

这使 Action form 能表达“上下文敏感的可选值”，而不是静态表单。

### 6. Overrides

Parameter Overrides 是 Action form 动态逻辑的重要能力。每个 override block 包含：

- IF：根据之前的 Parameters 构造条件；
- THEN：改变当前 Parameter 的 constraints、visibility、requiredness、default value 等。

多个 block 同时为真时，只有第一个 block 生效，因此排序属于行为语义的一部分。

### 7. Form Sections

Palantir 支持把 Parameters 组织到 Sections 中，并配置一/两列布局、描述、折叠、隐藏和条件 override。这说明 Action Type 不仅定义后端操作，也定义一部分可复用交互契约。

### 8. 参数依赖与性能

官方特别提醒：Parameter default、Object Set options、override 等可能形成依赖链。依赖层级过深会导致 Action form 必须串行加载多个数据请求。

建议：

- 尽量让 Parameter 依赖扁平；
- 能直接依赖上游对象就不要间接依赖另一个计算出来的参数；
- 对大 Object Set 做限制和分页；
- 把复杂计算转移到 Functions，而不是构造深层前端依赖链。

### 9. Security

Object Parameter 下拉结果会遵守用户对象/属性读取权限，但静态过滤值等配置可能对能查看 Action Type 的用户可见。设计时不能把敏感值硬编码到可查看配置中。

### 10. 规模限制

Parameter list 有明确规模限制，应在设计器中提前提示，而不是在执行后失败。

### 11. 自研建议

- Parameter Schema 应独立于 Form Schema；同一参数可以有不同应用层呈现，但业务约束必须统一。
- 对 default/options/override 建依赖 DAG 并做循环检测。
- 支持 Parameter 的“来源”追踪，方便解释某值来自用户、默认、对象属性还是上下文。
- Value Type 约束应在参数层和 Property 层共享实现。

### 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/parameter-overview
- https://www.palantir.com/docs/foundry/action-types/parameters-default-value
- https://www.palantir.com/docs/foundry/action-types/parameters-filter
- https://www.palantir.com/docs/foundry/action-types/parameters-override
- https://www.palantir.com/docs/foundry/action-types/configure-sections
- https://www.palantir.com/docs/foundry/action-types/parameter-performance-considerations
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits

## 2. 自研平台产品定位
**元素类别：Action Type 关键子元素。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 定义 Action 的强类型输入契约
- 支持默认值、过滤、隐藏/只读、动态 override 和 Value Type 校验
- 为表单、API、Rules、Criteria 提供统一参数模型
- 保证参数依赖与性能可预测

### 2.2 非目标 / 边界
- 参数本身不执行 Ontology edit
- 前端组件配置不替代参数后端类型校验
- 不允许动态 override 构成循环依赖

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
1. **Action > Parameters 列表**
2. **Parameter Detail**
3. **Type & Validation**
4. **Default Value**
5. **Filter**
6. **Override/Dependencies**
7. **Form Presentation**
8. **Usage**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| id | ID | 是 | 不可变 | Action内唯一 | 内部ID |
| displayName | String | 是 | 可编辑 | 1..120 | 表单名称 |
| apiName | String | 是 | 受限编辑 | Action内唯一 | API字段 |
| description | String | 否 | 可编辑 | ≤2000 | 帮助文本 |
| type | TypeRef | 是 | 高风险编辑 | 支持 primitive/object/objectSet等 | 输入类型 |
| valueType | ValueTypeRef | 否 | 受控编辑 | 类型兼容 | 复用校验 |
| required | Boolean | 是 | 可编辑 | 赋值路径必须存在 | 必填 |
| hidden | Boolean | 是 | 可编辑 | hidden须有系统赋值路径 | 不展示 |
| readOnly | Boolean | 是 | 可编辑 | 必须有预填值 | 不可编辑 |
| defaultValue | Expression | 否 | 可编辑 | 类型兼容 | 默认值 |
| filter | Expression | 否 | 可编辑 | 仅相关对象/枚举类型 | 候选约束 |
| overrideRules | Override[] | 否 | 可编辑 | DAG无环 | 动态配置 |
| multiValue | Boolean | 是 | 高风险编辑 | 类型支持 | 多选 |
| formSection | SectionRef | 否 | 可编辑 | section存在 | 表单布局 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 07-FR-001 | P0 | 参数按拖拽顺序决定默认表单展示顺序 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-002 | P0 | 支持 primitive、Object、Object Set/Array（如平台支持）、枚举/Value Type 等类型 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-003 | P0 | Object 参数支持基于其他参数/当前用户/上下文动态过滤候选对象 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-004 | P0 | 默认值支持静态值、对象属性、当前用户、上下文、Function（如支持） | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-005 | P0 | Override 可动态修改可见性、必填性、默认值、过滤器等受支持配置 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-006 | P0 | 系统自动绘制参数依赖 DAG | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-007 | P0 | 参数被 Rule/Criteria/SideEffect 引用时显示 usage | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-008 | P0 | 删除参数前必须处理所有引用 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-009 | P0 | 表单预览支持不同输入上下文和角色 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-010 | P0 | 性能警告：避免低效的单对象逐个解析或巨大候选集合 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-011 | P1 | API schema 明确 required/nullable/array/valueType constraint | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 07-FR-012 | P1 | 参数错误在表单端可提前提示，同时服务端重复校验 | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 07-VAL-001 | Blocking | apiName Action 内唯一 |
| 07-VAL-002 | Blocking | required+hidden 参数必须有 default/override/context 赋值 |
| 07-VAL-003 | Blocking | readOnly 参数必须有值来源 |
| 07-VAL-004 | Blocking | defaultValue 类型必须兼容 |
| 07-VAL-005 | Blocking | filter 只能引用已定义且依赖顺序可解析的参数 |
| 07-VAL-006 | Blocking | override 依赖图不可成环 |
| 07-VAL-007 | Blocking | Value Type constraint 必须通过 |
| 07-VAL-008 | Blocking | 删除参数前引用数为0 |
| 07-VAL-009 | Blocking | 对象参数引用的 Object Type/Interface 必须 active/可访问 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Configured -> Invalid -> Deprecated（参数级可选）
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Action Type
- Value Type
- Object Type/Interface
- Rules
- Submission Criteria
- Form Renderer
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
- `GET /action-types/{id}/parameters`
- `POST /action-types/{id}/parameters`
- `PATCH /action-parameters/{id}`
- `POST /action-parameters/{id}:validate`
- `GET /action-parameters/{id}/usage`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| actionParameter.created | 对应资源状态或定义发生变化时 |
| actionParameter.type.changed | 对应资源状态或定义发生变化时 |
| actionParameter.override.changed | 对应资源状态或定义发生变化时 |
| actionParameter.deleted | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 07_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 07_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 07_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 07_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 07_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 07_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 07-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 参数按拖拽顺序决定默认表单展示顺序 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 primitive、Object、Object Set/Array（如平台支持）、枚举/Value Type 等类型 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | Object 参数支持基于其他参数/当前用户/上下文动态过滤候选对象 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | 默认值支持静态值、对象属性、当前用户、上下文、Function（如支持） | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | Override 可动态修改可见性、必填性、默认值、过滤器等受支持配置 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | 系统自动绘制参数依赖 DAG | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | 参数被 Rule/Criteria/SideEffect 引用时显示 usage | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 删除参数前必须处理所有引用 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 07-AC-009 | Validation | 用户提交违反规则：apiName Action 内唯一 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 07-AC-010 | Validation | 用户提交违反规则：required+hidden 参数必须有 default/override/context 赋值 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 07-AC-011 | Validation | 用户提交违反规则：readOnly 参数必须有值来源 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 07-AC-012 | Validation | 用户提交违反规则：defaultValue 类型必须兼容 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 07-AC-013 | Validation | 用户提交违反规则：filter 只能引用已定义且依赖顺序可解析的参数 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 07-AC-014 | Validation | 用户提交违反规则：override 依赖图不可成环 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/action-types/configure-sections
- https://www.palantir.com/docs/foundry/action-types/parameter-overview
- https://www.palantir.com/docs/foundry/action-types/parameter-performance-considerations
- https://www.palantir.com/docs/foundry/action-types/parameters-default-value
- https://www.palantir.com/docs/foundry/action-types/parameters-filter
- https://www.palantir.com/docs/foundry/action-types/parameters-override
- https://www.palantir.com/docs/foundry/action-types/scale-property-limits

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
