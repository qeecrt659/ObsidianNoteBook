# Palantir 中 Space、Project、Ontology 与 Object Type 的关系

## 结论先说

在 Palantir 当前常见的 **Project-based permissions** 模型下：

> **Object Type 在逻辑和语义上属于 Ontology，但在资源存储和权限管理上，需要保存到某个 Project 中。**

因此，新建 Object Type 时要求选择 Project，并不代表：

> Object Type 只能被这个 Project 使用。

更准确地说，这个 Project 是该 Ontology Resource 的：

- 保存位置
- 权限边界
- 管理位置

其他拥有相应权限的 Project 仍然可以引用或导入这个 Object Type。

---

## 1. Space、Project、Ontology、Object Type 的基本关系

可以先用下面这张结构图理解：

```text
Space A
│
├── Ontology A
│     │
│     ├── Object Type: Customer
│     ├── Object Type: Order
│     ├── Action Type: Create Order
│     └── Link Type: Customer -> Order
│
├── Project 1
│     ├── Customer        ← Object Type 的保存位置
│     ├── Create Order    ← Action Type 的保存位置
│     └── Dataset A
│
└── Project 2
      ├── App / Workshop
      ├── Function
      └── import Customer ← 可以使用 Project 1 中的 Object Type
```

这里需要区分三个不同概念：

1. **Space**：更上层的资源和权限边界。
2. **Ontology**：定义业务语义模型。
3. **Project**：承载具体资源，并参与权限管理。

---

## 2. Object Type 真正“属于”谁？

从语义模型角度看：

```text
Ontology
    ↓
Object Type
```

Object Type 属于 Ontology。

例如：

```text
Ontology: Sales Ontology

Object Types
├── Customer
├── Order
├── Product
└── Store
```

Palantir 的 Ontology API 也是按照这种层级组织的：

```text
/api/v2/ontologies/{ontology}/objectTypes/{objectType}
```

因此，不应该把关系简单理解成：

```text
Project
    ↓
Object Type
```

更准确的理解应该是：

```text
Ontology
    ↓
Object Type
    ↓
saved in
    ↓
Project
```

也就是说：

> Ontology 决定 Object Type 的语义归属，而 Project 决定这个资源被保存在哪里以及如何进行权限管理。

---

## 3. 为什么新建 Object Type 时要选择 Project？

在 Palantir 的 Project-based permissions 模型中，Ontology Resource 需要保存到某个 Project。

例如新建：

```text
Object Type: Customer
```

选择：

```text
Save location:
Space A / Project CRM Core
```

那么实际关系是：

```text
Ontology A
   │
   └── Customer
          │
          └── 保存到 Project CRM Core
```

Project CRM Core 主要负责：

```text
Project CRM Core
│
├── Viewer
│     └── 可以查看相关 Ontology Resource
│
├── Editor
│     └── 可以编辑相关 Ontology Resource
│
└── Owner / Manager
      └── 可以管理相关权限
```

所以 UI 中类似下面这样的字段：

```text
Create Object Type

Name: Customer
Ontology: Enterprise Ontology
Project: CRM Core
```

其中的 Project 更接近：

> **Resource Location + Permission Boundary**

而不是：

> Customer 只能由 CRM Core 这个 Project 使用。

---

## 4. Object Type 是否必须绑定一个 Project？

如果使用的是 Palantir 当前新的 **Project-based permissions** 模型，那么可以理解为：

> **是的，Object Type 需要有一个 Project 作为保存位置。**

但这里的“绑定”主要是资源保存和权限管理层面的绑定。

可以表示成：

```text
Object Type
    │
    ├── 语义归属 → Ontology
    │
    └── 保存位置 → Project
```

因此，它不是传统意义上：

```text
Object Type = Project 私有资源
```

---

## 5. Project 与 Space 的关系

Project 本身位于一个 Space 中。

例如：

```text
Space A
├── Project A1
├── Project A2
└── Ontology A

Space B
└── Project B1
```

在 Project-based permissions 模型下，同一个 Ontology 的资源通常需要保存到与该 Ontology 位于同一 Space 的 Project 中。

例如：

```text
Ontology A
└── Customer Object Type
```

可以保存到：

```text
✅ Space A / Project A1
✅ Space A / Project A2
```

而不能把它的保存位置直接设为：

```text
❌ Space B / Project B1
```

因此可以理解为：

```text
Space
│
├── Ontology
│
└── Projects
      └── 保存该 Ontology 的资源
```

---

## 6. 保存到一个 Project，是否意味着只能由这个 Project 使用？

**不是。**

这是最重要的一点。

例如：

```text
Space A
│
├── Ontology
│      └── Customer
│
├── Project CRM
│      └── Customer  ← 保存位置
│
├── Project Sales App
│      └── import Customer
│
└── Project Analytics
       └── import Customer
```

虽然 Customer Object Type 保存于 Project CRM，但：

- Project Sales App 可以使用它
- Project Analytics 可以使用它
- 其他有权限的 Project 也可以使用它

因此：

```text
                 Customer
                    │
            Object Type RID
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
 Project CRM   Project Sales   Project Analytics
   保存位置        使用             使用
```

它们通常引用的是 **同一个 Object Type**，而不是各自复制一份。

---

## 7. Import Object Type 是什么意思？

假设：

```text
Project CRM
└── Customer Object Type
```

另一个 Project：

```text
Project Sales
```

希望在 Functions、应用或其他资源中使用 Customer，那么可能需要将 Customer 作为 Ontology dependency / import 添加到 Project Sales 中。

逻辑上类似：

```text
Project Sales
│
└── Ontology Imports
      └── Customer
```

这里的 import 更接近：

> 给当前 Project 建立对某个 Ontology Resource 的依赖和使用关系。

而不是：

> 把 Customer Object Type 复制到 Project Sales。

---

## 8. Object Type Schema 权限与 Object 数据权限不是一回事

这是另一个非常容易混淆的地方。

例如：

```text
Customer Object Type
保存于：
Project Ontology-Core

Backing Dataset:
Project Raw-CRM / customer_dataset
```

这里实际上存在两套权限：

### 8.1 Object Type Schema 权限

```text
Customer Object Type
        │
Project Ontology-Core
        │
 ├── view
 ├── edit
 └── manage
```

它控制：

- 谁能看 Object Type 定义
- 谁能改属性
- 谁能修改 Link、Action 等 Ontology 配置
- 谁能管理该 Ontology Resource

### 8.2 Object 实例 / 数据权限

```text
Customer Objects
张三
李四
王五
   │
customer_dataset
   │
数据访问权限 / Object Security Policy
```

它控制：

- 谁能看到哪些 Customer 数据
- 谁能查询哪些对象实例
- 是否有行级或对象级安全策略

因此：

> 把 Object Type 保存到某个 Project，并不会自动决定全部 Object 实例的数据访问权限。

---

## 9. “绑定 Project”应该如何理解？

可以拆成下面几种问题：

| 问题 | 答案 |
|---|---|
| Object Type 属于哪个 Ontology？ | 一个具体 Ontology |
| Object Type 是否需要保存到 Project？ | Project-based permissions 模型下，是 |
| Project 是否是 Object Type 的主要保存位置？ | 是 |
| Project 是否影响 Object Type 的查看和编辑权限？ | 是 |
| Object Type 是否只能被保存它的 Project 使用？ | 不是 |
| 其他 Project 能否使用它？ | 可以，在权限允许的情况下引用或 import |
| import 后是否复制成新的 Object Type？ | 通常不是，仍然引用同一个 Ontology Resource |
| Object Type 的数据实例权限是否完全由该 Project 决定？ | 不是，还涉及 backing datasource 和 object security |

---

## 10. 完整关系图

推荐用下面这张图记忆：

```text
                         SPACE
                           │
              ┌────────────┴────────────┐
              │                         │
          ONTOLOGY                   PROJECTS
              │                         │
              │                  ┌──────┼──────┐
              │                  │             │
              │              Project A     Project B
              │                  │             │
              │             保存资源         使用资源
              │                  │             │
              ├── Object Type ───┘─────────────┘
              │
              ├── Action Type
              │
              ├── Link Type
              │
              ├── Interface
              │
              └── Shared Property
```

最核心的记忆方式是：

```text
Space
  │
  ├── Ontology
  │      └── 定义业务语义
  │
  └── Project
         └── 保存资源 + 管理权限
```

对于 Object Type：

```text
Object Type
│
├── 语义上属于 → Ontology
│
├── 资源上保存于 → Project
│
└── 数据来自 → Backing Dataset / Datasource
```

---

## 11. 一句话总结

> **Object Type 在逻辑和语义上属于 Ontology，在物理资源和权限管理上保存在某个 Project 中；这个 Project 通常需要与 Ontology 位于同一个 Space，但 Object Type 并不是只能被该 Project 使用，其他具有相应权限的 Project 可以 import 或引用它。**

---

## 参考资料

- Palantir Ontology Permissions  
  https://www.palantir.com/docs/foundry/object-permissioning/ontology-permissions

- Palantir Migrate to Project-based Permissions  
  https://www.palantir.com/docs/foundry/ontology-manager/migrate-to-project-based-permissions

- Palantir Functions - Ontology Imports  
  https://www.palantir.com/docs/foundry/functions/ontology-imports

- Palantir Ontologies API - Object Types  
  https://www.palantir.com/docs/foundry/api/ontologies-v2-resources/object-types/get-object-type
