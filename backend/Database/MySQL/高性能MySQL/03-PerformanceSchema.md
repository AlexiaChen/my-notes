
# 第 3 章　Performance Schema——读书笔记

> 💡 这一章的本质，不是教你"查几张表"，而是给你一套**关于 MySQL 内部运行态的可编程探针系统**的心智模型。书里从"我改了配置，怎么知道有没有效？"这个可靠性工程的第一性问题切入——查询变快了吗？锁还在拖慢应用吗？内存涨了没？磁盘等待变了没？Performance Schema（以下简称 P_S）就是为回答这些问题而生的内置数据库。

---

## 🎯 本章核心脉络（先建立"第一性原理"）

P_S 的设计可以用一句话贯穿：**它在 MySQL 源代码里埋了无数个"测量点"（instrument），再把测量结果写到内存表里（consumer）**。所有的配置、开销、局限性，都是这两个概念衍生出来的。

- **instrument（插桩）**​ = MySQL 代码里任何我们想采集信息的点位。例如要看元数据锁，就要打开 `wait/lock/metadata/sql/mdl` 这个插桩。
- **consumer（消费者表）**​ = 真正存储插桩结果的表。比如给查询模块加了插桩，对应的 consumer 表就会记录执行总数、未用索引次数、耗时等。

📌 **我的洞见**：很多人把 P_S 当成一个"监控工具"来用，但它的本质更接近**可配置的遥测 SDK**——插桩是"生产数据的代码"，消费者是"存储数据的 sink"。这决定了三件事：

1. **插桩越多 = CPU 开销越大**（每次插桩调用会注入两个宏）
2. **consumer 层级决定了你能看到多细的历史**
3. **线程/对象过滤决定了"对谁"观测**

这也是为什么工业界现在的共识是：**默认配置已经覆盖了 80% 的高价值场景，剩下的 20% 是你按需临时开启的**。

---

## Performance Schema 介绍

P_S 是 MySQL 内置的一个**存储引擎为 PERFORMANCE_SCHEMA 的数据库**，所有表都是内存表，不持久化。

书中给出的关键背景：

- 5.7 起 P_S **默认启用**，但大多数插桩默认关闭，只开了全局、线程、语句、事务类插桩
- 8.0 起**默认额外启用元数据锁和内存插桩**
- `_history` 表存每个线程最近 10 个事件，`_history_long` 存最近 10000 个
- SQL 文本和摘要最大长度均为 1024 字节，超出截断

📌 **洞见**：为什么默认不全开？因为每个插桩的调用都会在热路径上增加两个宏调用，**wait 类插桩的调用频率远高于 statement 类**——扫描一个百万行 InnoDB 表，引擎就要设置和释放百万次行锁。这就是为什么"全开 P_S"在工业界是反模式。

---

## 插桩元件（Instrument Elements）

插桩的命名是有层级的，类似于 `wait/io/file/innodb/innodb_data_file`、`statement/sql/select`、`memory/innodb/%`。这种命名方式支持**通配符批量配置**：

```
-- 启动参数形式（8.0 官方文档）
--performance-schema-instrument='wait/synch/cond/%=COUNTED'
--performance-schema-instrument='%=OFF'   -- 全关
```

每个插桩有三个值可选：

- `OFF/FALSE/0`：禁用
- `ON/TRUE/1`：启用并计时
- `COUNTED`：启用但只计数不计时（省一次高精度时钟读取）

> ⚠️ 长名称优先级高于短模式，无论顺序。这是配置插桩时的一个隐蔽坑。

📌 **洞见**：`TIMED=YES` 听起来很合理，但在某些硬件上**读取高精度时钟本身就是瓶颈**。工业界一个常见做法是：先用 `COUNTED` 摸底"哪些插桩事件频次最高"，再针对性地对高频事件开启 `TIMED`。

---

## 消费者表的组织

这是全章最重要的"地图"。`setup_consumers` 表里的消费者形成**严格的层级结构**：

```
global_instrumentation
└── thread_instrumentation
    ├── events_waits_current
    │   ├── events_waits_history
    │   └── events_waits_history_long
    ├── events_stages_current
    │   ├── events_stages_history
    │   └── events_stages_history_long
    ├── events_statements_current
    │   ├── events_statements_history
    │   └── events_statements_history_long
    ├── events_transactions_current
    │   ├── events_transactions_history
    │   └── events_transactions_history_long
    └── statements_digest
```

**关键规则**：上层为 `NO` 时，下层全部忽略。这意味着：

- 只开 `global_instrumentation` → 只有全局聚合表有数据
- 开了 `thread_instrumentation` → 才有按线程的明细
- 开了 `events_statements_current` → 才有当前语句事件

📌 **洞见**：这个层级不是技术债，而是**开销控制阀**。工业界的典型配置是——**保留 statements_digest 和 events_statements_history_long，关掉绝大多数 waits 类 history**。原因很简单：`events_statements_summary_by_digest` 这一张表就能告诉你"哪些 SQL 家族占用了最多总延迟"，这是 8.0 默认配置里性价比最高的一张表。

---

## 资源消耗

书里在这一节讲的是 P_S 自身的开销模型，结合官方文档和工业实践，要点如下：

- P_S 启用时，**实例数量直接影响 MySQL 内存占用**
- 许多参数**自动伸缩**（autoscale）：启动时分配最小内存，按需增长
- **但一旦分配，即使你后来禁用插桩或 truncate 表，内存也不会释放**——除非重启
- 官方 8.0 文档明确：如果初始化时无法分配内部缓冲，P_S 会**自我禁用**并把 `performance_schema` 设为 OFF，服务器在无插桩状态下运行

📌 **洞见**：这是一个反直觉的工程事实——**P_S 的内存只增不减**。所以生产环境配置 P_S 内存参数时，不能只看"当前用了多少"，而要预估峰值。Percona 的实践中会显式设置 `performance_schema_max_thread_instances`、`performance_schema_accounts_size` 等上限来约束内存上限。

> 💡 一个工业级经验值：在繁忙的 OLTP 服务器上，P_S 的开销通常在 1-5% 之间，前提是**不要全开插桩**。这个开销远低于"因为盲目调优导致的故障"的成本。

---

## 局限性

官方文档列出的硬限制：

- P_S **避免使用互斥锁**来收集/产生数据，所以**没有一致性保证**，结果有时可能不正确
- `performance_schema` 表中的事件值是**不确定且不可重复的**——你 SELECT 出来存到临时表后再 JOIN 原表，可能匹配不上
- `mysqldump` 和 `BACKUP DATABASE` **忽略**​ P_S 下的表
- P_S 表**不能用 LOCK TABLES 锁定**（setup_* 表除外）
- P_S 表**不能建索引**
- 对 P_S 表的查询结果**不进查询缓存**
- P_S 表**不参与复制**
- 嵌入式服务器 libmysqld 中不可用
- 定时器类型随平台变化
- **第三方存储引擎的插桩由其维护者负责**——意味着不是所有引擎都插桩完整

📌 **洞见**：这些局限性看似琐碎，实则揭示了一个深层事实——**P_S 是为"实时诊断"设计的，不是为"历史审计"设计的**。如果你需要长期留存 MySQL 内部指标，正确做法是**周期性地把 P_S 数据抽取到外部时序数据库**（Prometheus + mysqld_exporter 是工业界主流方案），而不是试图让 P_S 自己承担存储职责。

> ⚠️ 一个 2026 年 Oracle 工程师披露的惨痛教训：在 5.7→8.0 升级后，P_S 的 `performance_schema.data_locks` 表在某些内存压力场景下**反而成为死锁崩溃的根因**——监控本身引发了它要监控的问题。这提醒我们：P_S 不是零成本的，它的内存表在高并发下也会成为竞争者。

---

## sys Schema

5.7 起，标准 MySQL 发行版内置了 sys Schema，它**全部基于 P_S 的视图和存储过程组成，自身不存任何数据**。

sys Schema 的价值在于把 P_S 里那些 80 多张原始表**包装成人类可读的视图**，例如：

|sys 视图|回答的问题|
|---|---|
|`statement_analysis`|哪些语句家族总延迟最高|
|`statements_with_full_table_scans`|哪些查询做了全表扫描|
|`statements_with_runtimes_in_95th_percentile`|哪些查询落在慢尾部|
|`statements_with_temp_tables`|哪些查询用了临时表|
|`statements_with_errors_or_warnings`|哪些语句带错误或警告|
|`memory_by_thread_by_current_bytes`|哪个线程吃内存最多|
|`schema_table_lock_waits`|谁在等锁、被谁阻塞|

此外 sys Schema 还提供了一组**存储过程来简化 P_S 配置**：

```
CALL sys.ps_setup_enable_instrument('wait/lock/metadata/sql/mdl');
CALL sys.ps_setup_disable_consumer('events_waits_history');
CALL sys.ps_setup_show_enabled();
```

📌 **洞见**：日常操作中，__几乎永远应该通过 sys Schema 查 P_S，而不是直接查 performance_schema._ 表__。但书里也提醒了一句很重要的话：__如果 sys Schema 里找不到你要的数据，去 P_S 基表里找_*——sys 只是视图，不是数据的全部。

---

## 理解线程

这是本章最容易被跳过、但工业界排查问题时最关键的一节。

`performance_schema.threads` 表包含服务器中所有线程，但有三个 ID 经常被人混淆：

- **THREAD_ID**：MySQL 内部线程 ID
- **PROCESSLIST_ID**：SHOW PROCESSLIST 里的 ID（用来 KILL 连接）
- **THREAD_OS_ID**：操作系统线程 ID

```
SELECT NAME, THREAD_ID, PROCESSLIST_ID, THREAD_OS_ID 
FROM performance_schema.threads;
```

8.0 的 threads 表还额外包含 `RESOURCE_GROUP`、`PARENT_THREAD_ID` 等列，并且数据与 SHOW PROCESSLIST 同源。

📌 **洞见**：排查"哪个会话持有了锁"这类问题时，**THREAD_ID 是用来 JOIN P_S 其他表的键，PROCESSLIST_ID 是用来 KILL 的句柄**——搞混这两个是新手最常见的错误。另外，P_S 的线程模型揭示了 MySQL 的一个本质：**用户连接、IO 线程、Purge 线程、Master 线程等都是平等的 P_S 线程**，这给了我们一个统一的视角去观察整个服务器。

---

## 配置

这一节书里讲的是 P_S 的两阶段配置模型：**启动期配置**（写在 my.cnf 里）+ **运行期配置**（改 setup_* 表）。

### 启用或禁用 Performance Schema

```
[mysqld]
performance_schema = ON
```

⚠️ **这是启动期变量，不能在运行时从 OFF 切到 ON，必须重启**。

### 启用或禁用插桩

启动期：

```
--performance-schema-instrument='wait/lock/metadata/sql/mdl=ON'
```

运行期：

```
UPDATE performance_schema.setup_instruments 
SET ENABLED='YES', TIMED='YES' 
WHERE NAME LIKE 'statement/sql/%';
```

### 启用或禁用消费者表

启动期：

```
--performance-schema-consumer-events-statements-history-long=ON
```

运行期：

```
UPDATE performance_schema.setup_consumers 
SET ENABLED='YES' 
WHERE NAME='events_statements_history_long';
```

📌 **洞见**：这里有个工业界踩坑最多的点——__运行期改 setup__ 表立即生效，但不持久化__。下次重启，你的精心调校全部回滚到默认值。所以__任何你认为应该长期保留的配置，都必须写回 my.cnf_*。正确的工作流是：运行时临时开启插桩做诊断 → 确认有价值 → 把对应配置固化到 my.cnf。

---

### 优化特定对象的监控

`setup_objects` 表控制对哪些 schema/table 进行插桩：

```
SELECT * FROM performance_schema.setup_objects;
```

可以针对特定 schema 下的表开启插桩：

```
UPDATE performance_schema.setup_objects 
SET ENABLED='YES' 
WHERE OBJECT_SCHEMA='my_schema' AND OBJECT_TYPE='TABLE';
```

📌 **洞见**：这是 P_S 精细化控制的精华所在——**你可以只对生产环境里最关键的几张表做深度插桩，而对其他表保持轻量**。在多租户或 SaaS 场景下尤其有用：只对你最核心的业务 schema 开 `wait/io/table` 插桩，其他的保持默认。

---

### 优化线程的监控

`setup_threads` 表控制哪些类型的线程被插桩，`setup_actors` 表控制哪些用户连接被插桩：

```
-- 只对特定用户开启历史记录
UPDATE performance_schema.setup_actors 
SET ENABLED='YES', HISTORY='YES' 
WHERE USER='app_user';
```

📌 **洞见**：`setup_actors` 是生产环境的神器。你可以做到——**对应用服务账户全量插桩，对运维账号只做全局聚合，对临时调试账号完全关闭**。这样既保证了核心业务的可观测性，又把开销压到最低。

---

### 调整 Performance Schema 的内存大小

8.0 提供了大量 `performance_schema_*` 系统变量来控制内存分配：

```
SHOW VARIABLES LIKE 'perf%';
-- performance_schema_digests_size = 200
-- performance_schema_events_statements_history_long_size = 10000
-- performance_schema_max_cond_instances = 1000
-- ...
```

工业界常见调优项：

```
performance_schema_events_statements_history_size = 20
performance_schema_events_statements_history_long_size = 10000
performance_schema_digests_size = 10000
```

📌 **洞见**：8.0 的 P_S 很多参数是**自动伸缩**的——启动时分配最小值，按需增长。但正如前面"资源消耗"一节所说，**增长后不释放**。因此生产环境的关键决策是：

- **核心表**（digests、statements_history_long）给足上限
- **细粒度 waits 表**保持默认小值或干脆关闭 consumer
- **监控 P_S 自身内存**：定期查 `memory_summary_global_by_event_name` 表里 `memory/performance_schema/%` 的开销

---

### 默认值

书里强调：8.0 的默认值已经过 Oracle 工程团队的权衡，**对绝大多数 OLTP 负载是合理的**：

- ✅ 默认开：statement 插桩、statements_digest consumer、memory 插桩、metadata lock 插桩
- ❌ 默认关：绝大多数细粒度 wait/stage 插桩、waits 类 history 表

📌 **洞见**：不要被"P_S 开销大"的旧观念误导。这个观念来自 5.5 时代，当时默认全关，大家习惯手动全开——然后抱怨慢。8.0 的默认配置本身就是"够用且不贵"的状态。**工业界现在的共识是：Stock 8.0 的 P_S 配置，已经能回答"最坏查询家族是哪个"、"谁持有了元数据锁"、"哪块内存涨了"这三大问题，零配置**。

---

## 使用 Performance Schema

这一节是全书最"实战"的部分，书里按使用场景逐一展开。我用"问题 → P_S 解法 → 工业实践"的三段式重写。

### 检查 SQL 语句

核心表：`events_statements_summary_by_digest`——按语句指纹聚合。

```
SELECT DIGEST_TEXT, COUNT_STAR AS exec_count, 
       ROUND(AVG_TIMER_WAIT/1e9, 2) AS avg_ms,
       ROUND(SUM_TIMER_WAIT/1e9, 2) AS total_ms,
       SUM_ROWS_EXAMINED, SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

Datadog 的文档里给出一个典型行的例子：`SELECT * FROM employees WHERE emp_no > ?` 执行 2 次，平均 325ms——digest 把参数归一化了，所以**同一族 SQL 的不同参数值会被聚合到一起**，这正是找出"热点 SQL 家族"的关键。

sys Schema 等价写法：

```
SELECT * FROM sys.statement_analysis 
ORDER BY total_latency DESC LIMIT 20;
```

📌 **洞见**：`events_statements_summary_by_digest` 是 P_S 里**投资回报率最高的一张表**。工业界的做法是——**把它周期性地抓取出来送进 Prometheus/时序库**，做 SQL 性能回归检测。一个真实场景：某次发布后，某条核心 SQL 的 `AVG_TIMER_WAIT` 从 2ms 涨到 15ms，digest 表在 5 分钟内就能暴露这个回归，远早于慢查询日志（因为慢查询日志有 `long_query_time` 阈值，2ms→15ms 不会触发慢日志，但 digest 表能）。

---

### 检查读写性能

通过 `events_statements_summary_by_digest` 里的 `SUM_ROWS_EXAMINED` vs `SUM_ROWS_SENT` 比值，可以识别"读了很多、返回很少"的低效查询。更细粒度可以用 `events_waits_summary_*` 来看 IO 等待分布：

```
SELECT EVENT_NAME, COUNT_STAR, 
       ROUND(SUM_TIMER_WAIT/1e12, 3) AS total_seconds
FROM performance_schema.events_waits_summary_global_by_event_name 
WHERE COUNT_STAR > 0 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

📌 **洞见**：读写性能的瓶颈定位，本质是**找到"时间花在了哪个等待事件上"**。是等 InnoDB 行锁？是等磁盘 IO？还是等网络 flush？P_S 的 waits 表能精确告诉你。但要注意——**默认配置下 waits 类插桩大部分是关的**，需要你主动开启：

```
UPDATE performance_schema.setup_instruments 
SET ENABLED='YES', TIMED='YES' 
WHERE NAME LIKE 'wait/io/file/%';
```

工业界的最佳实践是：**平时不开，排查 IO 瓶颈时临时开，排查完再关**。

---

### 检查元数据锁

8.0 新增了 `performance_schema.metadata_locks` 表（8.0 默认启用元数据锁插桩）：

```
SELECT * FROM performance_schema.metadata_locks;
```

sys Schema 提供了更易读的视图：

```
SELECT * FROM sys.schema_table_lock_waits;
```

📌 **洞见**：元数据锁是 DDL 操作的隐形杀手——`ALTER TABLE` 阻塞了后续所有查询，但 SHOW PROCESSLIST 里看到的只是"Waiting for table metadata lock"，看不到**谁持有了这个锁**。P_S 的 metadata_locks 表能直接告诉你持有者，这是 5.7 时代做不到的。

> ⚠️ 但要记住前文提到的 Oracle 2026 年披露的那个 bug：在某些内存压力场景下，`data_locks` 表本身会引发死锁崩溃。**监控锁的代价，可能比锁本身更高**——这是 P_S 设计哲学里"插桩不是免费的"的最深刻注解。

---

### 检查内存使用情况

`memory_summary_global_by_event_name` 和 `memory_summary_by_thread_by_event_name` 是两张核心表：

```
-- 全局内存消费者 Top 10
SELECT EVENT_NAME, 
       CURRENT_NUMBER_OF_BYTES_USED/1024/1024 AS current_mb
FROM performance_schema.memory_summary_global_by_event_name 
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 10;

-- 查看 P_S 自身内存开销
SELECT FORMAT(SUM(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024, 2) AS MB_used
FROM performance_schema.memory_summary_global_by_event_name 
WHERE EVENT_NAME LIKE 'memory/performance_schema/%';
```

sys Schema 等价视图：

```
SELECT * FROM sys.memory_by_thread_by_current_bytes;
```

📌 **洞见**：内存插桩是 8.0 才默认开启的，这是 P_S 演进史上的一大进步。工业界用它来回答两个关键问题：

1. **哪个线程在泄漏内存？**​ → `memory_summary_by_thread_by_event_name`
2. **InnoDB 的 buffer pool 之外，内存都去哪了？**​ → `memory_summary_global_by_event_name`

一个真实案例：某服务内存缓慢增长，传统手段（看 OS 内存、看 InnoDB buffer pool）都正常，最后通过 P_S 发现是 `memory/sql/THD::main_mem_root` 持续增长——定位到一个连接池泄漏 bug。**没有 P_S 的内存插桩，这个问题几乎无解**。

---

### 检查变量

`performance_schema.global_variables` 和 `performance_schema.session_variables` 提供了**带版本的变量视图**：

```
SELECT * FROM performance_schema.global_variables 
WHERE VARIABLE_NAME LIKE 'innodb%';
```

📌 **洞见**：和 `SHOW GLOBAL VARIABLES` 相比，P_S 里的变量表的优势是**可以被 JOIN、可以被程序化查询、可以通过 sys Schema 做差异对比**。更重要的是，配合 `performance_schema.variables_info` 表（8.0.16+），你可以看到**每个变量是默认值、还是配置文件设置的、还是命令行指定的**——这对排查"为什么这个参数没生效"类问题极为关键。

---

### 检查最常见的错误

`events_statements_summary_by_digest` 里有 `SUM_ERRORS` 和 `SUM_WARNINGS` 列，可以直接定位高频错误 SQL：

```
SELECT DIGEST_TEXT, COUNT_STAR, SUM_ERRORS, SUM_WARNINGS
FROM performance_schema.events_statements_summary_by_digest 
WHERE SUM_ERRORS > 0 OR SUM_WARNINGS > 0
ORDER BY SUM_ERRORS DESC;
```

sys Schema 等价视图：

```
SELECT * FROM sys.statements_with_errors_or_warnings;
```

📌 **洞见**：错误 SQL 的定位，传统做法是扫错误日志。但错误日志是文本，难以聚合。P_S 的 digest 表把"同类错误 SQL"聚合在一起，让你一眼看出"是哪种 SQL 在频繁报错"，这对于捕获应用层 bug（比如某次代码发布引入了空指针导致 SQL 报错）远比错误日志高效。

---

### 检查 Performance Schema 自身

这一节书里讲的是"如何监控监控器自己"——P_S 的开销、内存占用、内部状态：

```
-- P_S 自身内存使用
SELECT * FROM performance_schema.memory_summary_global_by_event_name 
WHERE EVENT_NAME LIKE 'memory/performance_schema/%' 
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC;

-- P_S 内部状态
SHOW ENGINE PERFORMANCE_SCHEMA STATUS;
```

📌 **洞见**：这是 P_S 使用纪律里最重要的一环——**你必须定期审视 P_S 自己吃了多少资源**。工业界的标准做法是把它纳入监控面板的常驻指标。一个经验法则：如果 `memory/performance_schema/%` 的内存占用超过了 `innodb_buffer_pool_size` 的 1%，就该重新审视你的插桩配置了——你可能开了太多细粒度 wait 插桩。

---

## 小结

这一章教会我们的，远不止"怎么查几张表"。它的深层逻辑是：

**1. P_S 不是监控工具，是可编程的遥测系统**

插桩和消费者的二元模型，意味着你拥有对 MySQL 内部可观测性的**完全编程控制权**。这不是开箱即用的监控，而是让你自己定义"我想观测什么"。

**2. 默认配置已经够用，按需开启才是王道**

8.0 的默认 P_S 配置经过了 Oracle 工程团队的精心权衡：statement digest 开、细粒度 wait 关。工业界现在的主流做法是——**stock 配置起步，遇到具体问题再临时开启对应插桩**。

**3. sys Schema 是 P_S 的"人体工学外壳"**

日常操作走 sys，深入排查查 P_S 基表。这套组合拳让你既能快速回答常见问题，又能在必要时下钻到原始数据。

**4. P_S 不是免费的**

它的内存只增不减、它的插桩有 CPU 开销、它在极端情况下甚至能引发死锁。**把它当作"生产环境的听诊器"，而不是"全天候全景摄像头"**——诊断时拿起，诊断完放下。

**5. 配置要持久化**

运行期改 setup_* 表不持久，重启即丢。**任何你认为有价值的配置，都要写回 my.cnf**。

> 💡 一个现代工业实践的完整闭环：
> 
> 1. 让 8.0 的默认 P_S 配置跑着
> 2. 用 `sys.statement_analysis` 周期性抓取热点 SQL，送入 Prometheus
> 3. 出现性能问题时，**临时**开启对应插桩（如 metadata lock、wait/io）
> 4. 定位根因后，**选择性**把有价值的插桩固化到 my.cnf
> 5. 始终监控 `memory/performance_schema/%` 的自体开销
> 
> 这套方法论，让 P_S 从"看起来很美的内部特性"变成"真正的生产级可观测性基础设施"。

📌 **最后一句话的洞见**：P_S 的精髓不在于它"能看多细"，而在于它"让你决定看什么"。这种**可编排的可观测性**，正是现代可靠性工程（SRE）理念在 MySQL 内核层的落地——测量不是目的，**基于测量的迭代**才是。这也是为什么本书第 2 章讲 SRE 监控理念、第 3 章立刻讲 P_S：**P_S 是 MySQL 践行"用数据驱动调优"的第一性原理工具**。