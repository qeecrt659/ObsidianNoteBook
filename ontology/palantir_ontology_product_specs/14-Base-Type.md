# Base Type 产品说明书

> 资料范围：Palantir 官方 Foundry / Ontology 文档；整理日期：2026-09-05。
> 说明：本文是对多个官方页面的结构化产品化整理与中文解释，不是对官网的逐字复制。所有关键结论均附官方来源链接。

## 1. 定义

Base Type 定义 Property 中可以存储的数据种类，并影响应用对该 Property 能执行的操作。它类似编程语言中的 primitive/structural type，是 Ontology Property 类型系统的底层基础。

Base Type 与 Value Type 不同：

- Base Type：技术数据形态；
- Value Type：基于 Base Type 的业务语义和约束。

## 2. 常用 Base Types

官方 Property 文档覆盖：

### 标量类型

- String
- Boolean
- Byte / Short / Integer / Long
- Float / Double / Decimal
- Date / Timestamp

### 集合/结构

- Array
- Struct

### 搜索与 AI

- Vector：用于语义搜索等向量场景。

### 地理空间

- Geopoint
- Geoshape

### 文件/媒体

- Attachment
- Media Reference

### 时间相关高级类型

- Time Series
- Geotemporal Series

### 安全/特殊类型

- Marking
- Cipher 等。

## 3. Primary Key / Title Key 兼容性

不是所有 Base Type 都能作为 key。

例如：

- String、Integer、Short 可作为 Primary Key/Title Key；
- Date/Timestamp 技术上可做主键但官方不推荐，容易产生碰撞或表示差异；
- Float/Double/Decimal 不可作为 Primary Key；
- Vector、Struct、Attachment、Time Series 等不可作为 key；
- Array 可在有限条件下作 Title，但不可作 Primary Key。

因此建模工具应把 key compatibility 作为 Base Type schema 的元数据，而不是写死在页面里。

## 4. Base Type 决定应用能力

Base Type 会决定：

- 可用 filter operators；
- aggregation；
- formatting；
- sorting/searching；
- Action parameter type；
- Value Type 可使用的 constraints；
- UI widget 能否使用该 Property。

## 5. Array / Struct 注意点

官方指出：

- Array 不能包含 null elements；
- OSv2 不支持 nested arrays；
- Struct 不支持任意嵌套，且字段能力存在限制。

自研类型系统应显式描述嵌套规则与 storage compatibility。

## 6. JavaScript 与 Long

Palantir 特别提醒 Long 在 JavaScript 中存在大整数精度/表示问题，因此很多场景更适合使用 String 作为稳定标识。这说明 Ontology 类型选择不仅取决于数据库，也取决于 SDK/应用生态。

## 7. 与 Value Type 的关系

Value Type 必须基于 Base Type。Constraint 的合法集合由 Base Type 决定。例如 String 支持 Regex，数值支持 Range，Array 支持 uniqueness/nested constraints。

## 8. 自研建议

- 建立统一 Type Registry，记录 storage、SDK、UI、filter、key、constraint 能力。
- 类型兼容性由 schema 驱动，不散落在代码分支。
- 对跨语言不安全类型提供 lint，例如 Long、Timestamp primary key。
- Base Type 变更视为潜在 breaking change。

## 官方原始资料

- https://www.palantir.com/docs/foundry/object-link-types/base-types
- https://www.palantir.com/docs/foundry/object-link-types/properties-overview
- https://www.palantir.com/docs/foundry/object-link-types/value-type-constraints
