# Ontology（顶层本体） — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Ontology（顶层本体） |
| 元素分类 | 顶层容器 / Ontology Artifact |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 产品定义

Ontology 是 Palantir 平台中用于承载 Ontology resources 的顶层工件。它不是单纯的 schema registry，也不是单纯的知识图谱，而是把企业的真实数据、语义关系、业务逻辑、可执行动作和安全治理统一到一个可供人和 AI 使用的操作模型中。

官方定义的 Ontology resources 包括 Object Types、Link Types、Action Types、Interfaces、Shared Properties 和 Object Type Groups。Property 则作为 Object Type 的关键组成部分存在。

### 2. 产品定位

从传统架构看，Ontology 位于“原始数据/业务系统”与“分析、应用、AI、自动化”之间。其价值是：

- 把底层表、流、文件、API 等技术数据模型映射为业务世界中的实体与事件；
- 把跨系统关系转化为 Link；
- 把可执行操作转化为 Action；
- 把逻辑通过 Functions 等能力暴露给应用、用户和 Agent；
- 用统一安全模型保证同一语义资源在不同应用中保持一致的访问和操作约束。

### 3. Ontology 与 Space 的关系

Palantir 官方说明，一个 Ontology 与一个 Space 之间是 **1:1 映射**。创建 Space 时会同步创建同名 Ontology，并继承 Space 的组织标记。Ontology 可以是 private，也可以是 shared；shared ontology 用于在多个组织之间安全共享数据和工作流。

因此 Ontology 的边界同时承担：

- 模型命名空间；
- 组织/协作边界；
- 权限和资源归属边界；
- Value Type 等空间级资源的消费边界。

### 4. 决策中心模型

Palantir 在新的 Ontology 产品叙事中，把企业决策拆成四部分：

#### Data

Object、Property、Link 把 ERP、MES、WMS、IoT、流数据、非结构化数据等形成企业现实的语义表示，并可以持续更新。

#### Logic

业务规则、算法、预测、优化、模型和 Functions 形成决策逻辑层。Ontology 的作用不是强制所有逻辑都运行在同一个引擎中，而是提供一致的上下文和接口把异构逻辑资产连接起来。

#### Action

Action Type 把现实中的“动词”建模为受控、可审计、可执行的操作。例如批准订单、重新分配资源、修改工单状态、写回 SAP 等。

#### Security

安全不是外围 ACL，而是贯穿 Object、Property、Action、Function、写回和 Agent 工具调用的运行时约束。

### 5. 生命周期与治理

Ontology 不是一次性生成的静态模型。Palantir 通过 Ontology Manager、branching/change management、status metadata、依赖视图和 Marketplace 等能力支持持续演进。

在产品层面，应至少考虑：

- draft/experimental/active/deprecated 等生命周期信号；
- breaking change 检测；
- API name 稳定性；
- 上游 datasource 变更与 reindex 影响；
- 下游应用、Function、Action 的依赖；
- 分支、评审、发布、恢复；
- 共享 Ontology 的组织边界。

### 6. 运行时视角

Ontology 元数据由 Ontology Metadata Service 等后台组件维护；对象的查询、搜索、聚合由对象存储和 Object Set Service 等能力处理；Actions 负责受控写入；Functions on Objects 处理自定义逻辑。

这意味着自研平台需要把 **设计态 metadata** 与 **运行态 object data** 分离建模，但在用户体验上保持统一。

### 7. 设计建议

1. Ontology 必须成为业务 API 和应用的稳定契约，而不是底层表的镜像。
2. 类型、行为、安全、版本必须在同一治理框架下演进。
3. 不要把业务表一表一 Object Type 机械映射；应围绕真实业务实体、事件、决策对象建模。
4. Action 应与 Object/Link 同等重要，否则 Ontology 会退化为只读语义层。
5. API Name 和稳定标识符应与 Display Name 分离。

### 官方原始资料

- https://www.palantir.com/docs/foundry/ontologies/ontologies-overview
- https://www.palantir.com/docs/foundry/ontology/overview/
- https://www.palantir.com/docs/foundry/ontology/why-ontology
- https://www.palantir.com/docs/foundry/object-backend/overview

## 2. 自研平台产品定位
**元素类别：顶层容器 / Ontology Artifact。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 创建和治理独立 Ontology 空间
- 维护 Ontology 级元数据、组织边界、权限、版本和资源目录
- 为 Object/Link/Action/Interface 等资源提供统一命名空间与发布边界
- 支持 private/shared Ontology 的治理模型

### 2.2 非目标 / 边界
- 不在本模块直接编辑对象实例数据
- 不把 Ontology 当作单一数据库或图数据库
- 不在本模块实现具体 Object Type 字段编辑

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
1. **Ontology 列表/切换器**
2. **Ontology Overview 仪表盘**
3. **Resources 资源目录**
4. **Dependencies/Impact 依赖总览**
5. **Versions & Changes 发布与变更**
6. **Permissions & Organizations**
7. **Settings / Metadata**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| internalId | UUID/RID | 是 | 创建后不可变 | 全局唯一 | 平台内部资源主标识 |
| displayName | String(1..120) | 是 | 可编辑 | 同 Space 建议唯一 | 用户可读名称 |
| apiName | String | 是 | 受限编辑 | PascalCase/命名空间唯一/保留字校验 | SDK/API 契约 |
| description | Markdown | 否 | 可编辑 | ≤10000字符 | 说明业务范围、责任团队与边界 |
| visibility | Enum | 是 | 可编辑 | private/shared | 组织可见性模型 |
| spaceId | ID | 是 | 不可直接编辑 | 与 Ontology 1:1 | 绑定协作/权限空间 |
| organizations | ID[] | 条件必填 | 受权限编辑 | shared 时至少1个 | 允许访问的组织范围 |
| status | Enum | 是 | 可编辑 | experimental/active/deprecated 等 | 生命周期信号 |
| ownerTeam | Principal | 是 | 可编辑 | 必须有效 | 责任团队 |
| version | SemVer/Int | 是 | 系统维护 | 单调递增 | 发布版本 |
| createdAt/By | Audit | 是 | 只读 | 系统生成 | 审计 |
| updatedAt/By | Audit | 是 | 只读 | 系统生成 | 审计 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| ONT-FR-001 | P0 | 支持创建 private/shared Ontology，并在创建时绑定 Space/组织边界 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-002 | P0 | 支持在多个 Ontology 之间切换，切换后所有资源列表严格作用于当前 Ontology | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-003 | P0 | Overview 展示资源数量、状态分布、最近变更、失败索引、待处理 breaking change | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-004 | P0 | Resources 目录按资源类型、状态、Owner、Group、Tag 过滤 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-005 | P0 | 支持 Ontology 级草稿分支、变更集、评审、发布和回滚 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-006 | P0 | 发布前执行跨资源一致性校验和依赖检查 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-007 | P0 | 支持导出当前版本元数据快照 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-008 | P0 | 支持 Ontology 级命名规范与默认状态策略 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-009 | P0 | 支持 Owner/Builder/Viewer 等角色授权 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-010 | P0 | shared Ontology 中所有资源访问必须受组织边界约束 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-011 | P1 | 支持复制/克隆 Ontology 元数据到新环境但重新生成内部 ID | 服务端必须可验证；UI 需提供对应状态/反馈 |
| ONT-FR-012 | P1 | 支持软删除/归档，默认禁止直接物理删除仍有消费方的 Ontology | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| ONT-VAL-001 | Blocking | displayName 不能为空且去除首尾空格 |
| ONT-VAL-002 | Blocking | apiName 必须符合命名规则且在命名空间内唯一 |
| ONT-VAL-003 | Blocking | shared Ontology 必须至少配置一个 Organization |
| ONT-VAL-004 | Blocking | 存在 active 下游消费者时禁止无迁移计划的删除 |
| ONT-VAL-005 | Blocking | Ontology 与 Space 必须保持 1:1 关系 |
| ONT-VAL-006 | Blocking | 发布时不得存在 unresolved blocking validation |
| ONT-VAL-007 | Blocking | Owner 不得为空 |
| ONT-VAL-008 | Blocking | Deprecated Ontology 不允许新增 active 资源，除非 Owner 显式解除 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Draft -> Experimental -> Active -> Deprecated -> Archived
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Space/Organization
- IAM/Principal
- Ontology Resource Registry
- Change Management
- Audit Log
- Search Index
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
- `GET /ontologies`
- `POST /ontologies`
- `GET /ontologies/{id}`
- `PATCH /ontologies/{id}`
- `POST /ontologies/{id}:validate`
- `POST /ontologies/{id}:publish`
- `GET /ontologies/{id}/resources`
- `GET /ontologies/{id}/dependencies`
- `GET /ontologies/{id}/versions`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| ontology.created | 对应资源状态或定义发生变化时 |
| ontology.metadata.updated | 对应资源状态或定义发生变化时 |
| ontology.permission.changed | 对应资源状态或定义发生变化时 |
| ontology.validation.completed | 对应资源状态或定义发生变化时 |
| ontology.published | 对应资源状态或定义发生变化时 |
| ontology.deprecated | 对应资源状态或定义发生变化时 |
| ontology.archived | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| ONT_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| ONT_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| ONT_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| ONT_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| ONT_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| ONT_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| ONT-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持创建 private/shared Ontology，并在创建时绑定 Space/组织边界 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持在多个 Ontology 之间切换，切换后所有资源列表严格作用于当前 Ontology | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | Overview 展示资源数量、状态分布、最近变更、失败索引、待处理 breaking change | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | Resources 目录按资源类型、状态、Owner、Group、Tag 过滤 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 Ontology 级草稿分支、变更集、评审、发布和回滚 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | 发布前执行跨资源一致性校验和依赖检查 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持导出当前版本元数据快照 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 Ontology 级命名规范与默认状态策略 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| ONT-AC-009 | Validation | 用户提交违反规则：displayName 不能为空且去除首尾空格 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| ONT-AC-010 | Validation | 用户提交违反规则：apiName 必须符合命名规则且在命名空间内唯一 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| ONT-AC-011 | Validation | 用户提交违反规则：shared Ontology 必须至少配置一个 Organization | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| ONT-AC-012 | Validation | 用户提交违反规则：存在 active 下游消费者时禁止无迁移计划的删除 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| ONT-AC-013 | Validation | 用户提交违反规则：Ontology 与 Space 必须保持 1:1 关系 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| ONT-AC-014 | Validation | 用户提交违反规则：发布时不得存在 unresolved blocking validation | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/object-backend/overview
- https://www.palantir.com/docs/foundry/ontologies/ontologies-overview
- https://www.palantir.com/docs/foundry/ontology/overview/
- https://www.palantir.com/docs/foundry/ontology/why-ontology

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
