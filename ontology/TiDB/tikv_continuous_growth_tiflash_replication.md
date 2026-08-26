# TiKV 数据持续增长时，TiFlash 是否会实时同步

## 核心结论

会。

只要某张 TiKV 表已经配置了 TiFlash 副本，后续对该表执行的新增、修改和删除操作，都会持续同步到 TiFlash，不需要定期全量导入，也不需要额外配置 CDC。

但是，需要准确理解 TiFlash 的“实时同步”机制：

- TiKV 到 TiFlash 是持续复制；
- 复制过程是异步的；
- TiKV 事务提交不需要等待 TiFlash；
- TiFlash 查询时会校验复制进度；
- 必要时，查询会等待 TiFlash 追赶到对应的一致性快照。

因此，TiFlash 更准确的机制是：

> 异步持续复制 + 查询时一致性校验

而不是：

> 事务提交时同时同步写入 TiKV 和 TiFlash。

---

## 1. 数据如何持续同步到 TiFlash

首先，需要为目标表创建 TiFlash 副本：

```sql
ALTER TABLE object_instance
SET TIFLASH REPLICA 1;
```

之后的数据链路如下：

```text
应用执行 INSERT / UPDATE / DELETE
              │
              ▼
          TiDB Server
              │
              ▼
             TiKV
              │
      Raft Learner 异步复制
              │
              ▼
           TiFlash
```

例如：

```sql
INSERT INTO object_instance(id, object_type, status)
VALUES (10001, 'Equipment', 'RUNNING');

UPDATE object_instance
SET status = 'STOPPED'
WHERE id = 10001;

DELETE FROM object_instance
WHERE id = 10001;
```

这些事务在 TiKV 提交后，相应变更会通过 Raft 日志持续复制并应用到 TiFlash。

TiFlash 本身不能被业务直接写入，它的数据来源是 TiKV。

---

## 2. “实时同步”不等于完全零延迟

TiFlash 使用 Raft Learner 副本接收 TiKV 的数据。

该复制过程是异步的：

```text
TiKV 事务提交成功
        │
        ├── 立即向应用返回成功
        │
        └── TiFlash 在后台继续同步
```

因此，在某一个瞬间，TiFlash 的物理数据可能比 TiKV 短暂落后。

正常负载下，这种延迟通常较低，可以理解为近实时。但是以下情况可能导致同步延迟明显增加：

- TiKV 写入速度非常高；
- TiFlash CPU、内存或磁盘资源不足；
- TiKV 与 TiFlash 之间网络带宽不足；
- TiFlash 同时执行大量复杂分析查询；
- 新创建 TiFlash 副本，正在同步历史数据；
- TiFlash 节点故障后正在恢复；
- 存在大量 Region Snapshot 传输；
- TiFlash 后台 Merge 或 Apply 处理能力不足。

因此，更准确的说法是：

> TiFlash 会持续近实时同步 TiKV 的数据，但物理复制是异步的，不保证每个时刻都完全零延迟。

---

## 3. 异步复制为什么还能保证查询一致性

虽然 TiKV 到 TiFlash 的物理复制是异步的，但是 TiFlash 查询可以提供与 TiDB 事务快照相匹配的一致性读取。

查询执行时，会携带需要读取的事务时间戳或快照信息。

TiFlash 会检查自己当前的 Raft Apply 进度是否已经覆盖该查询需要的时间点：

```text
查询需要读取到 TSO = 10000
             │
             ▼
TiFlash 检查本地是否已同步到该时间点
             │
       ┌─────┴─────┐
       │           │
   已同步        尚未同步
       │           │
   执行查询      等待追赶
```

只有 TiFlash 已经同步到满足查询快照的位置，查询才会正式读取数据。

例如：

```sql
UPDATE orders
SET status = 'PAID'
WHERE id = 1001;

SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */
       status
FROM orders
WHERE id = 1001;
```

如果 TiFlash 此时物理上落后几十毫秒，查询通常会等待 TiFlash 同步到对应快照，而不是直接返回更新前的旧状态。

但是，如果 TiFlash 落后过多，可能出现：

- 查询等待时间增加；
- 分析查询延迟升高；
- 查询因等待同步而超时；
- 优化器暂时无法使用 TiFlash 副本；
- 分析任务失败或被降级处理。

---

## 4. TiKV 数据持续增长，TiFlash 数据也会持续增长

如果配置了 TiFlash 副本，TiKV 表的数据不断增加，TiFlash 中对应的列式副本也会持续增长。

例如：

```text
TiKV 中的表：
1 TB → 5 TB → 20 TB → 100 TB
```

对应的 TiFlash 存储也会增长，包括：

```text
TiFlash
├── 列式数据
├── Delta 数据
├── MVCC 相关数据
├── 后台合并中的数据
├── 索引和元数据
└── 查询临时数据
```

TiFlash 是列式存储，压缩方式与 TiKV 不同，因此 TiFlash 的磁盘占用不一定与 TiKV 完全相同。

容量规划时，不能只计算逻辑数据量。

例如采用：

```text
TiKV：3 个副本
TiFlash：1 个副本
```

如果逻辑数据量为 10 TB，实际需要规划的总存储空间还包括：

- TiKV 的多副本空间；
- TiFlash 的列式副本空间；
- Region Snapshot 临时空间；
- Raft 和系统日志；
- Compaction 和 Merge 预留空间；
- 查询临时空间；
- 节点故障恢复预留空间。

可以按以下方式理解：

```text
集群总存储成本
≈ TiKV 多副本存储
+ TiFlash 列式副本
+ 日志和临时空间
+ 扩容与故障恢复预留
```

---

## 5. 不是所有 TiKV 表都会自动同步

部署 TiFlash 后，默认不会自动复制所有 TiKV 表。

只有显式配置了 TiFlash 副本的表，才会持续同步。

例如：

```sql
ALTER TABLE object_instance
SET TIFLASH REPLICA 1;

ALTER TABLE link_instance
SET TIFLASH REPLICA 1;
```

没有配置 TiFlash 副本的表仍然只存储在 TiKV 中。

可以通过以下 SQL 查看 TiFlash 副本配置和构建情况：

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    REPLICA_COUNT,
    AVAILABLE,
    PROGRESS
FROM information_schema.tiflash_replica;
```

字段含义：

| 字段 | 含义 |
|---|---|
| `TABLE_SCHEMA` | 数据库名称 |
| `TABLE_NAME` | 表名称 |
| `REPLICA_COUNT` | 配置的 TiFlash 副本数 |
| `AVAILABLE` | TiFlash 副本当前是否可用 |
| `PROGRESS` | 副本初始构建进度 |

需要注意：

- `PROGRESS = 1` 通常表示初始副本构建完成；
- `PROGRESS < 1` 表示副本仍在构建；
- 该字段不适合用于观察毫秒级同步延迟；
- 日常复制延迟需要通过 TiDB Dashboard 和相关监控指标判断。

---

## 6. 数据持续增加时的三个主要瓶颈

### 6.1 写入吞吐瓶颈

如果 TiKV 产生数据的速度长期高于 TiFlash 的接收和 Apply 速度，复制积压会不断增加。

例如：

```text
TiKV 每秒产生数据：500 MB
TiFlash 每秒处理数据：300 MB

每秒新增积压：200 MB
```

如果持续如此，TiFlash 会越来越落后。

需要重点关注：

- TiKV 写入吞吐；
- Raft 日志复制速度；
- TiFlash Apply 速度；
- TiFlash Delta 写入速度；
- 后台 Merge 速度；
- TiFlash 节点 CPU 和磁盘负载。

### 6.2 网络带宽瓶颈

TiKV 需要将 Raft 日志或 Snapshot 发送给 TiFlash。

如果数据写入量很大，网络可能成为主要瓶颈。

建议：

```text
TiKV ↔ TiFlash
├── 使用低延迟高速网络
├── 生产环境至少使用万兆网络
├── 大型集群考虑 25 GbE 或更高
└── 避免与备份、ETL、对象存储传输争抢链路
```

如果 TiKV 和 TiFlash 之间网络不足，可能导致：

- TiFlash 同步落后；
- Snapshot 传输耗时增加；
- TiKV 网络发送压力升高；
- 查询等待一致性快照的时间增加。

### 6.3 TiFlash 后台处理能力瓶颈

TiFlash 接收到数据后，还需要进行一系列后台处理：

- Raft Log Apply；
- Delta 数据写入；
- 列式数据组织；
- MVCC 版本处理；
- 数据压缩；
- 后台 Merge；
- 废弃版本清理；
- Snapshot 接收与恢复。

如果 TiFlash 同时执行大量分析查询，查询和同步可能争抢：

- CPU；
- 内存；
- 磁盘吞吐；
- 磁盘 IOPS；
- 网络；
- 后台线程。

可能形成以下链路：

```text
复杂查询占用大量资源
        ↓
Apply 和 Merge 速度下降
        ↓
TiFlash 复制延迟增加
        ↓
查询等待同步时间增加
        ↓
整体分析延迟进一步升高
```

因此，高写入和高分析并发的场景，需要为 TiFlash 保留足够的后台处理资源。

---

## 7. 首次创建 TiFlash 副本与日常同步不同

第一次执行以下命令时：

```sql
ALTER TABLE object_instance
SET TIFLASH REPLICA 1;
```

TiFlash 不只是同步后续增量数据，还需要同步该表已经存在的全部历史数据。

过程通常包括：

```text
TiKV 历史数据
      │
      ▼
生成并传输 Region Snapshot
      │
      ▼
TiFlash 接收并构建列式副本
      │
      ▼
追赶构建期间产生的增量数据
      │
      ▼
副本可用
```

如果表非常大，首次构建可能持续较长时间，并对以下资源产生影响：

- TiKV 磁盘读取；
- TiKV 网络发送；
- TiFlash 磁盘写入；
- TiFlash CPU；
- Region 调度；
- Snapshot 处理；
- OLTP P99 延迟。

因此，生产环境应当：

1. 在业务低峰期创建大表的 TiFlash 副本；
2. 按表分批创建，不要一次性覆盖全部数据库；
3. 提前规划 TiFlash 容量；
4. 监控 TiKV 和 TiFlash 的 CPU、磁盘、网络；
5. 避免与大规模数据导入、备份和扩容同时进行。

---

## 8. 查询 TiFlash 时如何避免返回旧数据

TiFlash 的一致性并不是通过事务提交时同步写入实现，而是通过查询时的快照校验实现。

因此：

- TiKV 事务成功后，不代表 TiFlash 物理副本已立即落盘；
- 但 TiFlash 查询不会随意返回不满足事务快照的数据；
- TiFlash 会等待数据追赶到所需位置；
- 如果同步落后严重，查询延迟会显著增加。

这意味着 TiFlash 更适合：

- 需要当前业务状态的一致性分析；
- 订单、库存、设备状态的实时统计；
- Object Instance 当前状态分析；
- Link Instance 当前关系分析；
- Action 当前状态和运行态分析。

不适合把 TiFlash 理解成一个完全独立、最终一致、可以任意读取旧副本的分析库。

---

## 9. 对 Ontology 平台的建议

### 9.1 适合保存在 TiKV 并同步到 TiFlash 的数据

以下数据通常属于当前运行态，需要事务能力，同时也需要实时分析：

```text
Ontology Runtime
├── Object Instance 当前状态
├── Object 当前属性值
├── Link Instance 当前关系
├── Action 当前状态
├── 工作流当前状态
├── 当前权限和可见性数据
└── 当前业务实体状态
```

推荐架构：

```text
业务操作
   │
   ▼
TiDB / TiKV
   │
   ├── 提供事务读写
   │
   └── Raft Learner 持续复制
            │
            ▼
         TiFlash
            │
            ▼
实时聚合、复杂 Join、运营分析
```

### 9.2 不建议永久全部保存在 TiKV + TiFlash 的数据

以下数据通常增长速度很快，而且以追加写和历史分析为主：

```text
历史数据
├── Object 属性变更历史
├── Link 关系变更历史
├── Action 执行流水
├── 审计日志
├── 用户行为事件
├── 设备遥测数据
├── 模型调用日志
├── 大模型 Token 使用记录
└── 长周期历史快照
```

如果这些数据长期全部保留在 TiKV，并配置 TiFlash 副本，会导致：

- TiKV 存储持续扩大；
- TiKV 多副本成本增加；
- TiFlash 列式副本持续扩大；
- Raft 复制流量持续增加；
- 备份恢复时间增长；
- Region 数量增长；
- Compaction 和 Merge 压力提高；
- 集群扩容成本增加。

更合理的架构是：

```text
TiDB / TiKV
    │
    ├── 保留当前数据和近期历史
    │
    └── TiCDC / Kafka
              │
              ├── ClickHouse
              │     └── 高频实时历史分析
              │
              └── Iceberg / 对象存储
                    └── 长期低成本归档
```

### 9.3 推荐的数据分层

| 数据类型 | 推荐存储 |
|---|---|
| 当前 Object Instance | TiKV + TiFlash |
| 当前 Link Instance | TiKV + TiFlash |
| 当前属性值 | TiKV + TiFlash |
| Action 当前状态 | TiKV + TiFlash |
| 近期操作历史 | TiKV，可按需同步 TiFlash |
| 长期 Action 历史 | ClickHouse 或 Iceberg |
| 审计日志 | ClickHouse + Iceberg |
| 设备遥测数据 | ClickHouse / 时序数据库 / Iceberg |
| 用户行为事件 | ClickHouse |
| 长期冷数据 | Iceberg + 对象存储 |

---

## 10. 生产环境监控建议

需要重点监控以下指标类别：

### TiKV

- CPU 使用率；
- 磁盘延迟和 IOPS；
- 网络发送带宽；
- Raft 日志处理延迟；
- Snapshot 生成与发送；
- Region 数量和调度情况；
- 写入吞吐；
- 事务 P95、P99 延迟。

### TiFlash

- CPU 和内存使用率；
- 磁盘吞吐和磁盘延迟；
- Raft Apply 速度；
- Delta 数据大小；
- Merge 速度；
- Snapshot 接收情况；
- 查询并发；
- MPP 任务数量；
- 查询等待同步时间；
- 副本可用状态。

### 网络

- TiKV 到 TiFlash 的带宽；
- 丢包率；
- 网络延迟；
- 是否存在链路拥塞；
- 是否与备份和批处理任务共享网络。

---

## 最终结论

当 TiKV 中的数据持续增加时，只要目标表配置了 TiFlash 副本，数据就会持续同步到 TiFlash。

需要同时理解以下三个概念：

1. **持续复制**

   TiKV 的新增、修改和删除会持续复制到 TiFlash。

2. **异步复制**

   TiKV 事务提交不会等待 TiFlash，因此 TiFlash 物理副本可能存在短暂延迟。

3. **一致读取**

   TiFlash 查询会检查自己是否已经同步到查询需要的快照；必要时会等待追赶，避免直接返回不符合事务快照的数据。

因此，TiFlash 的准确定位是：

> 以 Raft Learner 实现异步持续复制，并在查询阶段提供一致性快照校验的列式分析副本。

对于 Ontology 平台：

- 当前运行态数据适合使用 TiKV + TiFlash；
- 持续高速增长的历史数据更适合下沉到 ClickHouse 或 Iceberg；
- 不建议把所有历史流水永久保存在 TiKV 并全部复制到 TiFlash。

