# Palantir Foundry Ontology Manager 功能详细说明

> 文档名称：Palantir Foundry Ontology Manager 功能详细说明  
> 文档版本：1.0  
> 整理日期：2026-08-04  
> 产品名称：Ontology Manager  
> 其他名称：Ontology Management Application，简称 OMA  
> 文档定位：官方能力梳理 + 产品功能分解  
> 参考基准：Palantir Foundry 官方公开文档

---

## 1. 文档目的

本文详细说明 Palantir Foundry 中 **Ontology Manager（OMA）** 的产品定位、组成模块、主要页面、管理对象、操作流程、权限模型、版本管理、数据映射、运行监控和治理能力。

本文可用于：

- 理解 Palantir Ontology Manager 的完整功能范围；
- 规划自研 Ontology 管理平台；
- 编写 Ontology 管理平台 PRD；
- 拆分前后端研发模块；
- 设计 Ontology 元模型；
- 设计版本、分支、审批和发布流程；
- 建立 Ontology 治理规范。

需要注意：

> 本文对 Palantir 官方公开能力进行了产品化归类。Palantir 并未公开 Ontology Manager 的完整内部技术实现，因此部分“产品模块划分”是为了便于理解和自研落地，并不代表 Palantir 内部代码或微服务的实际拆分方式。

---

# 2. Ontology Manager 是什么

Ontology Manager 是 Foundry 中用于**构建和维护组织 Ontology** 的核心管理应用。

它支持的主要工作包括：

- 创建和管理 Object Type；
- 定义对象属性 Property；
- 建立 Link Type；
- 定义 Action Type；
- 管理 Interface；
- 管理 Shared Property；
- 查看和管理已发布 Function；
- 将 Dataset、流和其他数据源映射到 Ontology；
- 检查对象数据和索引状态；
- 查看资源依赖和使用情况；
- 管理 Ontology 变更；
- 创建分支和 Proposal；
- 审查、合并和恢复变更；
- 管理 Ontology 资源权限；
- 清理废弃或未使用的 Object Type。

官方对 Ontology Manager 的定位可以概括为：

```text
构建 Ontology
+
维护 Ontology
+
连接真实数据
+
管理业务操作
+
安全变更
+
监控运行状态
```

---

# 3. Ontology Manager 在 Foundry 中的位置

```text
Palantir Foundry
├── 数据层
│   ├── Data Connection
│   ├── Dataset
│   ├── Pipeline Builder
│   └── Code Repositories
│
├── Ontology 层
│   ├── Ontology Manager
│   ├── Ontology Engine
│   ├── Object Storage
│   ├── Object Set Service
│   ├── Actions
│   └── Functions
│
├── Ontology 消费层
│   ├── Object Explorer
│   ├── Workshop
│   ├── Quiver
│   ├── OSDK
│   └── AIP / Agent
│
└── 平台治理层
    ├── Compass
    ├── Control Panel
    ├── Approvals
    ├── Global Branching
    └── Resource Management
```

Ontology Manager 主要负责 **Ontology 定义和配置**。

它不直接承担所有相关能力：

| 能力 | 主要负责组件 |
|---|---|
| Ontology 元数据定义 | Ontology Manager |
| 对象索引和查询运行 | Ontology Engine / Object Storage |
| 数据管道开发 | Pipeline Builder / Code Repositories |
| Function 代码开发 | Functions Code Repository |
| 平台用户和组织管理 | Control Panel |
| 项目和文件权限 | Compass |
| 对象查询和业务探索 | Object Explorer |
| 低代码业务应用 | Workshop |
| 跨资源分支统一管理 | Global Branching |
| 资源成本管理 | Resource Management |

---

# 4. Ontology Manager 总体功能架构

```text
Ontology Manager
├── 1. 工作台与导航
│   ├── Discover
│   ├── 全局搜索
│   ├── 资源筛选
│   ├── 收藏与最近访问
│   └── 分支切换
│
├── 2. Ontology 资源目录
│   ├── Object Types
│   ├── Link Types
│   ├── Action Types
│   ├── Interfaces
│   ├── Shared Properties
│   ├── Functions
│   └── Object Type Groups
│
├── 3. Object Type 管理
│   ├── 元数据
│   ├── Properties
│   ├── Primary Key
│   ├── Title Key
│   ├── Datasources
│   ├── Action Types
│   ├── Link Graph
│   ├── Capabilities
│   ├── Object Views
│   ├── Usage
│   └── Security
│
├── 4. Property 管理
│   ├── 基础类型
│   ├── 值格式化
│   ├── 条件格式化
│   ├── 搜索和排序
│   ├── 必填和编辑控制
│   ├── Derived Property
│   ├── Struct
│   └── Shared Property
│
├── 5. Link Type 管理
│   ├── 两端 Object Type
│   ├── 基数
│   ├── 正向和反向名称
│   ├── Datasource 映射
│   ├── 键映射
│   └── 使用和依赖
│
├── 6. Action Type 管理
│   ├── 参数
│   ├── 规则
│   ├── Submission Criteria
│   ├── 对象和关系编辑
│   ├── Function-backed Action
│   ├── Side Effects
│   ├── 权限
│   └── Observability
│
├── 7. Interface 管理
│   ├── Interface Property
│   ├── Link Constraints
│   ├── Action Constraints
│   ├── 继承
│   └── Object Type 实现
│
├── 8. Function 管理
│   ├── 搜索
│   ├── 版本
│   ├── 输入输出
│   ├── 配置
│   ├── 使用历史
│   ├── Observability
│   └── 跳转代码仓库
│
├── 9. 数据映射与索引
│   ├── Backing Datasource
│   ├── Multi-datasource
│   ├── Batch Indexing
│   ├── Streaming Indexing
│   ├── Direct Datasource
│   ├── Schema 状态
│   ├── Data Liveness
│   └── 错误诊断
│
├── 10. 变更与版本管理
│   ├── Working State
│   ├── Review Edits
│   ├── Errors / Warnings
│   ├── Save
│   ├── History
│   ├── Restore
│   ├── Merge Conflict
│   └── Import / Export
│
├── 11. 分支与审批
│   ├── Main
│   ├── Branch
│   ├── Protected Resources
│   ├── Proposal
│   ├── Review
│   ├── Approval
│   └── Merge
│
├── 12. 安全与权限
│   ├── Project-based Permissions
│   ├── Resource Permissions
│   ├── Object Instance Permissions
│   ├── Action Permissions
│   ├── Security Policies
│   └── 权限迁移
│
├── 13. 依赖、使用和影响分析
│   ├── Dependents
│   ├── Usage Graph
│   ├── Reads
│   ├── Writes
│   ├── Active Users
│   └── Application Usage
│
├── 14. 生命周期与治理
│   ├── Status
│   ├── Promoted
│   ├── Experimental
│   ├── Active
│   ├── Deprecated
│   ├── Example
│   ├── Point of Contact
│   └── Cleanup
│
└── 15. 发布和复用
    ├── Marketplace Product
    ├── Ontology Resource Packaging
    ├── API Name
    └── JSON Import / Export
```

---

# 5. 用户角色

Ontology Manager 常见用户角色如下。

## 5.1 Ontology Owner

主要职责：

- 规划整体 Ontology；
- 定义核心领域模型；
- 审核关键变更；
- 管理保护资源；
- 管理资源位置和权限；
- 决定资源状态；
- 推动废弃和清理；
- 处理跨领域冲突；
- 管理 Ontology 治理规则。

## 5.2 Ontology Editor

主要职责：

- 创建和编辑 Object Type；
- 维护 Property；
- 创建 Link Type；
- 创建 Action Type；
- 管理数据映射；
- 提交变更；
- 创建分支和 Proposal；
- 修复数据和索引问题。

## 5.3 Data Engineer

主要职责：

- 准备数据集；
- 建设批处理或流式 Pipeline；
- 确保 Schema 稳定；
- 将数据接入 Ontology；
- 处理索引失败；
- 保证数据新鲜度和质量。

## 5.4 Application Developer

主要职责：

- 使用 Workshop、OSDK 或其他应用消费 Ontology；
- 反馈 Object、Property、Link 和 Action 需求；
- 评估 Ontology 变更对应用的影响；
- 配合升级 Function 和 API。

## 5.5 Security Administrator

主要职责：

- 管理 Project 和组织权限；
- 管理 Markings 和数据访问；
- 配置 Ontology 管理权限；
- 配置对象实例权限；
- 审计敏感变更。

## 5.6 Reviewer / Approver

主要职责：

- 审查 Proposal；
- 检查 breaking change；
- 检查数据源和安全影响；
- 审核资源状态；
- 批准或拒绝合并。

---

# 6. 工作台、导航和资源发现

## 6.1 Discover 首页

Discover 是 Ontology Manager 的可定制首页。

默认可展示：

- 收藏的 Object Type；
- 最近查看的 Object Type；
- 收藏的 Object Type Group；
- 最近修改的 Object Type；
- Promoted Object Type；
- 指定 Group 下的资源。

### 主要价值

- 快速进入常用模型；
- 让新用户发现核心对象；
- 提高大型 Ontology 的导航效率；
- 减少依赖复杂菜单。

### 可配置内容

- 首页区块；
- 区块显示顺序；
- 每个区块的资源数量；
- 特定 Group 专区；
- 收藏和取消收藏。

---

## 6.2 顶部导航栏

顶部导航栏主要提供：

- 全局搜索；
- 新建 Ontology 资源；
- 当前 Ontology 显示；
- 当前分支显示；
- 创建和切换分支；
- 保存未提交变更；
- 查看错误和警告。

---

## 6.3 全局搜索

可搜索的资源通常包括：

- Object Type；
- Property；
- Link Type；
- Action Type；
- Shared Property；
- Interface；
- Function；
- Object Type Group。

### 可匹配的字段

- Display Name；
- API Name；
- Description；
- Alias；
- RID；
- 其他元数据。

### 搜索交互

- 结果高亮匹配字段；
- 键盘上下选择；
- 预览选中资源；
- Enter 直接打开；
- 按类型筛选。

---

## 6.4 资源列表和筛选

资源列表支持按以下条件筛选：

- 资源类型；
- 可见性；
- Status；
- Group；
- Indexing Issue；
- Security Setup；
- 是否有数据源；
- 是否存在错误；
- 是否被弃用；
- 是否为 Promoted。

### 资源列表建议字段

| 字段 | 说明 |
|---|---|
| Display Name | 业务显示名称 |
| API Name | SDK 和 API 中使用的稳定名称 |
| Resource Type | Object、Link、Action 等 |
| Status | Active、Experimental 等 |
| Group | 业务分类 |
| Point of Contact | 负责人 |
| Datasource | 数据来源 |
| Index Status | 索引状态 |
| Last Modified | 最近修改时间 |
| Modified By | 最近修改人 |
| Usage | 最近使用情况 |
| Security | 权限模式 |
| Issue | 当前问题 |

---

# 7. Object Type 管理

Object Type 是 Ontology 中对现实世界实体或事件的 Schema 定义。

例如：

- Customer；
- Order；
- Product；
- Equipment；
- Factory；
- Shipment；
- Alert；
- Maintenance Work Order。

---

## 7.1 创建 Object Type

创建 Object Type 通常使用引导式流程。

### 创建流程

```text
选择创建 Object Type
        ↓
选择 Backing Datasource
        ↓
填写 Object Type 元数据
        ↓
生成或配置 Properties
        ↓
配置 Primary Key
        ↓
配置 Title Key
        ↓
可选生成 Actions
        ↓
选择 Project 保存位置
        ↓
Review Changes
        ↓
Save
```

### 支持的创建方式

- 从已有 Dataset 创建；
- 从流式数据源创建；
- 从 Pipeline Builder 输出创建；
- 先创建空定义，再手动关联数据；
- 通过导入 Ontology JSON 创建；
- 通过 API 或产品安装创建。

---

## 7.2 Object Type 基础元数据

### 核心字段

| 字段 | 说明 |
|---|---|
| Display Name | 面向业务用户的名称 |
| Plural Display Name | 复数显示名称 |
| API Name | SDK、Function 和 API 中使用的名称 |
| Description | 业务定义和使用说明 |
| RID | Foundry 生成的唯一资源标识 |
| Icon | 图标 |
| Status | 开发和生命周期状态 |
| Aliases | 搜索和业务同义词 |
| Point of Contact | 责任人或责任组 |
| Group | 所属 Object Type Group |
| Project Location | 资源保存位置 |
| Visibility | 资源发现范围 |
| Primary Key | 对象唯一标识 |
| Title Key | 对象默认显示字段 |

### 管理要求

- API Name 应保持稳定；
- Display Name 可面向业务优化；
- Description 应说明业务定义，而不是数据库表说明；
- Primary Key 必须能稳定唯一识别对象；
- Title Key 应便于业务用户识别对象；
- 生产对象应明确 Point of Contact；
- 核心对象应设置适当状态。

---

## 7.3 Object Type Overview 页面

官方 Object Type Overview 通常包括以下区域：

1. Object Type Metadata；
2. Properties；
3. Action Types；
4. Link Type Graph；
5. Dependents；
6. Data；
7. Usage。

### 建议页面结构

```text
Object Type
├── Overview
├── Properties
├── Datasources
├── Links
├── Actions
├── Interfaces
├── Capabilities
├── Object Views
├── Usage
├── Security
├── History
└── Observability / Issues
```

---

## 7.4 Primary Key

Primary Key 用于唯一标识一个对象实例。

### 关键要求

- 必须唯一；
- 应保持稳定；
- 不应因为业务显示名称变化而变化；
- 必须与数据源字段映射；
- 多数据源 Object Type 必须保证统一身份；
- 不合理的 Primary Key 会造成对象重复或覆盖。

### 常见选择

- 业务系统全局 ID；
- UUID；
- 复合业务键经过标准化后的 ID；
- 主数据平台统一 ID。

---

## 7.5 Title Key

Title Key 决定对象在应用中的默认显示值。

例如：

- Customer：客户名称；
- Equipment：设备编码；
- Employee：员工姓名；
- Flight：航班号；
- Order：订单编号。

Title Key 不承担唯一性职责，它主要服务于用户体验。

---

## 7.6 Object Type Datasources

Object Type 可以由一个或多个数据源提供数据。

### 数据源配置内容

- Backing Datasource；
- 数据源类型；
- Primary Key 映射；
- Property 字段映射；
- Schema；
- 数据新鲜度；
- 索引方式；
- 写回数据集；
- 安全来源；
- 数据限制；
- 索引错误。

### 支持的典型数据源

- Dataset；
- Restricted View；
- Streaming Dataset；
- Pipeline Builder 输出；
- Direct Datasource；
- 多数据源对象。

---

## 7.7 Multi-datasource Object Type

一个 Object Type 可以融合多个来源的数据。

例如：

```text
Customer Object Type
├── CRM Customer Dataset
├── Billing Customer Dataset
├── Support Customer Dataset
└── User Edits
```

### 主要目的

- 构建统一业务实体；
- 避免按源系统建立重复对象；
- 合并不同系统的补充属性；
- 保留统一对象身份；
- 支持跨系统业务应用。

### 关键控制

- 统一 Primary Key；
- 属性冲突处理；
- 数据覆盖优先级；
- Schema 演进；
- 权限合并；
- 更新延迟；
- 写回策略。

---

## 7.8 Object Type Capabilities

Capabilities 用于配置对象在平台中的特殊能力。

可能包括：

- 地理空间能力；
- 媒体能力；
- 时间序列能力；
- 可创建能力；
- 特定应用集成；
- 特定渲染行为；
- Gotham / Gaia 集成；
- 其他平台级特性。

Capabilities 逐步承接过去通过 Type Classes 表达的配置。

---

## 7.9 Object Views

创建 Object Type 后，Foundry 会提供标准 Object View。

标准 Object View 通常展示：

- Prominent Properties；
- 普通 Properties；
- Linked Objects；
- 时间序列；
- 地图；
- 媒体；
- 可执行 Actions。

Ontology Manager 中可：

- 预览 Object View；
- 选择预览对象；
- 查看 Full View；
- 查看 Panel View；
- 测试明暗主题；
- 进入 Object View Editor；
- 管理默认 Object View。

需要注意：

> Object View 的复杂页面内容通常由 Workshop 模块承载。Ontology Manager 提供入口、预览和对象类型级管理，但并非所有布局编辑都直接在 OMA 基础表单中完成。

---

# 8. Property 管理

Property 定义 Object Type 的一个业务特征。

例如：

```text
Equipment
├── equipmentId
├── equipmentName
├── status
├── location
├── installationDate
└── riskScore
```

---

## 8.1 Property 基础配置

| 配置项 | 说明 |
|---|---|
| Display Name | 属性显示名称 |
| API Name | API 和代码使用名称 |
| Description | 业务定义 |
| Base Type | 底层数据类型 |
| Value Type | 领域值类型 |
| Datasource Column | 映射字段 |
| Required | 是否必填 |
| Editable | 是否允许用户编辑 |
| Searchable | 是否支持搜索 |
| Sortable | 是否支持排序 |
| Prominent | 是否重点展示 |
| Hidden | 是否隐藏 |
| Value Formatting | 值格式 |
| Conditional Formatting | 条件样式 |
| Aliases | 同义词 |
| Shared Property | 是否绑定共享属性 |

---

## 8.2 Property Base Type

常见基础类型包括：

- String；
- Boolean；
- Integer；
- Long；
- Float；
- Double；
- Decimal；
- Date；
- Timestamp；
- Array；
- Struct；
- Geopoint / Geoshape；
- Media Reference；
- Time Series；
- User / Resource Reference；
- 其他平台支持类型。

不同 Base Type 决定：

- 可执行的过滤操作；
- 聚合方式；
- UI 渲染；
- 格式化方式；
- 是否可作为 Primary Key；
- 是否可作为 Title Key；
- 是否支持搜索排序。

---

## 8.3 Value Formatting

用于把原始数据转成更易读的展示格式。

典型配置：

- 数字千分位；
- 小数位数；
- 百分比；
- 货币；
- 日期格式；
- 时区；
- 用户 ID 转用户信息；
- 资源 ID 转资源名称；
- 单位显示。

---

## 8.4 Conditional Formatting

根据 Property 值应用不同展示样式。

例如：

```text
riskScore >= 80  → 高风险
50 <= riskScore < 80 → 中风险
riskScore < 50 → 低风险
```

应用场景：

- 风险等级；
- 告警状态；
- KPI 达成情况；
- 任务逾期；
- 库存预警；
- 设备健康状态。

---

## 8.5 Required Property

Required 表示对象应具有该属性值。

需要区分：

- 数据源 Schema 非空；
- Ontology 层 Required；
- Action 提交时必填；
- 编辑时必填；
- 历史对象是否满足新约束。

对生产对象新增 Required Property 可能形成 breaking change，应在分支中验证。

---

## 8.6 Edit-only Property

Edit-only Property 主要用于承载用户操作产生的数据，而不一定来自原始 Datasource。

例如：

- 人工备注；
- 审核状态；
- 处理结论；
- 负责人；
- 手工标签；
- 决策原因。

它通常通过 Action 写入并保存在对象编辑或写回体系中。

---

## 8.7 Derived Property

Derived Property 根据已有属性、关系或逻辑计算得到。

例如：

- 订单总金额；
- 设备是否逾期；
- 客户风险等级；
- 距离下一次维护的天数。

设计时需要考虑：

- 计算时机；
- 实时还是物化；
- 性能；
- 空值；
- 依赖字段变化；
- 是否可搜索和排序。

---

## 8.8 Property Reducer

Reducer 用于在多个来源或多个候选值之间确定最终 Property 值。

适用于：

- 多数据源对象；
- 多条记录归并；
- 选择最新值；
- 选择最大或最小值；
- 合并数组；
- 处理来源优先级。

---

# 9. Struct 管理

Struct 用于表达一个由多个字段组成的复杂值。

例如：

```text
Address
├── country
├── province
├── city
├── district
├── street
└── postalCode
```

适合：

- 地址；
- 联系信息；
- 坐标；
- 度量值与单位；
- 复合状态；
- 嵌套业务数据。

## 9.1 Struct 配置

- Struct Type 名称；
- 字段列表；
- 字段 Display Name；
- 字段 API Name；
- 字段 Description；
- 字段 Base Type；
- 主显示字段；
- 数据源自动映射；
- Shared Property 绑定。

## 9.2 Struct 设计注意事项

- 不要用 Struct 代替真正需要独立生命周期的 Object Type；
- 需要被单独查询、关联或执行 Action 的实体应建为 Object Type；
- Struct 适合“作为一个整体属于对象”的复合值；
- Struct Schema 变更也需要评估下游兼容性。

---

# 10. Shared Property 管理

Shared Property 是可被多个 Object Type 复用的 Property 定义。

例如：

```text
Shared Property: startDate
├── Employee.startDate
├── Contractor.startDate
└── Project.startDate
```

## 10.1 主要价值

- 统一属性语义；
- 统一名称和描述；
- 统一格式；
- 集中维护元数据；
- 降低重复定义；
- 支持 Interface 和跨类型复用；
- 改善数据治理。

## 10.2 Shared Property 配置

- Name；
- Description；
- API Name；
- Base Type；
- Value Formatting；
- Render Hints；
- Aliases；
- Struct Fields；
- Usage；
- Project Location；
- Permissions。

## 10.3 主要操作

- 新建 Shared Property；
- 将现有 Property 转换为 Shared Property；
- 将 Shared Property 应用到 Object Type；
- 查看使用它的 Object Type；
- 修改共享元数据；
- 从 Object Type Detach；
- 删除 Shared Property。

删除 Shared Property 后，使用它的属性可退回为普通 Property，但应先评估一致性和下游影响。

---

# 11. Value Type 管理

Value Type 是对基础数据类型增加业务语义和约束的可复用类型。

例如：

- EmailAddress；
- URL；
- UUID；
- CurrencyCode；
- CountryCode；
- OrderStatus；
- ProductSKU。

## 11.1 主要能力

- 定义领域名称；
- 绑定基础字段类型；
- 定义验证约束；
- 定义枚举；
- 定义版本；
- 在 Property 和 Pipeline 中复用；
- 管理权限；
- 提升类型安全。

## 11.2 与 Shared Property 的区别

| 对比项 | Value Type | Shared Property |
|---|---|---|
| 核心目的 | 约束和表达值本身 | 统一 Property 元数据 |
| 是否包含对象属性语义 | 较弱 | 强 |
| 是否跨 Pipeline 使用 | 可以 | 主要用于 Ontology |
| 是否包含 Display Name 等属性元数据 | 以类型元数据为主 | 包含 Property 元数据 |
| 示例 | EmailAddress | Customer Email |

---

# 12. Link Type 管理

Link Type 定义两个 Object Type 之间的关系。

例如：

```text
Customer --places--> Order
Order --contains--> Product
Equipment --locatedAt--> Factory
Employee --manages--> Employee
```

---

## 12.1 Link Type 基础配置

| 配置项 | 说明 |
|---|---|
| Display Name | 关系名称 |
| Reverse Display Name | 反向关系名称 |
| API Name | API 名称 |
| Description | 业务定义 |
| Source Object Type | 起点对象 |
| Target Object Type | 终点对象 |
| Cardinality | 基数 |
| Backing Datasource | 关系数据源 |
| Source Key Mapping | 来源对象键映射 |
| Target Key Mapping | 目标对象键映射 |
| Status | 生命周期状态 |
| Security | 权限 |
| Project Location | 保存位置 |

---

## 12.2 Link Cardinality

典型基数：

- One-to-one；
- One-to-many；
- Many-to-one；
- Many-to-many。

设计时应从业务语义出发，而不是只按数据库 Join 结果选择。

---

## 12.3 Link 创建方式

- 在 Object Type Link Graph 中创建；
- 从 Link Type 列表创建；
- 从 Datasource 创建；
- 从 Pipeline Builder 输出创建；
- 通过 JSON 或 API 创建。

---

## 12.4 Link Datasource 映射

Many-to-many Link 通常需要独立关系数据源。

例如：

```text
OrderProductLink Dataset
├── orderId
├── productId
├── quantity
└── createdAt
```

如果关系本身存在需要管理的属性，例如 `quantity`、`role`、`validFrom`，通常需要评估是否应将关系实体化为独立 Object Type，而不是只用简单 Link。

---

## 12.5 Link Type View

主要页面：

- Overview；
- Datasources；
- Usage；
- Security；
- History。

主要展示：

- 两端 Object Type；
- Link 图；
- 基数；
- 数据源；
- 映射字段；
- 依赖资源；
- 使用情况；
- 最近变更；
- 索引问题。

---

# 13. Interface 管理

Interface 描述一组 Object Type 共同具有的结构和能力。

例如：

```text
Interface: Facility
├── facilityName
├── location
├── owner
├── facilityToRegion Link Constraint
└── Update Facility Action Constraint
```

可能由以下 Object Type 实现：

- Airport；
- Manufacturing Plant；
- Warehouse；
- Maintenance Hangar。

---

## 13.1 Interface 组成

- Interface Property；
- Link Type Constraint；
- Action Type Constraint；
- Metadata；
- Parent Interface；
- Child Interface；
- Implementing Object Types。

## 13.2 Interface 主要能力

- 定义公共 Property；
- 定义公共 Link 能力；
- 定义公共 Action 能力；
- 被多个 Object Type 实现；
- 支持一个 Object Type 实现多个 Interface；
- 支持 Interface 继承；
- 支持跨 Object Type 统一查询和应用。

## 13.3 Interface 校验

当 Interface 新增必需 Property、Link Constraint 或 Action Constraint 时：

- 所有实现该 Interface 的 Object Type 必须满足新约束；
- 不满足的实现需要修复；
- 可能产生 breaking change；
- 应在 Branch 中修改并通过 Proposal 审核。

---

# 14. Object Type Group 管理

Object Type Group 是用于分类和发现 Ontology 资源的管理原语。

例如：

```text
Supply Chain
├── Supplier
├── Purchase Order
├── Shipment
└── Inventory Item
```

## 14.1 主要能力

- 创建 Group；
- 编辑 Group；
- 将 Object Type 加入一个或多个 Group；
- 在搜索中发现 Group；
- 按 Group 筛选；
- 在 Discover 页面展示 Group；
- 在 Object Explorer 中展示 Group；
- 管理 Group 所在 Project 和权限。

## 14.2 Group 与业务 Domain 的关系

Group 可以表达：

- 业务域；
- 用例；
- 团队；
- 数据产品；
- 生命周期阶段。

但 Group 本身通常不承担完整 Domain 的权限、版本和依赖边界，仍需结合 Project、Space 和治理制度。

---

# 15. Action Type 管理

Action Type 定义用户或系统可以一次性对对象、属性和 Link 执行的一组业务变更。

例如：

- Assign Employee；
- Approve Order；
- Close Alert；
- Create Maintenance Task；
- Change Equipment Status；
- Reassign Shipment。

Action 是 Foundry 从“查看数据”走向“执行业务”的核心能力。

---

## 15.1 Action Type 组成

```text
Action Type
├── Metadata
├── Parameters
├── Rules
├── Submission Criteria
├── Permission
├── Side Effects
├── Function-backed Logic
├── Validation
├── Writeback
└── Observability
```

---

## 15.2 Action Metadata

- Display Name；
- API Name；
- Description；
- Status；
- Object Type；
- Interface；
- Icon；
- Form Instructions；
- Project Location；
- Point of Contact。

---

## 15.3 Parameters

Action Parameter 是用户或系统提交 Action 时输入的参数。

常见类型：

- String；
- Number；
- Boolean；
- Date / Timestamp；
- Object；
- Object Set；
- Enum；
- Array；
- Struct；
- Attachment。

### 参数配置

- Display Name；
- Description；
- Required；
- Default Value；
- Validation；
- Dropdown Options；
- Object Filter；
- Dynamic Value；
- Hidden；
- Read-only；
- Conditional Visibility。

### 安全要求

对象参数的下拉选项必须遵守对象权限，不应把用户无权查看的对象暴露在候选列表中。

---

## 15.4 Rules

Action Rules 定义 Action 提交后执行的变化或副作用。

典型规则：

- Create Object；
- Modify Object；
- Delete Object；
- Create Link；
- Delete Link；
- Modify Property；
- 触发 Notification；
- 调用 Function；
- 调用 Webhook；
- 写入外部系统；
- 创建后续任务。

---

## 15.5 Submission Criteria

Submission Criteria 决定 Action 是否允许提交。

可以依据：

- 当前用户；
- 当前对象状态；
- Property 值；
- Link；
- Function 计算结果；
- 参数值；
- 组织和权限；
- 时间条件。

例如：

```text
只有满足以下条件才能审批：
- 当前订单状态为 Pending
- 当前用户具有 Approver 权限
- 订单金额低于用户审批额度
- 供应商未被列入风险名单
```

---

## 15.6 Function-backed Action

复杂 Action 可以由 Function 实现。

适合：

- 多对象事务；
- 复杂计算；
- 动态业务规则；
- 批量处理；
- 外部 API 调用；
- 优化模型调用；
- 复杂 Link 更新；
- 条件分支。

---

## 15.7 Side Effects

Action 除了编辑 Ontology，还可以触发副作用：

- 发送通知；
- 调用外部服务；
- 记录审计；
- 启动自动化；
- 更新其他系统；
- 触发模型；
- 发送消息。

副作用设计要考虑：

- 幂等性；
- 重试；
- 失败补偿；
- 超时；
- 审计；
- 与主事务的一致性。

---

## 15.8 Action Observability

Action Type View 可提供 Observability 页面，查看：

- 最近 30 天调用；
- 成功和失败；
- 使用趋势；
- 监控规则；
- 规则状态；
- 调用应用；
- 错误。

适合监控：

- Action 失败率；
- 延迟；
- 异常高频调用；
- 长时间无调用；
- 关键业务动作健康状态。

---

# 16. Function 管理

Ontology Manager 可以查看和管理已发布 Function，但 Function 的代码修改通常在 Functions Code Repository 中进行。

---

## 16.1 Function 列表

支持按以下元数据搜索：

- Function Name；
- Description；
- API Name；
- RID；
- Version；
- Repository；
- 输入输出类型。

---

## 16.2 Function Overview

主要展示：

- Function 元数据；
- 输入参数；
- 输出类型；
- 最新版本；
- 历史版本；
- 所属 Repository；
- 依赖的 Ontology 类型；
- 使用历史；
- 发布信息。

---

## 16.3 Function Version

支持：

- 查看最新版本；
- 切换查看历史版本；
- 查看应用使用了哪个版本；
- 识别需要升级的应用；
- 追踪稳定版本和非稳定版本。

---

## 16.4 Function Configuration

某些 Function 类型可配置：

- Timeout；
- Memory Limit；
- Snapshot Behavior；
- 其他运行资源。

配置通常按 Function Version 生效。

---

## 16.5 Usage History

记录：

- 哪些应用使用 Function；
- 使用哪个 Function Version；
- 最近使用时间；
- 需要升级的依赖应用。

---

## 16.6 Function Observability

可查看：

- 最近调用；
- 成功率；
- 失败率；
- 延迟；
- 监控规则；
- 运行状态；
- 版本使用分布。

---

## 16.7 跳转 Functions Repository

Ontology Manager 提供跳转入口。

但需要明确：

> Function 的代码内容、依赖、测试和发布不在 Ontology Manager 中直接编辑，而在 Functions Code Repository 中完成。

---

# 17. 数据映射、索引和运行状态

Ontology Manager 不只是编辑抽象 Schema，还负责将 Ontology 定义连接到真实数据。

---

## 17.1 Backing Datasource

每个数据驱动的 Object Type 需要配置 Backing Datasource。

主要内容：

- 数据源；
- 主键列；
- Property Mapping；
- 数据类型；
- 索引目标；
- 安全；
- 更新状态。

---

## 17.2 Batch Indexing

批处理数据通过构建和索引进入 Ontology。

需要监控：

- Dataset 最近更新时间；
- Pipeline Build 状态；
- Schema 是否匹配；
- 索引是否完成；
- 对象数量；
- 失败记录；
- 数据新鲜度。

---

## 17.3 Streaming Indexing

流式对象需要监控：

- Stream 是否运行；
- 最新事件时间；
- 消息延迟；
- Schema；
- 无效记录；
- 流任务状态；
- 回放和重置风险。

---

## 17.4 Direct Datasource

Direct Datasource 用于低延迟写入 Ontology，适合：

- 流式对象编辑；
- 小批量高频更新；
- 对延迟要求高的运营场景。

Ontology Manager 可查看：

- Direct Writer；
- Pipeline Builder 来源；
- Data Liveness；
- Schema Liveness；
- Latest Edit；
- Stream Metrics；
- Job History；
- Invalid Writes；
- Error Logs。

---

## 17.5 Data Liveness

用于判断当前对象视图是否新鲜。

典型指标：

- 最后数据写入时间；
- 最后 Schema 更新时间；
- 最后用户编辑时间；
- 最新索引时间；
- Pipeline 状态。

---

## 17.6 索引问题诊断

常见问题：

- Datasource 被删除；
- Datasource 未注册；
- Schema 不兼容；
- Primary Key 为空；
- Primary Key 重复；
- 数据类型错误；
- 记录违反限制；
- 索引任务失败；
- Direct Write 无效；
- 数据长期未更新。

Ontology Manager 应提供：

- 错误标记；
- 错误详情；
- 受影响资源；
- 跳转上游 Pipeline；
- 日志入口；
- 修复建议；
- 重新索引入口或指引。

---

# 18. 资源依赖和使用分析

## 18.1 Dependents

用于查看哪些资源依赖当前 Ontology 资源。

可能包括：

- Link Type；
- Action Type；
- Interface；
- Function；
- Workshop Application；
- Quiver；
- Object View；
- OSDK Application；
- AIP Logic；
- Automate；
- API Consumer。

其主要价值是支持 breaking change 评估。

---

## 18.2 Usage Graph

Object Type 和 Link Type Overview 可以展示最近 30 天使用概览。

主要指标：

- Reads；
- Writes；
- Interactions；
- Active Users。

---

## 18.3 Reads

Read 表示应用加载或查询某种 Object Type。

例如：

- Workshop 表格加载对象；
- Object Search；
- Property 聚合；
- OSDK 查询；
- Object Set 查询。

一次请求可能加载多个对象，但计为一次 Read 请求。

---

## 18.4 Writes

Write 表示应用对对象进行编辑。

来源可能包括：

- Action；
- Function；
- Foundry Form；
- Object Explorer 直接编辑；
- API 调用。

---

## 18.5 Usage 明细

Usage 页面可以查看：

- 哪些用户使用；
- 哪个时间使用；
- 哪个 Foundry 应用使用；
- Read 还是 Write；
- 最近使用趋势；
- 活跃用户数量。

---

## 18.6 使用分析的治理作用

在执行以下操作前应检查 Usage：

- 删除 Property；
- 修改 API Name；
- 修改 Base Type；
- 删除 Object Type；
- 改变 Primary Key；
- 删除 Link Type；
- 修改 Action 参数；
- 弃用 Function Version；
- 修改 Interface 约束。

---

# 19. Status 和生命周期管理

每个 Object Type、Property、Link Type、Action 或 Interface 都可以具有开发状态。

## 19.1 Promoted

适用于 Object Type。

表示：

- 核心可信资源；
- 已经过 Ontology Owner 审核；
- 推荐在正式业务中使用；
- API Name 等关键标识受到更严格保护。

## 19.2 Active

表示：

- 正式使用；
- 已被用户应用依赖；
- 不应随意进行 breaking change；
- 修改需要谨慎评估。

## 19.3 Experimental

表示：

- 仍在开发；
- 可能变化；
- 不保证兼容；
- 适合试验和早期用例。

## 19.4 Deprecated

表示：

- 即将删除；
- 不应再被新应用使用；
- 应迁移到替代资源。

Deprecated 元数据可以包括：

- 弃用原因；
- 删除期限；
- 替代资源。

## 19.5 Example

表示：

- 示例或培训资源；
- 不应用于生产；
- 可能来自 Starter Pack 或教程。

---

# 20. Point of Contact 和责任治理

Object Type 等关键资源应设置 Point of Contact。

可用于：

- 明确负责人；
- 变更通知；
- 故障通知；
- 数据质量问题归属；
- breaking change 协调；
- 清理候选确认；
- 业务定义确认。

建议责任信息包括：

- Owner；
- Steward；
- Technical Contact；
- Business Contact；
- Support Channel。

---

# 21. 变更管理

Ontology Manager 采用 Working State 管理编辑。

---

## 21.1 Working State

用户的编辑首先保存在未发布的工作状态中。

特点：

- 不立即影响其他用户；
- 不立即影响生产应用；
- 可积累多项修改；
- 可统一 Review；
- 可撤销；
- 可保存到 Main 或 Branch。

---

## 21.2 Review Edits

保存前进入 Review Edits。

应展示：

- 新增资源；
- 修改资源；
- 删除资源；
- Property 变化；
- Link 变化；
- Action 变化；
- 数据源变化；
- 安全变化；
- Errors；
- Warnings。

---

## 21.3 Errors 和 Warnings

### Error

会阻止保存，例如：

- 缺少 Primary Key；
- API Name 冲突；
- 数据类型不兼容；
- Interface 约束未满足；
- Link 映射无效；
- 必要数据源缺失。

### Warning

不会阻止保存，但建议处理，例如：

- Description 缺失；
- 可能存在 breaking change；
- 下游依赖较多；
- Status 不匹配；
- 数据源新鲜度异常。

---

## 21.4 并发更新和 Merge Conflict

如果其他用户在当前用户编辑期间保存了 Ontology：

1. 当前用户需要 Update；
2. 系统合并远端更新；
3. 检测 Merge Conflict；
4. 用户选择保留最新版本或当前工作状态；
5. 再次 Review；
6. 保存。

---

# 22. History 和 Restore

## 22.1 全局 History

记录每次保存：

- 修改时间；
- 修改人；
- 修改内容；
- 涉及资源；
- 保存批次。

可以按作者合并展示连续变更。

---

## 22.2 单资源 History

每个 Ontology Resource 可查看：

- 当前未保存修改；
- 历史保存记录；
- 修改时间；
- 修改人；
- 字段级变化；
- 最后编辑信息。

---

## 22.3 Restore

可将 Object Type 恢复到历史版本。

恢复流程：

```text
选择历史版本
      ↓
点击 Restore
      ↓
确认
      ↓
恢复内容进入 Working State
      ↓
Review
      ↓
Save
```

恢复不会自动立即发布，需要再次 Save。

---

# 23. Branch、Proposal 和审批

## 23.1 Main Branch

Main 是正式 Ontology。

生产应用通常读取 Main 上已发布的资源定义。

---

## 23.2 Branch

Branch 用于隔离开发和测试。

可用于：

- 修改 Object Type；
- 修改 Action Type；
- 修改 Link Type；
- 修改 Interface；
- 修改 Shared Property；
- 测试跨资源变化；
- 验证下游应用；
- 避免直接影响生产。

---

## 23.3 Protected Resource

受保护资源不能直接修改 Main。

保护对象可包括：

- Object Type；
- Action Type；
- Link Type；
- Interface；
- Shared Property。

修改受保护资源必须：

1. 创建 Branch；
2. 保存变更；
3. 创建 Proposal；
4. 通过 Review；
5. 合并到 Main。

---

## 23.4 Proposal

Proposal 类似代码版本控制中的 Pull Request。

主要内容：

- Proposal Name；
- Description；
- Author；
- Reviewers；
- Branch；
- Changes；
- Tasks；
- Preview Status；
- Changelog；
- Review State；
- Merge State。

---

## 23.5 Proposal 分类

- My Proposals；
- Assigned to Me；
- In Review；
- Merged；
- Closed。

---

## 23.6 Review 内容

Reviewer 应检查：

- 资源状态；
- breaking change；
- 数据源；
- API Name；
- Primary Key；
- Link；
- Action；
- Interface；
- 权限；
- 下游依赖；
- 使用情况；
- Preview；
- 测试结果；
- 迁移方案。

---

## 23.7 Global Branching

Global Branching 可以统一管理：

- Pipeline；
- Dataset Schema；
- Ontology；
- Object View；
- Workshop；
- 其他支持分支的资源。

其价值是进行端到端变更验证，而不是只修改 Ontology 元数据。

---

# 24. 权限和安全管理

需要区分两类权限：

```text
Ontology Resource Permission
≠
Object Instance Permission
```

---

## 24.1 Ontology Resource Permission

控制用户能否：

- 查看 Object Type 定义；
- 编辑 Object Type；
- 管理 Link Type；
- 管理 Action Type；
- 管理 Interface；
- 管理 Shared Property；
- 修改资源权限。

当前推荐使用 Project-based Permissions。

Ontology 资源保存在 Project 中，并继承：

- Project Role；
- Project Classification；
- Project 安全规则。

---

## 24.2 Object Instance Permission

控制用户能否读取具体对象和 Link 数据。

通常依赖：

- Backing Datasource 所在位置；
- Dataset 权限；
- Restricted View；
- Object Security Policy；
- 行级和列级安全；
- Markings；
- 组织边界。

迁移 Ontology 资源权限不会自动改变对象实例权限。

---

## 24.3 Action Permission

控制：

- 谁能看到 Action；
- 谁能执行 Action；
- 对哪些对象执行；
- 在什么条件下执行；
- 是否可以修改特定 Property；
- 是否可以触发 Side Effect。

权限检查应在服务端执行，不能只依赖前端隐藏按钮。

---

## 24.4 Project-based Permissions

核心特点：

- Ontology 资源保存在指定 Project；
- 通过 Compass 管理；
- 与其他 Foundry Resource 采用一致权限模型；
- 可与 Datasource 放在同一 Use Case Project；
- 也可放入专用 Ontology Project；
- 支持混合组织模式。

### 组织策略

#### 方案一：与 Datasource 放在一起

优点：

- 权限一致；
- 用例资源集中；
- 管理简单。

#### 方案二：独立 Ontology Project

优点：

- 核心资源集中；
- 适合跨团队复用；
- 避免业务项目散乱。

#### 方案三：混合模式

- 核心共享类型放入核心 Ontology Project；
- 用例特有类型放入 Use Case Project。

---

# 25. Ontology Import 和 Export

Ontology Working State 可以导出为 JSON，并重新导入。

## 25.1 Export

可用于：

- 代码方式编辑；
- 备份 Working State；
- 复制到其他 Ontology；
- 批量修改；
- 自动化迁移。

导出内容包括当前 Working State 中未保存变更。

## 25.2 Import

导入 JSON 后：

- 重建 Working State；
- 显示变更数量；
- 需要 Review；
- 需要 Save；
- 可能出现兼容性错误。

## 25.3 风险

- JSON Schema 可能随产品演进变化；
- 不应把内部 JSON 格式视为永久稳定公共协议；
- Conditional Formatting 等资源可能不能跨 Ontology 直接迁移；
- 导入前应进行备份和校验。

---

# 26. Ontology Cleanup

Ontology Cleanup 用于安全识别和删除不再需要的 Object Type。

## 26.1 主要目标

- 减少冗余 Object Type；
- 改善导航体验；
- 提升查询和加载性能；
- 降低对象存储成本；
- 推动废弃资源治理。

## 26.2 Cleanup Queue

系统根据规则识别候选对象，并形成清理队列。

支持：

- 按 Flag 筛选；
- 按 Group 筛选；
- 自定义列；
- 按优先级排序；
- 批量选择。

## 26.3 清理操作

### Snooze

- 暂时从个人队列隐藏；
- 可设置时间；
- 不影响其他编辑者。

### Deprecate

- 将状态设为 Deprecated；
- 填写原因；
- 设置删除期限；
- 指定替代资源；
- 通知使用者迁移。

### Delete

- 删除 Object Type；
- 删除关联对象存储数据；
- 需要检查依赖和用户编辑；
- 需要通过正常 Save 或 Proposal 流程。

## 26.4 Cleanup Flags

典型 Flag：

- 已超过 Deprecation Date；
- Backing Datasource 已进入 Trash；
- Datasource 多日未更新；
- Description 缺失；
- Display Name 匹配测试或废弃命名；
- 已从旧对象存储取消索引；
- 无使用；
- 无数据；
- 无负责人；
- 长期 Experimental。

---

# 27. Marketplace 和产品化

Ontology Resource 可以被加入 Marketplace Product。

可产品化资源包括：

- Object Type；
- Link Type；
- Action Type；
- Interface；
- Shared Property；
- 配套 Pipeline；
- Workshop Application；
- Function。

主要用途：

- 跨环境复制；
- 跨团队复用；
- 行业解决方案；
- Starter Pack；
- 版本升级；
- 依赖管理。

产品化时应考虑：

- API Name 稳定；
- 数据源参数化；
- 安全模型；
- 环境差异；
- Migration；
- 版本兼容；
- 示例资源和生产资源区分。

---

# 28. Ontology Manager 页面信息架构建议

以下是基于官方功能整理的完整页面结构。

```text
Ontology Manager
├── Discover
├── Search
├── Object Types
├── Link Types
├── Action Types
├── Interfaces
├── Shared Properties
├── Functions
├── Groups
├── Proposals
├── Unsaved Changes
├── History
├── Cleanup
├── Advanced Settings
│   ├── Import
│   ├── Export
│   └── Configuration
└── Ontology Configuration
    ├── Permissions
    ├── Project Permission Migration
    ├── Metrics
    ├── Branching
    ├── Resource Protection
    └── Group Migration
```

Object Type 详情页建议：

```text
Object Type Detail
├── Overview
├── Properties
├── Datasources
├── Link Types
├── Action Types
├── Interfaces
├── Capabilities
├── Object Views
├── Dependents
├── Usage
├── Security
├── History
└── Issues
```

---

# 29. Ontology Manager 核心业务流程

## 29.1 创建对象模型

```text
准备 Dataset
   ↓
创建 Object Type
   ↓
定义 Metadata
   ↓
映射 Properties
   ↓
设置 Primary Key / Title Key
   ↓
配置 Status / Group / Owner
   ↓
保存
   ↓
索引对象
   ↓
在 Object Explorer 验证
```

## 29.2 建立对象关系

```text
确认业务关系
   ↓
选择两端 Object Type
   ↓
确定 Cardinality
   ↓
配置正向和反向名称
   ↓
选择 Link Datasource
   ↓
映射主键
   ↓
保存
   ↓
验证 Link
```

## 29.3 创建业务动作

```text
确定业务动作
   ↓
选择目标 Object / Interface
   ↓
配置 Parameters
   ↓
配置 Rules
   ↓
配置 Submission Criteria
   ↓
配置 Permission
   ↓
配置 Side Effects
   ↓
测试
   ↓
保存和发布
   ↓
在 Workshop / OSDK 使用
```

## 29.4 安全变更

```text
创建 Global Branch
   ↓
修改 Pipeline / Ontology / Application
   ↓
检查错误和警告
   ↓
验证对象和 Action
   ↓
创建 Proposal
   ↓
Reviewer 审核
   ↓
解决冲突
   ↓
批准
   ↓
合并 Main
   ↓
监控运行状态
```

## 29.5 资源弃用

```text
识别低使用资源
   ↓
检查 Usage 和 Dependents
   ↓
设置 Deprecated
   ↓
填写原因、期限和替代资源
   ↓
推动下游迁移
   ↓
确认无依赖
   ↓
Branch + Proposal
   ↓
删除
```

---

# 30. 关键非功能要求

自研 Ontology Manager 时，除业务功能外，还应关注以下非功能能力。

## 30.1 一致性

- Schema 修改应具备事务性；
- 关联资源不能保存为不一致状态；
- Action 定义和 Object Schema 必须兼容；
- Interface 实现必须通过校验。

## 30.2 性能

- 大型 Ontology 搜索应快速；
- 资源列表支持分页和筛选；
- Link Graph 应支持大图降级；
- Usage 统计应异步计算；
- History 应支持增量加载。

## 30.3 可审计性

- 所有保存记录用户和时间；
- 权限变更可审计；
- 删除可追踪；
- Action 定义变更可追踪；
- Proposal 审批过程可追踪。

## 30.4 可恢复性

- 支持历史恢复；
- 支持分支；
- 支持冲突解决；
- 支持导出备份；
- 删除操作应有充分确认和保护。

## 30.5 安全

- 服务端权限校验；
- 资源和实例权限分离；
- 敏感字段不泄露；
- 搜索结果遵守权限；
- Usage 信息遵守权限；
- API Name 和 RID 不应绕过授权。

## 30.6 可扩展性

- 新 Ontology Resource Type 可扩展；
- 新 Property Type 可扩展；
- 新 Action Rule 可扩展；
- 新 Datasource 类型可扩展；
- 新 Capability 可扩展；
- 新 Observability 指标可扩展。

---

# 31. 自研 Ontology Manager 的建议模块优先级

## P0：必须实现

1. Ontology 基础信息；
2. Object Type；
3. Property；
4. Link Type；
5. Datasource Mapping；
6. Primary Key / Title Key；
7. Action Type 基础能力；
8. Resource Search；
9. Working State；
10. Review and Save；
11. History；
12. Project Permission；
13. Audit；
14. API；
15. 基础依赖分析。

## P1：生产级治理

1. Branch；
2. Proposal；
3. Resource Protection；
4. Interface；
5. Shared Property；
6. Function 管理；
7. Object Type Group；
8. Status；
9. Usage Metrics；
10. Cleanup；
11. Observability；
12. Multi-datasource；
13. Object Views。

## P2：高级能力

1. Value Type；
2. Struct；
3. Derived Property；
4. Complex Action；
5. Function-backed Action；
6. Streaming Indexing；
7. Direct Datasource；
8. Marketplace；
9. Import / Export；
10. Global Branching；
11. AI 辅助建模；
12. 自动影响分析；
13. 自动 Schema 映射。

---

# 32. 与其他 Foundry 模块的边界

## 32.1 Ontology Manager 与 Object Explorer

| Ontology Manager | Object Explorer |
|---|---|
| 管理 Schema | 查询对象实例 |
| 创建 Object Type | 搜索 Object |
| 配置 Link Type | 浏览 Link |
| 配置 Action Type | 执行 Action |
| 管理数据源 | 查看对象数据 |
| 管理变更 | 业务探索 |

## 32.2 Ontology Manager 与 Workshop

| Ontology Manager | Workshop |
|---|---|
| 定义业务模型 | 构建业务应用 |
| 定义 Action | 在页面中调用 Action |
| 定义 Property | 展示 Property |
| 定义 Link | 展示和遍历 Link |
| 管理安全定义 | 遵守安全定义 |

## 32.3 Ontology Manager 与 Pipeline Builder

| Ontology Manager | Pipeline Builder |
|---|---|
| 定义业务语义 | 清洗和转换数据 |
| 映射 Datasource | 生成 Datasource |
| 检查索引状态 | 运行数据 Pipeline |
| 管理 Object Type | 生成或更新对象输出 |

## 32.4 Ontology Manager 与 Functions Repository

| Ontology Manager | Functions Repository |
|---|---|
| 查看 Function | 编写 Function |
| 查看版本 | 发布版本 |
| 配置运行参数 | 编写测试 |
| 查看 Usage | 管理代码依赖 |
| 查看 Observability | 构建和发布 |

## 32.5 Ontology Manager 与 Control Panel

| Ontology Manager | Control Panel |
|---|---|
| 管理 Ontology Resource | 管理平台 |
| 配置 Resource Security | 管理用户和组织 |
| 查看 Ontology Usage | 开启 Ontology Metrics |
| 管理分支和资源 | 管理应用访问和平台配置 |

---

# 33. 重要设计原则

## 33.1 Ontology 不是数据库表目录

不应按源系统逐表创建 Object Type。

错误示例：

```text
SAP_Customer
CRM_Customer
Billing_Customer
```

推荐：

```text
Customer
├── SAP Datasource
├── CRM Datasource
└── Billing Datasource
```

## 33.2 Object Type 应表达业务实体或事件

适合：

- Customer；
- Order；
- Equipment；
- Shipment；
- Alert。

不适合：

- 临时 Join 结果；
- 纯 ETL 中间表；
- 无业务意义的技术表；
- 仅为某张报表存在的数据集。

## 33.3 Link 应表达稳定业务关系

不要把所有数据库 Join 都建成 Link。

Link 应支持：

- 业务理解；
- 应用导航；
- 权限；
- 分析；
- Action；
- Agent 工具调用。

## 33.4 Action 应表达业务意图

推荐：

- Approve Purchase Order；
- Assign Technician；
- Close Alert。

不推荐：

- Update status column；
- Set field X；
- Insert row。

## 33.5 API Name 应保持稳定

Display Name 可以修改，但 API Name 被以下资源依赖：

- Function；
- OSDK；
- API；
- Application；
- Agent；
- Automation。

修改 API Name 必须进行完整影响分析。

---

# 34. 结论

Ontology Manager 是 Foundry 中负责 Ontology 设计、配置、治理和变更管理的核心应用。

它不是简单的“对象类型编辑器”，而是由以下能力共同组成：

```text
语义模型管理
+
真实数据映射
+
业务 Action 管理
+
Interface 和复用
+
权限与治理
+
版本和分支
+
依赖和影响分析
+
运行状态监控
+
生命周期清理
```

其最核心的管理对象包括：

- Object Type；
- Property；
- Link Type；
- Action Type；
- Interface；
- Shared Property；
- Function；
- Object Type Group；
- Datasource Mapping；
- Ontology Change。

Ontology Manager 与 Object Explorer、Workshop、Pipeline Builder、Functions Repository、Compass、Control Panel 和 Global Branching 共同形成完整的 Ontology 建设和运行体系。

---

# 35. 官方参考资料

1. Ontology Manager Overview  
   https://www.palantir.com/docs/foundry/ontology-manager/overview

2. Ontology Manager Navigation  
   https://www.palantir.com/docs/foundry/ontology-manager/navigation

3. Viewing Ontology Usage  
   https://www.palantir.com/docs/foundry/ontology-manager/view-usage

4. Save Changes to the Ontology  
   https://www.palantir.com/docs/foundry/ontology-manager/save-changes

5. Review and Restore Changes  
   https://www.palantir.com/docs/foundry/ontology-manager/restore-changes

6. Export, Edit, and Import an Ontology  
   https://www.palantir.com/docs/foundry/ontology-manager/export-import

7. Ontology Cleanup  
   https://www.palantir.com/docs/foundry/ontology-manager/cleanup

8. Migrate to Project-based Permissions  
   https://www.palantir.com/docs/foundry/ontology-manager/migrate-to-project-based-permissions

9. Object Types Overview  
   https://www.palantir.com/docs/foundry/object-link-types/object-types-overview

10. Create an Object Type  
    https://www.palantir.com/docs/foundry/object-link-types/create-object-type

11. Properties Overview  
    https://www.palantir.com/docs/foundry/object-link-types/properties-overview

12. Shared Properties Overview  
    https://www.palantir.com/docs/foundry/object-link-types/shared-property-overview

13. Link Types Overview  
    https://www.palantir.com/docs/foundry/object-link-types/link-types-overview

14. Value Types Overview  
    https://www.palantir.com/docs/foundry/object-link-types/value-types-overview

15. Object Type Groups  
    https://www.palantir.com/docs/foundry/object-link-types/type-groups

16. Action Types Overview  
    https://www.palantir.com/docs/foundry/action-types/overview

17. Action Rules  
    https://www.palantir.com/docs/foundry/action-types/rules

18. Interfaces Overview  
    https://www.palantir.com/docs/foundry/interfaces/interface-overview

19. Ontology Branching  
    https://www.palantir.com/docs/foundry/ontologies/branching-ontology

20. Review Ontology Proposals  
    https://www.palantir.com/docs/foundry/ontologies/review-ontology-proposals

21. Manage Published Functions  
    https://www.palantir.com/docs/foundry/functions/manage-functions

22. Object Views  
    https://www.palantir.com/docs/foundry/object-views/standard-object-views

23. Direct Datasources  
    https://www.palantir.com/docs/foundry/object-indexing/direct-datasources

24. Ontology Core Concepts  
    https://www.palantir.com/docs/foundry/ontology/core-concepts

25. Ontology Design Anti-patterns  
    https://www.palantir.com/docs/foundry/ontology/ontology-anti-patterns

## 相关笔记

- [[palantir_foundry_modules_overview|Foundry 模块全景]]
- [[palantir_quiver_deep_dive|Quiver 研究]]
- [[企业本体方法论_对外完整版_v2|企业本体方法论]]
