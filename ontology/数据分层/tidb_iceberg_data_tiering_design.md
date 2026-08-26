# TiDB 与 Iceberg 的数据分层设计

## 1. 核心结论

使用 TiDB 后，不需要把 TiDB 中的所有数据都复制到 Iceberg。

推荐按用途分层：

```text
TiDB：
保存“当前是什么状态”
负责在线查询、Action、事务、约束和权限校验

Iceberg：
保存“过去发生过什么”
负责完整历史、长期归档、对象快照和大规模分析
```

对于 Ontology Runtime，可将两者定位为：

```text
TiDB = 当前态权威数据库
Iceberg = 历史事实库和分析数据湖
```

---

## 2. 应保留在 TiDB 中的数据

### 2.1 Ontology 当前元数据

包括：

```text
Object Type
Property
Link Type
Action Type
Interface
权限策略
存储映射
Ontology 当前版本
草稿状态
发布状态
```

对应表可以包括：

```text
ontology_object_type
ontology_property
ontology_link_type
ontology_action_type
ontology_interface
ontology_storage_mapping
ontology_version
```

这些数据通常规模不大，但需要：

- 强事务；
- 唯一约束；
- 引用完整性；
- 低延迟查询；
- 发布过程的一致性。

因此，它们应保留在 TiDB 中。

---

### 2.2 对象当前状态

例如：

```text
equipment_current
customer_current
order_current
factory_current
supplier_current
```

只保存每个对象最新的业务状态：

| object_id | status | factory_id | updated_at |
|---|---|---|---|
| E001 | RUNNING | F001 | 2026-08-06 14:30 |

这些数据需要支持：

- 按 Object ID 快速读取；
- Action 实时修改；
- 乐观锁；
- 唯一性约束；
- 多表事务；
- 最新状态查询。

因此，对象当前状态必须保留在 TiDB 中。

---

### 2.3 当前有效关系

例如：

```text
equipment_factory_link_current
customer_order_link_current
object_link_current
```

保存当前有效关系：

```text
设备 E001 当前属于工厂 F001
客户 C001 当前拥有订单 O001
员工 P001 当前属于部门 D001
```

当前关系经常参与 Action 执行、权限判断和在线关系查询，应保留在 TiDB 中。

---

### 2.4 Action 运行态数据

例如：

```text
action_execution
action_lock
action_idempotency
workflow_instance
workflow_task
transaction_outbox
```

这些数据用于：

- 判断 Action 是否已执行；
- 防止重复提交；
- 保存执行中状态；
- 对对象加锁；
- 驱动工作流；
- 保证事务与消息一致性。

Iceberg 不适合低延迟单行事务，因此这些运行态数据必须保留在 TiDB 中。

---

### 2.5 当前权限和安全数据

例如：

```text
user_role
object_permission
data_scope
policy_binding
authorization_cache
```

权限数据参与在线请求链路，不能在执行 Action 时实时依赖 Iceberg 查询。

---

## 3. 应进入 Iceberg 的数据

### 3.1 对象属性变更历史

这是最应优先进入 Iceberg 的数据。

例如设备 E001 的状态变化：

```text
2026-08-01：STOPPED
2026-08-02：MAINTENANCE
2026-08-03：RUNNING
2026-08-05：FAILED
2026-08-06：RUNNING
```

TiDB 只保留：

```text
E001 → RUNNING
```

Iceberg 保存完整变更：

```text
object_change_history
```

推荐字段：

```text
tenant_id
object_type_id
object_id
operation
property_name
old_value
new_value
before_payload
after_payload
changed_at
action_execution_id
operator_id
source_system
commit_ts
```

---

### 3.2 对象周期快照

除逐条变更事件外，建议定期生成对象完整快照：

```text
equipment_snapshot_daily
customer_snapshot_daily
order_snapshot_daily
```

推荐字段：

```text
snapshot_date
tenant_id
object_type_id
object_id
完整对象属性
object_version
is_deleted
```

用途包括：

- 查询某一天的对象完整状态；
- 对比两个时间点的差异；
- 重建分析数据；
- 模型训练；
- 数据质量审计；
- 灾难恢复后的业务核对。

需要区分：

```text
Iceberg Snapshot：
Iceberg 表文件集合的版本

Object Snapshot：
某个业务时间点的完整对象状态
```

---

### 3.3 对象关系历史

TiDB 中只保存当前关系：

```text
设备 E001 当前属于工厂 F002
```

Iceberg 中保存关系历史：

```text
object_link_history
```

推荐字段：

```text
tenant_id
link_type_id
source_object_type
source_object_id
target_object_type
target_object_id
operation
valid_from
valid_to
changed_at
action_execution_id
```

---

### 3.4 Action 完整执行历史

TiDB 中只需保留：

- 正在执行的 Action；
- 最近一段时间的 Action；
- 需要在线重试的 Action；
- 幂等和状态控制信息。

长期历史写入：

```text
action_execution_history
```

推荐字段：

```text
action_execution_id
action_type_id
tenant_id
target_object_id
request_payload
result_payload
before_state
after_state
operator_id
started_at
completed_at
execution_status
failure_reason
trace_id
```

保留策略示例：

```text
TiDB：最近 3～6 个月
Iceberg：全部历史
```

---

### 3.5 审计日志

适合进入 Iceberg 的审计数据包括：

```text
对象查看记录
对象修改记录
权限变更记录
Ontology 发布记录
数据导出记录
Action 调用记录
管理员操作记录
Agent 操作记录
API 调用审计
```

审计日志通常只追加、数据量大、很少修改并需要长期保留，因此适合 Iceberg。

---

### 3.6 IoT、事件和时序历史

例如：

```text
设备遥测数据
传感器读数
设备告警
生产事件
车辆轨迹
系统运行指标
```

建议拆分为：

```text
TiDB：
equipment_current
保存设备当前温度、当前状态和最近心跳

Iceberg：
equipment_telemetry_history
保存全部温度、压力、振动和心跳历史
```

---

### 3.7 已删除对象和 Tombstone

对象删除后，TiDB 可以先软删除，在安全周期后物理删除。

Iceberg 应保留完整删除事实：

```text
tenant_id
object_type_id
object_id
operation = DELETE
before_payload
deleted_at
deleted_by
action_execution_id
deletion_reason
```

这样可以追溯对象删除时间、删除前状态、操作者和触发 Action。

---

### 3.8 原始接入数据

Ontology 平台通常从多个系统接入数据：

```text
ERP
CRM
MES
IoT
外部数据库
CSV 文件
API
Kafka
```

建议在 Iceberg 中保存接近源数据的原始层：

```text
raw_erp_order
raw_crm_customer
raw_mes_equipment
raw_iot_event
```

可采用三层结构：

```text
Bronze：
原始接入数据

Silver：
清洗和标准化后的对象、属性和关系

Gold：
面向分析的事实表、指标和宽表
```

---

### 3.9 分析型宽表和事实表

例如：

```text
customer_360_profile
equipment_failure_fact
order_revenue_fact
supplier_risk_fact
object_relationship_fact
```

这类表数据量大、以批量读取和聚合为主、不参与在线事务，适合放在 Iceberg。

---

### 3.10 从 TiDB 清理出去的冷数据

推荐流程：

```text
1. 确认数据已完整进入 Iceberg
2. 核对记录数量和校验结果
3. 等待安全保留期
4. 删除 TiDB 旧分区或旧数据
```

---

## 4. 不应以 Iceberg 作为在线权威来源的数据

| 数据 | 原因 |
|---|---|
| 对象当前状态 | 需要低延迟读取和事务更新 |
| 当前有效关系 | Action 和业务查询频繁使用 |
| Action 执行中状态 | 需要锁、重试和状态转换 |
| 幂等记录 | 需要快速判断重复请求 |
| Outbox 未发送记录 | 需要与业务更新同事务提交 |
| 当前权限 | 每次请求需要实时校验 |
| 工作流待办 | 高频读取和更新 |
| 唯一性约束数据 | 需要数据库实时保证 |
| Ontology 当前定义 | 引用关系复杂且需要事务一致性 |

Iceberg 可以保存这些数据的历史副本，但不应成为在线权威数据源。

---

## 5. 同一份数据可以同时存在于 TiDB 和 Iceberg

这是推荐模式。

例如：

```text
TiDB：
equipment_current
保存在线当前状态

Iceberg：
equipment_change_history
保存每一次变化

Iceberg：
equipment_snapshot_daily
保存每天的完整状态

Iceberg：
equipment_current_lake
保存当前对象的分析型镜像
```

本质上：

```text
TiDB：
为在线业务运行优化

Iceberg：
为历史、分析、审计和重算优化
```

---

## 6. 推荐的数据同步架构

```text
                 Action Service
                       │
                       ▼
                     TiDB
              当前对象、当前关系
                       │
                     TiCDC
                       ▼
                     Kafka
                       │
                     Flink
              ┌────────┴────────┐
              ▼                 ▼
     Iceberg 变更历史表   Iceberg 当前镜像表
        Append 写入          Upsert 写入
```

需要注意：

> TiCDC 直接写入对象存储后，生成的是 CDC 变更文件，不是已经建好的 Iceberg 表。

仍需要 Flink、Spark 或专门的消费程序完成：

```text
解析 CDC
处理 Insert、Update、Delete
去重
处理 DDL
映射数据类型
写入 Iceberg Catalog
执行小文件合并
维护快照和 Manifest
```

---

## 7. 建议建立两类 Iceberg 表

### 7.1 Append-only 历史表

例如：

```text
object_change_history
```

每次变化只追加一行。

优点：

- 实现简单；
- 审计完整；
- 容易重放；
- 不容易丢失中间变化；
- 适合事件分析。

这是最应优先建设的 Iceberg 表。

### 7.2 当前对象湖仓镜像表

例如：

```text
object_current_lake
```

保存 TiDB 当前对象的分析型副本，可用于：

- 全量对象分析；
- 跨 Object Type 统计；
- 大模型训练；
- 特征工程；
- 离线查询；
- 重建搜索索引；
- 重建分析数据库。

推荐主键：

```text
tenant_id
object_type_id
object_id
```

---

## 8. TiFlash 与 Iceberg 的职责区别

### TiFlash

适合：

- 查询最新对象状态；
- 近实时分析；
- 对 TiDB 当前表聚合；
- 与 TiKV 保持一致；
- 不希望等待数据湖同步。

### Iceberg

适合：

- 多年历史；
- 海量事件；
- 跨系统数据融合；
- 数据科学和模型训练；
- 多计算引擎共享；
- 低成本对象存储；
- 对象和关系历史；
- 长期审计。

可以理解为：

```text
TiKV：
在线当前态

TiFlash：
当前态的近实时分析副本

Iceberg：
长期历史和企业级数据湖
```

---

## 9. 推荐的表级映射

```text
TiDB                             Iceberg

equipment_current        →       equipment_change_history
                         →       equipment_snapshot_daily
                         →       equipment_current_lake

customer_current         →       customer_change_history
                         →       customer_snapshot_daily

object_link_current      →       object_link_history
                         →       object_link_snapshot_daily

action_execution         →       action_execution_history

object_change_event      →       object_change_history

audit_log_recent         →       audit_log_history

iot_event_recent         →       iot_event_history

source_staging_recent    →       raw_source_dataset
```

通常不需要同步到 Iceberg 的运行态内部表：

```text
transaction_lock
action_idempotency
outbox_pending
workflow_task_pending
session
authorization_cache
临时任务表
数据库内部状态表
```

如需长期留痕，应归档最终业务事件或最终执行结果，而不是持续保留每个内部状态。

---

## 10. 针对 Ontology 平台的推荐边界

### TiDB 保存

```text
Ontology 当前定义
当前对象
当前关系
Action 当前状态
工作流运行状态
当前权限
幂等记录
Outbox
近期事件
近期审计
```

### Iceberg 保存

```text
对象完整变更历史
对象周期快照
关系完整变更历史
Action 长期历史
长期审计
已删除对象记录
大规模业务事件
IoT 和时序数据
原始接入数据
清洗后的标准数据集
分析宽表和事实表
AI 训练及特征数据
```

---

## 11. 推荐保留周期示例

```text
TiDB：
当前状态永久保留
事件明细保留 30～180 天
Action 明细保留 90～180 天
审计明细保留 90～365 天

Iceberg：
根据合规和业务要求保留数年或长期保留
```

具体周期应综合考虑：

- 查询频率；
- 合规要求；
- 存储成本；
- 故障恢复目标；
- 重算需求；
- 审计要求。

---

## 12. 最终判断规则

满足以下任意条件的数据，应优先考虑进入 Iceberg：

```text
需要长期保存
主要用于分析
需要回看历史
数据量持续高速增长
很少做单行事务更新
需要被多个计算引擎共享
需要用于审计、训练或重算
从 TiDB 删除后仍需保留
```

满足以下条件的数据，应继续留在 TiDB：

```text
代表当前权威状态
参与 Action 事务
需要毫秒级点查询
需要唯一约束
需要锁或并发控制
经常进行单行更新
影响在线权限和业务决策
```

最终推荐架构：

```text
TiDB
只承担在线当前态和事务

TiCDC + Kafka + Flink
承担可靠的增量同步和格式转换

Iceberg
承担完整历史、快照、归档和分析
```

---

## 13. Iceberg 的运行维护要求

Iceberg 也需要维护。流式写入会持续产生：

- 新快照；
- 小文件；
- Manifest；
- 删除文件；
- 元数据版本。

需要定期执行：

```text
快照过期
孤儿文件清理
数据文件合并
删除文件合并
Manifest 重写
分区优化
统计信息更新
```

建议由独立的数据湖维护任务定期执行，避免小文件和元数据持续累积。

