# Submission Criteria — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Submission Criteria |
| 元素分类 | Action Type 提交治理子元素 |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Submission Criteria 是决定某次 Action 是否允许提交的条件。它原名 validations，但现在的定位更明确：把业务规则嵌入数据编辑权限与流程治理，保证 Ontology 数据质量和操作合规。

它和“谁能编辑 Action Type 配置”不是一回事；每个 Action Type 可以拥有独立的执行条件。

### 2. 条件模型

Submission Criteria 由 **Conditions + Logical Operators** 组成。

Condition 是两个值之间的一次比较；Logical Operator 用于把多个条件组合、嵌套成复杂逻辑。

### 3. 三类 Context Template

#### Current User

可以依据当前用户：

- User ID；
- Group IDs；
- Organization；
- 其他 Multipass attributes；

来决定是否允许执行。

官方特别提醒不要对 group/marking/organization membership 随意使用 NOT 条件，因为 scoped token 可能没有携带完整属性，错误的否定逻辑可能产生越权风险。

#### Parameter

根据 Action Parameters 以及 Object Parameter 的 Properties 判断业务条件。例如只有 Ticket.status = Open 才允许修改 Priority。

#### Execution Context

可以判断 Action 是否在 Ontology Scenario 等执行上下文中运行，从而为规划/仿真和生产操作设置不同规则。

### 4. Operators

不同 Parameter 类型支持不同比较运算，如：

- is / is not
- matches
- less than / greater than
- list/集合相关运算
- all / any / none 等逻辑组合

产品设计器应根据左右类型动态限制合法 operator，避免运行时才发现类型错误。

### 5. Failure Message

根级条件/逻辑块可以配置面向终端用户的失败消息。当 criteria 不满足时，Object Explorer、Workshop、Quiver 等可以展示具体原因。

这使 Submission Criteria 不只是访问控制，还承担“可解释业务约束”的职责。

### 6. 与 Permissions 的区别

建议严格区分：

- **Resource Permission**：谁可以查看/编辑 Action Type 定义；
- **Object/Property Security**：用户能读取哪些输入数据；
- **Submission Criteria**：在当前业务上下文下能不能提交；
- **Write Authorization**：允许 Action 写入何种安全级别的数据。

Submission Criteria 不能替代底层数据权限。

### 7. 支持与限制

官方说明 attachment 与 object set 参数不能直接用于 submission criteria。复杂判断要通过可支持的 Parameter / Function /预计算方式表达。

### 8. Test Run

Ontology Manager 支持用给定 Parameter 值验证 Submission Criteria；Action Test Run 也会按真实权限执行同一套 criteria。这是产品中非常重要的测试与治理能力。

### 9. 自研建议

- Criteria 使用类型化表达式树。
- 条件中允许引用 current user、parameter、object property、execution context。
- 所有拒绝必须返回可解释失败原因。
- 对 NOT + 权限属性做静态风险检测。
- 支持按用户/参数做策略模拟。
- Agent 执行 Action 时必须走同一 criteria evaluator，不能绕过。

### 官方原始资料

- https://www.palantir.com/docs/foundry/action-types/submission-criteria
- https://www.palantir.com/docs/foundry/action-types/test-run
- https://www.palantir.com/docs/foundry/action-types/permissions

## 2. 自研平台产品定位
**元素类别：Action Type 提交治理子元素。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 在 Action 执行前强制业务条件
- 将用户、参数、对象/关系、执行上下文组合成可审计逻辑
- 确保数据质量和编辑治理
- 与 IAM/Authorization 形成分层防线

### 2.2 非目标 / 边界
- 不替代 Action 执行权限/IAM
- 不只依赖前端按钮禁用
- 不在 criteria 中执行副作用

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
1. **Action > Submission Criteria Builder**
2. **Expression Tree**
3. **Condition Editor**
4. **Message/UX**
5. **Test Matrix**
6. **Audit/Usage**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| criteriaId | ID | 是 | 不可变 | Action内唯一 | 标识 |
| expression | Boolean AST | 是 | 可编辑 | 合法AST | 提交条件 |
| conditions | Condition[] | 是 | 可编辑 | 模板合法 | 叶子条件 |
| failureMessage | I18n String | 是 | 可编辑 | 不得泄露敏感信息 | 拒绝提示 |
| severity | Enum | 是 | 可编辑 | block/warn(若平台支持) | 处理级别 |
| enabled | Boolean | 是 | 可编辑 | — | 启用 |
| version | Int | 是 | 系统 | 递增 | 审计 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 09-FR-001 | P0 | 可视化表达式支持 AND/OR/NOT 与嵌套分组 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-002 | P0 | Condition 模板至少支持 Current User、Parameter、Execution Context；对象/关系条件按平台能力接入 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-003 | P0 | 每个条件允许定义用户可理解的失败原因 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-004 | P0 | Test Matrix 可批量输入不同 user/context/parameter 验证 pass/fail | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-005 | P0 | 执行时 criteria 必须在服务器、靠近 edit 提交点重新计算 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-006 | P0 | 执行日志记录 criteria 结果，但敏感数据脱敏 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-007 | P0 | 支持显示“为什么不可提交”的可解释结果 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-008 | P0 | 修改 active criteria 视为业务语义变更并进入发布评审 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-009 | P0 | 允许 criteria 引用 Action Parameter 当前值和对象当前状态 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 09-FR-010 | P0 | 与 authorization 结果分开展示，避免把“无权限”和“业务条件不满足”混淆 | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 09-VAL-001 | Blocking | 根表达式必须返回 Boolean |
| 09-VAL-002 | Blocking | 所有参数引用存在 |
| 09-VAL-003 | Blocking | 用户组/属性引用有效 |
| 09-VAL-004 | Blocking | 表达式无不可解析循环/函数 |
| 09-VAL-005 | Blocking | failureMessage 必填且不可暴露保密字段 |
| 09-VAL-006 | Blocking | 条件访问的数据必须遵守读取权限策略 |
| 09-VAL-007 | Blocking | blocking criteria 失败时服务器必须拒绝执行 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Enabled -> Disabled -> Invalid
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Action Type
- Action Parameter
- User/Group Directory
- Expression Engine
- Authorization
- Object Query
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
- `GET /action-types/{id}/submission-criteria`
- `PUT /action-types/{id}/submission-criteria`
- `POST /action-types/{id}/submission-criteria:test`
- `POST /actions/{apiName}:evaluate`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| submissionCriteria.changed | 对应资源状态或定义发生变化时 |
| action.submission.rejected | 对应资源状态或定义发生变化时 |
| action.submission.criteriaError | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 09_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 09_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 09_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 09_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 09_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 09_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 09-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 可视化表达式支持 AND/OR/NOT 与嵌套分组 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | Condition 模板至少支持 Current User、Parameter、Execution Context；对象/关系条件按平台能力接入 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | 每个条件允许定义用户可理解的失败原因 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | Test Matrix 可批量输入不同 user/context/parameter 验证 pass/fail | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | 执行时 criteria 必须在服务器、靠近 edit 提交点重新计算 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | 执行日志记录 criteria 结果，但敏感数据脱敏 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持显示“为什么不可提交”的可解释结果 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 修改 active criteria 视为业务语义变更并进入发布评审 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 09-AC-009 | Validation | 用户提交违反规则：根表达式必须返回 Boolean | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 09-AC-010 | Validation | 用户提交违反规则：所有参数引用存在 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 09-AC-011 | Validation | 用户提交违反规则：用户组/属性引用有效 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 09-AC-012 | Validation | 用户提交违反规则：表达式无不可解析循环/函数 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 09-AC-013 | Validation | 用户提交违反规则：failureMessage 必填且不可暴露保密字段 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 09-AC-014 | Validation | 用户提交违反规则：条件访问的数据必须遵守读取权限策略 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/action-types/permissions
- https://www.palantir.com/docs/foundry/action-types/submission-criteria
- https://www.palantir.com/docs/foundry/action-types/test-run

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
