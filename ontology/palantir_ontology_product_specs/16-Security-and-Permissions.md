# Ontology Security & Permissions 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定位

Security 不是单独的 Ontology resource，但在 Palantir 的最新 Ontology 模型中与 Data、Logic、Action 并列，是决策系统的四个核心组成部分之一。

对自研平台而言，安全必须进入元模型和 runtime，而不是只在应用层做菜单权限。

## 2. 需要区分的安全层

### Resource-level permission

控制谁可以查看或编辑 Object Type、Link Type、Action Type、Interface、Value Type 等定义资源。

### Object-level security

Object Security Policy 对对象实例做 row-level filtering。

### Property-level security

Property Security Policy 对字段做 column-level filtering。与 Object policy 结合后形成 cell-level security。

### Mandatory Controls

Markings、Organizations、Classifications 等强制控制可以附着到资源/数据并传播。

### Action execution security

包括 Submission Criteria、Read/Write Authorizations、对象数据权限和 Action 本身权限。

## 3. Object Security Policy

对象可见通常要求：

- 用户对 Object Type 有 Viewer 等必要资源权限；
- 通过 granular policy；
- 通过 markings / organization / classification checks。

新模型允许对象安全策略独立于 backing datasource 权限配置，降低过去“必须直接拥有数据集权限”的耦合。

## 4. Property Security Policy

前提是已经存在 Object Security Policy。限制包括：

- Primary Key 不能加入 Property Security Policy；
- 一个非主键 Property 最多属于一个 Property Security Policy；
- 用户必须同时通过 Object policy 和对应 Property policy 才能读取字段；
- 通过对象策略但未通过属性策略时，对应字段值会被隐藏/置空。

## 5. Read-time Enforcement

官方明确提醒 Object/Property Security Policy 主要在读取时过滤。读取后的数据若继续导出/下游传播，并不会自动携带完整策略语义。因此需要结合 Marking 或 Classification-based Access Control 维持下游保护。

这是设计安全体系时非常关键的边界。

## 6. Action Security

用户能够读取某个对象，并不意味着能通过 Action 修改它。Action 还要经过：

- Parameters 输入权限；
- Submission Criteria；
- Read/Write Authorization；
- Action 配置的用户/组限制；
- 相关 Object/Link 编辑能力。

反之，单纯隐藏 Action 按钮也不能替代后端授权。

## 7. Policy Testing

2026 年官方已提供在 Ontology Manager 中测试 Object/Property Security Policies 的能力，可以选定 User + Object，查看该用户能看到哪些 Properties。

自研产品应内置“策略模拟器”，否则复杂 cell-level policy 很难安全运营。

## 8. 外部引用资源

某些 Property Value 可能只是指向其他受权限控制资源的引用，例如 Media。Ontology Property 安全只控制这个引用值本身，不一定自动控制外部 Media Set。因此需要确保外部资源权限同步配置。

## 9. 自研建议

建立四层授权模型：

```text
Resource ACL
  ↓
Object/Property Read Policy
  ↓
Action Submission / Write Authorization
  ↓
Mandatory Control Propagation
```

同时：

- 权限检查全部在服务端；
- UI visibility 仅作为体验优化；
- Agent 与人使用同一 policy engine；
- 提供 policy explain/test；
- 记录所有 Action/Function 的授权决策和审计信息。

## 官方原始资料

- https://www.palantir.com/docs/foundry/ontology/why-ontology
- https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies
- https://www.palantir.com/docs/foundry/object-permissioning/managing-object-security
- https://www.palantir.com/docs/foundry/security/access-control-propagation
- https://www.palantir.com/docs/foundry/action-types/permissions
- https://www.palantir.com/docs/foundry/announcements/2026-06
