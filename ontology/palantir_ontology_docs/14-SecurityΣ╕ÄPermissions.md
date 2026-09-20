# Security 与 Permissions

## 1. 为什么它不是普通“附属功能”

Palantir 的 Ontology 不只描述语义，还强调 granular security and governance。企业级 Ontology 如果没有资源权限和实例数据权限，就很难真正进入生产环境。

## 2. 两类治理对象

### A. Ontology resources / metadata
例如：

- Object Type
- Link Type
- Action Type
- 以及它们的 metadata

Palantir 当前文档正在使用 project-based / Compass filesystem permissioning 来统一治理这些资源。

### B. Ontology data
例如：

- Object instances
- Links
- Property values

这部分需要更细粒度的数据访问控制。

## 3. Object / Property Security Policies

Palantir 当前支持：

- Object security policy：行级控制，决定用户是否能看到某个 Object。
- Property security policy：列级控制，决定用户是否能看到某些 Property values。
- 两者组合：可以实现 cell-level security。

## 4. Action 的安全

Action 还涉及：

- 谁能执行 Action
- Submission Criteria
- Read / write authorizations
- Side effect permissions

这几层最好分开建模，不要只用一个 RBAC 判断解决所有问题。

## 5. 官方页面

- Object permissioning overview: https://www.palantir.com/docs/foundry/object-permissioning/overview
- Ontology permissions: https://www.palantir.com/docs/foundry/object-permissioning/ontology-permissions
- Object and property security policies: https://www.palantir.com/docs/foundry/object-permissioning/object-security-policies
- Action permissions: https://www.palantir.com/docs/foundry/action-types/permissions

## 6. 自研建议

建议权限模型至少分为：

1. Ontology resource metadata permissions
2. Object instance permissions
3. Property-level permissions
4. Action execution permissions
5. Submission Criteria / contextual business validation
6. 外部系统 writeback / webhook authorization

不要把它们全部合并成“用户是否有某角色”这一种判断。
