# Action Rules — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Action Rules |
| 元素分类 | Action Type 执行逻辑子元素 |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Rules 定义 Action Type 的执行逻辑，即如何把 Parameters 转化为 Ontology edits 或其他 Foundry effects。它是 Action Type 的“输出/执行层”。

### 2. 两大类规则

#### Ontology Rules

直接改变 Ontology 数据：

- Create Object
- Modify Object(s)
- Delete Object(s)
- Create Link
- Delete Link

对于 one-to-many / one-to-one 等外键关系，创建/删除 Link 往往通过修改对象 foreign key Property 完成。

#### Other Effects / Integrations

可以调用 Functions、Webhook、通知、build 等能力，实现超出简单对象编辑的流程。

### 3. 值映射

修改 Object 时，每个目标 Property 需要定义值来自哪里。官方支持的典型来源包括：

- **From parameter**：从同类型 Parameter 获取；
- **Object parameter property**：从 Object Reference Parameter 的某个 Property 获取；
- **Static value**：规则内部常量；
- **Current User / Time**：运行时上下文值；
- Webhook response / Function output 等更高级来源。

产品实现应把 value source 设计成可扩展 expression，而不是固定几种硬编码字段。

### 4. Create Object

创建对象时 Primary Key 是必填的，因为新 Object 必须有唯一标识。其他 Property 可由 Parameter、静态值、上下文或 Function 计算得到。

### 5. Modify Object(s)

目标对象一般由 Object Reference Parameter / list Parameter 提供。规则指定可以修改哪些 Property，以及每个新值如何计算。

### 6. Link Rules

Link 的创建/删除受到其 cardinality 和 storage model 影响。many-to-many 可直接编辑 Link storage；foreign-key backed links 则可能转化为 Property edit。

因此 Rule engine 必须与 Ontology metadata 联动，不能脱离 Link 定义独立执行。

### 7. Function-backed Rules

当普通 declarative rule 不足以表达：

- 跨多个关联对象修改；
- 基于复杂计算修改属性；
- 一次创建多个不同类型对象和 Link；
- 复杂循环/聚合逻辑；

可以使用 Ontology Edit Function。Function-backed Action 灵活度更高，但受到 Functions runtime 与 Action limits 的双重约束。

### 8. Rule 组合

一个 Action Type 可以组合多个普通 Rules，从而在一次业务动作中完成多个受控变化。Function rule 是特殊情况，不能简单与所有 Ontology rules 并列组合。

产品层应明确 Rule 执行阶段、依赖、失败处理和原子性边界。

### 9. 安全与权限

Rule 能做什么不等于执行用户一定有权限做什么。运行时仍必须评估 Action permissions、Submission Criteria、Object/Property Security、Read/Write Authorization 等。

### 10. 测试与解释

规则设计器应该支持：

- 给定一组 Parameter 值进行 test run；
- 展示 proposed object/link changes；
- 标注每个目标值的来源；
- 展示未满足的权限或 criteria；
- 解释 Function/Webhook 等外部依赖。

### 11. 自研建议

- Rule 使用声明式 AST / IR 表达，不要只存 UI JSON。
- value source 统一抽象。
- Rule 与 Ontology schema 做静态类型检查。
- 任何 schema 变化都应重新验证受影响 Rules。
- 支持 execution plan / proposed changes，方便 Agent 与用户审计。

### 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/rules
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/function-actions-overview
- https://www.palantir.com/docs/foundry/action-types/test-run

## 2. 自研平台产品定位
**元素类别：Action Type 执行逻辑子元素。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 声明 Parameters 如何转换为 Ontology edits 或平台 effects
- 提供可视化、可验证、可审计的简单业务变更逻辑
- 减少简单 CRUD 对自定义代码的依赖
- 与 Function-backed action 建立清晰边界

### 2.2 非目标 / 边界
- 不承担是否允许用户提交的职责（由 Submission Criteria/Auth负责）
- 不支持无界循环/任意脚本作为规则
- 不把复杂算法强塞进声明式规则

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
1. **Action > Rules Canvas/List**
2. **Rule Detail**
3. **Input Mapping**
4. **Target Object/Link**
5. **Preview Edits**
6. **Validation**
7. **Usage**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| ruleId | ID | 是 | 不可变 | Action内唯一 | 规则ID |
| ruleType | Enum | 是 | 高风险编辑 | create/modify/delete object/link/effect | 规则类别 |
| targetType | ResourceRef | 是 | 高风险编辑 | 资源存在 | 操作目标 |
| selector | Expression | 条件必填 | 可编辑 | 能唯一/合法定位目标 | 目标对象选择 |
| assignments | Map<Property,Expression> | 条件必填 | 可编辑 | 属性存在且类型兼容 | 写入内容 |
| linkEndpoints | Expression[] | 条件必填 | 可编辑 | 两端类型兼容 | Link编辑 |
| condition | Expression | 否 | 可编辑 | 可编译 | 规则条件 |
| order | Integer | 是 | 可编辑 | 无冲突 | 执行/展示顺序 |
| failurePolicy | Enum | 是 | 可编辑 | fail-all/skip?按平台能力 | 失败语义 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 08-FR-001 | P0 | Rule Builder 按 rule type 只展示合法配置 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-002 | P0 | Create Object 必须要求 PK 赋值或可自动生成策略 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-003 | P0 | Modify Object 必须明确目标对象来源 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-004 | P0 | Delete Object 显示级联/Link影响预览 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-005 | P0 | Link create/delete 必须校验 endpoints 和 cardinality | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-006 | P0 | Property assignment 支持 parameter、常量、对象当前属性、上下文等表达式 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-007 | P0 | 多条 rules 组成一个 Action transaction 时必须明确原子性语义 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-008 | P0 | 支持 Expected Edits 预览 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-009 | P0 | 系统检测同一 Action 内对同一 Property 的冲突写入 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-010 | P0 | 复杂规则推荐迁移 Function，并显示原因 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-011 | P1 | 规则变更进入 breaking/semantic change 风险评估 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 08-FR-012 | P1 | Rule 使用的所有资源引用必须参与 dependency graph | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 08-VAL-001 | Blocking | 所有资源引用必须存在且版本兼容 |
| 08-VAL-002 | Blocking | Property assignment 类型必须兼容目标 Property/Value Type |
| 08-VAL-003 | Blocking | Create Object 必须满足 required/PK约束 |
| 08-VAL-004 | Blocking | Link 规则必须满足 cardinality |
| 08-VAL-005 | Blocking | 同一事务中不允许无法定义顺序的冲突 edits |
| 08-VAL-006 | Blocking | 删除规则后 Action 仍必须有有效 effect |
| 08-VAL-007 | Blocking | Condition 表达式必须可编译且无非法引用 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Valid -> Invalid -> Disabled（可选）
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Action Parameter
- Object Type
- Property
- Link Type
- Expression Engine
- Transaction/Edit Runtime
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
- `GET /action-types/{id}/rules`
- `POST /action-types/{id}/rules`
- `PATCH /action-rules/{id}`
- `POST /action-rules/{id}:validate`
- `POST /action-types/{id}:preview-edits`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| actionRule.created | 对应资源状态或定义发生变化时 |
| actionRule.changed | 对应资源状态或定义发生变化时 |
| actionRule.validation.failed | 对应资源状态或定义发生变化时 |
| actionRule.deleted | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 08_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 08_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 08_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 08_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 08_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 08_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 08-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | Rule Builder 按 rule type 只展示合法配置 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | Create Object 必须要求 PK 赋值或可自动生成策略 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | Modify Object 必须明确目标对象来源 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | Delete Object 显示级联/Link影响预览 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | Link create/delete 必须校验 endpoints 和 cardinality | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | Property assignment 支持 parameter、常量、对象当前属性、上下文等表达式 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | 多条 rules 组成一个 Action transaction 时必须明确原子性语义 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 Expected Edits 预览 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 08-AC-009 | Validation | 用户提交违反规则：所有资源引用必须存在且版本兼容 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 08-AC-010 | Validation | 用户提交违反规则：Property assignment 类型必须兼容目标 Property/Value Type | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 08-AC-011 | Validation | 用户提交违反规则：Create Object 必须满足 required/PK约束 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 08-AC-012 | Validation | 用户提交违反规则：Link 规则必须满足 cardinality | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 08-AC-013 | Validation | 用户提交违反规则：同一事务中不允许无法定义顺序的冲突 edits | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 08-AC-014 | Validation | 用户提交违反规则：删除规则后 Action 仍必须有有效 effect | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/action-types/explore-action-types
- https://www.palantir.com/docs/foundry/action-types/function-actions-overview
- https://www.palantir.com/docs/foundry/action-types/rules
- https://www.palantir.com/docs/foundry/action-types/test-run

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
