# Base Type — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Base Type |
| 元素分类 | Ontology 数据类型系统基础能力 |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Base Type 定义 Property 中可以存储的数据种类，并影响应用对该 Property 能执行的操作。它类似编程语言中的 primitive/structural type，是 Ontology Property 类型系统的底层基础。

Base Type 与 Value Type 不同：

- Base Type：技术数据形态；
- Value Type：基于 Base Type 的业务语义和约束。

### 2. 常用 Base Types

官方 Property 文档覆盖：

#### 标量类型

- String
- Boolean
- Byte / Short / Integer / Long
- Float / Double / Decimal
- Date / Timestamp

#### 集合/结构

- Array
- Struct

#### 搜索与 AI

- Vector：用于语义搜索等向量场景。

#### 地理空间

- Geopoint
- Geoshape

#### 文件/媒体

- Attachment
- Media Reference

#### 时间相关高级类型

- Time Series
- Geotemporal Series

#### 安全/特殊类型

- Marking
- Cipher 等。

### 3. Primary Key / Title Key 兼容性

不是所有 Base Type 都能作为 key。

例如：

- String、Integer、Short 可作为 Primary Key/Title Key；
- Date/Timestamp 技术上可做主键但官方不推荐，容易产生碰撞或表示差异；
- Float/Double/Decimal 不可作为 Primary Key；
- Vector、Struct、Attachment、Time Series 等不可作为 key；
- Array 可在有限条件下作 Title，但不可作 Primary Key。

因此建模工具应把 key compatibility 作为 Base Type schema 的元数据，而不是写死在页面里。

### 4. Base Type 决定应用能力

Base Type 会决定：

- 可用 filter operators；
- aggregation；
- formatting；
- sorting/searching；
- Action parameter type；
- Value Type 可使用的 constraints；
- UI widget 能否使用该 Property。

### 5. Array / Struct 注意点

官方指出：

- Array 不能包含 null elements；
- OSv2 不支持 nested arrays；
- Struct 不支持任意嵌套，且字段能力存在限制。

自研类型系统应显式描述嵌套规则与 storage compatibility。

### 6. JavaScript 与 Long

Palantir 特别提醒 Long 在 JavaScript 中存在大整数精度/表示问题，因此很多场景更适合使用 String 作为稳定标识。这说明 Ontology 类型选择不仅取决于数据库，也取决于 SDK/应用生态。

### 7. 与 Value Type 的关系

Value Type 必须基于 Base Type。Constraint 的合法集合由 Base Type 决定。例如 String 支持 Regex，数值支持 Range，Array 支持 uniqueness/nested constraints。

### 8. 自研建议

- 建立统一 Type Registry，记录 storage、SDK、UI、filter、key、constraint 能力。
- 类型兼容性由 schema 驱动，不散落在代码分支。
- 对跨语言不安全类型提供 lint，例如 Long、Timestamp primary key。
- Base Type 变更视为潜在 breaking change。

### 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/base-types
- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/value-type-constraints

## 2. 自研平台产品定位
**元素类别：Ontology 数据类型系统基础能力。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 定义 Property/Parameter 等可存储的数据形态
- 驱动可用运算、过滤、格式、Key兼容性、索引能力和 UI 控件
- 维护平台级 capability matrix
- 为高级类型提供一致 schema

### 2.2 非目标 / 边界
- Base Type 不是用户可随意创建的 Ontology Resource
- 不承载领域语义（由 Value Type/Property 描述承担）
- 不允许页面散落硬编码兼容规则

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
1. **Admin/Developer > Base Type Catalog**
2. **Type Detail**
3. **Capability Matrix**
4. **Key Compatibility**
5. **Constraint Compatibility**
6. **Serialization/API Mapping**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| typeId | Enum/ID | 是 | 平台内置不可变 | 唯一 | 类型标识 |
| displayName | String | 是 | 平台维护 | — | 显示 |
| category | Enum | 是 | 平台维护 | scalar/array/struct/geo/media/time/security/ai | 分类 |
| keyCompatibility | Object | 是 | 平台维护 | schema | PK/Title支持 |
| searchCapabilities | Object | 是 | 平台维护 | schema | filter/sort/search/aggregate |
| formatCapabilities | Object | 是 | 平台维护 | schema | 格式 |
| constraintCapabilities | ConstraintKind[] | 是 | 平台维护 | 合法 | Value Type支持 |
| serialization | Object | 是 | 平台维护 | API兼容 | JSON/SDK表示 |
| storageMapping | Object | 是 | 平台维护 | 后端兼容 | 物理类型 |
| deprecated | Boolean | 是 | 平台维护 | — | 生命周期 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 14-FR-001 | P0 | 平台维护统一 Base Type registry | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-002 | P0 | Property/Parameter/Value Type 所有编辑器从 registry 动态读取能力 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-003 | P0 | 提供 scalar、Array、Struct、Vector、Geo、Attachment/Media、Time Series 等支持矩阵 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-004 | P0 | Key 选择器根据 compatibility 自动禁用不合法类型 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-005 | P0 | Filter/Sort/Aggregation UI 根据能力矩阵生成 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-006 | P0 | Value Type Constraint Builder 根据 compatibility 生成 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-007 | P0 | API schema/SDK generator 使用统一 serialization mapping | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-008 | P0 | 后台可逐版本扩展新 Base Type，而不要求前端发版硬编码（通过 capability API） | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 14-FR-009 | P0 | 标记 deprecated 类型并给出迁移目标 | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 14-VAL-001 | Blocking | registry schema 必须自洽 |
| 14-VAL-002 | Blocking | 每个 Base Type 必须定义 serialization |
| 14-VAL-003 | Blocking | 允许做 Key 的类型必须有稳定 equality/hash 语义 |
| 14-VAL-004 | Blocking | Vector/Struct/Media 等高级类型禁止误用为 PK |
| 14-VAL-005 | Blocking | constraint matrix 不得声明不存在的 constraint kind |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Supported -> Deprecated -> Experimental
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Property
- Value Type
- Action Parameter
- Query Engine
- Index Engine
- SDK Generator
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
- `GET /type-system/base-types`
- `GET /type-system/base-types/{id}`
- `GET /type-system/capabilities`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| baseType.capability.added | 对应资源状态或定义发生变化时 |
| baseType.deprecated | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 14_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 14_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 14_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 14_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 14_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 14_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 14-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 平台维护统一 Base Type registry | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | Property/Parameter/Value Type 所有编辑器从 registry 动态读取能力 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | 提供 scalar、Array、Struct、Vector、Geo、Attachment/Media、Time Series 等支持矩阵 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | Key 选择器根据 compatibility 自动禁用不合法类型 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | Filter/Sort/Aggregation UI 根据能力矩阵生成 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | Value Type Constraint Builder 根据 compatibility 生成 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | API schema/SDK generator 使用统一 serialization mapping | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 后台可逐版本扩展新 Base Type，而不要求前端发版硬编码（通过 capability API） | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 14-AC-009 | Validation | 用户提交违反规则：registry schema 必须自洽 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 14-AC-010 | Validation | 用户提交违反规则：每个 Base Type 必须定义 serialization | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 14-AC-011 | Validation | 用户提交违反规则：允许做 Key 的类型必须有稳定 equality/hash 语义 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 14-AC-012 | Validation | 用户提交违反规则：Vector/Struct/Media 等高级类型禁止误用为 PK | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 14-AC-013 | Validation | 用户提交违反规则：constraint matrix 不得声明不存在的 constraint kind | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/object-link-types/base-types
- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/value-type-constraints

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
