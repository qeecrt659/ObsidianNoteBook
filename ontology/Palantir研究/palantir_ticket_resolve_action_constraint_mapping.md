# Palantir Interface Action Type Constraint 详解：Ticket.Resolve → Bug.ResolveBug

## 1. 核心关系

在 Palantir Ontology 中：

```text
Ticket.Resolve
      │
      │ mapping
      ▼
Bug.ResolveBug
```

表示的是：

> 当 `Bug` 实现 `Ticket` Interface 时，`Ticket` Interface 所要求的 `Resolve` 能力，由 `Bug` 自己的 concrete Action Type `ResolveBug` 来满足。

这里要区分三个层次：

```text
1. Ticket Interface
2. Ticket.Resolve Action Type Constraint
3. Bug.ResolveBug Concrete Action Type
```

真正可执行的是 `Bug.ResolveBug`；`Ticket.Resolve` 更像一个能力契约。

## 2. 定义 Ticket Interface

```text
Interface: Ticket

Properties:
├── id: String
├── title: String
├── status: String
└── owner: User
```

它表达的是：任何实现 `Ticket` Interface 的 Object Type，都应该具备这些共同属性。

例如：

- Bug
- FeatureRequest
- SupportCase

都可以实现 `Ticket` Interface。

## 3. 在 Ticket Interface 中定义 Resolve Constraint

```text
Interface: Ticket

Action Type Constraint:
└── Resolve
    ├── ticket
    │    type: InterfaceReference<Ticket>
    │    required: true
    │
    └── resolution
         type: String
         required: true
```

概念上，可以把它理解为：

```typescript
interface TicketActions {
    resolve(
        ticket: Ticket,
        resolution: string
    )
}
```

但这只是类比。`Ticket.Resolve` 本身并不是一个真正可直接执行的 Action Type。它表达的是：

> 所有实现 Ticket 的 Object Type，都应该提供一个满足这个结构和能力要求的 concrete Action Type。

## 4. Action Type Constraint 不负责真正的业务逻辑

`Ticket.Resolve` 主要描述 capability contract：

```text
Resolve

Parameters:
├── ticket
└── resolution
```

它不负责真正定义：

```text
status = "Resolved"
sendEmail()
writeSAP()
createAuditLog()
updateRootCause()
```

这些具体业务逻辑应该定义在 concrete Action Type 中。

所以：

```text
Ticket.Resolve
```

回答的是：

> 一个 Ticket 应该能够被 Resolve。

而不是：

> Resolve 时到底执行哪些业务规则。

## 5. 创建具体 Object Type：Bug

```text
Object Type: Bug

Properties:
├── bugId
├── summary
├── status
├── owner
├── severity
└── rootCause
```

然后：

```text
Bug implements Ticket
```

此时需要建立 Property Mapping：

```text
Ticket                  Bug
──────────────────────────────

id          ────────→ bugId
title       ────────→ summary
status      ────────→ status
owner       ────────→ owner
```

所以从 Interface 的抽象角度来看：

```text
Bug
IS-A
Ticket
```

也就是说，Bug 可以被当作 Ticket 来消费。

## 6. Bug 自己定义真正的 Action Type

假设 Bug 已经有：

```text
Action Type:
ResolveBug
```

参数为：

```text
Bug.ResolveBug

Parameters:
├── bug
│    type: Bug
│    required: true
│
├── resolutionText
│    type: String
│    required: true
│
├── rootCause
│    type: String
│    required: false
│
└── notifyOwner
     type: Boolean
     required: false
```

这个才是真正可以执行的 concrete Action Type。

## 7. ResolveBug 可以包含真正的 Action Rules

例如：

```text
ResolveBug
│
├── 修改 bug.status
│      → "Resolved"
│
├── 修改 bug.rootCause
│      → rootCause parameter
│
├── 写入 resolution
│      → resolutionText
│
├── 写入 resolvedBy
│      → Current User
│
├── 写入 resolvedAt
│      → Current Time
│
└── 如果 notifyOwner = true
       → 发送通知
```

因此：

```text
Ticket.Resolve
```

定义的是 Resolve capability；

而：

```text
Bug.ResolveBug
```

定义的是具体如何 Resolve Bug。

## 8. 建立 Action Mapping

因为：

```text
Bug implements Ticket
```

而 Ticket Interface 中有 required constraint：

```text
Ticket.Resolve
```

因此需要选择一个 concrete Action Type 来满足它：

```text
Ticket.Resolve
      │
      │ Action Mapping
      ▼
Bug.ResolveBug
```

这表示：

> 对于 Bug 这个 Object Type，Ticket Interface 所要求的 Resolve capability，由 ResolveBug Action Type 实现。

## 9. 建立 Parameter Mapping

左边是 Interface contract：

```text
Ticket.Resolve

ticket:
    InterfaceReference<Ticket>

resolution:
    String
```

右边是 concrete Action：

```text
Bug.ResolveBug

bug:
    Bug

resolutionText:
    String

rootCause:
    String

notifyOwner:
    Boolean
```

可以建立：

```text
Ticket.Resolve                     Bug.ResolveBug
────────────────────────────────────────────────

ticket       ────────────────────→ bug

resolution   ────────────────────→ resolutionText
```

对应关系：

| Ticket.Resolve Constraint | Bug.ResolveBug |
|---|---|
| `ticket` | `bug` |
| `resolution` | `resolutionText` |

## 10. 为什么 ticket 可以映射到 bug？

因为：

```text
Bug implements Ticket
```

所以：

```text
Bug
```

是 `Ticket` Interface 的一个 concrete implementation。

抽象关系：

```text
Ticket
  ▲
  │ implements
  │
 Bug
```

因此，当 Interface Constraint 要求：

```text
InterfaceReference<Ticket>
```

时，在 `Bug implements Ticket` 这个具体实现上下文中，可以由对应的 `Bug` object parameter 满足。

## 11. 为什么 resolution 可以映射到 resolutionText？

因为两者表达的是同一种数据能力：

```text
resolution:
    String
```

映射到：

```text
resolutionText:
    String
```

参数名称不必相同，关键是 implementation mapping 明确对应关系并且参数类型兼容。

所以：

```text
resolution → resolutionText
```

是合法的映射思路。

## 12. Concrete Action 可以有额外参数

Interface Constraint：

```text
Resolve(
    ticket,
    resolution
)
```

而 Bug Action：

```text
ResolveBug(
    bug,
    resolutionText,
    rootCause,
    notifyOwner
)
```

Concrete Action 比 Constraint 多了：

```text
rootCause
notifyOwner
```

这通常没有问题。

可以理解成：

```text
Interface 定义共同能力
        │
        ▼
Concrete Action 可以拥有更丰富的实现细节
```

其中：

```text
ticket       → bug
resolution   → resolutionText
```

是 Interface 层关心的；

而：

```text
rootCause
notifyOwner
```

是 Bug-specific 的业务能力。

## 13. 为什么额外 required 参数需要谨慎？

假设：

```text
ResolveBug(
    bug: Bug,                required
    resolutionText: String,  required
    rootCause: String        required
)
```

但 Interface 只有：

```text
Resolve(
    ticket,
    resolution
)
```

此时：

```text
ticket       → bug
resolution   → resolutionText
???          → rootCause
```

从通用 Interface capability 的角度看，`rootCause` 没有对应的 Interface input。

因此，如果希望上层应用能够通过 Interface 的统一能力驱动不同 concrete Action，那么 concrete Action 中额外的 mandatory input 会削弱这种通用性。

设计 Interface Constraint 时，最好把所有 implementation 都真正共享的输入提升到 Interface Constraint 层。

## 14. 再加入 FeatureRequest，就能看出多态价值

```text
FeatureRequest implements Ticket
```

它自己的 Action Type 可能不是 `ResolveBug`，而是：

```text
CloseFeatureRequest
```

参数：

```text
CloseFeatureRequest(
    request: FeatureRequest,
    closeReason: String,
    releaseVersion: String
)
```

可以建立：

```text
Ticket.Resolve                     FeatureRequest.CloseFeatureRequest

ticket       ────────────────────→ request

resolution   ────────────────────→ closeReason
```

于是：

```text
                        Ticket Interface
                              │
                    Constraint: Resolve
                              │
                   Resolve(ticket,
                           resolution)
                              │
              ┌───────────────┴────────────────┐
              │                                │
              ▼                                ▼

             Bug                        FeatureRequest
       implements Ticket               implements Ticket
              │                                │
              ▼                                ▼

         ResolveBug                   CloseFeatureRequest
              │                                │
 ticket → bug                          ticket → request
 resolution → resolutionText           resolution → closeReason
```

## 15. 这就是 Interface Action Capability 的多态

从上层看：

```text
Ticket
│
└── Resolve
```

从底层看：

```text
                       Ticket.Resolve
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
           Bug                        FeatureRequest
              │                             │
              ▼                             ▼
        ResolveBug()             CloseFeatureRequest()
```

Interface 并不要求所有 Object Type 使用同一个 Action Type。

它要求的是：

> 所有实现这个 Interface 的 Object Type，都能够提供满足同一个 capability contract 的 Action。

## 16. 为什么不直接要求 Action 都叫 Resolve？

因为不同 Object Type 的业务语义和业务流程可能不同：

```text
Bug
└── ResolveBug

FeatureRequest
└── CloseFeatureRequest

SupportCase
└── CompleteCase
```

虽然三个 Action 的名字、参数甚至内部规则不同，但从抽象业务能力来看，它们可能都属于：

```text
Ticket.Resolve
```

因此 Palantir 使用：

```text
Constraint
    ↓
Explicit Mapping
    ↓
Concrete Action
```

而不是要求所有 Action 必须同名。

这样可以保持：

- Interface 的统一抽象
- Object Type 的业务独立性
- Concrete Action 的实现自由
- 多态消费能力

## 17. 运行时到底执行哪个 Action？

如果当前对象是：

```text
Bug #123
```

并且它被作为 `Ticket` 来消费，应用需要执行 Resolve capability。

概念上过程是：

```text
Bug #123

typed as Ticket
     │
     ▼
需要执行 Ticket.Resolve
     │
     ▼
查看 Bug 对 Ticket.Resolve 的 mapping
     │
     ▼
发现：

Ticket.Resolve
     ↓
Bug.ResolveBug
     │
     ▼
执行 Bug.ResolveBug
```

因此最终真正执行的是：

```text
Bug.ResolveBug
```

而不是把 `Ticket.Resolve` 当成 concrete Action Type 直接执行。

## 18. Action Type Constraint 本身不是 concrete executable Action

```text
Ticket.Resolve
```

不是一个真正业务 Action，而是：

```text
Interface-level capability contract
```

所以模型是：

```text
Ticket.Resolve
      │
      │ resolve implementation
      ▼
Bug.ResolveBug
      │
      ▼
真正执行
```

## 19. 和 Java Interface 的类比

可以粗略类比：

```java
interface Ticket {
    void resolve(String resolution);
}
```

然后：

```java
class Bug implements Ticket {
    public void resolve(String resolution) {
        // Bug-specific logic
    }
}
```

但 Palantir 比普通 Java Interface 多了一层显式 mapping：

```text
Interface Ticket

constraint:
Resolve(
    ticket,
    resolution
)
```

然后：

```text
Object Type: Bug

Concrete Action:
ResolveBug(
    bug,
    resolutionText,
    rootCause,
    notifyOwner
)
```

再配置：

```text
Ticket.Resolve
      ↓
Bug.ResolveBug
```

以及：

```text
ticket       → bug
resolution   → resolutionText
```

这层显式 mapping 是 Palantir Interface Action Type Constraint 模型的重要特征。

## 20. 完整关系图

```text
┌───────────────────────────────────────────────┐
│              INTERFACE: Ticket                │
│                                               │
│ Properties                                    │
│   id                                          │
│   title                                       │
│   status                                      │
│                                               │
│ Action Type Constraint                        │
│                                               │
│   Resolve                                     │
│      │                                        │
│      ├── ticket: Ticket                       │
│      └── resolution: String                   │
└───────────────────┬───────────────────────────┘
                    │
                    │ implements
                    │
                    ▼
┌───────────────────────────────────────────────┐
│             OBJECT TYPE: Bug                  │
│                                               │
│ Property mappings                             │
│                                               │
│ Ticket.id       → Bug.bugId                   │
│ Ticket.title    → Bug.summary                 │
│ Ticket.status   → Bug.status                  │
│                                               │
│ Action mapping                                │
│                                               │
│ Ticket.Resolve ─────────→ Bug.ResolveBug       │
│                              │                │
│ Parameter mappings           │                │
│                              │                │
│ ticket      ───────────────→ bug              │
│ resolution  ───────────────→ resolutionText   │
│                                               │
│ Other Bug-specific parameters                 │
│                              rootCause         │
│                              notifyOwner      │
└───────────────────────────────────────────────┘
```

## 21. 一句话总结

`Ticket.Resolve` 表达的是：

> 所有 `Ticket` implementation 都应该具备 `Resolve` 这种能力。

而 `Bug.ResolveBug` 表达的是：

> 对于 `Bug` 这种具体 Object Type，这个能力由 `ResolveBug` 这个 concrete Action Type 来真正实现。

最终 mapping：

```text
Ticket.Resolve
      │
      │ mapping
      ▼
Bug.ResolveBug
```

就是在告诉 Palantir：

> 当 `Bug` 作为 `Ticket` 时，`Ticket` Interface 所要求的 `Resolve` capability，由 `Bug.ResolveBug` 这个 concrete Action Type 来满足。

参数映射进一步告诉 Palantir：

```text
Ticket.Resolve.ticket
      ↓
Bug.ResolveBug.bug

Ticket.Resolve.resolution
      ↓
Bug.ResolveBug.resolutionText
```

因此整个机制可以压缩成：

```text
Interface
   ↓
定义能力契约

Action Type Constraint
   ↓
要求 implementation 提供对应能力

Object Type implements Interface
   ↓
建立 Action Mapping

Concrete Action Type
   ↓
真正定义并执行具体业务逻辑
```
