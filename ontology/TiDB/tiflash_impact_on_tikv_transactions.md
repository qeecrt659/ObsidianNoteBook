# TiFlash 分析对 TiKV 事务性能的影响

## 结论

使用 TiFlash 做分析时，**会对 TiKV 的事务操作产生一定影响，但在合理部署和隔离的情况下，通常影响较小且可控**。

TiFlash 的分析查询主要消耗 TiFlash 节点自己的 CPU、内存、磁盘和网络资源，不会像直接在 TiKV 上执行大范围扫描那样明显挤占 TiKV 的事务资源。不过，TiKV 与 TiFlash 仍然属于同一个 TiDB 集群，两者之间存在数据复制、网络传输、TiDB SQL 层调度以及集群级资源共享，因此不能认为完全没有影响。

---

## 1. 正常情况下，为什么影响较小

典型查询链路如下：

```text
OLTP 请求
   ↓
TiDB
   ↓
TiKV：事务读写

OLAP 请求
   ↓
TiDB
   ↓
TiFlash：扫描、Join、聚合、MPP
```

只要执行计划真正选择了 TiFlash，大规模扫描、聚合和 Join 就主要在 TiFlash 节点完成，不会大量占用 TiKV 的扫描线程、Block Cache 和磁盘读取能力。

推荐的物理部署方式：

```text
服务器组 A：TiDB-OLTP
服务器组 B：TiKV
服务器组 C：TiFlash
服务器组 D：TiDB-OLAP
```

在这种部署下，日常 TiFlash 查询对 TiKV 事务的影响通常较小。

---

## 2. TiFlash 可能影响 TiKV 的主要场景

### 2.1 TiKV 向 TiFlash 复制数据会消耗资源

应用写入 TiKV 后，数据需要继续复制到 TiFlash：

```text
应用写入
   ↓
TiKV Leader
   ├── 复制给 TiKV Follower
   └── 复制给 TiFlash Learner
```

TiFlash 使用 Raft Learner 副本。Learner 不参与 TiKV 事务提交的多数派投票，因此：

- TiFlash 宕机不会直接阻塞 TiKV 事务提交；
- TiFlash 复制延迟通常不会要求事务等待；
- TiFlash 副本落后时，TiKV 事务仍然可以继续执行。

但 TiKV Leader 仍然需要：

- 向 TiFlash 发送 Raft 日志；
- 消耗一定 CPU；
- 占用网络带宽；
- 在 TiFlash 严重落后时生成和发送 Snapshot。

如果事务写入量不高，复制开销通常不明显；如果持续进行高吞吐写入、批量导入或大量 UPDATE，复制流量可能变得明显。

建议重点监控：

- TiKV 网络发送带宽；
- TiKV CPU 和磁盘负载；
- TiFlash replication lag；
- Raft Wait Index Duration；
- TiFlash apply log 速度；
- TiKV 与 TiFlash 之间的网络延迟。

---

### 2.2 第一次创建 TiFlash 副本时影响可能较明显

例如：

```sql
ALTER TABLE object_instance
SET TIFLASH REPLICA 2;
```

第一次为大表创建 TiFlash 副本时，通常需要：

1. TiKV 扫描对应表的数据；
2. 生成 Region Snapshot；
3. 通过网络发送到 TiFlash；
4. TiFlash 接收并写入列式存储。

这一过程可能造成：

- TiKV 磁盘读取增加；
- 网络带宽占用增加；
- TiKV CPU 使用率上升；
- Region 调度增加；
- OLTP P99 延迟上升。

对于几十 TB 甚至更大的表，首次创建副本可能明显影响事务性能。因此应当：

- 在业务低峰期创建；
- 分批为表开启 TiFlash 副本；
- 不要一次性对整个数据库的大量表创建副本；
- 控制 Snapshot 和调度速度；
- 持续观察 TiKV P95/P99 延迟。

---

### 2.3 分析 SQL 可能错误地落到 TiKV

TiDB 优化器可以在 TiKV 和 TiFlash 之间选择执行引擎。如果出现以下情况：

- 统计信息不准确；
- TiFlash 副本尚未就绪；
- 某些算子或函数无法下推；
- 优化器判断 TiKV 成本更低；
- 查询 Hint 或会话配置不正确；

分析 SQL 就可能部分或全部在 TiKV 上执行。

这时，大范围扫描、排序、聚合或 Join 会直接消耗 TiKV 的：

- Coprocessor 线程；
- CPU；
- 磁盘 I/O；
- Block Cache；
- 网络资源。

这种情况可能明显影响事务操作。

建议使用：

```sql
EXPLAIN ANALYZE
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id;
```

确认执行计划中出现：

```text
cop[tiflash]
mpp[tiflash]
```

而不是：

```text
cop[tikv]
```

对于专用分析会话，可以限制读取引擎：

```sql
SET SESSION tidb_isolation_read_engines = 'tiflash,tidb';
```

也可以对特定 SQL 使用 Hint：

```sql
SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */
       customer_id,
       SUM(amount)
FROM orders
GROUP BY customer_id;
```

这样可以降低重型分析查询意外落到 TiKV 的风险。

---

### 2.4 TiDB SQL 层仍然可能形成资源竞争

即使扫描和聚合都下推到 TiFlash，查询仍然需要经过 TiDB Server：

```text
客户端
  ↓
TiDB Server
  ├── SQL 解析
  ├── 优化执行计划
  ├── 调度 MPP
  ├── 接收中间或最终结果
  └── 返回结果
       ↓
TiFlash
```

复杂分析查询可能消耗 TiDB Server 的：

- CPU；
- 内存；
- 网络连接；
- SQL 优化器资源；
- 最终排序和聚合资源；
- 大结果集传输能力。

如果 OLTP 和 OLAP 共用同一组 TiDB Server，那么即使 TiKV 本身没有被大量扫描，TiDB Server 也可能成为瓶颈，从而间接增加事务请求延迟。

推荐将入口分离：

```text
OLTP 应用
   ↓
TiDB-OLTP 节点
   ↓
TiKV

BI / 分析应用
   ↓
TiDB-OLAP 节点
   ↓
TiFlash
```

两组 TiDB 节点仍然可以属于同一个 TiDB 集群，但承担不同的工作负载。

---

### 2.5 TiKV 和 TiFlash 混部会放大影响

不建议在同一台生产服务器上同时部署 TiKV 和 TiFlash：

```text
同一台物理机
├── TiKV
└── TiFlash
```

两者可能争抢：

- CPU；
- 内存；
- NVMe IOPS；
- 文件系统缓存；
- 网络带宽。

TiFlash 的大扫描、排序和聚合更偏向吞吐型负载，TiKV 的事务处理更关注低延迟。如果混部，分析查询很容易拉高 TiKV 的 P99 延迟。

生产环境建议：

```text
TiKV：独立服务器、独立 NVMe、优先保障低延迟
TiFlash：独立服务器、大内存、多核 CPU、独立 NVMe
```

---

## 3. 不同场景下的影响程度

| 场景 | 对 TiKV 事务的影响 |
|---|---|
| TiKV 与 TiFlash 独立部署，普通分析查询 | 很小 |
| 查询全部下推 TiFlash，写入量不大 | 很小 |
| TiKV 持续大量写入并同步 TiFlash | 小到中等 |
| TiKV 和 TiFlash 部署在同一台服务器 | 中到明显 |
| 第一次为几十 TB 大表创建 TiFlash 副本 | 明显 |
| TiFlash 严重落后，需要传输大量 Snapshot | 中到明显 |
| 分析 SQL 错误地落到 TiKV | 非常明显 |
| OLTP 与 OLAP 共用已饱和的 TiDB Server | 明显 |
| TiKV 与 TiFlash 之间网络带宽不足 | 明显 |

---

## 4. 生产环境最佳实践

### 4.1 TiKV 和 TiFlash 分开部署

生产环境中应优先保证物理资源隔离：

```text
TiKV 节点
- 独立服务器
- 独立 NVMe
- 低延迟网络
- 优先保障事务性能

TiFlash 节点
- 独立服务器
- 大内存
- 多核 CPU
- 独立 NVMe
- 优先保障扫描和聚合吞吐
```

---

### 4.2 分离 OLTP 和 OLAP 的 TiDB 入口

可以设置不同的访问入口：

```text
tidb-oltp.example.com
    ↓
事务应用
    ↓
主要访问 TiKV

tidb-olap.example.com
    ↓
BI 和分析系统
    ↓
主要访问 TiFlash
```

分析账户可默认限制存储引擎：

```sql
SET SESSION tidb_isolation_read_engines = 'tiflash,tidb';
```

---

### 4.3 使用资源组隔离 OLTP 和 OLAP

可以为事务和分析创建不同 Resource Group：

```sql
CREATE RESOURCE GROUP oltp_group
RU_PER_SEC = 10000
PRIORITY = HIGH;

CREATE RESOURCE GROUP olap_group
RU_PER_SEC = 3000
PRIORITY = LOW;
```

基本原则：

```text
OLTP：高优先级，保证最低资源
OLAP：低优先级，限制最大资源消耗
```

具体 RU 数值需要结合真实业务压测确定，不能直接照搬示例。

---

### 4.4 只为必要的大表创建 TiFlash 副本

不建议无差别给所有表建立 TiFlash 副本。

适合建立 TiFlash 副本的表：

- Object Instance 大表；
- Link Instance 大表；
- 订单、交易、设备、生产等事实表；
- 需要频繁聚合和 Join 的宽表；
- 实时运营分析表。

通常不必建立 TiFlash 副本的表：

- 小型用户表；
- 权限配置表；
- 元模型配置表；
- 小型字典表；
- 数据量很小且主要按主键访问的表。

---

### 4.5 在低峰期执行副本创建和重建

以下操作应安排在低峰期：

- 第一次创建 TiFlash 副本；
- 新增 TiFlash 节点；
- TiFlash 故障后的副本重建；
- 大规模历史数据导入；
- 调整 TiFlash 副本数量；
- 对大量表同时开启 TiFlash。

---

### 4.6 对分析 SQL 做持续检查

建议对核心分析 SQL 建立基线：

- 定期检查 `EXPLAIN ANALYZE`；
- 确认大表扫描使用 TiFlash；
- 关注无法下推的算子；
- 保持统计信息准确；
- 限制返回超大结果集；
- 避免无条件全表扫描；
- 对重复报表考虑预计算或结果缓存。

---

## 5. 对 Ontology Runtime 的建议

可以把数据分为两类。

### 5.1 当前运行态数据

包括：

- Object Instance；
- Link Instance；
- 当前属性值；
- Action 当前状态；
- 权限和业务状态；
- 当前设备、订单和流程状态。

建议架构：

```text
TiDB + TiKV：事务写入和当前状态维护
TiFlash：对当前业务事实进行强一致分析
```

这类数据通常需要：

- 频繁更新；
- 事务一致性；
- 提交后立即查询；
- 复杂关联分析；
- 当前状态统计。

因此 TiKV + TiFlash 比直接使用独立 ClickHouse 更自然。

### 5.2 历史事件和分析数据

包括：

- 属性变更历史；
- Action 执行历史；
- 审计日志；
- 用户行为事件；
- 模型调用日志；
- 长周期统计明细；
- 超大规模事件流。

建议根据规模下沉到：

```text
ClickHouse：高并发实时分析、日志和事件查询
Iceberg：低成本长期存储、离线计算和数据湖分析
```

推荐的分层架构：

```text
                   ┌── TiFlash：当前状态实时分析
应用 → TiDB/TiKV ──┤
                   └── TiCDC / Kafka → ClickHouse / Iceberg
                                      历史事件与长期分析
```

---

## 6. 最终判断

正常情况下，TiFlash 分析不会明显拖慢 TiKV 事务。真正需要重点防范的是：

1. 首次创建或重建大规模 TiFlash 副本；
2. TiKV 持续高吞吐写入导致复制压力增加；
3. TiKV 和 TiFlash 混部造成资源竞争；
4. OLTP 和 OLAP 共用已经饱和的 TiDB Server；
5. 分析 SQL 未正确下推，意外在 TiKV 上执行；
6. TiKV 与 TiFlash 之间网络带宽不足；
7. 大量历史事件长期同时保存在 TiKV 和 TiFlash。

因此，合理的生产方案应当是：

```text
TiKV 与 TiFlash 物理隔离
+ OLTP 与 OLAP TiDB 入口隔离
+ Resource Group 限流和优先级控制
+ 只为必要的大表创建 TiFlash 副本
+ 持续检查分析 SQL 的执行计划
+ 将超大历史事件下沉到 ClickHouse 或 Iceberg
```

在这样的架构下，可以同时获得 TiKV 的事务能力和 TiFlash 的实时分析能力，并将对事务性能的影响控制在可接受范围内。

---

## 参考资料

- [TiFlash Overview](https://docs.pingcap.com/tidb/stable/tiflash-overview/)
- [Create TiFlash Replicas](https://docs.pingcap.com/tidb/stable/create-tiflash-replicas/)
- [Use TiDB to Read TiFlash](https://docs.pingcap.com/tidb/stable/use-tidb-to-read-tiflash/)
- [TiFlash Performance Tuning Methods](https://docs.pingcap.com/tidb/stable/tiflash-performance-tuning-methods/)
- [TiDB Resource Control](https://docs.pingcap.com/tidb/stable/tidb-resource-control-ru-groups/)

