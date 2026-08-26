
@docs/SAP_FI-AP(应付账款)模块功能清单.md 这个md是SAP的AP模块的功能清单，我现在要针对这些功能清单，整理一个符合palantir标准的ontology模型，包含：objecttype（含属性），link type，action type（含rules，submission criteria等），我想通过Opus 5自身含有的知识去提炼这个ontology模型（而不是去网上搜索信息），请给我提出一个执行计划或者workflow，提炼出最全的SAP的AP模型的ontology，输出的格式为md，按照每个object type一个md文件输出,你可以用subagents来执行。注意：这里提炼的ontology模型不是为了给业务看，而是用于重新开发一个SAP系统的AP模块，这个AP模块的功能清单为“@docs/SAP_FI-AP(应付账款)模块功能清单.md”，所以ontology模型的描述可以技术些。

@docs/SAP_FI-AP(应付账款)模块功能清单.md 这个md是SAP的AP模块的功能清单 




# 目标

对 `ontology/` 下已挖掘的 SAP FI-AP Ontology 模型做一次**完整的工程级审查**，产出三份可直接执行的结论：
(1) 正确性与一致性缺陷清单；(2) Object Type 补全建议；(3) Action Type 补全建议。

# 背景

- **唯一需求来源**：`docs/SAP_FI-AP(应付账款)模块功能清单.md`（5 大章：后台配置 / 主数据 / 前台业务 / 报表分析 / 周期作业）。
- **被审查物**：`ontology/` 目录，遵循 Palantir Foundry Ontology 建模范式：
  - `00-conventions.md` — 强制建模契约（命名、baseType 白名单、主键策略、状态机、表达式语法）。**契约优先于任何单个文件的写法。**
  - `01-object-type-inventory.md` — 权威范围定义，90 个 Object Type，分 8 个域（D1 凭证内核 → D8 报表期末）。
  - `object-types/<域>/<ApiName>.md` — **唯一事实来源**。每个文件含：概述 / Object Type 定义 / Properties / 状态机 / §5 Link Types / §6 Action Types / 校验规则 / 索引与查询模式 / 未决问题。
  - `interfaces/`、`value-types/`、`link-types/README.md`、`action-types/README.md` — 后两者是 `tools/generate_registries.py` 的**派生产物，不要作为事实来源，也不要直接改**。
- **模型用途**：**不是给业务方看的概念模型，而是重新开发一套 AP 模块的实现蓝图**。读者是要照此写代码的工程师。因此描述可以且应当技术化：字段精度、主键、幂等语义、并发与锁、状态迁移前后置条件、批处理边界、异常与补偿路径，都属于本模型该说清楚的内容。

# 硬性约束

1. **只使用 Opus 5 自身的 SAP FI-AP 领域知识进行校验，禁止联网检索**（不调用 WebSearch / WebFetch）。凡涉及 SAP 标准表结构、事务码、配置语义的判断，直接依据内部知识给出，并在不确定时显式标注置信度（高 / 中 / 低），而非去查。
2. **只读审查，不修改仓库文件**，除非我另行指示。所有产出以报告形式给出。
3. 结论必须**锚定到具体文件与行号**（`ontology/object-types/04-invoice/VendorInvoice.md:132` 形式），禁止泛泛而谈。
4. 区分三类问题，不要混为一谈：**缺陷**（与契约或 SAP 语义冲突，必错）/ **风险**（可实现但会在某场景下出问题）/ **建议**（可选增强）。
5. 覆盖率优先于修辞：宁可条目多而短，不要少而长。

# 对于挖掘的结果做优化的提示词
 将如下提示词写成一个符合claude goal的提示词。提示词：“@docs/SAP_FI-AP(应付账款)模块功能清单.md 这个md是SAP的AP模块的功能清单， @ontology/ 目录里是根据这个SAP的AP模块功能清单挖掘出来符合palantir要求的ontology模型。现在的需求：1）校验下 @ontology/ 挖掘出来的内容的正确性和逻辑的一致性，对于不一致和逻辑有问题的内容进行优化，2）基于@docs/SAP_FI-AP(应付账款)模块功能清单.md 中的功能，分析 @ontology 目录所挖掘出来的object type是否有遗漏或多余的，对于遗漏和多余的进行添加、修改或删除；3）补充在 @ontology/中挖掘出来的object type中的action type，使得每个object type中的action type要覆盖更多的场景。 注意：1）我只想使用Opus 5模型自身含有的知识，千万不要去网上搜索信息；2）这里提炼的ontology模型不是为了给业务看，而是用于重新开发一个SAP系统的AP模块，这个AP模块的功能清单为“@docs/SAP_FI-AP(应付账款)模块功能清单.md”，所以ontology模型的描述可以技术些。”