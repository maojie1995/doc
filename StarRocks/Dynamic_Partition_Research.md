# StarRocks 3.1 分区自动化功能全面分析

本文涵盖 StarRocks 3.1 的两种分区自动化机制：**动态分区（Dynamic Partition）** 和 **自动分区（Auto Partition）**。

---

## 一、前置知识

### 1.1 分区基础概念

**分区（Partition）** 是数据库中将大表按某种规则拆分成多个小表的管理策略。StarRocks 支持 **Range 分区**，即按分区列的值范围将数据分布到不同分区，如按日期将日志表分为每天的分区。

分区带来的好处：
- **查询裁剪**：查询条件只涉及部分分区时，跳过无关分区的扫描
- **数据生命周期管理**：按时间过期删除旧分区，避免存储膨胀
- **运维便利**：可对单个分区执行备份、修复、迁移等操作

### 1.2 静态分区的痛点

传统 Range 分区需要用户手动执行 `ALTER TABLE ADD/DROP PARTITION` 维护分区：
- **运维负担大**：每天/每月都要手动创建未来分区、删除过期分区
- **遗漏风险**：忘记创建分区会导致数据写入失败
- **时效性差**：分区创建不及时，数据可能无法入库

### 1.3 动态分区解决的问题

**动态分区（Dynamic Partition）** 让 StarRocks 自动管理分区的生命周期——按配置的时间粒度和偏移量，周期性地预创建未来分区、删除过期分区，消除人工维护负担。

### 1.4 自动分区解决的问题

**自动分区（Auto Partition）** 是 v3.1 引入的另一种分区自动化机制。与动态分区不同，它不需要预定义时间范围，而是在**数据写入时**由 BE 自动检测缺失分区并通过 RPC 向 FE 请求创建。适用于分区键不可预测的场景（如城市、类别、不规则时间）。

---

## 二、功能使用详解

### 2.1 建表时开启动态分区

```sql
CREATE TABLE site_access(
    event_day DATE,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY RANGE(event_day)(
    PARTITION p20200321 VALUES LESS THAN ("2020-03-22"),
    PARTITION p20200322 VALUES LESS THAN ("2020-03-23"),
    PARTITION p20200323 VALUES LESS THAN ("2020-03-24"),
    PARTITION p20200324 VALUES LESS THAN ("2020-03-25")
)
DISTRIBUTED BY HASH(event_day, site_id)
PROPERTIES(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-3",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "32",
    "dynamic_partition.history_partition_num" = "0"
);
```

### 2.2 属性参数详解

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `dynamic_partition.enable` | 否 | `true` | 是否开启动态分区 |
| `dynamic_partition.time_unit` | **是** | - | 时间粒度：`HOUR`/`DAY`/`WEEK`/`MONTH`/`YEAR`。决定分区名后缀格式 |
| `dynamic_partition.time_zone` | 否 | 系统时区 | 分区边界计算的时区 |
| `dynamic_partition.start` | 否 | `Integer.MIN_VALUE`(-2147483648) | 保留的历史分区偏移（**负整数**）。在此偏移之前的分区被自动删除。默认值表示永不删除历史分区 |
| `dynamic_partition.end` | **是** | - | 预创建的未来分区偏移（**正整数**） |
| `dynamic_partition.prefix` | 否 | `p` | 分区名前缀 |
| `dynamic_partition.buckets` | 否 | `0`（用表默认值） | 每个动态分区的分桶数 |
| `dynamic_partition.replication_num` | 否 | `-1`（用表默认值） | 动态分区的副本数 |
| `dynamic_partition.history_partition_num` | 否 | `0` | 预创建历史分区的个数 |
| `dynamic_partition.start_day_of_week` | 否 | `1`(周一) | WEEK 粒度时，指定每周起始日(1-7) |
| `dynamic_partition.start_day_of_month` | 否 | `1`(1号) | MONTH 粒度时，指定每月起始日(1-28) |

**各粒度的分区名后缀格式**：

| time_unit | 分区列类型要求 | 后缀格式 | 示例 |
|-----------|--------------|----------|------|
| HOUR | 仅 DATETIME | yyyyMMddHH | `p2020032101` |
| DAY | DATE/DATETIME/INT | yyyyMMdd | `p20200321` |
| WEEK | DATE/DATETIME/INT | yyyy_ww | `p2020_13` |
| MONTH | DATE/DATETIME/INT | yyyyMM | `p202003` |
| YEAR | DATE/DATETIME/INT | yyyy | `p2020` |

### 2.3 FE 全局配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `dynamic_partition_enable` | `true` | 全局开关，关闭后所有表的动态分区调度停止 |
| `dynamic_partition_check_interval_seconds` | `600`(10分钟) | 调度周期 |
| `max_dynamic_partition_num` | `500` | 单表最大动态分区数（`end + history_partition_num` 不能超过此值） |

### 2.4 修改动态分区属性

```sql
-- 暂停动态分区
ALTER TABLE site_access SET("dynamic_partition.enable"="false");

-- 重新开启
ALTER TABLE site_access SET("dynamic_partition.enable"="true");

-- 修改预创建分区数
ALTER TABLE site_access SET("dynamic_partition.end"="10");
```

### 2.5 查看动态分区状态

```sql
-- 查看分区列表
SHOW PARTITIONS FROM site_access;

-- 查看动态分区调度状态
SHOW DYNAMIC PARTITION TABLES;

-- 查看建表属性
SHOW CREATE TABLE site_access;
```

`SHOW DYNAMIC PARTITION TABLES` 输出列：
TableName | Enable | TimeUnit | Start | End | Prefix | Buckets | ReplicationNum | StartOf | LastUpdateTime | LastSchedulerTime | State | LastCreatePartitionMsg | LastDropPartitionMsg | InScheduler

### 2.6 手动触发调度（HTTP API）

```
curl "http://fe_host:8030/api/trigger?type=dynamic_partition&db=db_name&tbl=tbl_name"
```

---

## 三、原理与架构

### 3.1 整体架构

动态分区由 FE 端的 **`DynamicPartitionScheduler`** 守护线程驱动，运行在 **Leader FE** 上。核心流程：

```
Leader FE 启动
  → DynamicPartitionScheduler 守护线程启动（继承 FrontendDaemon）
  → [每 check_interval 秒循环]
      → 如果距上次扫描超过 max(300秒, check_interval)，执行 findSchedulableTables()
        → 遍历所有 DB，注册 dynamic_partition.enable=true 的表到调度集合
      → 如果 Config.dynamic_partition_enable=true，执行 scheduleDynamicPartition()
        → 遍历调度集合，对每个表调用 executeDynamicPartitionForTable()
          → 读锁阶段：计算需要 ADD/DROP 的分区
          → 写锁阶段：逐个执行 ADD/DROP 分区操作
      → 执行 scheduleTTLPartition()
        → 处理 partition_ttl / partition_ttl_number / partition_live_number 属性的表
```

### 3.2 核心类与职责

| 类 | 路径 | 职责 |
|----|------|------|
| **DynamicPartitionProperty** | `catalog/DynamicPartitionProperty.java` | 数据类，持有单表的全部动态分区配置属性 |
| **DynamicPartitionUtil** | `common/util/DynamicPartitionUtil.java` | 工具类：参数校验、分区范围计算、分区名格式化、注册/移除调度 |
| **DynamicPartitionScheduler** | `clone/DynamicPartitionScheduler.java` | 守护线程：定时调度，执行分区增删 |
| **TableProperty** | `catalog/TableProperty.java` | 表属性容器，包含 DynamicPartitionProperty |
| **LocalMetastore** | `server/LocalMetastore.java` | ALTER TABLE SET 动态分区属性的元数据修改 |
| **OlapTableFactory** | `server/OlapTableFactory.java` | CREATE TABLE 时的动态分区校验与设置 |
| **SchemaChangeHandler** | `alter/SchemaChangeHandler.java` | ALTER TABLE SET 动态分区属性的入口处理 |
| **ShowExecutor** | `qe/ShowExecutor.java` | SHOW DYNAMIC PARTITION TABLES 的执行逻辑 |
| **TriggerAction** | `http/rest/TriggerAction.java` | HTTP API 手动触发调度 |

### 3.3 分区创建逻辑（getAddPartitionClause）

核心代码位于 `DynamicPartitionScheduler.java:184-285`：

1. **计算偏移范围**：`idx = max(start, -historyPartitionNum)`，从 idx 到 end 循环
2. **计算分区边界**：对每个偏移，调用 `DynamicPartitionUtil.getPartitionRangeString()` 计算 `[lower, upper)` 日期字符串
3. **构建 PartitionKey 范围**：`Range.closedOpen(lowerBound, upperBound)`
4. **处理重叠**：
   - 如果新范围完全被现有分区包含 → 跳过
   - 如果新范围与现有分区部分重叠，且现有分区的上界在新范围内 → **自动截断**，新范围从现有分区上界开始
   - 这处理了"周分区与月分区重叠"等场景
5. **构造分区名**：`prefix + getFormattedPartitionName(tz, dateStr, timeUnit)`
6. **设置副本数**：如果指定了 replication_num 则用指定值，否则用表默认值
7. **设置分桶数**：如果 buckets > 0，构造 DistributionDesc；否则使用表默认

### 3.4 分区删除逻辑（getDropPartitionClause）

核心代码位于 `DynamicPartitionScheduler.java:310-358`：

1. 如果 `start == Integer.MIN_VALUE`（默认值），**不删除任何分区**
2. 计算保留范围 `[start_offset_date, current_date)`
3. 遍历所有分区（按上界排序），**上界 <= 保留范围下界** 的分区被标记删除
4. 通过 `DropPartitionClause(false, partitionName, false, true)` 执行删除（true = force drop）

### 3.5 TTL 分区管理

除动态分区外，Scheduler 还支持 TTL（Time To Live）分区清理：

| 属性 | 适用对象 | 行为 |
|------|----------|------|
| `partition_ttl` | MV | 按持续时间，删除上界 <= now()-ttl 的分区 |
| `partition_ttl_number` | MV | 保留最近 N 个分区，删除其余 |
| `partition_live_number` | 基础表（含表达式分区） | 保留最近 N 个分区 |

### 3.6 建表时的立即执行

建表开启动态分区后，除了注册到调度集合，`OlapTable.onCreate()` 还会**立即创建一个后台线程**执行 `executeDynamicPartitionForTable()`，确保分区在建表后立刻可用，无需等待调度周期。

---

## 四、关键代码路径

### 4.1 参数校验链路

```
CREATE TABLE:
  OlapTableFactory.create()
    → DynamicPartitionUtil.checkIfAutomaticPartitionAllowed()  // 阻止动态+自动分区冲突
    → DynamicPartitionUtil.checkInputDynamicPartitionProperties()  // 校验必填参数、类型约束
    → DynamicPartitionUtil.checkAndSetDynamicPartitionProperty()  // 分析+设置属性
      → DynamicPartitionUtil.analyzeDynamicPartition()  // 各参数校验+分区数上限检查
        → checkTimeUnit() / checkPrefix() / checkStart() / checkEnd() / ...

ALTER TABLE SET:
  SchemaChangeHandler.process()
    → DynamicPartitionUtil.checkDynamicPartitionPropertiesExist()
    → GlobalStateMgr.modifyTableDynamicPartition()
      → LocalMetastore.modifyTableDynamicPartition()
        → DynamicPartitionUtil.analyzeDynamicPartition()
        → tableProperty.modifyTableProperties()
        → tableProperty.buildDynamicProperty()
        → DynamicPartitionUtil.registerOrRemovePartitionScheduleInfo()
        → EditLog.logDynamicPartition()  // 持久化
```

### 4.2 调度执行链路

```
DynamicPartitionScheduler.runAfterCatalogReady()  // 守护线程主循环
  → findSchedulableTables()  // 扫描所有DB，注册可调度表
  → scheduleDynamicPartition()
    → 遍历 dynamicPartitionTableInfo 集合
    → executeDynamicPartitionForTable(dbId, tableId)
      → db.readLock()  // 读锁：分析阶段
        → 确认表存在且动态分区启用
        → 确认表状态为 NORMAL（否则跳过 ADD）
        → 获取分区列与格式
        → getAddPartitionClause()  // 计算需要创建的分区
        → getDropPartitionClause()  // 计算需要删除的分区
      → db.readUnlock()
      → db.writeLock()  // 写锁：逐个执行DROP
        → GlobalStateMgr.dropPartition()
      → db.writeUnlock()
      → db.writeLock()  // 写锁：逐个执行ADD
        → GlobalStateMgr.addPartitions()
      → db.writeUnlock()
```

### 4.3 分区范围计算

`DynamicPartitionUtil` 中各粒度的分区范围计算函数：

| 函数 | 粒度 | 核心逻辑 |
|------|------|----------|
| `getPartitionRangeOfHour()` | HOUR | `current.plusHours(offset)` → 去掉分秒 |
| `getPartitionRangeOfDay()` | DAY | `current.plusDays(offset)` → 去掉时分秒 |
| `getPartitionRangeOfWeek()` | WEEK | `current.plusWeeks(offset)` → `with(previousOrSame(startOfWeek))` |
| `getPartitionRangeOfMonth()` | MONTH | `current.plusMonths(realOffset)` → `withDayOfMonth(startOfMonth.day)` |
| `getPartitionRangeOfYear()` | YEAR | `current.plusYears(realOffset)` → `withDayOfYear(startDayOfYear)` |

WEEK 中的 **weekOfYear 边界处理**（`DynamicPartitionUtil.java:536-538`）：
```java
if (weekOfYear <= 1 && calendar.get(Calendar.MONTH) >= 11) {
    // JDK认为 2019-12-30 是2020年第1周，调整为2019年第53周
    weekOfYear += 52;
}
```

MONTH 中的 **当前日 < startOfMonth.day 处理**（`DynamicPartitionUtil.java:625-629`）：
```java
if (currentDay < startOf.day) {
    realOffset -= 1;  // 如今天是5月20日，startOf是25日，offset=0应返回4月25日
}
```

---

## 五、注意点与常见问题

### 5.1 使用限制

1. **仅支持单列 Range 分区**：多列 Range 分区、List 分区不支持动态分区
2. **HOUR 粒度 + DATE 列 = 不允许**：分区列必须是 DATETIME 类型
3. **动态分区与自动分区互斥**：`checkIfAutomaticPartitionAllowed()` 阻止两者共存
4. **手动 ADD/DROP 分区被阻止**：在动态分区启用时，不能手动增删分区，必须先 `SET("dynamic_partition.enable"="false")`
5. **start_day_of_month 仅支持 1-28**：避免闰年/大小月问题
6. **分区数上限**：`end + history_partition_num` 不能超过 `max_dynamic_partition_num`(默认500)

### 5.2 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| 新分区未创建 | enable=false 或 FE 调度卡死 | 检查 SHOW CREATE TABLE；检查 FE 日志；必要时重启 FE |
| 写入报 "partition not found" | end 值太小或为0 | 增加 end（如设为3，预创建3天分区） |
| 旧分区不删除 | start 设为 MIN_VALUE 或过大 | 调整 start 到期望保留天数 |
| 数据写入错误分区 | 时区不匹配 | 显式设置 `dynamic_partition.time_zone` |
| 手动 DROP 的分区被重建 | 调度器下次周期重建 | 调整 start/end，而非手动操作 |
| 分区过多导致元数据膨胀 | HOUR 粒度+大保留窗口 | 减少保留窗口，切换粒度，或归档旧数据 |
| ALTER 分区列时失败 | 动态分区与 schema change 冲突 | 先禁用动态分区，改完再启用 |
| Colocate 表修改 buckets 被拒 | Colocate 组要求各表分桶数一致 | 不能为 Colocate 表单独设 dynamic_partition.buckets |
| 建表后分区短暂缺失 | 调度器还没跑完第一轮 | 建表有立即执行的线程，通常不会出现；增加 end 缓冲 |

### 5.3 最佳实践

1. **务必显式设置 `dynamic_partition.time_zone`**：这是最常见的分区路由错误源头
2. **end >= 1**：确保"当前时间附近"的数据有目标分区
3. **分桶数按 1GB/桶 估算**：避免小分区过度分桶
4. **考虑冷热分层**：用 `hot_partition_num` + `history_replication_num` 降低冷数据副本数（3.1 版本文档中未提及，但代码中存在相关字段）
5. **reserved_history_partitions 作为安全网**：防止 start 偏移设置过于激进导致误删

### 5.4 调度器设计要点

1. **单线程串行**：所有动态分区表逐个处理，多表场景可能有调度延迟
2. **Leader FE 独占**：Follower FE 不调度，通过 EditLog 接收变更；Leader 切换时有短暂中断
3. **发现间隔最小5分钟**：`findSchedulableTables()` 的间隔为 `max(300秒, check_interval)`
4. **表状态必须 NORMAL**：Schema Change 等状态下只执行 DROP，不执行 ADD
5. **重叠自动截断**：新分区范围与现有分区重叠时，自动截断新范围避免冲突

---

## 六、自动分区（Auto Partition）详解

### 6.1 建表语法

**Range 自动分区**（按时间函数，不预建分区）：

```sql
CREATE TABLE t1 (
    event_day DATE NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY date_trunc('day', event_day)
DISTRIBUTED BY HASH(event_day, site_id) BUCKETS 32;
```

**Range 自动分区**（批量预建 + 自动分区）：

```sql
CREATE TABLE t2 (
    event_day DATE NOT NULL,
    site_id INT DEFAULT '10'
)
DUPLICATE KEY(event_day, site_id)
PARTITION BY date_trunc('month', event_day)(
    START ("2023-05-01") END ("2023-10-01") EVERY (INTERVAL 1 month)
)
DISTRIBUTED BY HASH(event_day) BUCKETS 32
PROPERTIES("partition_live_number" = "3");
```

**List 自动分区**（按枚举列）：

```sql
CREATE TABLE t3 (
    city VARCHAR NOT NULL,
    site_id INT DEFAULT '10',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(city, site_id)
PARTITION BY city
DISTRIBUTED BY HASH(city) BUCKETS 32;
```

### 6.2 支持的分区函数

| 函数 | 粒度参数 | 说明 | 支持批量预建（EVERY） |
|------|----------|------|----------------------|
| `date_trunc('hour/day/month/year', col)` | hour/day/month/year | 时间截断到指定粒度 | 支持 |
| `time_slice(col, interval)` | INTERVAL | 时间切片 | 不支持 |

### 6.3 参数详解

#### FE 全局配置（`fe.conf`）

| 配置项 | 默认值 | mutable | 说明 |
|--------|--------|---------|------|
| `max_automatic_partition_number` | `4096` | `true` | 单表自动分区数量上限，超出写入报错 |
| `enable_display_shadow_partitions` | `false` | - | 是否在 `SHOW PARTITIONS` 中显示影子分区 |

#### 表级属性

| 属性 | 适用对象 | 说明 |
|------|----------|------|
| `partition_live_number` | 基础表 | 保留最近 N 个分区，超出自动删除旧分区 |
| `partition_ttl_number` | MV | 保留最近 N 个分区 |
| `partition_ttl` | MV/基表 | 基于时间的 TTL（如 `"7d"`、`"1m"`），过期分区自动清理 |

#### Thrift 属性（FE→BE 传递）

| 属性 | 说明 |
|------|------|
| `enable_automatic_partition` | 建表时传递给 BE，标识该表为自动分区表 |

### 6.4 影子分区

建表时，如果 `partitionInfo.isAutomaticPartition()` 为 true，`OlapTableFactory` 会自动创建一个名为 **`$shadow_automatic_partition`** 的影子分区，范围为 `[shadowKey, shadowKey)`（空范围）。其作用是确保 BE 可以识别分区结构，即使尚无真实分区存在。影子分区在 `SHOW PARTITIONS` 中默认不显示（需设置 `enable_display_shadow_partitions=true`）。

### 6.5 分区创建流程（写入时触发）

```
数据写入（INSERT/LOAD）
  → BE: OlapTableSink.send_chunk()
    → _vectorized_partition->find_tablets()  // 查找分区
    → 如果分区不存在，填充 partition_not_exist_row_values
    → _automatic_create_partition()  // 异步提交到线程池
      → 构建 TCreatePartitionRequest (txn_id, db_id, table_id, partition_values)
      → RPC 调用 FrontendService.createPartition() 到 FE
  → FE: FrontendServiceImpl.createPartition()
    → 检查分区数限制: partitions + new_count <= max_automatic_partition_number
    → AnalyzerUtils.getAddPartitionClauseFromPartitionValues()
      → Range: 提取函数粒度，格式化分区名，创建 [beginTime, beginTime+interval) 范围
      → List: 格式化分区名，创建 MultiItemListPartitionDesc
    → GlobalStateMgr.addPartition()  // 执行分区添加
    → 返回分区信息（tablet位置、节点）给 BE
  → BE: 收到分区信息后，打开新 node channel，继续写入数据
```

**Pipeline 引擎**：数据 chunk 在分区创建期间被缓冲（`_automatic_partition_chunk`），创建完成后重新发送。
**旧引擎**：阻塞等待分区创建完成，然后用新分区重新评估数据。

### 6.6 TTL 清理机制

自动分区表的旧分区清理由 `DynamicPartitionScheduler` 的 `scheduleTTLPartition()` 处理：

**partition_live_number（基表）**：
- 按分区范围上界排序
- 排除影子分区（下界 == shadowPartitionKey）
- 排除未来分区（下界 > 当前时间）
- 保留最近 N 个分区，其余生成 `DropPartitionClause`

**partition_ttl（基于时间）**：
- 计算 TTL 边界：`now - ttl_duration`
- 删除上界 <= TTL 边界的分区
- 排除影子分区

### 6.7 核心类与职责

| 类 | 路径 | 职责 |
|----|------|------|
| **ExpressionRangePartitionInfo** | `catalog/ExpressionRangePartitionInfo.java` | Range 自动分区元数据（标记为 @Deprecated，V2 替代） |
| **ExpressionRangePartitionInfoV2** | `catalog/ExpressionRangePartitionInfoV2.java` | Range 自动分区元数据 V2，`automaticPartition` 字段 |
| **ListPartitionInfo** | `catalog/ListPartitionInfo.java` | List 自动分区元数据，`automaticPartition` 字段 |
| **ExpressionPartitionDesc** | `sql/ast/ExpressionPartitionDesc.java` | AST 描述符，`PARTITION BY functionCall(...)` |
| **RangePartitionDesc** | `sql/ast/RangePartitionDesc.java` | `isAutoPartitionTable` 标志 |
| **ListPartitionDesc** | `sql/ast/ListPartitionDesc.java` | `isAutoPartitionTable` 标志 |
| **AnalyzerUtils** | `sql/analyzer/AnalyzerUtils.java` | 分区值→AddPartitionClause 转换、粒度校验 |
| **PartitionExprAnalyzer** | `sql/analyzer/PartitionExprAnalyzer.java` | 分区表达式函数分析 |
| **FrontendServiceImpl** | `service/FrontendServiceImpl.java:1908-1932` | 处理 BE 的 createPartition RPC |
| **OlapTableFactory** | `server/OlapTableFactory.java` | 建表时创建影子分区 |
| **LocalMetastore** | `server/LocalMetastore.java` | 手动 ALTER ADD PARTITION 的自动分区限制 |
| **OlapTableSink** | `be/src/exec/tablet_sink.cpp` | BE 端自动分区创建逻辑 |
| **OlapTableSinkOperator** | `be/src/exec/pipeline/olap_table_sink_operator.cpp` | Pipeline 引擎的自动分区缓冲 |
| **DynamicPartitionUtil** | `common/util/DynamicPartitionUtil.java:494` | `checkIfAutomaticPartitionAllowed()` 阻止动态+自动分区冲突 |

### 6.8 注意点与常见问题

**使用限制**：
1. **自动分区与动态分区互斥**：不能同时设置动态分区属性
2. **分区数上限**：不能超过 `max_automatic_partition_number`(默认4096)
3. **`time_slice` 不支持批量预建**：不能用 `EVERY` 语法
4. **手动 ALTER ADD PARTITION 限制**：Range 自动分区表只支持批量添加（步长为1）
5. **分区列必须有 NOT NULL 约束**：List 自动分区要求分区列非空

**常见陷阱**：

| 问题 | 原因 | 解决 |
|------|------|------|
| 写入报 "exceeded maximum limit" | 分区数超过 4096 | 调大 `max_automatic_partition_number` 或设置 `partition_live_number` |
| 分区创建延迟导致写入慢 | BE→FE RPC + FE 创建分区耗时 | Pipeline 引擎会缓冲数据；旧引擎阻塞等待 |
| 旧分区堆积不清理 | 未设置 `partition_live_number` 或 `partition_ttl` | 配置 TTL 属性 |
| 高并发写入同一新分区 | 多个 BE 同时请求创建 | FE 有锁保护，只创建一次；其他请求等待 |
| 影子分区误操作 | 误删 `$shadow_automatic_partition` | 不要手动删除影子分区 |

**最佳实践**：
1. **务必设置 `partition_live_number` 或 `partition_ttl`**：防止分区无限增长
2. **选择合适的分区粒度**：`date_trunc('day', ...)` 比 `date_trunc('hour', ...)` 分区数可控
3. **批量预建 + 自动分区结合**：对可预测的时间段用 `EVERY` 预建，其余交给自动分区
4. **List 自动分区适合枚举维度**：如城市、类别等不可预测的分区键

---

## 七、动态分区与自动分区对比

| 特性 | 动态分区 | 自动分区（v3.1+） |
|------|----------|-----------------|
| 自动化方式 | FE 定时调度创建/删除 | 数据写入时 BE→FE RPC 创建 |
| 触发时机 | 调度周期（默认10分钟） | INSERT/LOAD 时即时触发 |
| 分区类型 | Range（时间） | Range + List |
| 需预定义范围 | 是（start/end 偏移） | 否（写入时按需创建） |
| 自动清理 | 是（start 偏移删除过期） | 否（需 `partition_live_number` / `partition_ttl`） |
| 分区数上限 | 500（`max_dynamic_partition_num`） | 4096（`max_automatic_partition_number`） |
| 分区创建延迟 | 等待调度周期 | 写入时即时创建 |
| 适用场景 | 时序数据，规律增长 | 不可预测分区键（城市、类别） |
| 互斥关系 | 两者不能同时启用 | 两者不能同时启用 |

---

## 八、问答

### Q1: 这个功能的设计动机是什么？怎么识别出这个优化方向？

**答**：时序数据场景（日志、监控、IoT）下，分区是刚需（查询裁剪+生命周期管理），但手动维护分区是高频低价值的运维操作。动态分区把这个重复劳动自动化，动机来自运维痛点识别——"忘建分区导致数据无法写入"是常见事故。

### Q2: 动态分区调度的正确性如何保证？

**答**：
- **分区边界计算**基于 `ZonedDateTime` + 时区，避免时区偏移错误
- **重叠检测**：ADD 前检查所有现有分区范围，有重叠则截断或跳过
- **DROP 语义**：仅删除上界 <= 保留范围下界的分区，不会误删保留范围内的分区
- **EditLog 持久化**：所有分区变更写入 EditLog，Follower 通过日志回放保持一致
- **锁保护**：读锁分析 → 写锁执行，防止并发冲突
- **状态检查**：表非 NORMAL 时跳过 ADD，避免 Schema Change 中途冲突

### Q3: 性能方面的考量？

**答**：
- **单线程串行调度**：简单可靠但多表时有延迟。生产环境中数百张动态分区表时，10分钟周期可能不够。可调整 `dynamic_partition_check_interval_seconds`
- **分区数上限 500**：防止元数据膨胀。HOUR 粒度下 `end=168`(7天) 就消耗 168 个名额，需要权衡
- **发现扫描最小间隔 300秒**：避免频繁全库扫描
- **建表后立即执行**：避免首次调度延迟导致数据无法写入

### Q4: 可靠性方面的考量？

**答**：
- **Leader FE 单点调度**：Leader 切换时有短暂中断，新 Leader 接管后恢复
- **失败重试**：ADD/DROP 失败不退出调度，下个周期继续尝试；失败信息记录在 runtimeInfo 中供排查
- **分区删除无宽限期**：一旦分区超出 start 偏移就立即删除，没有"最近删除"缓冲。建议用 `reserved_history_partitions` 或增大 start 偏移作为安全网
- **手动操作冲突**：用户手动 DROP 的分区在下一个调度周期会被重建（如果仍在范围内）

### Q5: 测试覆盖情况？

**答**：测试文件包括：
- `DynamicPartitionTableTest.java`：建表校验（缺少必填参数、类型约束、多列分区拒绝等）
- `DynamicPartitionSchedulerTest.java`：调度逻辑（TTL属性、自动分区冲突、分区存活数等）
- `DynamicPartitionUtilTest.java`：分区范围计算（各粒度、各偏移、startOfWeek/startOfMonth）
- `ShowDynamicPartitionStmtTest.java`：SQL 解析
- `ModifyDynamicPartitionInfoTest.java`：EditLog 序列化

覆盖了参数校验、范围计算、调度逻辑等核心路径，但对并发场景和大量分区表的性能场景缺少测试。

### Q6: 自动分区的设计动机是什么？

**答**：动态分区要求预定义时间范围（start/end），适用于日志等规律增长的时序数据。但对于分区键不可预测的场景（如按城市、按类别分区），动态分区无法覆盖——你无法预知未来会出现哪些城市或类别。自动分区通过"写入时按需创建"解决了这个问题，分区只在数据实际到来时才创建，无需人工预判。

### Q7: 自动分区的正确性如何保证？

**答**：
- **FE 端锁保护**：`FrontendServiceImpl.createPartition()` 在创建分区时加锁，防止多个 BE 并发创建同一分区
- **分区数限制**：创建前检查 `partitions + new_count <= max_automatic_partition_number`，防止分区无限增长
- **粒度校验**：批量创建分区时 `checkAutoPartitionTableLimit()` 确保步长与函数粒度一致
- **影子分区**：建表时创建空范围影子分区，确保 BE 可以识别分区结构
- **Pipeline 缓冲**：分区创建期间数据被缓冲，创建完成后重新发送，避免数据丢失

### Q8: 自动分区的性能考量？

**答**：
- **RPC 开销**：每个新分区需要一次 BE→FE 的 Thrift RPC。高并发写入大量新分区时有延迟
- **线程池**：BE 使用 `ExecEnv.automatic_partition_pool()` 线程池异步提交创建请求，避免阻塞主写入流程
- **分区数上限 4096**：比动态分区(500)宽松，但 HOUR 粒度下仍可能快速增长，需配合 TTL
- **影子分区开销**：建表时多一个空分区，对查询无影响但增加元数据条目

### Q9: 自动分区 vs 动态分区，如何选择？

**答**：
- **时序数据、规律增长**（日志、监控）→ 动态分区：可预创建未来分区，写入无延迟
- **不可预测分区键**（城市、类别）→ List 自动分区：写入时按需创建
- **时间维度但不需要预建**→ Range 自动分区：比动态分区更灵活，但首次写入有 RPC 延迟
- **两者不能共存**：同一张表只能选择一种机制

### Q10: 扩展性方向？

**答**：
- **多列 Range 分区支持**：当前仅单列，扩展需要修改范围计算和重叠检测逻辑
- **并行调度**：单线程可改为多线程/分区表级并行，减少多表调度延迟
- **宽限期机制**：分区删除前设置 grace period，允许数据回补
- **与自动分区融合**：3.1 已有表达式分区，未来可能统一动态分区和自动分区为更通用的自动生命周期管理
- **冷热分层自动化**：结合 `hot_partition_num` 自动调整副本数和存储介质

---

## 九、核心代码清单

### 动态分区

| 文件 | 关键行 | 作用 |
|------|--------|------|
| `DynamicPartitionProperty.java` | 全文(227行) | 动态分区属性定义与解析 |
| `DynamicPartitionUtil.java:88-97` | checkTimeUnit | 时间粒度校验 |
| `DynamicPartitionUtil.java:209-264` | checkInputDynamicPartitionProperties | 建表/ALTER时的属性校验入口 |
| `DynamicPartitionUtil.java:315-404` | analyzeDynamicPartition | 各参数校验+分区数上限检查 |
| `DynamicPartitionUtil.java:406-414` | checkAlterAllowed | 阻止动态分区表的手动ADD/DROP |
| `DynamicPartitionUtil.java:516-543` | getFormattedPartitionName | 分区名后缀格式化（含weekOfYear边界处理） |
| `DynamicPartitionUtil.java:546-561` | getPartitionRangeString | 分区范围日期计算分发 |
| `DynamicPartitionScheduler.java:184-285` | getAddPartitionClause | 计算需创建的分区（含重叠截断） |
| `DynamicPartitionScheduler.java:310-358` | getDropPartitionClause | 计算需删除的分区 |
| `DynamicPartitionScheduler.java:360-371` | scheduleDynamicPartition | 遍历调度集合 |
| `DynamicPartitionScheduler.java:373-467` | executeDynamicPartitionForTable | 单表调度执行（读锁分析+写锁执行） |
| `DynamicPartitionScheduler.java:714-735` | runAfterCatalogReady | 守护线程主循环 |
| `DynamicPartitionScheduler.java:669-706` | findSchedulableTables | 扫描全库注册可调度表 |
| `Config.java:289` | max_dynamic_partition_num=500 | 分区数上限配置 |
| `Config.java:1188` | dynamic_partition_enable=true | 全局开关 |
| `Config.java:1194` | dynamic_partition_check_interval_seconds=600 | 调度周期 |

### 自动分区

| 文件 | 关键行 | 作用 |
|------|--------|------|
| `ExpressionRangePartitionInfoV2.java:65-66` | automaticPartition 字段 | Range 自动分区标识 |
| `ListPartitionInfo.java:73-74` | automaticPartition 字段 | List 自动分区标识 |
| `ExpressionPartitionDesc.java:97-98` | isAutoPartitionTable | AST 描述符 |
| `RangePartitionDesc.java:44` | isAutoPartitionTable | Range 分区 AST 标志 |
| `ListPartitionDesc.java:51` | isAutoPartitionTable | List 分区 AST 标志 |
| `AnalyzerUtils.java:991` | getAddPartitionClauseFromPartitionValues | 分区值→AddPartitionClause 转换 |
| `AnalyzerUtils.java:1240` | checkAutoPartitionTableLimit | 批量创建粒度校验 |
| `FrontendServiceImpl.java:1908-1932` | createPartition | 处理 BE 的 createPartition RPC |
| `OlapTableFactory.java:131` | 影子分区创建 | 建表时创建 $shadow_automatic_partition |
| `LocalMetastore.java:944,977` | 自动分区 ALTER 限制 | 手动 ADD PARTITION 限制 |
| `DynamicPartitionUtil.java:494` | checkIfAutomaticPartitionAllowed | 阻止动态+自动分区冲突 |
| `OlapTableSink.cpp:1066,1347,1505-1543` | BE 端自动分区 | 检测缺失分区 + RPC 创建 |
| `OlapTableSinkOperator.cpp:111,159` | Pipeline 自动分区 | 数据缓冲 + 分区创建 |
| `Config.java:1985` | max_automatic_partition_number=4096 | 分区数上限 |
| `Config.java:2380` | enable_display_shadow_partitions=false | 影子分区可见性 |

---

## 十、总结

StarRocks 动态分区是一个成熟的时序数据分区自动化方案，通过 FE Leader 上的守护线程周期性调度，自动预创建未来分区和删除过期分区。核心设计简洁——单线程串行、基于时间偏移计算范围、重叠自动截断。适用于日志、监控等规律增长的时序场景。

**关键设计选择**：
- 偏移量模型（start/end）而非绝对时间，灵活适应不同粒度
- 时区独立计算，避免时区陷阱
- 建表后立即执行，消除首次调度延迟
- EditLog 持久化保证多 FE 一致性

**主要局限**：
- 仅单列 Range 分区
- 单线程调度，多表场景可能有延迟
- 无分区删除宽限期
- 与自动分区互斥

**自动分区关键设计选择**：
- 写入时按需创建（BE→FE RPC），无需预判分区键
- 支持 Range + List 两种分区类型
- 影子分区确保空表也能正确路由
- Pipeline 引擎缓冲数据，避免分区创建期间数据丢失
- TTL 属性（`partition_live_number` / `partition_ttl`）实现旧分区自动清理

**自动分区主要局限**：
- 首次写入有 RPC 延迟（分区创建需等待 FE 处理）
- 分区数上限 4096，高粒度时间分区仍需注意
- 与动态分区互斥，同一表只能选择一种机制
- 无分区删除宽限期