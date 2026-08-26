# Palantir Foundry 模块全景说明

Palantir Foundry 不是一个单纯的 BI 工具，而是一套从**数据接入、数据加工、语义建模、分析、业务应用到安全治理**的企业级数据操作平台。

Palantir 官方并没有发布一个永久固定的“模块清单”，其应用会持续调整，但可以按当前能力体系分成以下模块。

---

## 一、Foundry 核心模块总览

| 层级  | 模块          | 主要功能                            | 代表性应用/组件                                           |
| --- | ----------- | ------------------------------- | -------------------------------------------------- |
| 1   | 数据连接与接入     | 连接数据库、ERP、CRM、API、文件、对象存储和实时数据源 | Data Connection、Connectors、Streaming               |
| 2   | 数据资产管理      | 管理结构化与非结构化数据、文件、数据集和项目资源        | Datasets、Media Sets、Projects、Files                 |
| 3   | 数据集成与加工     | 清洗、关联、聚合、标准化数据，建立批处理或流式管道       | Pipeline Builder、Code Repositories、Transforms      |
| 4   | 数据目录与血缘     | 搜索数据资产、追踪上下游依赖、查看数据来源和变更影响      | Data Lineage、Catalog、Search                        |
| 5   | Ontology 本体 | 把数据转成业务对象、关系、行为和业务规则            | Ontology Manager、Object Type、Link Type、Action Type |
| 6   | 数据分析        | 表格分析、对象分析、时间序列分析、图表和探索式分析       | Contour、Quiver、Insight、Code Workbook               |
| 7   | 业务应用开发      | 基于数据和 Ontology 快速构建业务应用         | Workshop、Slate、Carbon                              |
| 8   | 工作流与业务操作    | 任务流转、审批、数据修改、规则触发和业务闭环          | Actions、Functions、Workflow Builder、Foundry Rules   |
| 9   | 地理空间分析      | 地图、空间对象、轨迹、区域和空间关系分析            | Map、Geospatial、Object Explorer                     |
| 10  | 机器学习与模型     | 模型训练、部署、推理、评估和生产反馈              | Model Studio、Code Workspaces、Model Assets          |
| 11  | 开发者平台       | 使用代码、SDK、API和自定义前端扩展 Foundry    | OSDK、Platform SDK、Functions、VS Code Extension      |
| 12  | 安全与治理       | 用户、组织、权限、字段级安全、审计和数据合规          | Projects、Roles、Groups、Markings、Organizations       |
| 13  | 运维与质量管理     | 调度、构建、数据质量、失败监控、告警和问题管理         | Schedules、Builds、Checks、Issues、Data Health         |
| 14  | 复用与发布       | 将数据管道、应用和 Ontology 能力打包复用       | Marketplace、Foundry DevOps、Products                |

---

## 二、数据连接与数据资产模块

### 1. 数据连接与接入

负责把企业不同来源的数据接入 Foundry，例如：

- Oracle、MySQL、PostgreSQL、SQL Server 等数据库
- SAP、Salesforce 等业务系统
- S3、Azure Blob 等对象存储
- REST API、文件、消息队列
- 实时流数据
- 外部数据湖和数据仓库

该模块不仅负责读取数据，也负责：

- 连接配置
- 凭据管理
- 增量同步
- Schema 发现
- 同步状态监控
- 数据源健康检查

### 2. 数据资产与项目管理

Foundry 使用 `Project` 组织数据、代码、应用和用户协作。

Project 同时也是重要的安全边界，项目内部可以包含：

- 文件夹
- 数据集
- 代码库
- 应用
- 模型
- Ontology 资源
- 分析结果

主要能力包括：

- Project 和文件夹管理
- Dataset 数据集管理
- 文件与非结构化数据管理
- 数据预览和 Schema 查看
- 标签、描述、负责人等元数据
- 数据资产搜索
- 项目成员和角色管理

---

## 三、数据管道与加工模块

### 3. Pipeline Builder

Pipeline Builder 是 Foundry 的主要可视化数据集成工具，允许开发人员和非开发人员通过拖拽或配置方式建立数据转换流程。

典型能力包括：

- 数据过滤、排序和去重
- Join 和 Union
- 聚合与窗口计算
- Schema 映射
- 数据质量规则
- 批处理和流处理
- 管道预览和调试
- 管道调度和增量构建
- 上下游依赖管理
- 失败重试和运行监控

### 4. Code Repositories

Code Repositories 是 Foundry 内置的代码开发环境，可用于开发：

- 数据转换
- Functions
- 机器学习模型
- 数据质量规则
- 自定义业务逻辑

常见语言包括：

- Python
- Java
- SQL
- TypeScript

主要能力包括：

- Git 分支和版本管理
- Pull Request 和代码审查
- 自动补全和静态检查
- Transform 预览和调试
- 依赖库管理
- 构建和发布
- 自动化测试
- CI/CD 集成

---

## 四、数据目录、血缘和质量模块

这一层主要解决以下问题：

- 数据从哪里来
- 经过了什么处理
- 谁在使用
- 出现问题会影响什么
- 数据是否完整、准确和及时

主要包括：

- 数据资产目录
- 字段和 Schema 元数据
- 上下游血缘
- 代码与数据之间的依赖
- 数据更新时间和新鲜度
- 构建历史
- 数据质量检查
- 异常和 Issue 管理
- 变更影响分析

例如，当一个源数据字段发生变化时，可以通过血缘查看它会影响哪些：

- 数据集
- Ontology 对象
- 分析结果
- 业务应用
- 机器学习模型
- 工作流

---

## 五、Ontology 本体模块

Ontology 是 Foundry 最核心、最有差异化的模块。

它位于数据资产之上，把技术数据映射成企业能够理解和操作的业务世界。很多情况下，可以将其理解为企业的数字孪生或业务操作层。

### 5. Ontology Manager

Ontology Manager 用于定义和管理 Ontology 元素，包括：

- Object Type
- Property
- Link Type
- Action Type
- Function
- Interface
- Shared Property Type
- Ontology 权限
- 数据映射
- 版本和发布

### 5.1 Object Type

定义业务实体，例如：

- 客户
- 订单
- 工厂
- 设备
- 产品
- 员工
- 供应商

Object Type 通常包括：

- 主键
- 属性
- 显示名称
- 图标
- 状态
- 数据来源
- 安全策略
- 生命周期
- 搜索配置
- 排序配置
- 默认展示配置

### 5.2 Property

对象的属性，例如：

- 订单金额
- 设备状态
- 客户名称
- 交付日期
- 库存数量

Property 通常需要定义：

- 属性名称
- 数据类型
- 是否必填
- 是否唯一
- 默认值
- 枚举值
- 显示格式
- 数据来源字段
- 权限规则
- 是否可编辑

### 5.3 Link Type

定义对象之间的业务关系，例如：

- 客户“创建”订单
- 订单“包含”产品
- 设备“位于”工厂
- 员工“负责”客户
- 供应商“提供”物料

Link Type 通常包括：

- 来源 Object Type
- 目标 Object Type
- 关系名称
- 反向关系名称
- 一对一、一对多或多对多基数
- 关系数据来源
- 关系权限
- 关系约束

### 5.4 Action Type

定义用户可以对对象执行的业务动作，例如：

- 创建订单
- 修改设备状态
- 分配负责人
- 批准申请
- 关闭告警
- 取消交付

Action 不仅可以修改数据，还可以：

- 执行业务校验
- 判断执行权限
- 调用 Function
- 写入一个或多个数据源
- 修改对象属性
- 创建或删除对象
- 创建或删除对象关系
- 触发后续业务流程
- 记录审计日志

### 5.5 Function

用于实现可复用业务逻辑，例如：

- 计算库存缺口
- 计算订单风险
- 推荐供应商
- 判断是否允许审批
- 生成生产计划
- 计算业务指标
- 查询对象集合
- 调用外部系统

### 5.6 Interface 和 Shared Property Type

用于在不同 Object Type 之间建立统一的能力和属性规范。

例如多个对象都可以实现：

- 可定位对象
- 可审批对象
- 具有负责人的对象
- 具有生命周期的对象
- 具有风险等级的对象
- 具有时间区间的对象

这有利于：

- 统一查询
- 统一应用组件
- 统一权限控制
- 统一 Action 和 Function 复用
- 降低重复建模

### 5.7 Ontology 安全

可根据用户身份、对象属性、对象关系或业务条件动态决定：

- 哪些对象可以查看
- 哪些属性可以查看
- 哪些 Action 可以执行
- 哪些对象可以修改
- 哪些关系可以访问
- 哪些对象可以导出

---

## 六、分析模块

### 6. Contour

Contour 主要用于大规模表格数据的可视化分析，包括：

- 数据过滤
- 分组聚合
- Pivot 分析
- 数据关联
- 图表生成
- 结果输出为 Dataset
- 无代码数据探索

### 7. Quiver

Quiver 更偏向基于 Ontology 对象的分析，包括：

- 对象趋势分析
- 时间序列
- 对象关系分析
- 指标计算
- 仪表板
- 对象集合分析
- 对象属性对比
- 业务状态监控

### 8. Insight

Insight 主要用于交互式、面向业务用户的数据探索与分析。

通常适用于：

- 业务人员自助分析
- 图表和仪表盘
- 指标查看
- 数据筛选
- 多维分析
- 分析结果共享

### 9. Code Workbook / Notepad

面向数据分析师和数据科学家，支持：

- SQL 分析
- Python 分析
- 数据可视化
- 临时实验
- 分析结果输出
- 模型原型验证
- 数据质量验证

---

## 七、业务应用与工作流模块

### 10. Workshop

Workshop 是 Foundry 最主要的低代码业务应用开发工具。

它直接使用 Ontology 中的对象、关系、Actions 和 Functions 构建交互式应用。

常见功能包括：

- 表格和对象列表
- 表单
- 对象详情页
- 图表
- 地图
- 筛选器
- Action 按钮
- 审批界面
- 业务工作台
- 运营驾驶舱
- 对象关系展示
- 批量业务操作

Workshop 与普通 BI 的关键区别是：

> 用户不仅可以查看数据，还可以直接执行 Action，修改业务状态，形成业务闭环。

### 11. Slate

Slate 也是应用开发工具，但比 Workshop 更偏向技术人员，支持：

- 高度定制化界面
- 自定义前端逻辑
- 数据集直接访问
- API 调用
- 更自由的页面布局
- 复杂交互逻辑

可以简单理解为：

- Workshop：面向低代码、Ontology 驱动应用
- Slate：面向高度定制化应用
- OSDK：面向完全自定义开发的外部应用

### 12. Carbon

Carbon 用于把多个 Foundry 资源组合成面向最终用户的统一工作空间，例如：

- Workshop 应用
- Slate 应用
- 分析结果
- 对象视图
- 操作入口
- 文档和说明

可以用于构建统一业务门户和角色化工作台。

### 13. Workflow Builder 与 Foundry Rules

用于定义：

- 审批流程
- 条件规则
- 任务分派
- 状态流转
- 通知
- 人工审核
- 自动化业务动作
- 超时处理
- 升级处理
- 事件触发

---

## 八、机器学习和 AI 模块

### 14. Model Studio

Model Studio 是 Foundry 的无代码或低代码机器学习开发工具，支持：

- 选择训练数据
- 选择机器学习任务
- 配置算法和参数
- 模型训练
- 模型评估
- 模型部署
- 预测结果写回
- 生产模型监控

### 15. 模型开发和模型资产

Foundry 可以管理多种模型：

- 机器学习模型
- 预测模型
- 优化模型
- 物理模型
- 业务规则模型
- 风险评分模型

模型可以在以下环境中开发：

- Code Repositories
- 代码工作区
- Model Studio
- 外部模型平台

模型可以：

- 部署为 API
- 接入 Ontology
- 被 Function 调用
- 被 Workshop 应用使用
- 参与业务工作流
- 将结果回写到数据集或对象

### 16. AIP

严格来说，AIP 是与 Foundry 深度集成的平台能力，而不是 Foundry 传统数据平台内部的单一模块。

AIP 负责把大语言模型和 AI 能力连接到：

- Foundry 数据
- Ontology
- Actions
- Functions
- 业务流程
- 企业权限体系

主要包括：

- 大模型接入
- Prompt 和 AI Logic
- Agent
- 文档检索
- 工具调用
- Ontology 对象查询
- Action 执行
- 人工审批
- 模型安全与审计
- AI 应用开发
- 多模型管理
- AI 评测和监控

---

## 九、开发者平台

### 17. Ontology SDK

OSDK 允许开发人员在自己的开发环境中访问 Ontology，并构建 Foundry 之外的自定义应用。

常见语言包括：

- TypeScript
- Java
- Python

主要能力包括：

- 查询对象
- 查询对象关系
- 查询对象集合
- 调用 Action
- 调用 Function
- 订阅数据变化
- 构建 Web 应用
- 构建移动应用
- 与外部系统集成

### 18. Platform SDK 和 API

用于管理平台资源和开发自动化能力，例如：

- Dataset
- 构建和调度
- 项目资源
- 治理工作流
- 媒体资源
- 用户和权限
- 平台管理
- 自动化运维
- 外部系统集成

### 19. Functions

Functions 用于编写低延迟业务逻辑，并直接读取 Ontology 对象。

常见用途包括：

- 实时计算
- 表单校验
- Action 前置判断
- 推荐和评分
- 应用动态数据
- 调用外部服务
- 查询对象集合
- 执行业务规则

---

## 十、安全与治理模块

Foundry 的安全不是独立附加功能，而是贯穿 Dataset、Ontology、模型和应用的底层能力。

### 20. Projects 与 Roles

Project 既是协作空间，也是主要的权限边界。

用户和用户组通过角色获得不同权限，例如：

- Viewer
- Editor
- Owner
- Developer
- Manager

### 21. Organizations、Spaces、Users 和 Groups

用于管理：

- 租户和组织边界
- 用户
- 用户组
- 工作空间
- 企业目录集成
- 统一身份认证
- 多组织隔离
- 角色分配

Organizations 提供严格的数据与工作隔离，Groups 用于批量管理权限和用户体验。

### 22. Markings

Markings 是 Foundry 非常重要的权限机制，可以给以下资源附加安全条件：

- 项目
- 文件
- Dataset
- 字段
- Ontology 对象
- 对象属性
- 应用

例如：

- 财务敏感数据
- 个人身份信息
- 医疗数据
- 特定国家可见数据
- 特定项目成员可见数据

用户只有满足全部 Marking 要求，才能访问相应资源。

### 23. 动态和属性级安全

可以实现：

- Dataset 级权限
- 行级权限
- 属性或字段级权限
- Object 级权限
- Action 执行权限
- 基于对象关系的权限
- 基于业务条件的动态权限
- 导出权限
- 下载权限
- API 调用权限

---

## 十一、运维、质量和 DevOps 模块

主要能力包括：

- 数据任务调度
- 构建管理
- 失败重试
- 运行日志
- 数据质量检查
- 数据新鲜度监控
- 告警通知
- Issue 管理
- 变更发布
- 环境管理
- 版本管理
- CI/CD
- 回滚
- 依赖影响分析
- 平台健康监控

Foundry DevOps 主要用于将资源在不同环境之间进行：

- 开发
- 测试
- 验收
- 生产发布
- 版本升级
- 配置迁移
- 回滚

---

## 十二、复用、产品化和发布模块

Foundry 支持将平台资源打包为可复用产品，例如：

- 数据产品
- 数据管道模板
- Ontology 模型
- Workshop 应用
- Functions
- 模型
- 分析模板
- 连接器

主要作用包括：

- 跨团队复用
- 标准化交付
- 模板化开发
- 降低重复建设
- 管理依赖和版本
- 形成企业内部能力市场

---

## 十三、Foundry 整体架构

```text
┌─────────────────────────────────────────────┐
│ 业务应用层                                  │
│ Workshop / Slate / Carbon / 自定义 OSDK 应用│
├─────────────────────────────────────────────┤
│ AI 与业务流程层                             │
│ AIP / Agents / Workflow / Rules / Actions   │
├─────────────────────────────────────────────┤
│ Ontology 运营语义层                         │
│ Object / Property / Link / Action / Function│
├─────────────────────────────────────────────┤
│ 分析与模型层                                │
│ Contour / Quiver / Model Studio / ML        │
├─────────────────────────────────────────────┤
│ 数据加工层                                  │
│ Pipeline Builder / Code Repositories        │
├─────────────────────────────────────────────┤
│ 数据资产层                                  │
│ Dataset / Catalog / Lineage / Quality       │
├─────────────────────────────────────────────┤
│ 平台基础层                                  │
│ Security / Governance / Compute / DevOps    │
└─────────────────────────────────────────────┘
```

---

## 十四、Foundry 最核心的六大模块

如果只看 Foundry 最不可缺少的核心能力，可以归纳为六大模块：

1. **数据接入与数据资产管理**
2. **数据管道与计算引擎**
3. **数据目录、血缘和质量治理**
4. **Ontology 语义与业务操作层**
5. **Workshop 业务应用与工作流**
6. **统一权限、安全、版本和运维体系**

---

## 十五、开发 Palantir 风格 Ontology 平台的建议优先级

如果目标不是完整复制 Foundry，而是先开发 Palantir 风格的 Ontology 平台，建议按以下顺序实现。

### P0：Ontology 核心管理能力

- Ontology Manager
- 元模型管理
- Object Type
- Property
- Link Type
- Action Type
- Function
- Interface
- Shared Property Type
- 数据映射
- Ontology 版本管理
- 发布和回滚
- 权限和审计
- Ontology API

### P1：查询与应用能力

- 对象查询
- 对象搜索
- Object Explorer
- 图关系展示
- Action 执行
- 低代码业务应用
- Workflow
- 规则引擎
- SDK

### P2：分析和智能能力

- 指标分析
- 可视化分析
- 地理空间
- 机器学习
- AIP
- Agent
- 优化和仿真
- Marketplace

---

## 十六、核心结论

真正决定一个平台能否成为“Palantir 式平台”的，不是单独的 Pipeline Builder 或图表工具，而是以下五部分共同形成的业务闭环：

> **数据资产 + Ontology + Action + 应用工作流 + 动态安全**

其完整闭环为：

```text
企业数据
   ↓
数据清洗和治理
   ↓
Ontology 业务语义建模
   ↓
对象、关系、规则和业务动作
   ↓
Workshop 或自定义业务应用
   ↓
业务人员执行 Action
   ↓
结果写回业务系统和数据平台
```

这也是 Foundry 与传统数据仓库、BI 平台和普通低代码平台之间最核心的区别。
