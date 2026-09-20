# Ontology 平台中 Project 的可扩展设计

## 1. 结论

如果当前正在开发一个 Ontology Management Platform，Project 第一阶段完全可以只支持 Ontology 管理。

但从底层模型设计上，建议不要把 Project 定义成“Ontology 专属容器”，而应该从一开始就把它设计成：

> **Project = 通用 Resource Governance Container（通用资源治理容器）**

当前阶段只开放 Ontology Resources，未来再逐步增加 Dataset、Pipeline、Repository、Application、Workflow 等其他资源类型。

这样可以保证平台未来从：

```text
Ontology Management Platform
```

平滑演进到：

```text
Foundry-like Platform
```

而不需要推翻 Project、Space、Resource、Permission 等核心模型。

---

## 2. 第一版推荐的抽象

第一版可以采用：

```text
Space
│
├── Ontology
│
└── Project
     │
     ├── Resource
     │    ├── Object Type
     │    ├── Link Type
     │    ├── Action Type
     │    ├── Interface
     │    └── Shared Property
     │
     ├── Members
     ├── Roles
     └── Permissions
```

此时 Project 只承载 Ontology 相关资源。

但是 Project 本身的定义应该保持通用。

---

## 3. 未来扩展后的形态

未来可以直接扩展为：

```text
Space
│
├── Ontology
│
└── Project
     │
     ├── Resource
     │    │
     │    ├── Ontology Resources
     │    │    ├── Object Type
     │    │    ├── Link Type
     │    │    ├── Action Type
     │    │    └── Interface
     │    │
     │    ├── Data Resources
     │    │    ├── Dataset
     │    │    ├── Table
     │    │    └── Data Source
     │    │
     │    ├── Engineering Resources
     │    │    ├── Repository
     │    │    ├── Pipeline
     │    │    └── Job
     │    │
     │    └── Application Resources
     │         ├── Application
     │         ├── Workflow
     │         └── Dashboard
     │
     ├── Members
     ├── Roles
     └── Permissions
```

Project 本身不需要变化，只需要不断增加新的 Resource 类型。

---

## 4. 不建议的设计方式

不建议把 Project 设计成 Ontology 专属实体，例如：

```text
ontology_project
----------------
id
ontology_id
name
```

也不建议：

```text
project
----------------
id
ontology_id
object_type_ids
link_type_ids
action_type_ids
```

这种设计把 Project 与 Ontology 强耦合。

未来如果要加入：

```text
Dataset
Pipeline
Repository
Application
```

就会发现 Project 从一开始就被限定成“只能管理 Ontology”，从而需要进行较大重构。

---

## 5. 更推荐的方式：统一 Resource 抽象

建议从第一版就建立一个统一的 Resource 抽象。

例如：

```text
project
-------------------------
id
space_id
name
description
status
created_by
created_at
updated_at
```

Project 本身不要强绑定：

```text
ontology_id
```

而通过 Resource 关联具体资源。

例如：

```text
resource
-------------------------
id
space_id
project_id
resource_type
resource_rid
name
created_at
```

其中：

```text
resource_type
```

第一版只需要支持：

```text
OBJECT_TYPE
LINK_TYPE
ACTION_TYPE
INTERFACE
SHARED_PROPERTY
```

未来可以增加：

```text
DATASET
DATA_SOURCE
PIPELINE
REPOSITORY
APPLICATION
WORKFLOW
DASHBOARD
```

这样 Project 数据模型本身无需改变。

---

## 6. Ontology Resource 本身单独建模

例如 Object Type 可以单独建立：

```text
object_type
-------------------------
id
ontology_id
api_name
display_name
description
...
```

而 Project 归属通过通用 Resource 层管理：

```text
resource
-------------------------
id
project_id
resource_type
resource_id
```

例如：

```text
resource

id = R001
project_id = HR_PROJECT
resource_type = OBJECT_TYPE
resource_id = EMPLOYEE_OBJECT_TYPE_ID
```

这意味着：

```text
Employee
```

在语义上属于：

```text
Enterprise Ontology
```

而：

```text
resource.project_id
```

表示：

```text
Employee 当前由 HR Project 管理
```

这个分离非常重要。

---

## 7. 推荐的三层模型

可以采用：

```text
Resource Definition
        ↓
Resource Registration
        ↓
Project Governance
```

例如：

```text
ObjectType
------------------
id
ontology_id
api_name
...

        ↓

Resource
------------------
id
resource_type
resource_id

        ↓

ProjectResource
------------------
project_id
resource_id
```

如果未来希望一个 Resource 可以同时出现在多个 Project 的不同视图或协作空间中，这种模型扩展性更强。

如果明确一个 Resource 永远只能归属一个 Project，则可以进一步简化为：

```text
Resource
------------------
id
project_id
resource_type
resource_id
```

第一版实现成本更低。

---

## 8. Project 权限模型也要保持通用

不要分别建立：

```text
object_type_permission
link_type_permission
action_type_permission
```

否则未来容易继续产生：

```text
dataset_permission
pipeline_permission
repository_permission
```

更合理的方式是统一采用：

```text
project_member
---------------------
project_id
principal_id
principal_type
role
```

例如：

```text
HR Project
│
├── HR Ontology Team       OWNER
├── HR Developers          EDITOR
└── HR Analysts            VIEWER
```

Project 中的 Resource 默认继承 Project Role。

未来当 Dataset、Pipeline 等资源进入 Project 时：

```text
HR Project
│
├── Employee Object Type
├── Department Object Type
├── Employee Dataset
└── HR Pipeline
```

原有：

```text
OWNER
EDITOR
VIEWER
```

权限体系仍然可以继续使用。

---

## 9. Resource 级权限 Override 建议提前预留

第一版可能只实现：

```text
Project Permission
```

但建议权限模型中提前预留：

```text
Resource Permission Override
```

例如：

```text
HR Project
    HR Developers = Editor
```

但其中：

```text
Compensation Object Type
```

可能具有更严格权限，只允许：

```text
Compensation Team = Editor
```

未来可以形成：

```text
Project Role
        ↓
Default Permission
        ↓
Resource Override
```

例如：

```text
project_permission
resource_permission_override
```

第一版前端和业务能力可以暂时不开放，但底层模型不要把扩展路径堵死。

---

## 10. Space 也不要只为 Ontology 服务

不建议建模成：

```text
Ontology
└── Project
```

更推荐：

```text
Space
├── Ontology
├── Project
├── Project
└── Project
```

即：

```text
Space
```

是更高层的治理边界。

```text
Ontology
```

和：

```text
Project
```

都是 Space 内的重要实体，但不是简单的父子语义关系。

当前：

```text
Space
│
├── Ontology
│
└── Projects
     ├── HR Ontology Project
     ├── Procurement Ontology Project
     └── Finance Ontology Project
```

未来可以演进为：

```text
Space
│
├── Ontology
│
└── Projects
     ├── HR Semantic Project
     ├── HR Data Project
     ├── HR Pipeline Project
     └── HR Application Project
```

Project 本身完全无需变化。

---

## 11. Resource Category 的设计

第一版可以只支持：

```text
Resource Category
    ONTOLOGY
```

下面包括：

```text
OBJECT_TYPE
LINK_TYPE
ACTION_TYPE
INTERFACE
SHARED_PROPERTY
```

未来逐渐增加：

```text
Resource Category
├── ONTOLOGY
├── DATA
├── ENGINEERING
└── APPLICATION
```

例如：

```text
ONTOLOGY
├── OBJECT_TYPE
├── LINK_TYPE
└── ACTION_TYPE

DATA
├── DATASET
├── TABLE
└── DATA_SOURCE

ENGINEERING
├── REPOSITORY
├── PIPELINE
└── JOB

APPLICATION
├── APPLICATION
├── WORKFLOW
└── DASHBOARD
```

这样前端也非常容易扩展。

---

## 12. Project 页面如何演进

### V1：只支持 Ontology

```text
HR Project

Overview

Resources
└── Ontology
     ├── Object Types
     ├── Link Types
     └── Action Types

Members

Permissions
```

### V2：加入 Data

```text
Resources
├── Ontology
└── Data
```

### V3：加入 Pipeline

```text
Resources
├── Ontology
├── Data
└── Pipelines
```

### V4：加入 Application

```text
Resources
├── Ontology
├── Data
├── Pipelines
└── Applications
```

Project 页面整体信息架构不需要推翻。

---

## 13. 实际演进示例

### V1：Ontology Management

```text
Project: HR

Resources
├── Employee
├── Department
├── Position
└── Employee → Department
```

Project 负责：

```text
Ownership
Permission
Members
Resource Organization
```

---

### V2：加入 Dataset

```text
Project: HR

Resources
├── Ontology
│    ├── Employee
│    ├── Department
│    └── Position
│
└── Data
     ├── employee_raw
     └── employee_clean
```

Project 不改。

---

### V3：加入 Pipeline

```text
Project: HR

Resources
├── Ontology
├── Data
└── Pipelines
     └── HR Employee Pipeline
```

Project 仍然不改。

---

### V4：加入 Application

```text
Project: HR

Resources
├── Ontology
├── Data
├── Pipelines
└── Applications
     └── Employee Management App
```

Project 仍然不需要重构。

---

## 14. 推荐的核心 ER 模型

推荐从第一版就采用：

```text
Organization
    │
    ▼
Space
    │
    ├──────────────► Ontology
    │                   │
    │                   ├── ObjectType
    │                   ├── LinkType
    │                   ├── ActionType
    │                   └── Interface
    │
    └──────────────► Project
                        │
                        ▼
                  ProjectResource
                        │
                        ▼
                     Resource
```

Resource 未来可以扩展为：

```text
Resource
├── OntologyResource
├── DataResource
├── PipelineResource
├── CodeResource
└── ApplicationResource
```

权限模型：

```text
Project
│
├── ProjectMember
├── ProjectRole
└── ProjectPermission
```

未来再增加：

```text
ResourcePermissionOverride
```

---

## 15. 最关键的语义分离

### Ontology 的关系

```text
Object Type
    belongs to
Ontology
```

### Project 的关系

```text
Object Type
    governed by
Project
```

不要混成：

```text
Object Type
    belongs to
Project
    belongs to
Ontology
```

因为这会错误地把 Project 放进 Ontology 的语义层级。

更准确的是两个独立维度：

```text
                Object Type
                /         \
               /           \
    Semantic ownership    Governance ownership
            ↓                    ↓
        Ontology              Project
```

这能够显著提升未来扩展性。

---

## 16. 第一版建议能力范围

| 能力 | 第一版是否实现 | 底层模型是否预留 |
|---|---:|---:|
| Project | 是 | 是 |
| Object Type | 是 | 是 |
| Link Type | 是 | 是 |
| Action Type | 是 | 是 |
| Interface | 是 | 是 |
| Project Role | 是 | 是 |
| Project Member | 是 | 是 |
| Project Resource | 是 | 是 |
| Dataset | 否 | 是 |
| Pipeline | 否 | 是 |
| Repository | 否 | 是 |
| Application | 否 | 是 |
| Resource-level Override | 可暂缓 | 是 |

---

## 17. 最终推荐

第一版只支持 Ontology 管理是完全合理的 MVP 路线。

但建议从一开始将 Project 定义为：

> **通用 Resource Governance Container**

而不是：

> **Ontology 专属容器**

这样未来从：

```text
Ontology Management Platform
```

扩展到：

```text
Foundry-like Platform
```

核心的：

```text
Space
Project
Resource
Permission
```

模型都可以保持稳定。

后续只需要不断扩充 Resource 类型即可。

最终可以把核心设计原则概括为：

> **Ontology 决定资源的语义归属；Project 决定资源的治理归属。**

也就是：

```text
Ontology
    = Semantic Boundary

Project
    = Governance / Ownership / Collaboration Boundary

Resource
    = 可被 Project 管理的统一资源抽象
```

这套设计最适合当前“先做 Ontology Management、未来再扩展成更完整 Foundry-like 平台”的演进路线。
