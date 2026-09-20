# Palantir Project-based Ontology Permissions：跨 Space 导入 Ontology Resource 所需权限

## 1. 核心结论

在 Palantir 最新的 **Project-based Ontology permissions** 模型下，如果要让 **Space B 中的 Project** 使用或导入 **Space A 的 Ontology Resource**，通常至少需要同时满足以下条件：

1. 对源 Ontology Resource 拥有 `compass:import-resource-from`
2. 对目标 Project 拥有 `compass:import-resource-to`
3. 满足 Source Space 和 Destination Space 的 Organization 访问要求
4. 满足相关 Markings / Mandatory Controls
5. 如果启用了 CBAC / Classification，还需要满足对应分类访问要求
6. 如果还要读取 Object instances，则需要额外的数据访问权限

在默认角色模型下，经常可以简化为：

```text
Source Project: Viewer
    +
Destination Project: Editor
```

但最准确的判断应该看底层 capability，而不只是角色名。

---

## 2. 先澄清：跨 Space import 的到底是什么

例如：

```text
Space A
│
├── Ontology A
│
└── Project CRM
      ├── Customer Object Type
      ├── Order Object Type
      └── Customer → Order Link Type


Space B
│
└── Project Analytics
      └── Functions Repository
```

在 Space B 的 Project Analytics 中配置 import 时，本质上不是：

```text
把 Ontology A 整体复制到 Space B
```

而是：

```text
Project Analytics
       │
       │ Project-level import/reference
       ▼
Ontology A
   ├── Customer
   ├── Order
   └── Link Type
```

因此，更准确的说法是：

> Space B 中的 Project 引用或消费 Space A 的 Ontology 中的 Ontology Resources。

这些资源仍然属于原来的 Ontology 和 Space。

---

## 3. 第一层：必须满足 Space / Organization 访问要求

假设：

```text
Space A
   ↓
Ontology A
```

Space 本身可能受 Organization 限制。

例如：

```text
Space A
Allowed Organizations:
├── Org A
└── Org B
```

而某个用户只属于：

```text
Org C
```

那么即使该用户在某些 Project 上拥有角色，也可能因为 Organization 这层 Mandatory Control 而无法访问 Space A / Ontology A。

可以理解为：

```text
User
 │
 │ Organization eligibility
 ▼
Space A
 │
 ▼
Ontology A
```

如果这层不满足，则后续 import 无法进行。

---

## 4. 第二层：源 Ontology Resource 需要 import-from 权限

例如：

```text
Space A
└── Project CRM
      └── Customer Object Type
```

在 Project-based Ontology permissions 下，Customer 是一个受 Project 权限管理的 Ontology Resource。

要从这里导入，一般需要：

```text
compass:import-resource-from
```

默认情况下，这个 capability 通常由源 Project 的：

```text
Viewer
```

角色提供。

即：

```text
Source side

Space A
└── Project CRM
      └── Customer
             ▲
             │
      compass:import-resource-from
             │
           User
```

因此，常见最小权限可以理解成：

```text
Source Project
     ↓
Viewer
```

---

## 5. 为什么是“通常 Viewer”而不是“必须 Viewer”

因为 Palantir 支持 Custom Roles。

例如企业管理员可以定义：

```text
Role: Ontology Consumer
```

其中只包含：

```text
compass:view-resource
compass:import-resource-from
```

此时该用户即使不是系统默认的 Viewer，也仍然可以从源 Project 导入资源。

因此，判断权限时最准确的问题不是：

```text
我是不是 Viewer？
```

而是：

```text
我有没有：
compass:import-resource-from
```

---

## 6. 第三层：目标 Project 需要 import-to 权限

假设目标是：

```text
Space B
└── Project Analytics
```

用户要在这里新增 Resource Import，本质上是在修改目标 Project 的资源依赖关系。

因此通常需要：

```text
compass:import-resource-to
```

默认情况下，这个 capability 一般来自：

```text
Editor
```

即：

```text
Destination side

             User
              │
     compass:import-resource-to
              │
              ▼
       Project Analytics
            Space B
```

所以最典型的组合是：

```text
Source Project A
Viewer
   +
Destination Project B
Editor
```

---

## 7. 三层合并后的最小权限模型

假设：

```text
SPACE A
│
├── Ontology A
│
└── Project CRM
      └── Customer
             │
             │
             ▼
          IMPORT
             │
             ▼
SPACE B
│
└── Project Analytics
```

那么用户通常需要：

```text
User
│
├── 满足 Space A 的 Organization access requirements
│
├── Source Project CRM
│      └── Viewer
│            └── compass:import-resource-from
│
├── 满足 Space B 的 Organization access requirements
│
└── Destination Project Analytics
       └── Editor
             └── compass:import-resource-to
```

结构可以画成：

```text
                    User
                      │
         ┌────────────┴────────────┐
         ▼                         ▼
     SOURCE                     DESTINATION
     Space A                      Space B
        │                            │
    Project CRM              Project Analytics
        │                            │
      Viewer                      Editor
        │                            │
import-resource-from        import-resource-to
        │                            │
        └───────── Customer ─────────┘
```

---

## 8. 第四层：Markings 仍然可以阻止 import

Project Role 属于可授予权限，但 Markings 属于 Mandatory Access Control。

例如：

```text
Space A
└── Project CRM
      └── Customer
             │
             └── Marking:
                 PII-Approved
```

某个用户拥有：

```text
Project CRM = Viewer
Project Analytics = Editor
```

但不具备：

```text
PII-Approved marking access
```

那么即使：

```text
Viewer + Editor
```

都具备，也不能绕过 Marking。

权限检查大致可以理解成：

```text
Viewer on source
       │
       ▼
Editor on destination
       │
       ▼
Organization access
       │
       ▼
Marking access
       │
       ├── NO
       ▼
IMPORT DENIED
```

所以：

> Project role 无法覆盖 Mandatory Controls。

---

## 9. 第五层：Organization 对跨 Space 尤其重要

例如：

```text
Space A
Organizations:
   Org Finance

Space B
Organizations:
   Org Analytics
```

用户 Alice：

```text
Member of Org Analytics
Guest of Org Finance
```

她可能同时满足：

```text
Space A access
Space B access
```

再加上：

```text
Project CRM Viewer
Project Analytics Editor
```

则可能完成跨 Space import。

如果另一个用户只属于：

```text
Org Analytics
```

且不满足 Space A 的 Organization requirement，则即使目标 Project 有 Editor，也可能无法访问源 Ontology Resource。

---

## 10. 第六层：CBAC / Classification

如果 Foundry 环境启用了：

```text
Classification-Based Access Controls
(CBAC)
```

那么 Ontology Resource 也可能带有分类。

例如：

```text
Customer Object Type

Classification:
SECRET
```

用户虽然拥有：

```text
Source Viewer
Destination Editor
```

但是 Clearance 只有：

```text
CONFIDENTIAL
```

那么仍然可能无法访问或导入该资源。

因此，不能简单写成：

```text
Viewer + Editor = 一定可以 import
```

更准确的是：

```text
compass:import-resource-from
               +
compass:import-resource-to
               +
Organization
               +
Markings
               +
CBAC / Classification
```

全部满足后才可能成功。

---

## 11. Import Object Type 和读取 Object instances 是两套权限

假设：

```text
Space A
└── Project Ontology
      └── Customer Object Type

Space A
└── Project Data
      └── customer_dataset
```

Space B 中：

```text
Space B
└── Project Functions
```

成功 import：

```text
Customer Object Type
```

这只说明用户或目标 Project 可以访问：

```text
Customer schema
```

并不代表自动获得：

```text
Customer object instances
```

例如：

```text
Customer
├── ID 001 Alice
├── ID 002 Bob
└── ID 003 Carol
```

要读取这些数据，还可能需要：

```text
Backing datasource access
```

或者：

```text
Object / Property Security Policy
```

因此可能出现：

```text
Import Customer Object Type          ✅
查看 Customer Schema                 ✅
生成 Customer TypeScript bindings   ✅
查询 Customer.objects.all()          ❌
```

原因可能只是 backing datasource 权限不足。

---

## 12. Functions Repository 场景还要区分用户权限和 Repository 权限

在 Functions Repository 中使用 Object Type 时，还需要理解：

```text
用户本人有权限
```

并不一定等于：

```text
Functions Repository 自动有权限
```

通常需要通过 Project-level resource imports，把：

```text
Ontology types
```

以及必要时的：

```text
backing datasource
```

显式加入 Repository 所在 Project 的依赖范围。

可以理解成：

```text
User permission
       ↓
允许你配置 import

Project Resource Import
       ↓
允许 Repository 引用 Object Type

Backing datasource import / access
       ↓
允许 Repository / Code Assist 实际读取数据
```

---

## 13. 完整例子

假设：

```text
SPACE A — CRM Space
│
├── Ontology CRM
│      └── Customer
│
└── Project Ontology-Core
       └── Customer
            Marking: Customer-PII


SPACE B — Analytics Space
│
└── Project Customer-Analytics
       └── Functions Repository
```

用户 Alice 想在：

```text
Customer-Analytics
```

使用：

```text
CRM Ontology → Customer
```

那么权限检查可以拆成：

| 层级 | Alice 需要什么 |
|---|---|
| Space A | 满足 Space A 的 Organization access requirement |
| Ontology A | 能发现并访问该 Ontology |
| Customer | `compass:import-resource-from` |
| Source Project | 默认通常为 `Viewer` |
| Customer Marking | 具有 `Customer-PII` marking access |
| Space B | 满足 Space B 的 Organization access requirement |
| Destination Project | `compass:import-resource-to` |
| Destination role | 默认通常为 `Editor` |
| Customer 数据 | 如需读实例，还需 datasource / object-security 权限 |
| Functions | 必要的 backing datasource 还可能需要加入目标 Project |

最终可以表示成：

```text
Alice
 │
 ├── Space A organization access ─────────── ✅
 │
 ├── Project Ontology-Core Viewer ───────── ✅
 │      └── import-resource-from
 │
 ├── Customer-PII marking ───────────────── ✅
 │
 ├── Space B organization access ─────────── ✅
 │
 ├── Customer-Analytics Editor ───────────── ✅
 │      └── import-resource-to
 │
 └── customer_dataset access ─────────────── ✅
                │
                ▼
        可以完整使用 Customer
```

如果最后一项不存在：

```text
customer_dataset access ❌
```

那么可能仍然可以：

```text
✅ 发现 Customer
✅ import Customer
✅ 查看 Customer schema
✅ 在 Functions 中生成相关类型
```

但不一定能够：

```text
❌ 读取 Customer instances
```

---

## 14. 只是消费源 Ontology 时，源 Project 一般不需要 Editor

如果用户只是在 Space B 中使用 Space A 的 Customer：

```text
Source Project
Viewer
   +
Destination Project
Editor
```

通常就符合最小权限原则。

原因是：

```text
Source:
只读取 / 引用
→ Viewer

Destination:
需要修改当前 Project 的 imports
→ Editor
```

可以画成：

```text
SOURCE                         DESTINATION

Customer                      Analytics Project
   │                                  │
   │ READ / EXPORT REFERENCE          │ MODIFY IMPORT SCOPE
   │                                  │
Viewer                             Editor
   │                                  │
import-resource-from          import-resource-to
```

---

## 15. 什么时候源 Project 也需要 Editor

如果用户不是简单地 import / consume，而是要修改源 Object Type：

```text
Customer
├── 新增 property
├── 修改 display name
├── 修改 schema
└── 修改 metadata
```

那么就需要对源 Project / Ontology Resource 的编辑权限。

因此：

```text
Import / consume:
Source Viewer

Edit source Ontology Resource:
Source Editor
```

这两个场景必须区分。

---

## 16. 最完整的权限图

```text
                         USER
                           │
                           ▼
             ┌─────────────────────────┐
             │ Organization / Space A  │
             │ eligibility             │
             └────────────┬────────────┘
                          │
                          ▼
                     SPACE A
                          │
                     ONTOLOGY A
                          │
                    PROJECT CRM
                          │
                     Customer
                          │
             compass:import-resource-from
                          │
                     usually Viewer
                          │
                          │
                    Resource Import
                          │
                          ▼
             ┌─────────────────────────┐
             │ Markings / CBAC / Org   │
             │ mandatory controls      │
             └────────────┬────────────┘
                          │
                          ▼
                     SPACE B
                          │
                 PROJECT ANALYTICS
                          │
              compass:import-resource-to
                          │
                     usually Editor
                          │
                          ▼
                  Resource Imports
                          │
                    Customer
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
         Schema only              Object data
              │                       │
       Object Type View        Datasource / OSP
              │                       │
              ✅                      ✅
```

---

## 17. 最值得记住的权限公式

对于：

> Space B 的 Project import Space A 中的 Project-based Ontology Resource

可以记成：

```text
跨 Space Ontology Resource Import
=
源资源 compass:import-resource-from
（默认通常 Source Project Viewer）

+

目标 Project compass:import-resource-to
（默认通常 Destination Project Editor）

+

满足 Source / Destination Space
Organization requirements

+

满足所有 Markings

+

满足 CBAC / Classification
（如启用）
```

如果还需要读取 Object instances，再加：

```text
+

Backing Datasource access
或
Object / Property Security Policy access
```

如果还要修改源 Object Type，则需要：

```text
Source Editor
```

而不是仅仅 Source Viewer。

---

## 18. 一句话总结

> **跨 Space 使用 Ontology Resource 时，最核心的权限组合是：源资源具备 `compass:import-resource-from`，目标 Project 具备 `compass:import-resource-to`；默认角色下通常对应 Source Viewer + Destination Editor。同时还必须通过 Organization、Marking、CBAC 等 Mandatory Controls。如果还要读取 Object 实例，则还需单独满足 backing datasource 或 Object Security Policy 的数据权限。**

---

## 参考资料

- Palantir — Project references / import permissions  
  https://www.palantir.com/docs/foundry/code-repositories/use-project-references

- Palantir — Ontology imports in Functions  
  https://www.palantir.com/docs/foundry/functions/ontology-imports

- Palantir — Organizations and Spaces  
  https://www.palantir.com/docs/foundry/security/orgs-and-spaces

- Palantir — Projects and Roles  
  https://www.palantir.com/docs/foundry/security/projects-and-roles

- Palantir — Ontology permissions  
  https://www.palantir.com/docs/foundry/object-permissioning/ontology-permissions

- Palantir — Functions permissions  
  https://www.palantir.com/docs/foundry/functions/permissions
