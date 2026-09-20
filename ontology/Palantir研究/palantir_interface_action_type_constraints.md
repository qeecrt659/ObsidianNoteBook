# Palantir Interface 中 Action Type Constraint 可以定义什么

## 1. 核心结论

在 Palantir 的 Interface 中，**Action Type Constraint** 用来定义一个“Action 能力契约”，而不是定义真正的执行逻辑。

除了 **Parameter Constraints** 之外，主要还可以定义：

- **Display name**
- **Description**
- **API name**
- **Required / Optional**

而真正的执行行为，例如 Rules、Submission Criteria、Side Effects 等，仍然定义在具体的 **Concrete Action Type** 中。

---

## 2. Action Type Constraint 可以定义的内容

| 可定义项 | 作用 |
|---|---|
| Display name | 给用户和 Ontology 建模者看的显示名称 |
| Description | 描述这个 Action 能力应该表达的业务语义 |
| API name | 在代码、配置或 OSDK 中引用该 Constraint 的名称 |
| Required / Optional | 是否要求所有实现该 Interface 的 Object Type 都必须映射一个具体 Action Type |
| Parameter Constraints | 约束具体 Action Type 必须提供哪些兼容参数 |

---

## 3. 示例

假设有：

```text
Interface: Ticket
```

可以定义：

```text
Action Type Constraint:
    Display name: Resolve Ticket
    API name: resolveTicket

    Description:
        Resolve the ticket and record the resolution.

    Required: true

    Parameter Constraints:
        ticket
            type: object reference
            required: true

        resolutionReason
            type: string
            required: true
```

可以把它理解成：

```text
Ticket Interface
        │
        └── Resolve Ticket
              │
              ├── Display Name
              ├── API Name
              ├── Description
              ├── Required / Optional
              └── Parameter Constraints
```

---

## 4. Required / Optional 的作用

假设：

```text
Ticket Interface
├── Resolve       [Required]
└── Escalate      [Optional]
```

并且：

```text
Bug implements Ticket
FeatureRequest implements Ticket
```

如果 `Resolve` 是 Required：

```text
Bug
    Resolve → ResolveBug

FeatureRequest
    Resolve → CompleteFeature
```

两个 Object Type 都必须提供一个符合 Constraint 的 Concrete Action Type。

如果某个 Object Type 没有映射：

```text
FeatureRequest
    Resolve → ???
```

则无法满足这个 Interface 的 Required Action Type Constraint。

而如果 `Escalate` 是 Optional：

```text
Bug
    Escalate → EscalateBug

FeatureRequest
    Escalate → 无
```

这种情况仍然是允许的。

---

## 5. Description 的实际意义

Action Type Constraint 本身不能完整表达复杂的业务执行要求。

例如业务上希望：

```text
Resolve Ticket
```

满足：

```text
1. status 必须修改为 RESOLVED
2. 必须记录 resolvedAt
3. 必须记录 resolvedBy
4. 必须发送通知
5. 只有未关闭的 Ticket 才能执行
```

这些逻辑本身不能直接作为 Constraint 的强制执行规则。

因此可以通过：

```text
Description
```

描述业务语义，例如：

```text
The satisfying action should mark the ticket as resolved,
record the resolver and timestamp,
and notify the ticket owner.
```

然后由不同的 Concrete Action Type 自己实现。

---

## 6. Action Type Constraint 不能定义什么

Action Type Constraint 主要定义的是契约，而不是行为。

通常不能直接定义：

```text
❌ Action Rules
❌ Modify / Create / Delete Object 的具体逻辑
❌ Submission Criteria
❌ Side Effects
❌ Notification
❌ Webhook
❌ Action Form Layout
❌ 具体业务执行逻辑
```

这些内容应该定义在：

```text
Concrete Action Type
```

中。

---

## 7. Constraint 和 Concrete Action Type 的关系

例如：

```text
Interface: Ticket

Action Type Constraint:
    Resolve
```

两个实现 Ticket 的 Object Type：

```text
Bug
FeatureRequest
```

可以分别映射：

```text
Ticket.Resolve
       │
       ├── Bug
       │      → ResolveBug
       │
       └── FeatureRequest
              → CompleteFeatureRequest
```

其中：

```text
Action Type Constraint
```

负责定义：

```text
- 这个能力叫什么
- API 名是什么
- 是否必须实现
- 需要什么参数
```

而：

```text
Concrete Action Type
```

负责定义：

```text
- Parameters
- Rules
- Submission Criteria
- Side Effects
- Form / UI 配置
```

---

## 8. 类比 Java Interface

可以把 Action Type Constraint 类比成 Java Interface 中的方法定义：

```java
interface Ticket {
    void resolve(String reason);
}
```

它可以规定：

```text
- 方法名称
- 参数
- 是否需要实现
```

但是不会定义：

```java
status = "RESOLVED";
sendEmail();
writeAuditLog();
```

这些属于具体实现：

```java
class Bug implements Ticket {
    void resolve(String reason) {
        ...
    }
}
```

对应到 Palantir：

```text
Interface Action Type Constraint
        ↓
Concrete Action Type
```

---

## 9. 最终总结

> Interface 中的 Action Type Constraint 定义的是一个 Action 的“能力契约”。

它主要可以定义：

```text
Display Name
API Name
Description
Required / Optional
Parameter Constraints
```

但不能定义真正的执行逻辑。

真正的：

```text
Rules
Submission Criteria
Side Effects
具体 Object / Link 修改逻辑
```

仍然由对应的 Concrete Action Type 负责。
