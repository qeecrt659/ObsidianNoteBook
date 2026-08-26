# TiDB 保留 30 天时查询近 60 天数据的实现方案

## 1. 核心结论

TiDB 只保留最近 30 天的事件明细，并不代表系统只能查询最近 30 天。

查询近 60 天时，可以把时间范围拆成两部分：

```text
近 60 天
├── 最近 30 天：TiDB
└── 第 31～60 天：Iceberg
```

然后将两部分结果合并、排序、去重并分页返回。

不过更推荐让 Iceberg 持续接收 TiDB 的增量数据，而不是等数据满 30 天后才归档。这样 Iceberg 本身也包含最近 60 天的完整数据。

---

## 2. 推荐的数据保存方式

假设当前时间为：

```text
2026-08-06 15:00:00
```

查询近 60 天，对应：

```text
开始时间：2026-06-07 15:00:00
结束时间：2026-08-06 15:00:00
```

TiDB 保留最近 30 天：

```text
2026-07-07 15:00:00
至
2026-08-06 15:00:00
```

Iceberg 保存完整历史：

```text
2026-06-07 15:00:00
至
2026-08-06 15:00:00
```

推荐数据链路：

```text
业务写入 TiDB
      ↓
TiCDC
      ↓
Kafka
      ↓
Flink
      ↓
实时或准实时写入 Iceberg
```

因此，Iceberg 不需要等 TiDB 清理旧数据时才收到历史记录。

---

## 3. 方案一：直接查询 Iceberg

如果业务允许几秒到几分钟的数据延迟，查询近 60 天事件时可以直接查询 Iceberg：

```sql
SELECT
    tenant_id,
    event_id,
    object_type_id,
    object_id,
    event_type,
    event_time,
    payload
FROM iceberg.ontology.event_history
WHERE tenant_id = 10001
  AND event_time >= TIMESTAMP '2026-06-07 15:00:00'
  AND event_time <  TIMESTAMP '2026-08-06 15:00:00'
ORDER BY event_time DESC;
```

适合：

- 历史事件列表；
- 统计分析；
- 报表；
- 趋势查询；
- 大范围导出；
- AI 分析；
- 对实时性要求不高的查询。

这是近 60 天分析查询的推荐方式。

---

## 4. 方案二：TiDB 与 Iceberg 联合查询

如果查询必须包含刚刚产生的事件，而 Iceberg 存在同步延迟，则需要同时查询两个数据源。

推荐不要固定按 30 天切割，而是使用 Iceberg 的同步水位 `Watermark`。

例如 Iceberg 已确认同步到：

```text
watermark = 2026-08-06 14:58:00
```

查询近 60 天时：

```text
Iceberg：
2026-06-07 15:00:00
到
2026-08-06 14:58:00

TiDB：
2026-08-06 14:58:00
到
2026-08-06 15:00:00
```

优势：

- TiDB 只查询很小的实时窗口；
- Iceberg 承担绝大部分历史查询；
- 避免因同步延迟遗漏数据；
- 减少 TiDB 的扫描和分析压力。

---

## 5. 固定 30 天边界的基础实现

如果第一版还没有 Watermark 管理，可以先按 30 天边界拆分。

定义：

```text
start_time  = 当前时间 - 60 天
cutoff_time = 当前时间 - 30 天
end_time    = 当前时间
```

查询 Iceberg：

```sql
SELECT
    tenant_id,
    event_id,
    object_type_id,
    object_id,
    event_type,
    event_time,
    payload
FROM iceberg.ontology.event_history
WHERE tenant_id = 10001
  AND event_time >= :start_time
  AND event_time <  :cutoff_time;
```

查询 TiDB：

```sql
SELECT
    tenant_id,
    event_id,
    object_type_id,
    object_id,
    event_type,
    event_time,
    payload
FROM ontology_runtime.event_recent
WHERE tenant_id = 10001
  AND event_time >= :cutoff_time
  AND event_time <  :end_time;
```

合并流程：

```text
Iceberg 结果
    +
TiDB 结果
    ↓
合并
    ↓
按 event_time、event_id 排序
    ↓
去重
    ↓
分页返回
```

时间边界必须统一采用半开区间：

```text
Iceberg：event_time < cutoff_time
TiDB：event_time >= cutoff_time
```

避免两边同时包含边界数据。

---

## 6. 通过 Trino 统一查询

可以部署 Trino，并配置：

```text
tidb catalog
iceberg catalog
```

统一 SQL 示例：

```sql
SELECT
    tenant_id,
    event_id,
    object_type_id,
    object_id,
    event_type,
    event_time,
    payload
FROM iceberg.ontology.event_history
WHERE tenant_id = 10001
  AND event_time >= TIMESTAMP '2026-06-07 15:00:00'
  AND event_time <  TIMESTAMP '2026-07-07 15:00:00'

UNION ALL

SELECT
    tenant_id,
    event_id,
    object_type_id,
    object_id,
    event_type,
    event_time,
    payload
FROM tidb.ontology_runtime.event_recent
WHERE tenant_id = 10001
  AND event_time >= TIMESTAMP '2026-07-07 15:00:00'
  AND event_time <  TIMESTAMP '2026-08-06 15:00:00'

ORDER BY event_time DESC, event_id DESC;
```

架构：

```text
应用
  ↓
Trino
  ├── TiDB Connector
  └── Iceberg Connector
```

优点是应用只发送一条查询，不需要自行连接两个数据源。

需要在生产使用前验证：

- TiDB 与 Trino MySQL Connector 的兼容性；
- JSON、时间、二进制 ID 等数据类型映射；
- 查询下推能力；
- 跨源查询性能；
- 超时和资源隔离。

---

## 7. 由 Ontology Query Service 进行查询路由

如果不希望引入 Trino，可以由 Ontology Query Service 完成路由和合并：

```text
Ontology Query Service
        │
        ├── 查询 TiDB
        ├── 查询 Iceberg
        └── 合并结果
```

推荐路由规则：

```text
查询范围全部在最近 30 天
→ 只查 TiDB

查询范围全部早于 30 天
→ 只查 Iceberg

查询范围跨越 30 天边界
→ 同时查询 TiDB 和 Iceberg

查询范围很长且允许少量延迟
→ 只查 Iceberg
```

Java 伪代码：

```java
public EventPage queryEvents(EventQuery query) {
    Instant cutoff = clock.instant().minus(30, ChronoUnit.DAYS);

    if (!query.getStartTime().isBefore(cutoff)) {
        return tidbEventStore.query(query);
    }

    if (!query.getEndTime().isAfter(cutoff)) {
        return icebergEventStore.query(query);
    }

    List<Event> historicalEvents =
        icebergEventStore.query(
            query.withTimeRange(query.getStartTime(), cutoff)
        );

    List<Event> recentEvents =
        tidbEventStore.query(
            query.withTimeRange(cutoff, query.getEndTime())
        );

    return mergeSortDeduplicateAndPage(
        historicalEvents,
        recentEvents
    );
}
```

生产实现中，建议使用 Watermark 代替固定 30 天边界。

---

## 8. 必须处理重复数据

TiDB 和 Iceberg 通常会有一段数据重叠：

```text
TiDB 保留最近 30 天
Iceberg 也已同步最近 30 天
```

如果同时查询重叠区间，就可能返回重复事件。

每条事件必须有全局唯一标识：

```text
event_id
```

推荐唯一键：

```text
tenant_id + event_id
```

或者：

```text
tenant_id + source_system + source_event_id
```

合并后可以按唯一键去重：

```sql
ROW_NUMBER() OVER (
    PARTITION BY tenant_id, event_id
    ORDER BY source_priority DESC
)
```

如果数据重复，可以规定：

```text
TiDB 的数据优先于 Iceberg
```

但更好的办法是严格使用 Watermark 切割：

```text
Iceberg：event_time < watermark
TiDB：event_time >= watermark
```

---

## 9. 分页不要简单使用 OFFSET

跨 TiDB 和 Iceberg 查询时，不建议使用：

```sql
LIMIT 100 OFFSET 100000
```

原因：

- 两边都可能扫描大量数据；
- 新事件持续写入，OFFSET 对应的位置会变化；
- 深分页性能差；
- 容易产生漏数据或重复数据。

推荐使用游标分页，排序字段为：

```text
event_time
event_id
```

第一页：

```sql
ORDER BY event_time DESC, event_id DESC
LIMIT 100;
```

下一页：

```sql
WHERE (
    event_time < :last_event_time
    OR (
        event_time = :last_event_time
        AND event_id < :last_event_id
    )
)
ORDER BY event_time DESC, event_id DESC
LIMIT 100;
```

返回给前端的游标示例：

```json
{
  "lastEventTime": "2026-07-20T15:31:12.123Z",
  "lastEventId": "EVT-100086"
}
```

---

## 10. TiDB 与 Iceberg 的 Schema 必须兼容

联合查询字段需要统一：

```text
字段名称一致
字段含义一致
数据类型兼容
时间统一使用 UTC
event_id 规则一致
枚举值一致
删除标记一致
```

推荐统一事件模型：

```text
tenant_id
event_id
object_type_id
object_id
event_type
operation
event_time
ingested_at
operator_id
action_execution_id
payload
source_system
```

例如：

```text
TiDB：
event_time DATETIME(6)

Iceberg：
event_time TIMESTAMP
```

需要明确二者采用相同的时间语义和时区。

---

## 11. 事件时间和同步时间必须分开

事件表至少应保存两个时间字段：

```text
event_time：
业务事件真正发生的时间

ingested_at / commit_ts：
系统接收或数据库提交的时间
```

例如，离线设备在 8 月 6 日重新联网，上报了 8 月 4 日发生的事件：

```text
event_time  = 2026-08-04
ingested_at = 2026-08-06
```

用户查询“8 月 4 日发生的事件”时，应按 `event_time` 查询。

判断 Iceberg 是否同步完整以及维护 Watermark 时，应依据：

```text
commit_ts
ingested_at
CDC offset
```

不能只依赖 `event_time` 判断同步进度。

---

## 12. Watermark 管理建议

建议维护一张同步水位表：

```text
iceberg_sync_watermark
```

示例字段：

```text
pipeline_name
source_database
source_table
tenant_id
last_commit_ts
last_kafka_offset
last_event_id
updated_at
sync_status
```

例如：

| pipeline_name | source_table | last_commit_ts | sync_status |
|---|---|---|---|
| event-to-iceberg | event_recent | 2026-08-06 14:58:00 | HEALTHY |

Query Service 查询前读取 Watermark：

```text
查询开始时间 → Watermark
    查 Iceberg

Watermark → 当前时间
    查 TiDB
```

Watermark 必须表示：

> 该时间点之前的所有变更已经完整、成功且可查询地写入 Iceberg。

不能只使用 Flink 任务最近收到事件的时间。

---

## 13. Watermark 异常时的处理

如果同步任务延迟或异常：

```text
Iceberg Watermark 落后 2 小时
```

则查询应自动扩大 TiDB 实时窗口：

```text
Iceberg：
开始时间到 Watermark

TiDB：
Watermark 到当前时间
```

如果 Watermark 早于 TiDB 的数据保留边界，则可能出现查询缺口：

```text
TiDB 最早只保留到 30 天前
Iceberg 只同步到 35 天前
```

此时中间 5 天的数据可能无法查询。

因此，清理 TiDB 数据前必须满足：

```text
Iceberg Watermark 已超过待清理分区结束时间
数据数量核对通过
抽样或校验和验证通过
安全等待期已结束
```

---

## 14. TiDB 数据清理的安全条件

删除 TiDB 中超过 30 天的事件前，建议检查：

1. TiCDC 和 Flink 任务状态正常；
2. Kafka 消费无明显积压；
3. Iceberg 对应分区已写入完成；
4. Watermark 已超过待清理时间；
5. TiDB 与 Iceberg 记录数量核对通过；
6. 删除事件和更新事件已正确处理；
7. Schema 变更已同步；
8. 已经过安全缓冲期；
9. Iceberg 快照已提交并可查询；
10. 有异常回放或重新同步方案。

推荐留出缓冲区：

```text
逻辑保留期：30 天
实际删除时间：32～35 天
```

避免在同步延迟时立即删除 TiDB 数据。

---

## 15. 对 Ontology 平台的推荐查询策略

### 15.1 普通在线查询

查询最近几小时或最近几天：

```text
Ontology Query API
        ↓
TiDB
```

优点：

- 实时；
- 延迟低；
- 适合在线事件列表。

### 15.2 历史和大范围查询

查询近 60 天、半年或几年：

```text
Ontology Query API
        ↓
Trino 或历史查询服务
        ↓
Iceberg
```

优点：

- 避免给 TiDB 带来大范围扫描；
- 适合历史分析和导出；
- 可跨数据源和跨 Object Type 查询。

### 15.3 严格实时的近 60 天查询

```text
Iceberg：
查询开始时间到 CDC Watermark

TiDB：
查询 CDC Watermark 到当前时间

Query Service 或 Trino：
合并、排序、去重、游标分页
```

推荐架构：

```text
                    Ontology Query API
                             │
                 根据时间范围和实时性路由
                 ┌───────────┴───────────┐
                 ▼                       ▼
               TiDB                    Trino
          最新实时事件查询                 │
                                      Iceberg
                                  历史及大范围查询
```

---

## 16. 最终建议

如果 Iceberg 已通过 TiCDC、Kafka 和 Flink 持续同步：

```text
查询近 60 天，允许少量延迟
→ 直接查询 Iceberg

查询近 60 天，要求包含刚刚产生的数据
→ Iceberg 查询历史部分
  + TiDB 查询 Watermark 之后的实时部分
  + 合并、排序、去重和游标分页
```

TiDB 保留 30 天的目的，是控制在线事务数据库的容量和扫描压力，并不会限制平台只能查询最近 30 天。

真正需要避免的是：

```text
近 60 天的大范围查询全部扫描 TiDB
```

这类查询应主要由 Iceberg 和 Trino 或专门的历史查询服务承担。

