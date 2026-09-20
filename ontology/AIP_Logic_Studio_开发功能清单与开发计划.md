# AIP Logic Studio 开发功能清单与开发计划

> **适用前提**
>
> 已经具备一个 Foundry-like 平台底座，包括：
>
> - Project / Folder / Resource
> - Identity / Authorization
> - Ontology
> - Object Type / Link Type / Interface
> - Function / Function Runtime
> - Action / Action Runtime
> - Dataset / Data Access
> - Model Gateway / Model Catalog
> - Audit / Lineage
>
> 本文只讨论在现有 Foundry 平台之上，建设 **AIP Logic Studio** 所需要开发的功能和推荐开发顺序。

---

# 1. AIP Logic Studio 的目标

AIP Logic Studio 的目标不是再做一个聊天机器人，而是建设一个：

> **面向企业业务的 AI Logic / AI Function 低代码开发与运行平台。**

它负责把：

```text
LLM
+
Prompt
+
Ontology
+
Function
+
Action
+
业务规则
+
权限
```

组合成一个可以：

```text
开发
→ 测试
→ 调试
→ 版本化
→ 发布
→ 被其他应用复用
```

的 AI Logic Function。

例如：

```text
AnalyzeServerConfiguration()
AnalyzeSupplierRisk()
GenerateQuotationRecommendation()
DiagnoseServerFailure()
RecommendAlternativePart()
```

这些都应该最终成为平台中的可复用 Function。

---

# 2. 总体架构

AIP Logic Studio 建议只划分为四个大模块：

```text
┌──────────────────────────────────────┐
│          AIP Logic Studio            │
│                                      │
│ Logic Design                         │
│ Prompt / Model / Tool Configuration  │
│ Run / Debug                          │
│ Version / Publish                    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│          AIP Logic Runtime           │
│                                      │
│ Logic Execution                      │
│ LLM Execution                        │
│ Tool Calling                         │
│ Function / Action Execution          │
│ Runtime Context                      │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│          Existing Foundry            │
│                                      │
│ Project / Authorization              │
│ Ontology                             │
│ Function / Action                    │
│ Model Gateway                        │
│ Data / Audit                         │
└──────────────────┬───────────────────┘
                   │
                   ▼
        Enterprise Data / Systems
```

核心原则：

> **AIP Logic Runtime 不直接绕开 Foundry 去操作数据库。**

应该通过：

```text
AIP Logic
   ↓
Ontology / Function / Action
   ↓
Authorization / Audit
   ↓
Data
```

---

# 3. 一期开发功能总览

建议将功能分成以下 12 个模块：

```text
01 Logic Resource Management
02 Input / Output Type System
03 Logic Designer
04 Core Blocks
05 LLM Block
06 Tool System
07 Binding & Variable System
08 Logic Compiler
09 Logic Runtime
10 Permission & Security
11 Run / Debug / Trace
12 Version / Publish
```

二期再增加：

```text
13 Loop / Parallel
14 Staged Write
15 Evals
16 Metrics
17 Branching
18 Automation
19 Human Approval
20 Custom Node SDK
```

---

# 4. 模块 1：Logic Resource Management


> **一句话说明：负责把每一个 AI Logic 当作 Foundry 中可管理、可检索、可归属、可追踪依赖的正式资源。**
## 4.1 目标

把每一个 AI Logic 当作 Foundry 中正式的一类 Resource 管理。

例如：

```text
Project
  └── Logic
       └── AnalyzeSupplierRisk
```

## 4.2 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Create Logic | 创建新的 Logic | P0 |
| Logic Name | 设置显示名称 | P0 |
| API Name | 设置稳定的程序标识 | P0 |
| Description | Logic 业务说明 | P0 |
| Project Binding | Logic 归属 Project | P0 |
| Owner | Logic Owner | P0 |
| Status | Draft / Published / Deprecated | P0 |
| Save Draft | 保存开发中版本 | P0 |
| Copy Logic | 复制已有 Logic | P1 |
| Delete / Archive | 删除或归档 | P1 |
| Tags | 分类与检索 | P1 |
| Dependency View | 查看依赖资源 | P1 |
| Used By | 查看被哪些应用/Agent/Logic 使用 | P1 |

---

# 5. 模块 2：Input / Output 与类型系统


> **一句话说明：负责定义 Logic 的输入、输出和强类型约束，让 AI Logic 具备稳定的软件接口契约。**
这是 AIP Logic 的核心基础。

## 5.1 Input

例如：

```text
AnalyzeServerConfiguration

Input
├── requirement: String
├── customer: Customer
├── budget: Decimal
└── deliveryDate: Date
```

## 5.2 Output

```text
Output
└── result: ServerConfigurationRecommendation
```

## 5.3 类型系统

至少支持：

```text
Primitive
├── String
├── Integer
├── Long
├── Decimal
├── Boolean
├── Date
└── Timestamp

Complex
├── Struct
├── List
└── Map

Ontology
├── Object
├── Object List
└── Object Set
```

## 5.4 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Input Definition | 定义输入参数 | P0 |
| Output Definition | 定义输出 | P0 |
| Required / Optional | 是否必填 | P0 |
| Default Value | 默认值 | P1 |
| Primitive Type | 基础类型 | P0 |
| Struct Type | 结构化类型 | P0 |
| List Type | 列表 | P0 |
| Ontology Object Type | Ontology Object | P0 |
| Object List | Object 集合 | P0 |
| Object Set | ObjectSet | P1 |
| Type Validation | 类型校验 | P0 |
| Output Schema | 输出结构 Schema | P0 |

---

# 6. 模块 3：Logic Designer


> **一句话说明：负责通过低代码界面可视化编排 Logic 的执行步骤、参数和流程结构。**
## 6.1 目标

提供低代码 Logic 编排界面。

建议第一版采用“受控流程式”设计，不要一开始做完全自由 DAG。

```text
Input
  ↓
Block
  ↓
Block
  ↓
Conditional
 ├── Path A
 └── Path B
  ↓
Output
```

## 6.2 UI 结构

建议：

```text
┌──────────────────────────────────────────────┐
│ Logic Name                     Run | Publish │
├──────────────┬─────────────────┬─────────────┤
│ Block List   │ Logic Canvas    │ Properties  │
│              │                 │             │
│ LLM          │ Input           │ Config      │
│ Function     │   ↓             │             │
│ Action       │ LLM             │             │
│ Condition    │   ↓             │             │
│ Variable     │ Function        │             │
│              │                 │             │
├──────────────┴─────────────────┴─────────────┤
│ Run / Debug / Trace                         │
└──────────────────────────────────────────────┘
```

## 6.3 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Block Palette | Block 列表 | P0 |
| Add Block | 新增 Block | P0 |
| Delete Block | 删除 Block | P0 |
| Reorder Block | 调整顺序 | P0 |
| Copy Block | 复制 Block | P1 |
| Config Panel | Block 属性配置 | P0 |
| Variable Picker | 变量选择 | P0 |
| Type Hint | 类型提示 | P0 |
| Validation Error | 错误提示 | P0 |
| Undo / Redo | 撤销恢复 | P1 |
| Search Resource | 搜索 Function / Action / Object | P1 |

---

# 7. 模块 4：Core Blocks


> **一句话说明：负责提供构成 Logic 的基础执行单元，例如 LLM、Function、Action、条件判断和变量。**
第一版建议只做最核心的 Block。

## 7.1 P0 Block

```text
Use LLM
Execute Function
Apply Action
Conditional
Create Variable
```

## 7.2 P1 Block

```text
Loop
Transform
```

## 7.3 后续

```text
Parallel
Wait
Human Approval
Sub Logic
Custom Block
```

## 7.4 Block 元模型

建议统一：

```text
BlockDefinition
│
├── blockId
├── blockType
├── displayName
├── inputs
├── outputs
├── config
├── timeout
├── retryPolicy
└── errorPolicy
```

---

# 8. 模块 5：Use LLM Block


> **一句话说明：负责配置并执行大模型推理，包括模型、Prompt、变量、Tool 和结构化输出。**
这是整个 AIP Logic 最重要的 Block。

## 8.1 功能模型

```text
LLM Block
│
├── Model
├── System Prompt
├── User Prompt
├── Input Binding
├── Tools
├── Output Schema
└── Model Parameters
```

## 8.2 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Model Selector | 从 Model Catalog 选择模型 | P0 |
| System Prompt | 系统提示词 | P0 |
| Prompt Editor | 用户 Prompt | P0 |
| Variable Insert | 引用 Logic Input / Block Output | P0 |
| Object Property Insert | 引用 Object Property | P0 |
| Prompt Preview | 查看最终 Prompt | P0 |
| Structured Output | 结构化返回 | P0 |
| Output Schema | JSON / Struct Schema | P0 |
| Tool Binding | 配置 Tools | P0 |
| Tool Description | Tool 使用说明 | P0 |
| Model Parameters | Temperature 等 | P1 |
| Token Limit | Token 限制 | P1 |
| Fallback Model | 备用模型 | P2 |

---

# 9. 模块 6：Tool System


> **一句话说明：负责让 LLM 在受控权限下安全调用 Ontology 查询、Function、Action 等企业能力。**
LLM Tool 是 AIP Logic 和普通 Prompt Workflow 最大的区别之一。

第一版至少支持：

```text
Query Objects
Call Function
Apply Action
Calculator
```

---

## 9.1 Query Objects Tool

作用：

```text
LLM
 ↓
Query Object
 ↓
Ontology Runtime
```

例如查询：

```text
Server
BOM
Component
Order
Supplier
Inventory
WorkOrder
Customer
```

功能：

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Object Type Selector | 选择 Object Type | P0 |
| Property Allowlist | 允许访问的 Property | P0 |
| Filter | 查询过滤 | P0 |
| Result Limit | 返回数量限制 | P0 |
| Authorization | 查询权限 | P0 |
| Link Traversal | 沿 Link 查询 | P1 |
| Aggregation | Sum / Count / Avg | P1 |

---

## 9.2 Call Function Tool

```text
LLM
 ↓
Function Tool
 ↓
Function Runtime
```

功能：

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Function Selector | 选择 Function | P0 |
| Signature Discovery | 获取参数定义 | P0 |
| Parameter Mapping | 参数映射 | P0 |
| Output Mapping | 结果返回 LLM | P0 |
| Permission Check | Function 权限 | P0 |
| Call Logic Function | 调用其他 Logic | P1 |

---

## 9.3 Apply Action Tool

```text
LLM
 ↓
Action Tool
 ↓
Authorization
 ↓
Action Runtime
 ↓
Ontology
```

功能：

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Action Selector | 选择 Action | P0 |
| Action Signature | 获取 Action 参数 | P0 |
| Parameter Mapping | 参数映射 | P0 |
| Permission Check | Action 权限校验 | P0 |
| Audit | 记录 Action | P0 |
| Edit Preview | 修改预览 | P1 |
| Staged Write | 暂存写入 | P1 |

---

# 10. 模块 7：Binding 与 Variable System


> **一句话说明：负责管理 Logic 各步骤之间的数据传递、变量引用、作用域和类型匹配。**
## 10.1 核心目标

解决 Block 之间数据如何传递。

```text
QuerySupplier.output.supplier
          ↓
AnalyzeRisk.input.supplier
```

## 10.2 Binding

```text
InputBinding
│
├── sourceType
├── sourceBlockId
├── sourceOutput
├── targetBlockId
├── targetInput
└── expression
```

## 10.3 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Variable Registry | 当前可用变量 | P0 |
| Input Binding | Logic Input → Block | P0 |
| Block Binding | Block → Block | P0 |
| Property Binding | Object Property | P0 |
| Type Validation | 类型检查 | P0 |
| Scope | 变量作用域 | P0 |
| Null Handling | Null / Optional | P1 |
| List Element | Loop 元素变量 | P1 |
| Expression | 简单表达式 | P1 |

---

# 11. 模块 8：Logic Compiler


> **一句话说明：负责在 Logic 真正运行前完成结构、类型、依赖、权限和流程合法性检查，并生成可执行计划。**
Compiler 是自研平台非常关键的一层。

Designer 保存的定义不建议直接执行。

```text
LogicDefinition
      ↓
Validation
      ↓
Type Check
      ↓
Binding Check
      ↓
Dependency Check
      ↓
Permission Check
      ↓
Compile
      ↓
ExecutionPlan
```

## 11.1 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Schema Validation | Logic 结构检查 | P0 |
| Block Validation | Block 配置检查 | P0 |
| Type Check | 类型检查 | P0 |
| Binding Check | Binding 检查 | P0 |
| Function Signature Check | Function 参数检查 | P0 |
| Action Signature Check | Action 参数检查 | P0 |
| Model Validation | 模型是否可用 | P0 |
| Tool Validation | Tool 合法性 | P0 |
| Permission Validation | 权限检查 | P0 |
| Control Flow Validation | 分支合法性 | P0 |
| Dependency Resolution | 依赖分析 | P1 |
| Execution Plan | 生成执行计划 | P0 |

---

# 12. 模块 9：Logic Runtime


> **一句话说明：负责按照执行计划真正运行 Logic，并调度 LLM、Function、Action 和控制流。**
这是系统后台核心。

## 12.1 Runtime 架构

```text
Logic Runtime
│
├── Run Manager
├── Execution Engine
├── Execution Context
├── Variable Context
└── Block Executor Registry
     │
     ├── LLM Executor
     ├── Function Executor
     ├── Action Executor
     ├── Conditional Executor
     ├── Variable Executor
     └── Loop Executor
```

## 12.2 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Execute Logic | 执行 Logic | P0 |
| Run Manager | Run 生命周期管理 | P0 |
| Execution Context | 用户/Project/Version 上下文 | P0 |
| Variable Context | Runtime 变量 | P0 |
| Sequential Execution | 顺序执行 | P0 |
| Conditional | 条件执行 | P0 |
| Function Executor | 执行 Function | P0 |
| Action Executor | 执行 Action | P0 |
| LLM Executor | 调用 LLM | P0 |
| Error Handling | 错误处理 | P0 |
| Timeout | Timeout | P0 |
| Retry | Retry Policy | P1 |
| Cancel | Cancel Run | P1 |
| Loop | 循环 | P1 |
| Parallel | 并行 | P2 |

---

# 13. 模块 10：Permission & Security


> **一句话说明：负责确保 Logic 继承 Foundry 的用户、项目、对象、函数、Action 和模型权限，不发生越权执行。**
AIP Logic 必须继承 Foundry 的权限体系。

## 13.1 P0

```text
User-scoped Execution
```

执行链：

```text
User
 ↓
Logic
 ↓
Query / Function / Action
 ↓
以 User 权限执行
```

## 13.2 P1

```text
Project-scoped Execution
```

## 13.3 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| User-scoped Execution | 使用调用用户权限 | P0 |
| Principal Propagation | 身份向下传递 | P0 |
| Object Permission | Object 权限 | P0 |
| Function Permission | Function 权限 | P0 |
| Action Permission | Action 权限 | P0 |
| Model Permission | 模型权限 | P0 |
| Tool Allowlist | Tool 白名单 | P0 |
| Audit | 全链路审计 | P0 |
| Sensitive Data Policy | Prompt 敏感数据控制 | P1 |
| Project-scoped Execution | Project 身份运行 | P1 |

---

# 14. 模块 11：Run / Debug / Trace


> **一句话说明：负责测试运行 Logic，并完整记录每一步输入、输出、Prompt、Tool 调用、错误和执行链路。**
建议 V1 就开发。

没有 Debugger，AIP Logic 很难进入生产。

## 14.1 Run

```text
Test Input
   ↓
Run
   ↓
Logic Runtime
   ↓
Output
```

## 14.2 Debugger

需要展示：

```text
LogicRun
│
├── Input
├── BlockRun 1
│   ├── Input
│   ├── Output
│   └── Duration
│
├── LLM Block
│   ├── Model
│   ├── Rendered Prompt
│   ├── Response
│   ├── Tool Calls
│   └── Token Usage
│
├── Function Block
├── Action Block
├── Error
└── Output
```

## 14.3 功能清单

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Run Draft | 执行 Draft | P0 |
| Input Form | 输入测试数据 | P0 |
| Run Status | 执行状态 | P0 |
| Final Output | 最终结果 | P0 |
| Logic Trace | 完整 Trace | P0 |
| Block Trace | Block 执行详情 | P0 |
| Input Snapshot | Block 输入 | P0 |
| Output Snapshot | Block 输出 | P0 |
| Rendered Prompt | 最终 Prompt | P0 |
| LLM Response | LLM 响应 | P0 |
| Tool Call Trace | Tool 调用 | P0 |
| Function Trace | Function 调用 | P0 |
| Action Trace | Action 调用 | P0 |
| Error Detail | 错误详情 | P0 |
| Duration | 耗时 | P1 |
| Token Usage | Token 使用 | P1 |
| Cost | 模型成本 | P1 |
| Run History | 历史运行 | P0 |

---

# 15. 模块 12：Version / Publish


> **一句话说明：负责管理 Logic 的草稿、历史版本和正式发布，并将发布后的 Logic 注册为可复用 Function。**
AIP Logic 必须像普通软件一样管理版本。

## 15.1 Version

```text
LogicDefinition
│
├── Draft
├── Version 1
├── Version 2
└── Version 3
```

功能：

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Save Draft | 保存草稿 | P0 |
| Version History | 历史版本 | P0 |
| Published Version | 发布版本 | P0 |
| Immutable Published Version | 发布后不可直接修改 | P0 |
| Compare | 版本 Diff | P1 |
| Rollback | 回滚 | P1 |
| Compatibility Check | Breaking Change 检查 | P1 |

---

## 15.2 Publish

发布流程：

```text
Draft
 ↓
Validate
 ↓
Compile
 ↓
Test
 ↓
Publish
 ↓
Logic Function
```

发布后的 Logic 应注册到平台：

```text
Foundry Function Registry
```

然后可以被：

```text
Agent
Application
Action
Other Logic
API
Automation
```

调用。

功能：

| 功能 | 描述 | 优先级 |
|---|---|---:|
| Publish Validation | 发布前检查 | P0 |
| Publish | 发布 | P0 |
| Function Registration | 注册为 Function | P0 |
| Version Number | 版本号 | P0 |
| API Signature | Input / Output Signature | P0 |
| Deprecate | 废弃旧版本 | P1 |
| Usage View | 查看调用方式 | P1 |

---

# 16. 二期功能


> **一句话说明：负责在核心闭环稳定后补充循环、并行、事务写入、评估、监控、分支协作和人工审批等企业增强能力。**
核心闭环稳定之后，再增加以下能力。

---

## 16.1 Loop / Parallel

```text
List
 ↓
Loop
 ├── Item 1
 ├── Item 2
 └── Item N
```

支持：

- Loop Variable
- Index
- Parallel Execution
- Error Handling
- Result Collection

---

## 16.2 Staged Write

如果 Logic 会修改 Ontology，建议增加：

```text
Logic Run
  ↓
Action A
  ↓
Stage Edit
  ↓
Action B
  ↓
Stage Edit
  ↓
Success
  ↓
Commit
```

失败：

```text
Rollback
```

功能：

- Staged Edit
- Read-your-writes
- Commit
- Rollback
- Edit Preview

---

## 16.3 AIP Evals

用于测试 AI Logic。

```text
EvaluationSuite
├── TestCase
├── Target Logic
├── Evaluator
└── Metrics
```

功能：

- Test Case
- Save Run as Test
- Batch Eval
- Model Comparison
- Version Comparison
- Repeated Runs
- Metrics
- LLM Judge
- Regression Gate

---

## 16.4 Metrics / Observability

建议增加：

```text
Success Rate
Failure Rate
P50 / P95 Duration
Token Usage
Model Usage
Tool Usage
Cost
Failure Category
```

---

## 16.5 Branching

如果 Foundry 有 Branching：

```text
main
 └── feature-logic-v2
```

Logic 应与：

```text
Object Type
Function
Action
```

一起 Branch。

---

## 16.6 Human Approval

高风险 Action：

```text
AI Logic
 ↓
Prepare Action
 ↓
Human Review
 ↓
Approve / Reject
 ↓
Execute
```

---

## 16.7 Custom Node SDK

允许平台团队扩展：

```text
HTTP Node
Search Node
SAP Node
Vector Search Node
OCR Node
Custom Business Node
```

但建议放到后期。

---

# 17. V1 功能范围


> **一句话说明：用于明确第一版必须完成的最小可生产功能边界，防止项目范围失控。**
第一版建议严格控制范围。

| 功能 | V1 |
|---|---:|
| Logic Resource | ✅ |
| Input / Output | ✅ |
| Strong Type System | ✅ |
| Logic Designer | ✅ |
| Use LLM Block | ✅ |
| Execute Function | ✅ |
| Apply Action | ✅ |
| Conditional | ✅ |
| Create Variable | ✅ |
| Query Object Tool | ✅ |
| Function Tool | ✅ |
| Action Tool | ✅ |
| Variable / Binding | ✅ |
| Logic Compiler | ✅ |
| Logic Runtime | ✅ |
| User-scoped Authorization | ✅ |
| Run | ✅ |
| Debugger / Trace | ✅ |
| Run History | ✅ |
| Version | ✅ |
| Publish as Function | ✅ |
| Loop | V1.1 |
| Project-scoped Execution | V1.1 |
| Staged Write | V1.1 |
| Evals | V1.1 |
| Metrics | V1.1 |
| Parallel | V2 |
| Branching | V2 |
| Automation | V2 |
| Human Approval | V2 |
| Custom Node SDK | V2 |

---

# 18. 推荐开发顺序


> **一句话说明：用于规定 AIP Logic 的正确建设先后关系，优先打通元模型和运行时，再建设低代码界面。**
不要从 UI 开始。

推荐顺序：

```text
Meta-Model
   ↓
Runtime
   ↓
Core Executors
   ↓
Compiler
   ↓
Trace / Debug
   ↓
Version / Publish
   ↓
Studio UI
   ↓
Evals
```

---

# 19. Phase 0：前置确认


> **一句话说明：负责确认现有 Foundry 底座是否已经具备 AIP Logic 所需的权限、Ontology、Function、Action、模型和审计能力。**
## 目标

确认 Foundry 已有底座是否满足 AIP Logic 建设要求。

需要检查：

```text
Project
Resource
Authorization
Ontology Query
Function Runtime
Action Runtime
Model Gateway
Audit
```

## 交付

```text
AIP Logic Integration Checklist
```

明确：

- 哪些能力可以复用；
- 哪些能力需要补齐；
- API / SDK 接口；
- Authorization Contract；
- Function / Action Signature Contract。

---

# 20. Phase 1：Meta-Model Specification


> **一句话说明：负责先定义 AIP Logic 的核心领域模型、状态和接口契约，为后续 Runtime 开发打基础。**
这是第一份必须完成的开发 Spec。

## 核心模型

```text
LogicDefinition
LogicVersion
LogicInput
LogicOutput
BlockDefinition
Binding
ToolDefinition
ExecutionPlan
LogicRun
BlockRun
```

## 同时定义

```text
BlockType
TypeSystem
ExecutionStatus
VersionStatus
ErrorModel
PermissionContext
```

## 交付

> 《AI Logic Meta-Model & Runtime Specification》

在这一阶段，不开发复杂 UI。

---

# 21. Phase 2：Runtime MVP


> **一句话说明：负责先实现一个不依赖可视化 UI、能够直接执行 Logic Definition 的最小运行时。**
这是整个项目最关键的阶段。

先支持 JSON Logic Definition。

例如：

```json
{
  "name": "AnalyzeSupplierRisk",
  "inputs": [],
  "blocks": [],
  "outputs": []
}
```

Runtime 完成：

```text
Load
 ↓
Validate
 ↓
Execute
 ↓
Trace
 ↓
Return Result
```

## 第一批 Executor

```text
LLMExecutor
FunctionExecutor
ActionExecutor
ConditionalExecutor
VariableExecutor
```

## 交付标准

即使没有 Logic Designer UI，也能够通过 API 执行完整 Logic。

---

# 22. Phase 3：Tool Runtime


> **一句话说明：负责把 LLM Tool Calling 安全连接到 Foundry 的 Object Query、Function 和 Action 能力。**
开发：

```text
QueryObjectTool
FunctionTool
ActionTool
CalculatorTool
```

同时完成：

```text
Tool Registry
Tool Schema
Permission Check
Tool Call Trace
```

重点保证：

```text
LLM
不能直接调用数据库
```

而只能：

```text
LLM
 ↓
Tool Runtime
 ↓
Foundry Authorization
 ↓
Ontology / Function / Action
```

---

# 23. Phase 4：Compiler


> **一句话说明：负责把用户设计的 Logic 转换成经过校验、可以被 Runtime 稳定执行的 Execution Plan。**
在 Runtime 能跑起来之后开发 Compiler。

完成：

```text
Schema Validation
Type Check
Binding Check
Function Signature Check
Action Signature Check
Permission Validation
Control Flow Validation
ExecutionPlan Generation
```

Compiler 完成后，才建议正式开发低代码 UI。

---

# 24. Phase 5：Run / Trace / Debugger


> **一句话说明：负责建立完整运行追踪和调试能力，让开发者能够看清每一次 Logic 执行发生了什么。**
开发完整运行观测能力。

模型：

```text
LogicRun
  │
  └── BlockRun[]
```

每个 BlockRun 保存：

```text
Input
Output
Status
StartTime
EndTime
Error
TraceId
```

LLM Block 额外保存：

```text
Model
Rendered Prompt
Response
Tool Calls
Token Usage
```

---

# 25. Phase 6：Version / Publish


> **一句话说明：负责把开发中的 Logic 版本化并正式发布为平台可复用的 Function。**
开发：

```text
Draft
Version
Published Version
Rollback
```

以及：

```text
Publish Logic
   ↓
Register Logic Function
```

完成后，其他平台模块才能正式调用 Logic。

---

# 26. Phase 7：Logic Studio UI


> **一句话说明：负责在后端能力稳定后提供低代码可视化编辑、配置、运行和发布界面。**
这时候再开发 Visual Studio。

推荐前端：

```text
React
TypeScript
React Flow
```

第一版 UI 重点：

```text
Logic List
Logic Editor
Input / Output
Block Editor
LLM Config
Tool Config
Variable Picker
Run Panel
Debugger
Version
Publish
```

不需要一开始就做复杂动画、自由布局和大量 UI 装饰。

---

# 27. Phase 8：V1.1 增强


> **一句话说明：负责在 V1 核心闭环之后补齐循环、项目级执行、Staged Write、Evals 和 Metrics 等重要能力。**
核心平台稳定后增加：

```text
Loop
Project-scoped Execution
Staged Write
Evals
Metrics
Version Diff
Rollback
```

---

# 28. Phase 9：V2 企业增强


> **一句话说明：负责建设并行、Branching、Automation、Human Approval 和自定义节点等更高级的平台能力。**
继续建设：

```text
Parallel
Branching
Automation
Human Approval
Custom Node SDK
Advanced Cost Control
Advanced Governance
```

---

# 29. 推荐里程碑


> **一句话说明：用于定义每个阶段必须达到的可验收结果，确保项目按能力闭环而不是按页面数量推进。**
## Milestone 1：Meta-Model 完成

完成：

```text
LogicDefinition
BlockDefinition
Binding
ToolDefinition
LogicRun
```

### 验收

所有核心概念有明确 Schema 和 API Contract。

---

## Milestone 2：Runtime MVP 完成

完成：

```text
LLM
Function
Action
Condition
Variable
```

### 验收

无 UI 情况下能执行完整 Logic。

---

## Milestone 3：Tool Runtime 完成

### 验收

LLM 能通过 Tool：

```text
Query Object
Call Function
Apply Action
```

且正确继承用户权限。

---

## Milestone 4：Compiler / Debug 完成

### 验收

能：

```text
Validate
Compile
Run
Trace
Debug
```

---

## Milestone 5：Studio MVP 完成

### 验收

用户无需写 JSON，可以通过 UI 创建和运行 Logic。

---

## Milestone 6：Publish 完成

### 验收

Logic 可以发布为 Function，并被其他应用调用。

---

## Milestone 7：V1 完成

完整闭环：

```text
Create
 ↓
Design
 ↓
Validate
 ↓
Compile
 ↓
Run
 ↓
Debug
 ↓
Version
 ↓
Publish
 ↓
Reuse
```

---

# 30. 建议一期业务验证场景


> **一句话说明：用于通过服务器配置、供应链风险和故障诊断等真实业务场景验证 AIP Logic 平台能力。**
对于服务器组装企业，建议使用真实业务场景验证平台能力。

---

## 30.1 Server Configuration Logic

```text
Customer Requirement
 ↓
Query Product / BOM
 ↓
Check Compatibility
 ↓
Check Inventory
 ↓
Calculate Cost
 ↓
LLM Recommend
 ↓
Output Configuration
```

用于验证：

```text
Ontology Query
Function
LLM
Structured Output
```

---

## 30.2 Supply Risk Logic

```text
Order
 ↓
BOM
 ↓
Inventory
 ↓
Supplier
 ↓
Function calculateSupplyRisk
 ↓
LLM Analysis
 ↓
Risk Result
```

用于验证：

```text
Object Link
Function
LLM
Condition
```

---

## 30.3 Server Failure Diagnosis Logic

```text
Server Serial Number
 ↓
Configuration
 ↓
Test History
 ↓
Repair History
 ↓
LLM
 ↓
Diagnosis
```

用于验证：

```text
Ontology Query
Tool Calling
Long Context
Structured Output
```

---

# 31. 不建议一期做的内容


> **一句话说明：用于明确一期暂缓建设的复杂能力，避免过早扩大范围和拖慢核心平台落地。**
为了防止项目失控，一期不要优先开发：

```text
完整 Agent Studio
完整 App Builder
复杂 BPM
复杂 Human Workflow
大量 Integration Connector
自由无限 DAG
复杂 Durable Workflow
完整 Marketplace
```

这些都可以在 AIP Logic 核心稳定后再做。

---

# 32. 最终开发顺序总结


> **一句话说明：用于再次明确整个项目从 Meta-Model 到 Runtime、Studio、Evals 和企业增强的整体建设路线。**
建议严格按照：

```text
① Meta-Model
      ↓
② Runtime
      ↓
③ Core Executors
      ↓
④ Tool Runtime
      ↓
⑤ Compiler
      ↓
⑥ Run / Trace / Debug
      ↓
⑦ Version / Publish
      ↓
⑧ Logic Studio UI
      ↓
⑨ Evals / Metrics
      ↓
⑩ Enterprise Enhancements
```

不要按照：

```text
先做一个很漂亮的拖拽页面
      ↓
再想后台怎么执行
```

这种顺序开发。

---

# 33. V1 最终完成标准


> **一句话说明：用于定义第一版真正完成的判定条件，即从创建、运行、调试到发布和复用形成完整闭环。**
AIP Logic V1 完成时，必须至少能够做到：

```text
Create Logic
    ↓
Define Input / Output
    ↓
Add Blocks
    ↓
Configure LLM
    ↓
Configure Tools
    ↓
Bind Variables
    ↓
Validate
    ↓
Compile
    ↓
Run
    ↓
Debug
    ↓
Save Version
    ↓
Publish
    ↓
Register as Function
    ↓
Called by App / Agent / Other Logic
```

做到这个闭环，才可以认为：

> **现有 Foundry 平台已经真正拥有了一个可生产使用的 AIP Logic Studio。**

---
