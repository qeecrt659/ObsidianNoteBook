# Palantir Quiver 深度功能研究

> 文档主题：Quiver 的产品定位、内部分析模型、详细功能、执行方式、应用边界及平台建设启示  
> 整理日期：2026-08-06

---

## 1. Quiver 到底是什么

Quiver 不是普通的 BI 图表工具，也不是单纯的自然语言问答工具。

更准确地说，Quiver 是：

> **建立在 Palantir Ontology 之上的、面向业务对象关系与时间序列的可视化分析环境。**

它把一次分析表示为由多个 **Card** 构成的、有明确输入和输出类型的分析图。用户可以从业务对象出发，筛选对象、沿关系查找关联对象、计算指标、处理时间序列、识别异常、比较事件、生成图表，最后把分析发布成 Dashboard，甚至通过 Action 将分析结论写回 Ontology。

Quiver 的整体工作方式可以概括为：

```text
Ontology 数据
    ↓
Object Set / Single Object / Time Series
    ↓
一系列具有明确输入输出类型的 Card
    ↓
筛选、关系穿透、计算、转换、异常检测
    ↓
Number / Object Set / Event Set / Chart / Table
    ↓
Dashboard
    ↓
Action 写回 Ontology
```

Quiver 真正解决的问题不是“怎样画一个图表”，而是：

> **怎样基于统一业务语义，把对象、关系、指标、时间序列和事件组织成一条可检查、可复用、可参数化、可发布、可执行的分析链路。**

---

## 2. Quiver 的核心价值

### 2.1 把 Ontology 变成可分析的数据模型

传统 BI 中，分析人员通常面对：

```text
数据表
字段
主键
外键
JOIN
GROUP BY
```

Quiver 中，分析人员面对的是业务概念：

```text
客户
订单
工厂
设备
传感器
维修事件
```

这些业务概念对应 Ontology 中的：

- Object Type；
- Object；
- Object Set；
- Property；
- Link Type；
- Time Series Property。

对象之间已经定义好的 Link，可以直接用于关系穿透。业务用户通常不需要重新识别主键、外键或者手工编写 Join。

例如：

```text
高风险订单
    ↓ Order → Product
相关产品
    ↓ Product → Factory
生产工厂
    ↓ Factory → Equipment
相关设备
    ↓ Equipment → Maintenance Event
维修事件
```

因此，Quiver 把传统的“数据表关联分析”提升成了“业务对象关系分析”。

### 2.2 把分析表示成强类型数据流图

Quiver 的分析由多个 Card 构成。每个 Card：

- 接收零个或多个输入；
- 执行一个明确的分析操作；
- 返回一种明确的数据类型；
- 可以作为其他 Card 的输入。

例如：

```text
Order Object Set
    ↓ Filter Object Set
Delayed Order Object Set
    ↓ Numeric Aggregation
Delayed Order Count
```

这里：

- 第一个 Card 输出 `Object Set`；
- 筛选 Card 输入和输出都是 `Object Set`；
- 聚合 Card 输入 `Object Set`，输出 `Number`。

这种设计意味着 Quiver 本质上是：

> **一个低代码、强类型、面向 Ontology 的分析 DAG。**

常见数据类型包括：

- Object Set；
- Single Object；
- Transform Table；
- Materialization；
- Ontology SQL Result；
- Time Series；
- Event Set；
- Number；
- String；
- Boolean；
- Time；
- Time Range；
- Array；
- Numeric Range。

只有上游输出类型与下游输入类型兼容时，Card 才能直接连接。

---

## 3. Quiver 的分析组织方式

## 3.1 Graph Mode：真实计算逻辑

Graph Mode 用节点和连线展示 Card 之间的依赖关系，适合：

- 查看指标的完整计算来源；
- 查看某个 Card 的所有上游输入；
- 查看某个 Card 的所有下游依赖；
- 检查复杂分析分支；
- 对比两个 Card 的输出；
- 发现重复逻辑；
- 定位分析错误；
- 理解整个分析 DAG。

例如：

```text
                         ┌→ 按工厂统计延期率
Order Set → Filter Delay ├→ 计算延期金额
                         ├→ 关联战略客户
                         └→ 关联设备异常
```

Graph Mode 展示的是分析真正的执行和依赖关系。

## 3.2 Canvas Mode：分析工作台和展示层

Canvas Mode 用于摆放、组织和展示 Card。

一个 Quiver Analysis 可以包含多个 Canvas，例如：

```text
Canvas 1：输入数据与参数
Canvas 2：对象筛选与关系分析
Canvas 3：时间序列与异常分析
Canvas 4：最终指标和图表
```

需要注意：

> Canvas 中 Card 的上下左右位置并不决定执行顺序。

移动 Card 只改变展示布局，真实数据流仍由 Card 之间的输入输出连接决定。

因此：

- Graph 是逻辑视图；
- Canvas 是工作台和展示视图。

中间计算 Card 可以从 Canvas 中隐藏，但仍保留在 Graph 中供下游继续使用。

---

## 4. Object Analytics：对象分析能力

## 4.1 添加对象数据

Quiver 可以将以下数据作为分析起点：

- 某个 Object Type 的全部对象；
- 单个 Object；
- 已保存的 Object Set；
- 由 Function 返回的 Object Set；
- 由筛选产生的 Object Set；
- 由关系穿透产生的 Object Set。

Object Set Card 通常可以展示：

- 对象总数；
- 对象属性；
- 每行一个对象；
- 关联的 Sensor；
- 时间序列属性；
- 对象预览和详情。

还可以从 Object Set 中继续弹出：

- 单个对象；
- 某个对象属性；
- 某个对象的 Time Series Property。

## 4.2 Object Set 筛选

Filter Object Set 是最基础、最常用的 Card。

典型筛选条件包括：

```text
status = "Delayed"
amount > 1,000,000
plannedDeliveryDate 在指定日期范围内
riskScore >= 风险阈值
factoryName 包含 "Shanghai"
property 为空
```

### Linked Property Filter

可以依据关联对象的属性筛选当前对象。

例如：

```text
筛选订单
条件：
订单关联客户.customerLevel = "Strategic"
```

也就是说，筛选条件不必来自 Order 自身，也可以来自与 Order 关联的 Customer。

### 嵌套 AND/OR

例如：

```text
订单金额 > 100万元
AND
(
    延期天数 > 7
    OR
    客户等级 = 战略客户
)
```

### 派生属性筛选

如果筛选条件不是 Ontology 中的原生属性，而是分析过程中临时计算出的字段，可以先使用 Transform Table 或 Materialization 生成派生列，再继续筛选。

## 4.3 Object Set 集合运算

Quiver 可以对多个 Object Set 执行集合运算：

```text
A ∩ B：同时属于两个集合
A ∪ B：属于任意一个集合
A - B：属于 A 但不属于 B
```

例如：

```text
A = 所有延期订单
B = 所有战略客户订单

A ∩ B
= 战略客户的延期订单
```

Object Set 是 Quiver 最重要的中间分析状态之一。很多业务分析的过程，本质上就是不断构造更有业务意义的 Object Set。

## 4.4 沿 Link 关系穿透

Quiver 可以通过 Search Around，从当前对象集合切换到关联对象集合。

例如：

```text
订单集合
    ↓ Search Around
产品集合
    ↓ Search Around
工厂集合
    ↓ Search Around
设备集合
```

多个 Search Around 可以连续串联，从而形成多跳关系分析。

### Switch to linked object set

输入：

```text
Order Object Set
```

输出：

```text
与这些订单关联的 Product Object Set
```

此时分析焦点从订单切换为产品，不再保留原订单表中的所有列。

### Join to linked objects

如果需要在一个表中同时保留订单和产品属性，则可以使用 Join：

```text
orderId
orderAmount
productId
productCategory
factoryName
```

因此，关系分析可以区分为：

- Search Around：切换业务分析对象；
- Join：保留多类对象属性，形成一张分析表。

## 4.5 对象聚合分析

Quiver 可以对 Object Set 执行：

- Count；
- Unique Count；
- Sum；
- Average；
- Min；
- Max；
- Percentile；
- Standard Deviation；
- Variance。

例如：

```text
按工厂分组
计算：
- 订单数量
- 延期订单数量
- 平均延期天数
- 延期金额总和
- 延期天数 95 百分位
```

## 4.6 对象图表和钻取

常见可视化包括：

- Bar Chart；
- Line Chart；
- Pie Chart；
- Categorical Scatter Plot；
- Numerical Scatter Plot；
- Heat Grid；
- Waterfall Plot；
- Map；
- Correlation Matrix；
- Events Timeline；
- Pivot Table；
- Vega Plot；
- Overlay Chart。

图表不仅用于查看结果，还可以用于创建新的 Object Set。

例如：

```text
柱状图：按工厂显示延期订单数量
    ↓ 用户选中上海工厂和成都工厂
Selection Object Set
    ↓
只包含这两个工厂的延期订单
```

Quiver 还支持 Cross Filter：一个图表中的选择可以同步过滤其他图表。

---

## 5. Transform Table：灵活的局部表格计算

Object Set 适合直接依据 Ontology 属性做分析，但复杂业务问题经常需要：

- 临时派生列；
- 多列计算；
- Join；
- Group By；
- 数据修正；
- 字符串处理；
- 日期处理；
- 批量时间序列操作。

Quiver 为此提供 Transform Table。

## 5.1 Transform Table 可以做什么

包括：

- 从 Object Set 生成本地表；
- 选择或删除列；
- 创建派生列；
- 编辑值；
- 查找替换；
- Filter；
- Group By；
- Joined Group By；
- Join；
- 处理空值和错误值；
- 字符串操作；
- 数值操作；
- 日期操作；
- 数组操作；
- 批量处理 Time Series 列；
- 将结果输出为图表、数字、表格或时间序列。

例如：

```text
输入：Order Object Set

派生列：
delayDays =
actualDeliveryDate - plannedDeliveryDate

派生列：
riskLevel =
if delayDays > 15 then "Critical"
else if delayDays > 7 then "High"
else "Normal"

关联：
Join Customer
Join Factory

聚合：
按 Factory 分组
计算平均 delayDays
```

## 5.2 Transform Table 的适用边界

Transform Table 适合：

- 中小规模数据；
- 强交互分析；
- 临时计算；
- 灵活调整；
- 需要逐步探索的问题。

其计算主要在浏览器侧完成，因此不适合无限规模的数据处理。复杂 Join、聚合和大量时间序列单元格会消耗较多资源。

---

## 6. Materialization：大规模对象分析

当数据规模超过 Transform Table 的能力时，Quiver 可以使用 Materialization。

Materialization 是一种后端执行、由数据集支撑的分析能力，主要用于：

- 大规模对象转换；
- 左连接、右连接、内连接和全连接；
- 派生列；
- 大规模筛选；
- 聚合；
- 集合运算；
- SQL；
- 大规模类别图表。

常见 Materialization Card 包括：

- Object Set Materialization；
- Expression；
- Filter Materialization；
- Join Materializations；
- Materialization SQL；
- Numeric Aggregation；
- Set Math；
- Unique Column Values；
- Categorical Plot。

例如：

```text
千万级订单对象
    ↓ Object Set Materialization
Expression：计算延期天数和风险等级
    ↓
Join Materialization：关联客户和工厂
    ↓
Filter Materialization：只保留高风险订单
    ↓
按工厂聚合
    ↓
Categorical Plot
```

Materialization 主要面向大规模对象和表格型分析，但并不替代专门的 Time Series 分析引擎。

### 计算层选择建议

| 数据形态 | 推荐方式 |
|---|---|
| Ontology 原生筛选与关系穿透 | Object Set |
| 中小规模派生、Join、编辑 | Transform Table |
| 大规模 Join、聚合、转换 | Materialization |
| 高频信号与时间窗口计算 | Time Series Engine |

---

## 7. SQL 能力

## 7.1 Ontology SQL

Quiver 可以直接对分析中的 Object Set 编写 SQL，例如：

```sql
SELECT
    factory_name,
    COUNT(*) AS delayed_orders,
    AVG(delay_days) AS avg_delay_days
FROM delayed_orders
GROUP BY factory_name
ORDER BY delayed_orders DESC
```

Ontology SQL 可以完成：

- Select；
- Filter；
- Join；
- Aggregate；
- 引用 Object Set；
- 引用上游 SQL 结果；
- 使用 Number、String、Date、Boolean 参数；
- 转换成 Transform Table；
- 通过 AIP 生成 SQL。

## 7.2 Materialization SQL

Materialization SQL 面向后端 Materialization 数据，适合处理更大规模的数据。

Quiver 的复杂度层次可以理解为：

```text
点选式 Object Set 操作
        ↓
Transform Table
        ↓
Formula
        ↓
Ontology SQL / Materialization SQL
        ↓
Foundry Code Function
```

---

## 8. Time Series Analytics：时间序列分析

时间序列是 Quiver 最重要的差异化能力之一。

## 8.1 时间序列来源

Time Series 可以来自：

- Ontology Time Series Property；
- Time Series Sync；
- 对象关联的 Sensor；
- 单个 Object；
- Object Set 中弹出的时间序列属性；
- 具有时间戳和值的对象表；
- Transform Table；
- Function 返回的 Time Series。

它既可以分析按日销售额，也可以处理亚秒级温度、压力、振动和设备信号。

## 8.2 Time Series Chart 的结构

Quiver 将时间序列可视化拆分为：

- Chart：时间序列图表容器；
- Plot：Chart 中的一条具体序列；
- Axis：时间轴、相对时间轴、数值轴或序数轴。

例如：

```text
Chart 1：
- 设备温度
- 温度移动平均

Chart 2：
- 设备振动
- 振动阈值

Chart 3：
- 设备功耗
```

多个图表的时间轴可以同步联动。用户缩放一个图表的时间窗口时，其他图表同步变化。

## 8.3 时间序列转换

常见 Time Series Card 包括：

- Rolling Aggregate；
- Periodic Aggregate；
- Formula；
- Derivative；
- Integral；
- Union；
- Coalesce；
- Combine；
- Filter；
- Sample；
- Time Shift；
- Bollinger Bands；
- Regression；
- DSP Filter；
- Linear Aggregation；
- Linked Series Aggregation。

例如：

```text
原始振动序列
    ↓ Rolling Aggregate
30分钟移动平均
    ↓ Derivative
振动变化速度
```

## 8.4 插值、采样和对齐

实际业务中，不同传感器可能具有不同采样频率：

```text
温度：每秒一次
振动：每100毫秒一次
能耗：每分钟一次
```

Quiver 需要处理：

- 时间点对齐；
- 缺失值；
- 插值；
- 重采样；
- 不同频率的聚合；
- 时间窗口；
- 序列合并。

不同 Card 可能使用不同插值和对齐语义，因此分析人员仍需理解计算规则。

---

## 9. Event Set 与异常分析

## 9.1 Time Series Search

Quiver 可以把满足某个条件的时间区间转成 Event Set。

例如：

```text
温度 > 90℃
AND
振动 > 7 mm/s
AND
持续时间 > 10分钟
```

Time Series Search 会逐点扫描：

- 条件首次满足时，事件开始；
- 条件持续满足时，事件继续；
- 条件不满足时，事件结束；
- 最终输出 Event Set。

支持的搜索方式包括：

### Threshold

```text
temperature > 90
pressure <= 30
status != "NORMAL"
```

### Bounded

判断序列是否超出动态上下界：

```text
实际温度超出动态上限或动态下限
```

### Formula

```text
$temperature > $temperatureLimit
&&
$vibration > $vibrationLimit
```

还可以限定事件的最短或最长持续时间。

## 9.2 Event Set 的来源

Event Set 可以来自：

- Time Series Search；
- 具有开始和结束时间的 Event Object；
- 关联对象；
- Transform Table；
- Materialization；
- 手工指定的时间范围。

事件可以携带：

```text
startTime
endTime
eventType
severity
equipmentId
batchId
operator
qualityScore
```

## 9.3 Event 分析能力

Event Set 可以用于：

- Event Statistics；
- Event Indicator；
- Filter Time Series by Event；
- Event Comparison Plot；
- Time Shift；
- Deduplicate；
- Reference Profile。

例如研究设备故障前一小时的共同规律：

```text
所有故障事件
    ↓ 向前扩展60分钟
故障前观察窗口
    ↓
截取每次事件对应的温度和振动
    ↓
以故障开始时间对齐
    ↓
叠加50次故障前的曲线
    ↓
寻找共同前兆
```

这类基于事件对齐的相对时间分析，是普通 BI 工具很难完成的。

## 9.4 批量时间序列分析

Quiver 可以对大量对象关联的时间序列进行批量处理，例如：

```text
每台设备：
原始振动
    ↓
30分钟移动平均
    ↓
一阶导数
    ↓
阈值搜索
    ↓
异常事件数量
```

还可以使用：

- Grouped Time Series Plot；
- 多序列搜索；
- Linear Aggregation；
- Transform Table 中的批量序列操作。

---

## 10. Formula、Visual Function 和 Code Function

## 10.1 Formula Language

Quiver 支持通过公式引用其他 Card：

```text
$A / $B
($sales - $cost) / $sales
rollingAverage($temperature)
```

公式输入可以是：

- Number；
- Categorical Chart；
- Time Series；
- Transform Table Column。

公式能力包括：

- 数值公式；
- 时间序列公式；
- 类别图表公式；
- Segment Formula；
- 自动补全。

## 10.2 Visual Function

Visual Function 是通过 Quiver Card 搭建的可复用分析逻辑。

例如统一的高风险订单函数：

```text
输入：
- Order Object Set
- 风险阈值 Number

内部：
- 筛选订单
- 关联客户
- 计算延期风险
- 过滤高风险对象

输出：
- High Risk Order Object Set
```

Visual Function 的价值包括：

- 无需代码；
- 跨分析复用；
- 可以发布和共享；
- 可以集中升级；
- 减少指标口径不一致；
- 把复杂逻辑封装成可复用模块。

## 10.3 Code Function

复杂算法可以在 Foundry Function 中实现，然后通过 Quiver 调用。

Function 可以返回：

- Object Set；
- Time Series；
- Number；
- String；
- Boolean；
- Array；
- Categorical Plot。

因此 Quiver 并不是完全无代码系统，而是可以把代码能力封装在低代码分析链路中。

---

## 11. 参数化和交互分析

Quiver Parameter 允许用户动态控制分析。

常见参数类型包括：

- Number；
- String；
- Boolean；
- Date/Time；
- Time Range；
- Numeric Range；
- Object Selector；
- Property Value Selector；
- String Selector；
- Transform Table Row Selector；
- Array；
- X/Y Range；
- Action Button。

参数可以控制：

- 筛选条件；
- 时间范围；
- 风险阈值；
- 当前对象；
- 分组维度；
- 图表坐标范围；
- 公式；
- Action 输入。

例如：

```text
工厂：上海工厂
时间范围：最近30天
设备类型：数控机床
异常阈值：7 mm/s
```

参数可以暴露到 Dashboard，使最终用户在不修改分析逻辑的情况下改变分析范围。

---

## 12. Dashboard 能力

Quiver 可以从一个 Analysis 创建一个或多个 Dashboard。

Dashboard 具有以下特点：

- 只读但可交互；
- 可以放置图表、表格、参数和指标；
- 可以包含多个页面或视图；
- 可以发布；
- 可以嵌入 Workshop；
- 可以嵌入 Notepad；
- 可以嵌入 Object View；
- 可以向外部应用暴露输入输出。

例如 Workshop 向 Quiver 传入：

```text
High Priority Aircraft Object Set
```

Quiver 根据输入执行分析，再输出：

```text
用户选中的 Aircraft Object Set
选中飞机数量
```

Workshop 可以把这些输出映射成自己的变量、表格和业务操作。

因此合理边界是：

```text
Quiver：复杂分析逻辑
Workshop：业务交互和流程执行
```

---

## 13. AIP 自然语言分析

Quiver 的 AIP 能力主要包括 AIP Generate 和 AIP Configure。

## 13.1 AIP Generate

用户可以输入自然语言：

```text
显示最近六个月每月的销售趋势
```

或者：

```text
按工厂统计延期订单数量，并只保留前三名
```

AIP 会根据当前 Card、Object Set 和 Ontology 语义，生成候选分析步骤。用户确认后，系统创建和配置对应 Card。

复杂问题可能生成多步分析图。

AIP 的本质不是直接输出一段不可检查的结论，而是：

```text
自然语言问题
    ↓
识别分析意图
    ↓
生成候选 Card 和连接关系
    ↓
用户确认
    ↓
形成可检查、可修改的分析 DAG
```

## 13.2 AIP Configure

AIP Configure 可以修改已有 Card，例如：

```text
把平均值改成总和
只显示华东地区
把时间窗口改成最近90天
按月而不是按天聚合
将图例移动到底部
```

AIP 会给出具体配置建议，用户可以接受或拒绝。

## 13.3 自然语言能力的前提

自然语言分析仍依赖预先建立的：

- 数据接入；
- Object Type；
- Property；
- Link Type；
- Time Series Property；
- 标准业务指标；
- 权限；
- 可复用函数。

因此 AIP 降低的是分析链路的搭建成本，而不是替代 Ontology 建模和业务治理。

---

## 14. Action 和写回 Ontology

Quiver 不只读取和展示数据，还可以通过 Action 写回 Ontology。

Action 可以：

- 创建新对象；
- 更新对象属性；
- 修改对象关系；
- 创建 Annotation；
- 创建维修工单；
- 修改订单优先级；
- 更新风险等级；
- 分配负责人。

例如：

```text
发现设备异常
    ↓
选择异常时间范围
    ↓
执行 Create Annotation Action
    ↓
创建异常事件对象
```

或者：

```text
筛选高风险订单
    ↓
执行 Update Priority Action
    ↓
批量修改订单优先级
```

在时间序列图中，用户框选的时间范围还可以映射为 Action 参数：

```text
框选左边界 → eventStart
框选右边界 → eventEnd
```

从而创建带开始和结束时间的事件对象。

最终形成闭环：

```text
观察
 → 分析
 → 识别对象
 → 做出判断
 → 执行 Action
 → 写回 Ontology
```

---

## 15. 保存、版本和协作

Quiver Analysis 可以保存版本，并查看和恢复已保存版本。

典型能力包括：

- 手工保存；
- 保存版本历史；
- 恢复旧版本；
- 保留未正式保存的工作状态；
- 多用户独立编辑。

需要注意，多人同时编辑并不一定等于 Google Docs 式的实时合并。多人分别保存时，后保存的版本可能覆盖前一个版本，因此正式平台中需要重视协作、锁定、分支和冲突解决机制。

---

## 16. 完整案例：设备异常根因分析

业务问题：

> 为什么生产线 Equipment-017 最近频繁停机，哪些批次和订单受到影响？

### 第一步：选择设备对象

```text
Equipment Object Set
    ↓ Filter
equipmentId = "Equipment-017"
    ↓ Object Selector
Equipment-017
```

### 第二步：读取时间序列

```text
Equipment-017
    ├→ Temperature Time Series
    ├→ Vibration Time Series
    ├→ Pressure Time Series
    └→ Power Consumption Time Series
```

### 第三步：信号处理

```text
Vibration
    ↓ Rolling Aggregate
振动移动平均
    ↓ Derivative
振动上升速度
```

```text
Temperature
    ↓ Rolling Aggregate
温度移动平均
```

### 第四步：异常搜索

```text
条件：
振动移动平均 > 7
AND
温度移动平均 > 85
AND
持续时间 > 10分钟
```

输出：

```text
Equipment Anomaly Event Set
```

### 第五步：事件分析

```text
异常事件
    ↓ 向前扩展30分钟
    ↓ Event Comparison Plot
比较每次异常前30分钟的：
- 温度
- 振动
- 功耗
```

可能发现：

```text
多数停机前：
振动先快速升高
15分钟后温度上升
最后功耗突然下降
```

### 第六步：关系穿透

```text
Equipment-017
    ↓ Search Around
Production Batch Object Set
    ↓ 按异常时间重叠筛选
受异常影响的批次
    ↓ Search Around
Order Object Set
    ↓ Search Around
Customer Object Set
```

### 第七步：计算业务影响

```text
受影响批次数量
受影响订单数量
受影响订单金额
战略客户数量
平均预计延期天数
```

### 第八步：发布 Dashboard

Dashboard 展示：

```text
异常次数
停机总时长
受影响订单
受影响金额
异常时间序列
事件对比图
订单明细
```

### 第九步：执行 Action

```text
Create Maintenance Work Order
Update Equipment Risk Level
Annotate Anomaly Event
Assign Maintenance Engineer
Mark Orders At Risk
```

完整流程为：

> **对象定位 → 关系穿透 → 时间序列处理 → 异常转事件 → 影响分析 → Dashboard → Action。**

---

## 17. Quiver 不负责什么

## 17.1 不是 Ontology 建模工具

Object Type、Property、Link Type、Action Type 和 Time Series Property 一般需要在 Ontology 管理工具中事先定义。

Quiver 是 Ontology 模型的消费者和分析应用，而不是 Ontology 定义本身的主要管理入口。

## 17.2 不是大型数据管道开发工具

虽然 Transform Table 和 Materialization 能做数据转换，但复杂、生产级 ETL 更适合 Pipeline Builder、Code Repository 或其他数据工程工具。

## 17.3 不是完整的机器学习实验平台

Quiver 可以调用 Function、处理时间序列、做回归和异常搜索，但复杂模型训练、调参、实验管理和模型治理不是其核心定位。

## 17.4 不是自动因果推断工具

Quiver 可以发现：

```text
设备告警增加
订单延期增加
两者在时间上相关
```

但不能仅凭图表自动证明：

```text
设备告警一定导致订单延期
```

因果判断仍需要业务逻辑、实验或额外验证。

## 17.5 不是 Workshop 的替代品

Quiver 擅长：

- 构建分析链路；
- 关系穿透；
- 时间序列分析；
- 根因分析；
- 交互式分析 Dashboard。

Workshop 更适合：

- 完整业务应用；
- 一线运营工作台；
- 审批和任务；
- 多步骤操作；
- 角色和权限交互；
- Action 执行流程。

---

## 18. Quiver 的产品子系统拆解

```text
Quiver
├── 1. Typed Analysis Graph
│   ├── Card 输入输出类型
│   ├── Graph 依赖关系
│   └── Canvas 展示组织
│
├── 2. Object Analytics
│   ├── Object Set
│   ├── Filter
│   ├── Set Math
│   ├── Search Around
│   ├── Aggregation
│   └── Drill-down
│
├── 3. Tabular Compute
│   ├── Transform Table
│   ├── Materialization
│   ├── Formula
│   └── Ontology SQL
│
├── 4. Time Series Analytics
│   ├── Plot 与 Axis
│   ├── Transform
│   ├── Aggregation
│   ├── Sampling 与 Interpolation
│   └── Batch Analysis
│
├── 5. Event Analytics
│   ├── Time Series Search
│   ├── Event Set
│   ├── Event Comparison
│   ├── Event Statistics
│   └── Reference Profile
│
├── 6. Reusable Logic
│   ├── Visual Function
│   ├── Code Function
│   └── Derived Series
│
├── 7. Consumption
│   ├── Parameter
│   ├── Cross Filter
│   ├── Dashboard
│   └── Workshop 嵌入
│
└── 8. Operational Closure
    ├── Action Button
    ├── Annotation
    ├── Object 更新
    └── Link 修改
```

---

## 19. 对自研 Ontology 平台的核心启示

如果要仿照 Quiver 开发分析平台，不应首先把重点放在图表数量，而应优先建设以下五项基础能力。

### 19.1 强类型 Card 和分析 DAG

每个分析节点必须定义：

- 输入类型；
- 输出类型；
- 参数；
- 执行引擎；
- 错误状态；
- 数据血缘；
- 版本信息。

### 19.2 Object Set 作为核心中间态

平台需要允许用户不断构造和转换 Object Set：

```text
全部订单
 → 延期订单
 → 战略客户延期订单
 → 受设备异常影响的战略客户延期订单
```

### 19.3 Link Traversal 与表 Join 分离

平台需要明确区分：

- 关系穿透：改变分析对象；
- 表 Join：保留多个对象的属性。

这两个操作的语义、性能和用户体验都不同。

### 19.4 Time Series 和 Event Set 作为一等类型

不能把时间序列简单处理成普通表格。需要独立支持：

- 时间轴；
- 采样；
- 插值；
- 滚动窗口；
- 信号处理；
- 事件搜索；
- 事件对齐；
- 相对时间分析。

### 19.5 分析结果通过 Action 回到运行态

分析平台不应止步于报表，而应形成：

```text
Object Set
 → 分析
 → 判断
 → Action
 → 写回 Ontology
 → 继续跟踪
```

---

## 20. 建议的自研功能优先级

### P0：对象分析基础

- Card 类型系统；
- 分析 DAG；
- Object Set 数据类型；
- Object Set Filter；
- Set Math；
- Link Traversal；
- 基础聚合；
- 表格和基础图表；
- 参数；
- 保存和版本。

### P1：可复用分析和 Dashboard

- Transform Table；
- Formula；
- Visual Function；
- Selection Object Set；
- Cross Filter；
- Dashboard；
- Workshop 或应用容器嵌入；
- 分析输入输出。

### P2：大规模计算

- Materialization；
- 后端执行引擎；
- 大规模 Join；
- 大规模聚合；
- SQL；
- 缓存；
- 计算状态和执行监控。

### P3：时间序列和事件分析

- Time Series 一等类型；
- Rolling Aggregate；
- Sampling；
- Interpolation；
- Time Series Search；
- Event Set；
- Event Comparison；
- Event Statistics；
- 批量时间序列分析。

### P4：AIP 和 Action 闭环

- 自然语言生成分析计划；
- 自然语言配置 Card；
- 可审查的 AIP 分析步骤；
- Action Button；
- Object 更新；
- Link 更新；
- Annotation；
- 审计和权限控制。

---

## 21. 最终总结

Quiver 不是普通的 Dashboard 工具，而是一套：

> **以 Ontology 为语义基础，以 Object Set、Time Series 和 Event Set 为核心数据类型，以强类型 Card 和分析 DAG 为执行模型，以 Dashboard 和 Action 为消费与业务闭环的低代码分析环境。**

它的核心竞争力不在于图表样式，而在于：

1. 强类型分析图；
2. Object Set 分析范式；
3. Ontology Link 关系穿透；
4. 深度时间序列与事件分析；
5. 可复用 Visual Function；
6. 分析输入输出和 Dashboard 嵌入；
7. AIP 生成可审查分析链路；
8. Action 写回运行态。

对于自研 Ontology 平台，最重要的不是复制 Quiver 的界面，而是复制它的分析抽象和执行模型。

---

## 参考资料

- Palantir Foundry Quiver Overview  
  https://www.palantir.com/docs/foundry/quiver/overview/
- Quiver Core Concepts  
  https://www.palantir.com/docs/foundry/quiver/core-concepts/
- Quiver Objects Overview  
  https://www.palantir.com/docs/foundry/quiver/objects-overview/
- Quiver Analysis Graph  
  https://www.palantir.com/docs/foundry/quiver/analysis-graph/
- Quiver Analysis Canvas  
  https://www.palantir.com/docs/foundry/quiver/analysis-canvas/
- Quiver Transform Table  
  https://www.palantir.com/docs/foundry/quiver/cards-transform-table/
- Quiver Materializations  
  https://www.palantir.com/docs/foundry/quiver/cards-index-materializations/
- Quiver Time Series Overview  
  https://www.palantir.com/docs/foundry/quiver/timeseries-overview/
- Quiver Event Analysis  
  https://www.palantir.com/docs/foundry/quiver/timeseries-analyze-events-data/
- Quiver Visual Functions  
  https://www.palantir.com/docs/foundry/quiver/visual-functions-overview/
- Quiver AIP  
  https://www.palantir.com/docs/foundry/quiver/quiver-aip/
- Quiver Dashboards  
  https://www.palantir.com/docs/foundry/quiver/dashboards-overview/
