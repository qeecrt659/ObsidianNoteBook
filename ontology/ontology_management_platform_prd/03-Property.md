# Property — PRD 级产品需求说明书

> **目标**：本文件用于直接指导 Ontology 管理平台的产品评审、交互设计、架构设计、接口设计、开发拆解和验收。
> 设计基线来自 Palantir Foundry / Ontology 官方文档；“自研产品要求”属于在官方概念基础上的工程化产品设计，不代表 Palantir 官方 UI 的逐字复刻。

## 0. 文档控制
| 字段 | 值 |
| --- | --- |
| 文档版本 | v1.0-PRD |
| 整理日期 | 2026-09-05 |
| 产品模块 | Property |
| 元素分类 | Object Type 关键组成元素 |
| 目标读者 | 产品经理、Ontology 架构师、前端、后端、测试、安全、FDE |
| 优先级说明 | P0=首版必须；P1=第二阶段；P2=增强 |
| 规范原则 | 术语优先与 Palantir 官方一致；自研扩展必须显式标记 |

## 1. Palantir 官方产品语义基线
### 1. 定义

Property 是 Object Type 中某个现实世界特征的 schema definition；Property Value 是某个具体 Object 上该特征的实际值。官方把 Property 与数据集列类比：Property 类似列的语义定义，Property Value 类似具体行中的字段值。

### 2. 产品作用

Property 不只是“字段”。它同时决定：

- 对象可携带什么数据；
- 该数据的 Base Type / Value Type；
- 搜索、排序、聚合、渲染等应用能力；
- 是否承担 Primary Key / Title Key；
- 如何映射底层 datasource；
- 是否可编辑；
- 是否受独立安全策略保护。

### 3. 主要元数据

官方 Metadata reference 的关键内容包括：

- ID
- RID
- Display Name
- Description
- Status
- API Name
- Base Type
- Keys：Primary Key / Title Key
- Visibility
- Value formatting
- Render hints
- Type classes
- 数据映射信息

在产品模型中，建议把“语义元数据”“数据类型”“展示配置”“索引能力”“数据映射”“安全配置”分成独立子结构，而不要堆在一个 JSON 中。

### 4. Base Type

Property Base Type 定义 Property 可以存储什么数据，以及用户应用能够进行什么操作。官方支持常见的 String、Integer、Short、Date、Timestamp、Boolean、Byte、Long、Float、Double、Decimal，也支持 Array、Struct、Vector、Geopoint、Geoshape、Attachment、Time Series、Geotemporal Series、Media Reference、Marking、Cipher 等扩展类型。

一些类型不适合作为 Primary Key；Vector、Struct、附件、时间序列等也不能作为 Title Key。设计器应基于 Base Type 动态限制合法配置。

### 5. Primary Key 与 Title Key

#### Primary Key

Property 可被指定为对象主键，用于唯一确定对象。该字段的稳定性和唯一性直接影响 Link 与 Action。

#### Title Key

用于在用户应用中显示对象的人类可读名称。通常应选择可读性强且相对稳定的字段。

自研平台应把两者明确分开，不能把“显示名称”默认当作主键。

### 6. Datasource Mapping

常规 Property 由 Object Type 的 backing datasource 中某一列提供值。映射变化会影响对象索引和下游应用。

Palantir 还支持 **Edit-only Property**：不直接映射到 backing dataset 的列，主要承载通过 Ontology 编辑产生的数据。在 OSv2 中，这允许先定义业务字段，再通过 Action 持久化用户输入，而不要求先改底层源表。

### 7. Value Formatting 与 Display

Property 可以配置数值、日期时间、User ID、Resource ID 等格式，使原始值在用户应用中更可读。Visibility 可提示应用优先或隐藏展示。

这类配置属于 Ontology 的“语义 UI 契约”，意味着应用不需要重复定义每一个字段的基本展示规则。

### 8. Render Hints 与性能

Render hints 会影响应用能力和索引，例如 searchable、sortable 等。官方文档明确指出，关闭不需要的搜索/排序能力可以改善 reindex 性能。

所以在自研平台里，不能假设所有字段都应建立全文索引/排序索引。Property 应携带“查询能力声明”。

### 9. Required / Edit-only / Derived 等扩展能力

官方 Property 文档还覆盖：

- Required Properties：用于表达对象字段的必需性；
- Edit-only Properties：只由 Ontology edits 提供值；
- Derived Properties：运行时或查询层派生；
- Mandatory Control Properties：用于把 markings、organization、classification 等安全控制关联到对象数据。

这些能力说明 Property 是运行时数据治理单元，而不是简单 schema column。

### 10. Property Security

Property Security Policy 可以在 Object Security Policy 的基础上进一步限制字段级可见性。用户通过 Object policy 但未通过 Property policy 时，可以看到对象但对应字段值会被隐藏/置空。

主键 Property 不能加入 Property Security Policy，因为对象标识需要支持对象访问和权限评估。

### 11. 变更风险

删除 Property 可能破坏 Object Views、Workshop、Functions、SDK 代码和 Action。active Property 不能直接删除。API Name 一旦作为程序契约使用，应保持稳定。

### 12. 设计建议

- Property 名称表达业务语义，不要照搬数据库缩写。
- API Name 与 Display Name 分离。
- 对可枚举、范围、正则等业务含义强的字段优先引入 Value Type。
- searchable/sortable 按真实用例开启。
- 对敏感字段单独配置安全策略，而不是只依赖 UI 隐藏。
- 把来源映射与 Property 定义解耦，为多数据源和 schema evolution 留空间。

### 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/property-metadata
- https://www.palantir.com/docs/foundry/object-link-types/edit-properties
- https://www.palantir.com/docs/foundry/object-link-types/edit-only-properties
- https://www.palantir.com/docs/foundry/object-link-types/base-types
- https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies

## 2. 自研平台产品定位
**元素类别：Object Type 关键组成元素。** 本模块在自研 Ontology 管理平台中负责把 Palantir 的相关语义落实为可治理、可版本化、可审计、可通过 API 管理的产品能力。

### 2.1 产品目标
- 定义对象特征的业务语义与技术类型
- 控制字段的 API、搜索、排序、格式、Key、Value Type 和安全能力
- 建立 datasource field 到 Ontology property 的稳定映射
- 支持普通、edit-only、共享/引用等属性模式

### 2.2 非目标 / 边界
- Property Visibility 不作为权限控制
- 不允许把所有字段默认设置为 Searchable/Sortable
- 不在 Property 模块单独决定对象行级安全

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
1. **Object Type > Properties 列表**
2. **Property Detail/Drawer**
3. **Data Mapping**
4. **Type & Validation**
5. **Display & Capabilities**
6. **Security**
7. **Usage/Dependencies**
8. **Change History**

### 5.1 通用详情页布局
- 顶部：Display Name、API Name、Status、Owner、版本、保存状态、发布状态。
- 左侧或页签：Overview / Definition / Usage / Dependencies / Security / Versions / Audit。
- 右侧固定区域：Validation、Breaking Change、Unsaved Changes、快捷跳转。
- active 资源发生高风险变更时，在页面顶部持续显示风险 Banner，直到变更被撤销或进入迁移流程。

## 6. 领域数据模型与字段字典
| 字段 | 类型 | 必填 | 可编辑性 | 校验/约束 | 产品含义 |
| --- | --- | --- | --- | --- | --- |
| internalId | ID | 是 | 不可变 | 类型内唯一 | 内部标识 |
| displayName | String | 是 | 可编辑 | 1..120 | 业务名称 |
| apiName | String | 是 | 受限编辑 | camelCase，类型内唯一 | 代码字段名 |
| description | Markdown | 否 | 可编辑 | ≤5000 | 字段口径 |
| baseType | Enum/Schema | 是 | 高风险编辑 | 必须为支持类型 | 技术类型 |
| valueTypeId | ValueTypeRef | 否 | 受控编辑 | Base Type 兼容 | 业务语义/验证 |
| nullable | Boolean | 是 | 高风险编辑 | PK=false | 空值约束 |
| keyRole | Enum | 是 | 高风险编辑 | none/primary/title | Key角色 |
| mapping | FieldMapping | 条件必填 | 受控编辑 | 字段类型兼容 | 数据来源 |
| editOnly | Boolean | 是 | 受控编辑 | 与存储后端兼容 | 仅编辑持久化属性 |
| searchable | Boolean | 是 | 可编辑 | Base Type/后端支持 | 建立搜索索引 |
| sortable | Boolean | 是 | 可编辑 | Base Type/后端支持 | 排序能力 |
| valueFormatting | Object | 否 | 可编辑 | 按 Base Type 校验 | 显示格式 |
| visibility | Enum | 是 | 可编辑 | prominent/normal/hidden | 应用展示 |
| renderHints | Object | 否 | 可编辑 | schema 校验 | 应用/索引提示 |
| status | Enum | 是 | 可编辑 | experimental/active/deprecated | 生命周期 |


### 6.1 标识符规则
- `internalId/RID`：系统生成、不可变，只用于内部引用和审计。
- `apiName`：程序化契约，必须稳定、可生成 SDK；active 后变更按 breaking change 处理。
- `displayName`：面向业务用户，可重命名；不得作为下游代码唯一引用。

## 7. 功能需求（Functional Requirements）
| 需求ID | 优先级 | 需求说明 | 验收原则 |
| --- | --- | --- | --- |
| 03-FR-001 | P0 | Properties 表格支持批量编辑 displayName/description/status/visibility/searchable/sortable | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-002 | P0 | 新建 Property 时根据 Base Type 动态展示允许的格式、Key、Constraint、Widget 能力 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-003 | P0 | 支持将普通 Property 转换为 Shared Property，并先执行兼容性与依赖检查 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-004 | P0 | 支持绑定 Value Type，并展示其 constraint 与版本 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-005 | P0 | 支持 datasource field 自动映射、手工重映射和 drift 检测 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-006 | P0 | 支持 edit-only property 模式，并清晰标识值来源 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-007 | P0 | 修改 Base Type 时提供兼容性矩阵与数据转换风险 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-008 | P0 | 配置 Searchable/Sortable 时预估索引成本并提示 reindex | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-009 | P0 | 允许设置 Primary Key/Title Key，但高风险操作需要 Object Type 级验证 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-010 | P0 | 支持 Property-level security policy 入口和当前策略展示 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-011 | P1 | 支持格式预览：数字、日期、资源 ID、用户 ID 等 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-012 | P1 | 支持查看 Property 被哪些 Actions、Links、Functions、Interfaces、Views 使用 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-013 | P1 | 批量导入字段时提供重复 API Name、类型冲突、不可映射字段报告 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-014 | P1 | active Property 删除必须走 deprecate→迁移→删除流程 | 服务端必须可验证；UI 需提供对应状态/反馈 |
| 03-FR-015 | P2 | 发布后若索引失败应定位到具体 Property/Value Type constraint | 服务端必须可验证；UI 需提供对应状态/反馈 |


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
| 03-VAL-001 | Blocking | apiName 必须 camelCase 且类型内唯一 |
| 03-VAL-002 | Blocking | Primary Key 不得 nullable |
| 03-VAL-003 | Blocking | Primary Key 不允许不兼容 Base Type |
| 03-VAL-004 | Blocking | Value Type 的 Base Type 必须与 Property Base Type 一致/兼容 |
| 03-VAL-005 | Blocking | Searchable/Sortable 仅允许后端支持的类型 |
| 03-VAL-006 | Blocking | Property Security Policy 不得包含 Primary Key |
| 03-VAL-007 | Blocking | editOnly=true 时不得要求普通 datasource field mapping |
| 03-VAL-008 | Blocking | 修改 Base Type 前检查已有数据可转换性 |
| 03-VAL-009 | Blocking | active API Name 修改默认阻断 |
| 03-VAL-010 | Blocking | 删除前 usageCount 必须为0或存在批准的迁移计划 |

### 10.1 校验结果模型
- `BLOCKING`：禁止保存/发布/执行。
- `WARNING`：允许保存草稿；发布时需显式确认或审批。
- `INFO`：最佳实践、性能、命名或治理建议。

## 11. 生命周期与状态机
```text
Draft -> Experimental -> Active -> Deprecated -> Removed
```
- 状态变更与内容变更分开授权。
- `Active` 表示已成为稳定消费契约，系统必须增强 breaking change 保护。
- `Deprecated` 必须允许填写 replacement resource、sunset date、migration guide。
- `Archived/Removed` 默认不可再被新资源引用，但历史版本和审计仍可读取。

## 12. 依赖关系与 Impact Analysis
### 12.1 直接依赖
- Object Type
- Datasource Field
- Value Type
- Shared Property
- Security Policy
- Index Service
- Actions
- Interfaces
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
- `GET /object-types/{ot}/properties`
- `POST /object-types/{ot}/properties`
- `PATCH /properties/{id}`
- `POST /properties/{id}:validate`
- `POST /properties/{id}:convert-to-shared`
- `GET /properties/{id}/usage`
- `GET /properties/{id}/data-profile`
### 15.1 API 通用约定
- 写 API 接收 `expectedVersion`，用于 optimistic concurrency control。
- 所有 mutation 支持 `requestId/idempotencyKey`。
- PATCH 返回 updated resource + validation summary + change risk。
- 404 不应泄露用户无权限资源是否真实存在；按安全策略返回 403/404。
- 发布 API 必须是事务性的 change-set publish，而非逐资源 best-effort。

## 16. 领域事件与审计
| 事件 | 触发时机 |
| --- | --- |
| property.created | 对应资源状态或定义发生变化时 |
| property.type.changed | 对应资源状态或定义发生变化时 |
| property.mapping.changed | 对应资源状态或定义发生变化时 |
| property.capability.changed | 对应资源状态或定义发生变化时 |
| property.valueType.changed | 对应资源状态或定义发生变化时 |
| property.security.changed | 对应资源状态或定义发生变化时 |
| property.deprecated | 对应资源状态或定义发生变化时 |
| property.deleted | 对应资源状态或定义发生变化时 |

审计记录最少字段：`auditId, actor, timestamp, ontologyId, resourceKind, resourceId, action, before, after, changeSetId, reason, client, traceId`。

## 17. 错误处理规范
| 错误码示例 | 场景 | 用户提示 |
| --- | --- | --- |
| 03_VALIDATION_FAILED | 字段/结构校验失败 | 指出字段、规则和修复方式 |
| 03_VERSION_CONFLICT | 并发修改冲突 | 提示重新加载并展示差异 |
| 03_BREAKING_CHANGE_BLOCKED | active 资源发生阻断性变更 | 展示消费者和迁移入口 |
| 03_DEPENDENCY_EXISTS | 删除仍有依赖 | 列出可见依赖和处理建议 |
| 03_PERMISSION_DENIED | 无权限 | 不泄露敏感资源细节 |
| 03_REFERENCE_NOT_FOUND | 引用失效 | 指出失效引用并提供重新选择入口 |


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
| 03-AC-001 | Happy Path | 已具备完成该功能所需权限和合法输入 | Properties 表格支持批量编辑 displayName/description/status/visibility/searchable/sortable | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-002 | Happy Path | 已具备完成该功能所需权限和合法输入 | 新建 Property 时根据 Base Type 动态展示允许的格式、Key、Constraint、Widget 能力 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-003 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持将普通 Property 转换为 Shared Property，并先执行兼容性与依赖检查 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-004 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持绑定 Value Type，并展示其 constraint 与版本 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-005 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 datasource field 自动映射、手工重映射和 drift 检测 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-006 | Happy Path | 已具备完成该功能所需权限和合法输入 | 支持 edit-only property 模式，并清晰标识值来源 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-007 | Happy Path | 已具备完成该功能所需权限和合法输入 | 修改 Base Type 时提供兼容性矩阵与数据转换风险 | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-008 | Happy Path | 已具备完成该功能所需权限和合法输入 | 配置 Searchable/Sortable 时预估索引成本并提示 reindex | 系统完成操作并持久化变更；审计日志可查询；刷新页面结果一致 |
| 03-AC-009 | Validation | 用户提交违反规则：apiName 必须 camelCase 且类型内唯一 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 03-AC-010 | Validation | 用户提交违反规则：Primary Key 不得 nullable | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 03-AC-011 | Validation | 用户提交违反规则：Primary Key 不允许不兼容 Base Type | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 03-AC-012 | Validation | 用户提交违反规则：Value Type 的 Base Type 必须与 Property Base Type 一致/兼容 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 03-AC-013 | Validation | 用户提交违反规则：Searchable/Sortable 仅允许后端支持的类型 | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |
| 03-AC-014 | Validation | 用户提交违反规则：Property Security Policy 不得包含 Primary Key | 点击保存/发布/执行 | 系统拒绝请求并返回明确 errorCode 与修复建议，不产生部分写入 |


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
- https://www.palantir.com/docs/foundry/object-link-types/edit-only-properties
- https://www.palantir.com/docs/foundry/object-link-types/edit-properties
- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/property-metadata
- https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies

## 25. 自研实现备注
本文中“建议 API、页面布局、审批流、错误码、SLA、Change Set”等属于为了把 Palantir 概念落地为可开发产品而补充的自研 PRD 设计。实现时应保持 Palantir 核心术语和语义不变，但不必机械复制其具体 UI。
