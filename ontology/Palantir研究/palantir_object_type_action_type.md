# Palantir 中新建 Object Type 时与 Action Type 的关系

## 1. 新建 Object Type 时是否可以选择 Action Type？

可以，但更准确地说：

> 在创建 Object Type 的向导中，可以选择 **Generate actions**，让 Palantir 自动为这个 Object Type 生成一组标准 Action Types。

这不是“从已有 Action Type 列表里选择并绑定”，而是“在创建 Object Type 的同时，自动创建新的 Action Types”。

典型流程：

```text
Ontology Manager
  ↓
New
  ↓
Create object type
  ↓
选择 datasource
  ↓
定义 metadata / properties
  ↓
定义 primary key / title
  ↓
Generate actions
  ↓
选择保存 Project
  ↓
Create / Save
```

例如创建：

```text
Object Type: Ticket
```

可以同时生成：

```text
Create Ticket
Modify Ticket
Delete Ticket
```

这些生成出来的 Action 仍然是 Ontology 中独立的 Action Type 资源。

---

## 2. Action Type 是否属于 Object Type？

不是。

不应把它理解为：

```text
Ticket
└── 内嵌方法
    ├── Create()
    ├── Modify()
    └── Delete()
```

更准确的是：

```text
Ontology
│
├── Object Types
│   └── Ticket
│
└── Action Types
    ├── CreateTicket
    ├── ModifyTicket
    └── DeleteTicket
```

Action Type 与 Object Type 是两个独立的 Ontology Resource。

---

## 3. Action Type 与 Object Type 的关系如何建立？

主要是在 Action Type 这一侧建立。

例如：

```text
Action Type: ResolveTicket

Parameter:
    ticket : Ticket

Rule:
    Modify Ticket
        status = "Resolved"
```

这里 `ResolveTicket` 通过：

- Parameter 引用 `Ticket`
- Rule 修改 `Ticket`

因此可以理解为：

```text
Action Type
    ↓
references / operates on
    ↓
Object Type
```

而不是：

```text
Object Type
    ↓
owns
    ↓
Action Type
```

---

## 4. Object Type 已经创建后，还能新增 Action Type 吗？

可以。

可以从 Object Type 页面进入其 Action Types 区域，然后选择：

```text
Create new action type
```

这会进入 Action Type 创建流程，并以当前 Object Type 作为主要操作对象。

也可以直接进入：

```text
Ontology Manager
  ↓
Action types
  ↓
New action type
```

手动创建。

---

## 5. 已有 Action Type 是否是在 Object Type 上“勾选绑定”？

通常不是。

假设已经存在：

```text
Action Type: ResolveTicket
```

它之所以适用于 `Ticket`，是因为 Action Type 自己的定义中包含：

```text
Parameter:
    ticket : Ticket
```

或 Rule 中明确操作：

```text
Modify Ticket
```

所以“Action Type 适用于哪个 Object Type”，主要由 Action Type 的 Parameters 和 Rules 决定。

---

## 6. Object Type 创建时的 Generate actions 应如何理解？

可以把它理解成一个“快捷生成器”。

例如：

```text
创建 Ticket Object Type
        │
        ├── Generate Create Action
        ├── Generate Modify Action
        └── Generate Delete Action
```

最终结果仍然是：

```text
Ontology
│
├── Object Type
│   └── Ticket
│
└── Action Types
    ├── CreateTicket
    ├── ModifyTicket
    └── DeleteTicket
```

这些 Action Type 后续可以单独继续配置、编辑和使用。

---

## 7. Action Type 定义完后在哪里被使用？

Action Type 可以被多个上层入口调用，例如：

```text
Action Type
   ▲
   │
   ├── Object View
   ├── Object Explorer
   ├── Workshop
   └── Ontology API
```

典型链路：

```text
Ontology Manager
    ↓
定义 Action Type
    ↓
应用层引用 Action
    ↓
用户点击 / API 调用
    ↓
传入 Parameters
    ↓
执行 Rules
    ↓
修改 Ontology 中的 Object / Link
```

因此，真正“使用” Action Type 的通常是应用层、UI 或 API，而不是 Object Type 自己主动调用它。

---

## 8. 一句话总结

> 新建 Object Type 时可以使用 **Generate actions** 自动生成标准 Action Types，但这不是从已有 Action Type 中选择并绑定。Action Type 仍然是 Ontology 中独立的资源，并通过自己的 Parameters 和 Rules 与 Object Type 建立关系。定义完成后，它可以被 Object View、Object Explorer、Workshop 或 API 调用。
