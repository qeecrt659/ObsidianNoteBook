# Palantir Foundry 中 Space、Project 与 Dataset 的关系

## 1. 核心结论

在 Palantir Foundry 中，**Dataset 不是直接创建在 Space 上，而是创建在某个 Project（通常是 Project 下的 Folder）中**。

可以使用下面的心智模型理解：

```text
Space
  ↓
Project
  ↓
Folder
  ↓
Dataset
```

因此，如果有一个 Dataset `Customer`，更准确的描述不是：

> “Customer Dataset 建在 Space A 中。”

而是：

> “Customer Dataset 属于 Space A 下的 Project A，并位于 Project A 的某个 Folder 中。”

例如：

```text
Space A
│
├── Project A1
│   ├── Folder /raw
│   │   └── Dataset Customer_Raw
│   │
│   └── Folder /curated
│       └── Dataset Customer
│
└── Project A2
    └── Dataset Orders
```

---

## 2. Space、Project 与 Dataset 分别承担什么职责？

### Space

Space 更适合作为较高层级的治理、组织和权限边界。

例如企业可以按业务域设计：

```text
Space: Customer Domain
Space: Sales Domain
Space: Finance Domain
```

### Project

Project 是具体资源的 ownership、协作和权限管理边界。

Project 中可以包含很多类型的资源，例如：

- Dataset
- Folder
- Pipeline / Transform
- Code Repository
- Function
- Data Connection 相关资源
- Analysis / Report
- Application
- Model
- Ontology Resources
- 其他 Foundry Resource

### Dataset

Dataset 是 Foundry 中承载数据的核心资源之一。

Dataset 本身拥有明确的资源身份，例如：

- RID
- 所属 Project
- 所属 Space
- 所属 Folder / Path
- Schema
- Version
- Permission
- Transaction / History 等

因此 Dataset 的 ownership 是明确的。

---

## 3. Dataset 是在哪里创建出来的？

Dataset 可以通过很多方式产生。

### 3.1 Data Connection / 数据接入

例如从 SAP、Snowflake、S3、API 或数据库接入：

```text
SAP
 │
 ▼
Data Connection
 │
 ▼
Space: Enterprise Data
 │
 └── Project: SAP Ingestion
       │
       └── Dataset: SAP_Customer_Raw
```

这里 `SAP_Customer_Raw` 的 owner 是：

```text
Space: Enterprise Data
└── Project: SAP Ingestion
```

---

### 3.2 Pipeline / Transform 输出

一个 Pipeline 或 Transform 可以读取上游 Dataset，并生成新的输出 Dataset：

```text
Project: Customer Data Product
│
├── Input:
│   └── Customer_Raw
│
├── Transform
│
└── Output:
    └── Customer_Canonical
```

通常推荐让：

> Transform / Pipeline 的逻辑和它产生的输出 Dataset 位于同一个 Project 中。

---

### 3.3 文件上传

CSV、Parquet 等文件也可以被导入并形成 Dataset。

例如：

```text
Space A
└── Project A
    └── uploads/
        └── Customer.csv
              ↓
        Dataset: Customer
```

---

## 4. Dataset 是否属于 Space？

可以说“属于”，但要注意层级。

更加准确的关系是：

```text
Space A
   │
   ▼
Project A
   │
   ▼
Folder
   │
   ▼
Dataset X
```

也就是说：

- Dataset **直接属于 Project**
- Project **属于 Space**
- 所以 Dataset **间接处于某个 Space 中**

因此下面两种说法中：

### 简化说法

> Dataset X 在 Space A 中。

没有大问题。

### 更准确的架构说法

> Dataset X 属于 Space A 下的 Project A。

后者更适合做系统设计、权限设计和治理设计。

---

## 5. Dataset 能否被其他 Project 使用？

可以。

一个 Dataset 有自己的 owner Project，但可以被其他 Project 引用和消费。

例如：

```text
Project A
└── Dataset: Customer
          │
          │ Reference / Import
          ▼
Project B
└── Sales Pipeline
```

这里：

```text
Dataset Customer
    ↓
Owner = Project A
```

而：

```text
Project B
    ↓
Consumer
```

Project B 使用了 Dataset，但并没有改变 Dataset 的 ownership。

所以可以记住：

> **Resource 通常只有一个归属位置，但可以有很多 Consumer。**

---

## 6. Dataset 能否跨 Space 使用？

可以，只要满足权限和治理要求。

例如：

```text
SPACE A
────────────────────────────

Project A1
└── Dataset: Customer
          │
          │ Cross-space
          │ Reference / Import
          ▼

SPACE B
────────────────────────────

Project B1
├── 使用 Customer
├── Pipeline
└── Dataset: Customer_Features
```

其中：

- `Customer` 仍然属于 Space A / Project A1
- Space B / Project B1 只是 consumer
- Project B1 可以基于它继续加工
- 新产生的 `Customer_Features` 属于 Project B1

这可以理解成：

```text
READ across Project / Space
        ✅

WRITE output locally
        ✅
```

---

## 7. 跨 Space 使用并不意味着资源被复制过去

假设：

```text
Space A
└── Project A
    └── Dataset X
```

然后：

```text
Space B
└── Project B
    └── 使用 Dataset X
```

并不意味着 Dataset X 同时属于两个 Project。

错误理解：

```text
Dataset X
├── belongs to Project A
└── belongs to Project B
```

更准确的理解：

```text
              Owner

Space A
└── Project A
    └── Dataset X
           │
           │ Reference / Import
           ▼
Space B
└── Project B
    └── Consumer
```

---

## 8. 跨 Project / 跨 Space 后，一般在哪里写输出？

推荐模式是：

```text
Project A
└── Dataset A
       │
       │ READ
       ▼
Project B
├── Transform
└── Dataset B
```

也就是说：

- Project B 可以读取 Project A 的 Dataset A
- Project B 的代码 / Pipeline 在 Project B 中运行
- Project B 生成 Dataset B
- Dataset B 属于 Project B

不建议把 Project B 的处理逻辑直接用于覆盖 Project A 管理的 Dataset。

因此可以简单概括：

```text
READ across Project   ✅
READ across Space     ✅（权限允许时）

WRITE output locally  ✅
跨 Project 改写上游资源  不作为推荐模式
```

---

## 9. 一个企业级 Data Product 架构例子

企业可以建立一个公共 Customer Data Product：

```text
                       Source System
                      SAP / CRM / DB
                            │
                            ▼

SPACE: DATA PLATFORM
──────────────────────────────────────────

Project: CRM Ingestion
        │
        └── Customer_Raw
                 │
                 ▼

Project: Customer Data Product
        │
        └── Customer_Canonical
                 │
        ┌────────┼────────────┐
        │        │            │
        ▼        ▼            ▼

SPACE SALES   SPACE SERVICE   SPACE FINANCE
──────────    ─────────────   ─────────────

Project       Project         Project
Sales         Service         Finance
  │              │               │
  └── use        └── use         └── use
      Customer       Customer        Customer
```

这样做的好处是：

- Customer 主数据只维护一次
- Sales、Service、Finance 可以按需消费
- 避免每个 Space 都复制一份 Customer
- Dataset 的 ownership 清晰
- 数据治理和依赖关系更加稳定

---

## 10. Dataset 与 Ontology Resource 的跨 Space 规则不同

这是一个非常重要的区别。

### Dataset

典型模式：

```text
Space A
└── Project A
    └── Dataset Customer
            │
            └───────────────► Space B / Project B
                               Consumer
```

Dataset 可以在权限允许时被其他 Project / Space 使用。

---

### Ontology Resources

Ontology 中常见资源包括：

- Object Type
- Link Type
- Action Type
- Interface
- Shared Property

这些资源的 ownership 与 Ontology / Space 的治理边界关系更紧密。

例如：

```text
Space A
├── Ontology A
│
└── Project A
    └── Customer Object Type
```

如果 `Customer Object Type` 属于 Ontology A，那么管理这个 Ontology Resource 的 Project 应位于 Ontology A 所属的 Space 中。

所以不要把：

> “Dataset 可以跨 Space 使用”

错误地理解为：

> “任何 Ontology Resource 也都可以跨 Space 随意管理。”

两者是不同的治理模型。

---

## 11. 推荐的心智模型

可以用下面四句话理解 Palantir Foundry：

> **Space 管治理边界。**

> **Project 管 Resource Ownership 和协作权限。**

> **Dataset / Code / Pipeline / App 等 Resource 都有明确的归属 Project。**

> **Reference / Import 负责跨 Project、跨 Space 的资源复用。**

整体结构：

```text
Foundry
│
├── Space A
│   ├── Ontology A
│   │
│   ├── Project A1
│   │   ├── Dataset
│   │   ├── Pipeline
│   │   └── Code
│   │
│   └── Project A2
│       └── Application
│
└── Space B
    ├── Ontology B
    │
    └── Project B1
        ├── Reference / Import → Space A Dataset
        ├── Transform
        └── Local Output Dataset
```

---

## 12. 一句话总结

对于问题：

> Dataset 到底在哪里建立？

答案是：

> **Dataset 建立在某个 Project（通常是 Project 下的 Folder）中，而这个 Project 位于某个 Space 中。**

对于问题：

> Dataset 能不能被其他 Space 的 Project 使用？

答案是：

> **可以。Dataset 保持原来的 ownership，同时其他 Project / Space 可以在权限允许的情况下通过 Reference / Import 来消费它。**

因此最重要的关系是：

```text
Space
  ↓
Project
  ↓
Owns Resource
  ↓
Dataset
  │
  └──────── Reference / Import ────────► Other Project / Other Space
```
