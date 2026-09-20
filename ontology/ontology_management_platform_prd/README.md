# Ontology 管理平台 PRD 文档集

本目录把上一版 Palantir Ontology 产品说明升级为可直接用于产品评审、研发拆解和验收的 PRD 级文档。

## 文档结构

- `00-Ontology元素地图与官方资料索引.md`：平台范围、元素地图、官方资料入口、研发分期。
- `01-Ontology.md`：顶层 Ontology 容器与治理。
- `02-Object-Type.md`：Object Type。
- `03-Property.md`：Property。
- `04-Shared-Property.md`：Shared Property。
- `05-Link-Type.md`：Link Type。
- `06-Action-Type.md`：Action Type。
- `07-Action-Parameter.md`：Action Parameters。
- `08-Action-Rules.md`：Action Rules。
- `09-Submission-Criteria.md`：Submission Criteria。
- `10-Action-Side-Effects.md`：Action Side Effects。
- `11-Interface.md`：Interface。
- `12-Object-Type-Group.md`：Object Type Group。
- `13-Value-Type.md`：Value Type。
- `14-Base-Type.md`：Base Type。
- `15-Functions-on-Objects.md`：Functions / Functions on Objects。
- `16-Security-and-Permissions.md`：Security & Permissions。
- `17-Common-Metadata.md`：Status / API Name / Visibility / Type Class / Render Hint。

## 每个 PRD 都包含

Palantir 官方语义基线、产品目标/非目标、角色、场景、页面 IA、字段字典、FR 编号、创建与编辑流程、Change Set/发布、校验规则、生命周期、依赖分析、权限、安全、列表搜索、API、事件/审计、错误码、NFR、运营指标、验收用例、测试与研发分期。

## 重要原则

1. 六类一级 Ontology Resources 与 Palantir 官方保持一致：Object Type、Link Type、Action Type、Interface、Shared Property、Object Type Group。
2. Property 和 Action Parameter/Rules/Submission Criteria/Side Effects 作为关键子元素独立产品化。
3. Value Type、Base Type、Functions、Security、Common Metadata 作为强绑定能力单独治理。
4. 自研扩展（例如 Change Set、统一错误码、GitOps）必须明确是“实现建议”，不要伪装成 Palantir 一级 Ontology 概念。
