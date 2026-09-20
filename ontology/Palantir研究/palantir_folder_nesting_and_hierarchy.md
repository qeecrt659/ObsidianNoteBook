# Palantir 中 Folder 的嵌套能力与 Portfolio / Project 层级关系

## 1. 核心结论

在 Palantir Foundry 中：

> **普通 Folder 支持嵌套，也就是 Folder 下面可以继续创建 Folder / Subfolder。**

因此，可以形成多级目录结构，例如：

```text
Space
└── Portfolio
    └── Project
        ├── Folder: ontology
        │   ├── Folder: object-types
        │   │   ├── Customer
        │   │   ├── Order
        │   │   └── Product
        │   │
        │   ├── Folder: link-types
        │   ├── Folder: action-types
        │   └── Folder: interfaces
        │
        ├── Folder: data
        │   ├── Folder: raw
        │   ├── Folder: cleaned
        │   └── Folder: ontology-ready
        │
        ├── Folder: pipelines
        └── Folder: documentation
```

Palantir 官方推荐的 Project 结构中也大量使用多级目录，例如：

```text
/data/transformed
/data/analysis
/models/templates
/applications/develop
```

这说明 Folder 可以形成层级树。

---

## 2. Palantir 中几个层级的职责

可以把主要结构理解为：

```text
Space
│
├── Portfolio
│     │
│     └── Project
│           │
│           ├── Folder
│           │     │
│           │     └── Folder
│           │           │
│           │           └── Folder
│           │                 └── Resource
│           │
│           └── Resource
```

其中：

```text
Portfolio
→ 用来组织多个 Projects

Project
→ 主要的资源与权限边界

Folder
→ 用来整理 Project 内部资源

Folder
→ 可以多级嵌套
```

而：

```text
Portfolio
→ 不支持 Portfolio 嵌套
```

---

## 3. Folder 与 Portfolio 的区别

虽然两者看起来都像“分组”，但它们服务于完全不同的层级。

### Portfolio

Portfolio 用于组织同一个 Space 中的多个 Project。

例如：

```text
Space
│
├── Portfolio: Customer Domain
│      ├── CRM Project
│      ├── MDM Project
│      └── Billing Project
│
└── Portfolio: Manufacturing Domain
       ├── MES Project
       ├── PLM Project
       └── QMS Project
```

Portfolio 更适合表达：

- Business Domain
- Department
- Product
- Workstream

---

### Folder

Folder 用来组织一个 Project 内部的资源。

例如：

```text
Project: MES
│
├── ontology/
│   ├── object-types/
│   ├── link-types/
│   ├── action-types/
│   └── interfaces/
│
├── data/
│   ├── raw/
│   ├── transformed/
│   └── ontology-ready/
│
├── pipelines/
└── documentation/
```

所以最简单的记忆方式是：

```text
Portfolio
=
组织 Projects

Folder
=
组织 Resources
```

---

## 4. Folder 可以继续嵌套 Folder

例如：

```text
Project
│
└── Folder: data
     │
     ├── Folder: raw
     │    ├── Folder: sap
     │    └── Folder: crm
     │
     ├── Folder: transformed
     │    ├── Folder: customer
     │    └── Folder: order
     │
     └── Folder: ontology-ready
          ├── Folder: domain-a
          └── Folder: domain-b
```

这类多级目录结构在大型 Foundry Project 中非常常见。

---

## 5. 不建议用 Folder 模拟业务组织边界

虽然 Folder 可以无限向下组织资源，但不建议用 Folder 去模拟：

```text
Business Domain
    ↓
Subdomain
    ↓
IT System
```

例如不要把整个企业 Discovery 环境做成：

```text
Project Enterprise Discovery
│
├── Folder Domain 1
│   ├── Folder Subdomain 1
│   │   ├── Folder SYS001
│   │   └── Folder SYS002
│   └── ...
│
└── Folder Domain 2
```

主要原因是：

> **Project 才是 Palantir 中主要的资源 ownership、协作和 discretionary permission 边界。**

如果不同 IT 系统由不同团队负责，或者需要不同的权限管理，那么应该拆成不同 Project。

---

## 6. 对 7 个业务域、81 个 IT 系统的推荐结构

对于：

```text
7 Business Domains
      │
      └── 81 IT Systems
```

推荐结构：

```text
Space
│
├── Portfolio D01
│   ├── Project SYS001
│   │   ├── ontology/
│   │   │   ├── object-types/
│   │   │   ├── link-types/
│   │   │   ├── action-types/
│   │   │   └── interfaces/
│   │   │
│   │   ├── data/
│   │   │   ├── raw/
│   │   │   ├── transformed/
│   │   │   └── ontology-ready/
│   │   │
│   │   ├── pipelines/
│   │   ├── mappings/
│   │   └── docs/
│   │
│   └── Project SYS002
│
├── Portfolio D02
│   └── ...
│
└── Portfolio D07
```

这里：

```text
Portfolio
=
业务域

Project
=
IT System / Source Ontology

Folder
=
系统 Project 内部的资源分类
```

---

## 7. 推荐的职责分工

可以用下面这张表快速记忆：

| 层级 | 推荐职责 |
|---|---|
| Space | 顶层协作范围 + Common Ontology |
| Portfolio | 组织业务域下的多个 Projects |
| Project | IT System、ownership、权限、资源生命周期边界 |
| Folder | Project 内部资源目录 |
| Subfolder | 更细粒度的资源分类 |
| Resource | Dataset、Pipeline、Object Type、Action Type、文档、应用等 |

---

## 8. 一句话总结

> **Palantir 的 Folder 可以多级嵌套；Portfolio 用来组织 Projects，Project 是主要资源与权限边界，而 Folder 负责整理 Project 内部资源。对于 7 个业务域、81 个 IT 系统的场景，推荐使用 Portfolio 表达业务域、Project 表达 IT System、Folder / Subfolder 管理系统内部的 Ontology、Data、Pipeline、Mapping 和 Documentation 等资源。**

## 参考资料

- Palantir — Recommended project structure  
  https://www.palantir.com/docs/foundry/building-pipelines/recommended-project-structure

- Palantir — Organizations and Spaces  
  https://www.palantir.com/docs/foundry/security/orgs-and-spaces

- Palantir — Filesystem API: Create Folder  
  https://www.palantir.com/docs/foundry/api/filesystem-v2-resources/folders/create-folder

- Palantir — Portfolios  
  https://www.palantir.com/docs/foundry/security/portfolios
