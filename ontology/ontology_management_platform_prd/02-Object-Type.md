# Object Type — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Object Type |
| 元素分类 | 一级 Ontology Resource |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义与定位

Object Type 是现实世界实体或事件的 **schema definition**。Object 是该类型的单个实例；Object Set 是同一类型的一组对象。

例如 `Employee` 是 Object Type，某个具体员工是 Object；`Flight` 也可以是 Object Type，某次具体航班是 Object。Palantir 明确允许实体与事件都成为 Object Type，因此设计时不应局限于“主数据实体”。

### 2. 核心职责

Object Type 同时承担四类职责：

1. **语义定义**：说明这个业务对象是什么。
2. **属性契约**：定义该对象拥有的 Properties。
3. **数据映射**：把 datasource 中的数据映射成对象实例。
4. **应用/API 契约**：为 Object Explorer、Workshop、Functions、Ontology SDK 等提供稳定类型接口。

### 3. 主要元数据

官方 Metadata reference 中的核心字段包括：

- **ID**：类型级唯一标识，用于平台内部及部分配置引用；创建后不应随意变化。
- **RID**：Foundry 自动生成的全局资源标识，常用于错误与资源追踪。
- **Display name / Plural display name**：面向最终用户的单数、复数名称。
- **Description**：给使用者解释类型业务含义。
- **Icon**：应用中的视觉标识。
- **Groups**：用于发现和分类。
- **API name**：代码/API 使用的稳定名称。
- **Visibility**：normal / prominent / hidden 等展示信号。
- **Status**：experimental、active、deprecated；Object Type 还可能使用 promoted。
- **Index status**：反映最近一次索引状态。
- **Writeback**：表示是否启用用户编辑/写回能力。

### 4. Properties、Primary Key 与 Title Key

一个 Object Type 由多个 Property 构成。

#### Primary Key

用于唯一标识对象实例。底层 datasource 中映射到主键的值必须能够唯一确定对象。主键选择会直接影响链接、API 访问、Action 编辑和索引，因此属于高风险 schema 决策。

#### Title Key

用于最终用户界面显示对象名称。例如 Employee 可以把 `fullName` 作为 title key。Title Key 与 Primary Key 的目的不同：前者面向人，后者面向稳定唯一标识。

### 5. Datasource Mapping

创建 Object Type 时，Palantir 推荐通过引导式流程选择 backing datasource。选择数据源后，可自动将列映射为 Properties，再由建模者删除不需要的映射或修正元数据。

Object Type 也可以在没有现成数据源时创建；在支持的存储模式中，可以先创建权限所需的数据集，再逐步补充数据。

产品实现时建议把以下对象分开：

- Object Type metadata
- Datasource reference
- Datasource field → Property mapping
- Primary/title key mapping
- Index/materialization state

### 6. 创建流程

官方创建向导大致覆盖：

1. 选择或创建 datasource；
2. 设置 Object Type metadata；
3. 创建/映射 Properties；
4. 配置 Primary Key / Title Key；
5. 可选生成 Actions；
6. 选择保存位置；
7. 保存到 Ontology。

这说明 Object Type 的产品创建体验不应只是一张“类型表单”，而应把数据接入、属性、键、行为一起串联。

### 7. 编辑与 Breaking Change

Palantir 特别警告 Object Type 和 Property 的修改可能破坏下游应用。高风险变化包括：

- 替换 backing datasource；
- 修改 primary key；
- 删除 Object Type；
- 删除被应用引用的 Property；
- 修改 active 资源的 API 契约。

部分变更会触发重新索引，旧 Object Storage 模式下甚至可能导致对象短时不可用。自研平台需要有依赖分析和变更预览，而不能直接保存。

### 8. 编辑与 Writeback

Object Type 可以支持用户通过 Action 编辑。OSv2 中可以直接启用编辑能力；旧 OSv1 依赖 writeback dataset。无论底层实现怎样，产品层应区分：

- source-of-truth 数据；
- 用户/Agent 产生的 edits；
- 合并后的当前对象状态；
- edit history / provenance。

### 9. 安全

Object Type 的可见性与数据读取权限不是同一个概念。运行时还可能通过 Object Security Policy 对对象实例做 row-level filtering，并通过 Property Security Policy 做 column/cell-level 控制。

因此需要至少区分：

- Resource permission：谁能看/编辑 Object Type 定义；
- Object data permission：谁能看哪些对象；
- Property data permission：谁能看哪些字段；
- Action permission：谁能修改对象。

### 10. 与其他元素关系

- Property：描述对象特征。
- Link Type：把对象连接到其他对象。
- Action Type：创建、修改、删除对象或改变与其他对象的关系。
- Interface：Object Type 可以实现一个或多个 Interface。
- Object Type Group：用于分类、发现。
- Functions：可读取、搜索、聚合、遍历和编辑对象。

### 11. 设计建议

- 一个 Object Type 应对应一个稳定的业务概念，而不是某张物理表。
- Primary Key 应稳定、不可变、避免时间戳/浮点数等脆弱主键。
- Display Name 可业务化，API Name 必须稳定且适合代码使用。
- active/promoted 后应提高 breaking change 门槛。
- 先设计“对象如何被消费”，再决定哪些底层字段暴露为 Property。

### 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/object-types-overview
- https://www.palantir.com/docs/foundry/object-link-types/create-object-type/
- https://www.palantir.com/docs/foundry/object-link-types/edit-object-type
- https://www.palantir.com/docs/foundry/object-link-types/object-type-metadata
- https://www.palantir.com/docs/foundry/object-link-types/allow-editing

## 2. 自研平台产品定位
**元素类别：一级 Ontology Resource。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 定义现实世界实体或事件的 schema
- 统一 Property、Keys、Datasource Mapping、Capabilities 和应用/API 契约
- 支持对象类型从草稿到正式发布的全生命周期治理
- 在变更时保护下游应用、Action、Link、Function

### 2.2 非目标 / 边界
- 不把底层数据表机械等同于 Object Type
- 不在管理平台中承担大规模对象实例查询引擎
- 不允许绕过版本治理直接修改 active 类型的高风险契约

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
1. **Object Types 列表**
2. **Object Type Overview**
3. **Properties**
4. **Datasources & Mappings**
5. **Links**
6. **Actions**
7. **Interfaces**
8. **Capabilities**
9. **Security**
10. **Dependencies**
11. **Versions & Audit**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| internalId/RID | ID | 是 | 不可变 | 全局唯一 | 内部标识 |
| displayName | String | 是 | 可编辑 | 1..120字符 | 单数显示名 |
| pluralDisplayName | String | 是 | 可编辑 | 1..120字符 | 复数显示名 |
| apiName | String | 是 | 受限编辑 | PascalCase，Ontology 内唯一 | 代码契约 |
| description | Markdown | 否 | 可编辑 | 建议≥20字符 | 业务定义 |
| icon | IconRef | 否 | 可编辑 | 合法图标 | 视觉标识 |
| status | Enum | 是 | 可编辑 | experimental/active/deprecated/promoted | 生命周期 |
| visibility | Enum | 是 | 可编辑 | normal/prominent/hidden | 应用展示提示 |
| primaryKeyPropertyId | PropertyRef | 是 | 高风险编辑 | 必须兼容 PK 且唯一非空 | 对象唯一标识 |
| titleKeyPropertyId | PropertyRef | 否 | 可编辑 | 必须兼容 Title Key | 人类可读标题 |
| groups | GroupRef[] | 否 | 可编辑 | 引用存在 | 发现分类 |
| datasources | DatasourceMapping[] | 条件必填 | 受控编辑 | 至少一个可产出对象数据的 mapping，edit-only 模式除外 | 对象实例来源 |
| writebackEnabled | Boolean | 是 | 可编辑 | 与存储/Action能力兼容 | 是否允许编辑 |
| interfaceImplementations | InterfaceRef[] | 否 | 受控编辑 | 必须满足 interface shape | 多态契约 |
| capabilities | Object | 否 | 可编辑 | 按 schema 校验 | 搜索/展示/高级能力 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 02-FR-001 | P0 | 列表支持按 Status、Group、Owner、Datasource、Interface、Writeback、Index Status 过滤 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-002 | P0 | 创建向导支持“从 Datasource 创建”和“无现成 Datasource 创建”两种路径 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-003 | P0 | 选择 Datasource 后可自动发现字段并生成 Property 草稿映射 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-004 | P0 | 创建时必须显式选择 Primary Key；若无法证明唯一性则阻止发布 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-005 | P0 | 支持配置 Title Key，预览典型对象在人机界面中的展示效果 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-006 | P0 | Properties 页显示字段类型、API Name、Key、Search/Sort、Value Type、安全策略和映射状态 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-007 | P0 | Datasources 页显示数据源、主键映射、刷新/索引状态、字段漂移告警 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-008 | P0 | Links/Actions/Interfaces 页展示所有依赖并允许跳转 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-009 | P0 | Capabilities 页集中配置历史 type-class/应用能力 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-010 | P0 | 编辑 active Object Type 时实时计算 breaking change 风险 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-011 | P1 | 删除/修改 Property、Primary Key、API Name 时显示受影响的应用/API/Action/Link/Function | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-012 | P1 | 支持一键复制类型定义用于新版本迁移，但新类型获得新内部 ID/API Name | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-013 | P1 | 支持数据样例预览和 PK 唯一性/空值率检查 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-014 | P1 | 保存草稿不触发生产索引；发布后触发 metadata + index rollout | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-015 | P2 | 支持索引失败状态、失败原因、重试入口 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 02-FR-016 | P2 | 支持生成/查看 SDK/API schema 预览 | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 02-VAL-001 | Blocking | Primary Key 必须引用本类型 Property |
| 02-VAL-002 | Blocking | Primary Key Base Type 必须在允许集合内 |
| 02-VAL-003 | Blocking | Primary Key 不允许 nullable，发布前数据唯一性校验必须通过 |
| 02-VAL-004 | Blocking | Title Key 不能引用不兼容类型 |
| 02-VAL-005 | Blocking | API Name 在 Ontology 内唯一且 active 后默认锁定 |
| 02-VAL-006 | Blocking | 删除 Property 前必须确认不再被 Link/Action/Function/Security/Interface 使用 |
| 02-VAL-007 | Blocking | 实现 Interface 时必须满足所需 Shared/Interface Properties 和能力约束 |
| 02-VAL-008 | Blocking | Datasource mapping 中每个 Property 至多一个有效主来源（除明确合并策略） |
| 02-VAL-009 | Blocking | promoted 状态只允许 Owner/Steward 设置 |
| 02-VAL-010 | Blocking | writebackEnabled=true 时必须存在兼容的 Action/存储策略 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Draft -> Experimental -> Active -> Promoted -> Deprecated -> Archived
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Datasource Catalog
- Property
- Link Type
- Action Type
- Interface
- Security Policies
- Index Service
- Ontology SDK Registry
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
- `GET /object-types`
- `POST /object-types`
- `GET /object-types/{id}`
- `PATCH /object-types/{id}`
- `POST /object-types/{id}:validate`
- `POST /object-types/{id}:publish`
- `GET /object-types/{id}/dependencies`
- `GET /object-types/{id}/schema`
- `POST /object-types/{id}:reindex`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| objectType.created | 对应资源状态或定义发生变化时 |
| objectType.property.changed | 对应资源状态或定义发生变化时 |
| objectType.mapping.changed | 对应资源状态或定义发生变化时 |
| objectType.key.changed | 对应资源状态或定义发生变化时 |
| objectType.validation.failed | 对应资源状态或定义发生变化时 |
| objectType.published | 对应资源状态或定义发生变化时 |
| objectType.reindex.started | 对应资源状态或定义发生变化时 |
| objectType.reindex.failed | 对应资源状态或定义发生变化时 |
| objectType.deprecated | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 02_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 02_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 02_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 02_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 02_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 02_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 02-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | 列表支持按 Status、Group、Owner、Datasource、Interface、Writeback、Index Status 过滤 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | 创建向导支持“从 Datasource 创建”和“无现成 Datasource 创建”两种路径 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | 选择 Datasource 后可自动发现字段并生成 Property 草稿映射 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | 创建时必须显式选择 Primary Key；若无法证明唯一性则阻止发布 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持配置 Title Key，预览典型对象在人机界面中的展示效果 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | Properties 页显示字段类型、API Name、Key、Search/Sort、Value Type、安全策略和映射状态 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | Datasources 页显示数据源、主键映射、刷新/索引状态、字段漂移告警 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | Links/Actions/Interfaces 页展示所有依赖并允许跳转 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 02-AC-009 | Validation | 用户提交违反规则：Primary Key 必须引用本类型 Property | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 02-AC-010 | Validation | 用户提交违反规则：Primary Key Base Type 必须在允许集合内 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 02-AC-011 | Validation | 用户提交违反规则：Primary Key 不允许 nullable，发布前数据唯一性校验必须通过 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 02-AC-012 | Validation | 用户提交违反规则：Title Key 不能引用不兼容类型 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 02-AC-013 | Validation | 用户提交违反规则：API Name 在 Ontology 内唯一且 active 后默认锁定 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 02-AC-014 | Validation | 用户提交违反规则：删除 Property 前必须确认不再被 Link/Action/Function/Security/Interface 使用 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/object-link-types/allow-editing
- https://www.palantir.com/docs/foundry/object-link-types/create-object-type/
- https://www.palantir.com/docs/foundry/object-link-types/edit-object-type
- https://www.palantir.com/docs/foundry/object-link-types/object-type-metadata
- https://www.palantir.com/docs/foundry/object-link-types/object-types-overview

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
