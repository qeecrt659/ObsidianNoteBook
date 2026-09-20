# Object Type Group

## 1. 定义

Object Type Group 是一种 classification primitive，用于帮助用户更好地搜索、过滤和探索 Ontology 中的 Object Types。

## 2. 作用

它更接近“组织和发现机制”，而不是业务关系建模。

典型用途：

- 按业务域分组：Supply Chain、Finance、HR
- 按平台分层：Master Data、Transactional、Reference
- 帮助 Ontology Manager / Object Explorer 搜索与筛选

## 3. 与 Link Type 的区别

- Object Type Group：分类标签 / 发现机制。
- Link Type：现实世界业务关系。

不要用 Group 替代 Link，也不要用 Link 模拟导航分类。

## 4. 官方页面

- Object type groups: https://www.palantir.com/docs/foundry/object-link-types/type-groups
- Types reference: https://www.palantir.com/docs/foundry/object-link-types/type-reference

## 5. 自研建议

如果平台会有大量业务部门、领域和项目，Object Type Group 很值得作为独立资源保留，便于浏览、权限治理和模型资产发现。
