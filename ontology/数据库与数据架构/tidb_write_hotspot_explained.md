# TiDB 热点写入详解

## 1. 什么是热点写入

**热点写入（Write Hotspot）**是指大量 `INSERT`、`UPDATE` 或 `DELETE` 请求没有均匀分散到多个 TiKV 节点，而是集中写入少数几个、甚至一个 Region。

可以把 TiDB 集群想象成有 10 个收费窗口：

- 正常写入：车辆均匀进入 10 个窗口；
- 热点写入：所有车辆都排在同一个窗口；
- 结果：一个窗口严重拥堵，其他窗口却比较空闲。

因此，热点写入的核心问题不是“总写入量太大”，而是：

> 大量写请求集中在相邻 Key、少量 Region 或同一个业务对象上，使分布式集群无法充分发挥并行能力。

---

## 2. TiDB 为什么会出现热点写入

TiDB 的数据实际存储在 TiKV 中。TiKV 会按照 Key 范围把数据切分成多个 **Region**。

每个 Region 通常包含多个副本，其中：

- Region Leader 负责处理主要读写请求；
- Leader 再通过 Raft 将数据复制到其他副本；
- 不同 Region 可以分布在不同 TiKV 节点上并行处理请求。

假设数据分布如下：

```text
TiKV-1：Region A，保存 ID 1～100万
TiKV-2：Region B，保存 ID 100万～200万
TiKV-3：Region C，保存 ID 200万～300万
```

如果当前所有新增数据的 ID 都在 300 万以后，新数据就会持续写入 Key 空间末端的 Region。

即使末端 Region 因为容量增长发生分裂，新的写入仍然会继续进入最新的末端 Region。因此热点可能沿着 Key 空间不断向右移动。

---

## 3. 最典型的热点：自增主键

例如订单表使用自增主键：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id BIGINT,
    amount DECIMAL(18,2)
);
```

新增订单的 ID 是连续递增的：

```text
100001
100002
100003
100004
...
```

这些主键在 TiKV 的 Key 空间中也是连续的，因此新订单会不断写向表的末端：

```text
旧 Region        旧 Region        当前末端 Region
1～100万    |  100万～200万  |  200万以后
                                  ↑
                           所有新 INSERT
```

如果末端 Region 的 Leader 位于 `TiKV-3`，可能出现：

```text
TiKV-1：CPU 20%
TiKV-2：CPU 18%
TiKV-3：CPU 95%
```

虽然整个集群还有大量空闲资源，但写入吞吐已经受到 `TiKV-3` 和热点 Region Leader 的限制。

这就是典型的**表数据写热点**。

---

## 4. 没有自增主键也可能产生热点

### 4.1 隐式 `_tidb_rowid` 热点

例如：

```sql
CREATE TABLE event_log (
    event_name VARCHAR(100),
    create_time DATETIME
);
```

如果表没有合适的聚簇主键，TiDB 可能使用隐式的 `_tidb_rowid` 标识每一行。

当 `_tidb_rowid` 按递增方式生成时，大量新增数据也可能集中写入 Key 空间的末端 Region。

### 4.2 单调递增索引热点

即使主键已经使用随机方式分散：

```sql
CREATE TABLE event_log (
    id BIGINT PRIMARY KEY AUTO_RANDOM,
    create_time DATETIME,
    INDEX idx_create_time(create_time)
);
```

如果 `create_time` 持续递增：

```text
2026-08-06 17:40:01
2026-08-06 17:40:02
2026-08-06 17:40:03
```

主表数据可能已经被打散，但 `idx_create_time` 的索引条目仍然会不断追加到索引 Key 空间末端。

这会形成**二级索引写热点**。

因此：

> `AUTO_RANDOM` 主要解决主键或表记录的顺序写热点，不一定能解决单调递增二级索引造成的热点。

### 4.3 大量请求更新同一行

例如秒杀库存：

```sql
UPDATE product_inventory
SET stock = stock - 1
WHERE product_id = 10001;
```

所有请求都在更新同一个商品库存，因此请求会集中到：

- 同一行；
- 同一个 Key；
- 同一个 Region；
- 同一个 Region Leader。

这种情况除了 Region 热点，还可能出现严重的事务锁冲突。

类似场景包括：

- 大量请求更新同一个账户余额；
- 所有任务更新同一个全局计数器；
- 大量订单更新同一个汇总记录；
- 所有日志都写入相同租户的连续 Key 范围；
- 高频写入单一状态值并维护相应索引。

---

## 5. 热点写入的常见表现

TiDB 出现热点写入后，通常会有以下现象：

- `INSERT`、`UPDATE` 或 `DELETE` 延迟升高；
- 写入 QPS 达到某个值后难以继续提高；
- 少数 TiKV 节点 CPU 很高，其他 TiKV 节点比较空闲；
- 个别 TiKV 节点磁盘写入和网络流量明显更高；
- Raftstore CPU 集中在少数节点；
- 扩容 TiKV 后，写入性能改善不明显；
- 批量写入时延迟突然升高；
- 同一张表或同一个索引持续出现在 Hot Region 中；
- 可能出现事务冲突、锁等待或请求重试。

### 为什么扩容后不一定有效

假设原来有 3 个 TiKV 节点，后来增加到 6 个 TiKV 节点。

如果所有写入仍然集中在一个 Region Leader 上，那么单个热点 Region 的处理能力仍然是系统瓶颈。

增加节点可以增加集群总容量，但不一定能突破一个热点 Region 的吞吐上限。

---

## 6. 写入量大和热点写入的区别

### 写入量大但分布均匀

```text
100 万次写入
    ↓
均匀分散到 100 个 Region
    ↓
多个 TiKV 节点并行处理
```

这种情况通常可以通过增加 TiKV 节点进行水平扩展。

### 热点写入

```text
10 万次写入
    ↓
集中到 1～2 个 Region
    ↓
受少数 Region Leader 限制
```

即使集群整体资源还有剩余，热点 Region 仍然可能成为系统瓶颈。

所以：

> 写入量大关注的是集群总容量，热点写入关注的是数据和请求是否均匀分布。

---

## 7. 如何判断有没有写热点

### 7.1 使用 TiDB Dashboard

进入：

```text
TiDB Dashboard
  → Key Visualizer
  → 查看 Write 维度
```

重点观察：

- 是否存在长期明显的高亮区域；
- 是否出现持续移动的亮色斜线；
- 是否有阶梯状热点轨迹；
- 热点对应的是表数据还是索引；
- 热点是否一直位于 Key 空间末端。

连续向右移动的亮线，通常是顺序递增主键或递增索引产生的末端写热点。

### 7.2 使用 Grafana

重点查看以下监控：

```text
TiKV-Trouble-Shooting
  → Hot Write
  → Raftstore CPU
```

还可以比较：

- 各 TiKV 实例 CPU；
- 磁盘写入吞吐；
- 网络流量；
- Raft 消息处理压力；
- 各 Store 的写入请求量。

如果某个 TiKV 的 Raftstore CPU 长期远高于其他 TiKV，需要重点检查该节点上的热点 Region。

### 7.3 查询热点 Region

在支持该系统表的 TiDB Self-Managed 环境中，可以查询：

```sql
SELECT
    DB_NAME,
    TABLE_NAME,
    INDEX_NAME,
    REGION_ID,
    TYPE,
    MAX_HOT_DEGREE,
    FLOW_BYTES
FROM INFORMATION_SCHEMA.TIDB_HOT_REGIONS
WHERE TYPE = 'write'
ORDER BY FLOW_BYTES DESC;
```

重点判断：

- 哪个数据库和表出现热点；
- 热点是表数据还是某个索引；
- 是否长期集中在少数 Region；
- 热点流量是否明显高于其他 Region；
- 热点是否一直集中在相同业务 Key 范围。

---

## 8. 常见解决方法

### 8.1 自增聚簇主键：考虑 `AUTO_RANDOM`

对于新表，可以考虑：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_RANDOM,
    customer_id BIGINT,
    amount DECIMAL(18,2)
);
```

`AUTO_RANDOM` 会在主键高位加入随机信息，使新行分散到不同 Key 范围，从而减少连续主键带来的末端写热点。

使用前需要确认：

- 业务不依赖连续递增 ID；
- 不使用 ID 大小代表数据创建顺序；
- 应用能够接受 ID 不连续；
- 对已有大表不要未经评估直接修改主键设计。

### 8.2 隐式 RowID 热点：使用 `SHARD_ROW_ID_BITS`

对于依赖隐式 `_tidb_rowid` 的表，可以考虑：

```sql
CREATE TABLE event_log (
    event_name VARCHAR(100),
    create_time DATETIME
) SHARD_ROW_ID_BITS = 4;
```

`SHARD_ROW_ID_BITS = 4` 表示将 RowID 打散到最多 16 个逻辑分片。

注意：

- 主要影响后续新增数据；
- 不会自动重写全部历史数据；
- 不适用于所有聚簇主键场景；
- 分片位数不能盲目设置得过大，否则可能产生过多 Region 和调度开销。

### 8.3 提前拆分并打散 Region

对于即将进行大批量并发写入的新表，可以提前拆分 Region，使写入从一开始就分布到多个 Region。

例如：

```sql
SPLIT TABLE orders
BETWEEN (0) AND (1000000000)
REGIONS 16;
```

还可以根据表结构、主键范围和业务数据分布设计更精确的 Region 拆分策略。

需要注意，预拆分 Region 只能为均匀写入创造条件。如果 Key 本身仍然严格顺序递增，长期热点仍可能继续出现在末端 Region。

### 8.4 同一行更新热点：业务分片

假设所有请求都更新一个全局计数器：

```sql
UPDATE global_counter
SET value = value + 1
WHERE counter_name = 'order_count';
```

可以改成多个分桶计数器：

```text
order_count_00
order_count_01
order_count_02
...
order_count_63
```

写入时根据业务 ID、随机数或哈希选择一个桶，查询总数时再聚合所有桶。

类似方法包括：

- 库存分桶；
- 账户子余额；
- 分片计数器；
- 按租户、业务 ID 或哈希前缀拆分；
- 将同步更新改为消息队列加异步汇总。

### 8.5 重新评估递增索引

如果热点来自时间字段或递增序列索引，可以考虑：

- 是否真的需要该索引；
- 是否可以调整联合索引列顺序；
- 是否能在索引前增加分散度更高的列；
- 是否可以按租户、业务类型或分片键建立联合索引；
- 是否可以使用表分区减少单一索引 Key 范围压力；
- 是否可以把分析型查询迁移到 TiFlash；
- 是否可以使用汇总表避免持续维护高成本索引。

例如单独使用：

```sql
INDEX idx_create_time(create_time)
```

可能形成索引末端热点。

如果业务查询通常同时包含租户 ID，可以评估：

```sql
INDEX idx_tenant_time(tenant_id, create_time)
```

这样不同租户的索引 Key 可以分散到更多范围。但是否有效仍然取决于租户数量、写入分布和查询模式。

### 8.6 调整 Region Leader 分布

如果热点 Region 的 Leader 分布不均，可以通过 PD 调度和集群均衡，让不同热点 Region 的 Leader 分散到不同 TiKV 节点。

但是需要注意：

- Leader 调度可以解决多个热点 Region 集中在一个节点的问题；
- 它不能把一个单独的热点 Region 同时分给多个 Leader 处理；
- 如果根因是所有请求都访问同一个 Key 或连续 Key，仍需要修改数据模型或写入方式。

---

## 9. 不同热点问题的处理对照表

| 热点原因 | 典型表现 | 主要处理方法 |
|---|---|---|
| 自增聚簇主键 | 表数据持续写向末端 Region | 新表考虑 `AUTO_RANDOM` |
| 隐式 `_tidb_rowid` | 无聚簇主键表顺序写入 | 评估 `SHARD_ROW_ID_BITS` |
| 递增时间索引 | 主表已分散，但索引仍热点 | 调整索引设计、增加分散列或重新评估索引 |
| 单一业务对象 | 同一商品、账户或计数器并发更新 | 业务分片、分桶、异步汇总 |
| 大批量导入新表 | 初始数据集中进入少数 Region | 预拆分并打散 Region |
| 多个热点 Leader 集中 | 少数 TiKV Raftstore CPU 很高 | 调整 Region Leader 分布 |
| 低基数索引集中写入 | 相同状态值产生大量索引记录 | 重新评估索引必要性和列顺序 |

---

## 10. 一个完整示例

假设有订单表：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    create_time DATETIME NOT NULL,
    INDEX idx_status(status),
    INDEX idx_create_time(create_time)
);
```

这个设计可能同时出现三类热点：

1. `id` 自增，造成主表末端热点；
2. `create_time` 递增，造成时间索引末端热点；
3. `status` 取值很少，大量订单写入相同状态索引范围。

可评估调整为：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_RANDOM,
    tenant_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    create_time DATETIME NOT NULL,
    INDEX idx_tenant_time(tenant_id, create_time),
    INDEX idx_tenant_status(tenant_id, status)
);
```

这只是一个设计方向，不代表所有业务都应这样修改。实际需要结合以下因素验证：

- 查询是否总是包含 `tenant_id`；
- 不同租户的写入量是否均匀；
- `status` 索引选择性是否足够；
- 是否需要按时间分区；
- 是否存在跨租户全量查询；
- 执行计划是否真正使用了新的联合索引。

---

## 11. 排查热点写入的推荐顺序

1. 在 TiDB Dashboard 的 Key Visualizer 中确认是否存在写入亮线；
2. 确认热点来自表数据还是二级索引；
3. 查看热点表的主键类型和聚簇方式；
4. 检查是否存在自增主键、隐式 RowID 或递增索引；
5. 检查是否有大量事务更新同一行或同一业务对象；
6. 比较各 TiKV 的 CPU、Raftstore CPU、磁盘和网络负载；
7. 判断问题属于单个热点 Region，还是多个热点 Region Leader 分布不均；
8. 再决定使用 `AUTO_RANDOM`、`SHARD_ROW_ID_BITS`、Region 预拆分、索引调整或业务分片；
9. 修改后通过压测、Key Visualizer 和实际写入延迟验证效果。

---

## 12. 最终结论

热点写入并不等于单纯的“写入量很大”。

真正的问题是：

> 大量写请求集中到少量 Key 范围、Region Leader、索引末端或同一个业务对象，导致部分 TiKV 节点过载，而其他节点无法参与分担。

因此，解决热点写入的关键不是简单增加服务器，而是让数据 Key 和写请求能够更均匀地分布。

常见手段包括：

- 使用 `AUTO_RANDOM` 分散主键写入；
- 使用 `SHARD_ROW_ID_BITS` 分散隐式 RowID；
- 调整递增索引和低基数索引设计；
- 提前拆分和打散 Region；
- 对热点业务对象进行分桶和分片；
- 调整 Region Leader 分布；
- 使用 Dashboard、Key Visualizer 和 Hot Region 信息持续验证。

## 相关笔记

- [[tidb_ontology_sharding_and_partitioning_design|TiDB 分片设计]]
- [[tiflash_impact_on_tikv_transactions|TiFlash 对 TiKV 影响]]
- [[tidb_30_day_retention_60_day_query_design|TiDB 保留30天查60天]]
