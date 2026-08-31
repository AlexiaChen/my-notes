
# 第 11 章　扩展 MySQL——读书笔记

> 💡 这一章的本质，不是教你"加机器"或"上中间件"，而是建立一个**第一性原理**：MySQL 扩展性的根本矛盾在于**数据是有状态的**——你可以随意往负载均衡后面加 Web 服务器，但你不能随意往数据库后面加 MySQL 实例，因为数据必须保持一致、可寻址、可关联。
> 
> 正如书中所言："扩展 MySQL 与其他类型的服务器非常不同，这在很大程度上是因为数据是有状态的"。这句话揭示了整个扩展议题的哲学起点：**所有的扩展策略，本质上都是在"数据有状态"这一约束下做权衡**。2026 年的工业现实进一步验证了这一点——Vitess 已成为 CNCF 毕业项目，ProxySQL 已成为读池路由的事实标准，但它们的存在恰恰说明：MySQL 单机写能力存在天花板，而跨越这道天花板的代价，远比加一台 Web 服务器高昂。

---

## 🎯 本章核心脉络

Silvia Botros 和 Jeremy Tinley 在这一章埋下了一条主线——**扩展之前，必须先回答"你在扩展什么"**：

1. **可扩展性 ≠ 高性能**：高性能是"车跑得快"，可扩展性是"加车道不减速"
2. **先判定读受限还是写受限**——这两类瓶颈对应完全不同的扩展杠杆
3. **读扩展的相对低成本**：副本 + 读池 + 感知延迟的路由
4. **写扩展的高昂代价**：分片（Sharding）引入跨分片查询、分片键选择、再平衡等硬核难题
5. **Vitess 与 ProxySQL 的定位差异**：ProxySQL 是轻量级读写分离与查询路由利器；Vitess 是完整的分片编排平台
6. **扩展的隐性成本**：运维复杂度、故障域扩大、跨分片事务——这些往往比硬件成本更致命

---

## 什么是可扩展性

### 第一性原理视角

书中给出的定义简洁有力：**可扩展性（Scalability）是系统支撑不断增长的流量的能力**。但书中更深入的洞见是：

> 📌 **可扩展性 ≠ 最大吞吐量**
> 
> "系统的最大吞吐量并不等同于容量。大多数基准测试能够衡量一个系统的最大吞吐量，但真实的系统一般不会被使用到极限。如果达到最大吞吐量，则性能会下降，响应时间会变得不可接受且非常不稳定。"

📌 **洞见 1：扩展性的衡量标准是"成本与简单性"**

书中明确指出："一个系统扩展能力的好坏可以用成本和简单性来衡量。如果增加系统的扩展能力十分昂贵或复杂，那么在达到天花板时可能需要花费更多的努力来解决相关问题。"

这意味着：

- **纵向扩展（Scale Up）**：升级单机硬件。成本低、简单，但有天花板（单机极限）
- **横向扩展（Scale Out）**：增加节点。理论上无限，但复杂度指数级上升
- **最优扩展路径**：先纵向后横向，在恰当的时机切换

📌 **洞见 2：容量必须可被有效利用**

书中强调"容量必须是可以被有效利用的"。一个 10000 QPS 容量的系统，如果查询模式不均（20% CPU 下 1000 QPS 并不意味着还能加 4000 QPS，因为并非每个查询都相等），那么理论容量毫无意义。

**查询的不均等性**：

- 主键查找 vs 全表扫描：成本差几个数量级
- 简单 INSERT vs 批量 INSERT...SELECT：资源消耗天差地别
- 读查询 vs 写查询：对 CPU、IO、锁的影响完全不同

📌 **洞见 3：扩展性的"道路隐喻"**

给出了一个精妙的类比：

- **性能（Performance）**​ = 车的速度
- **容量（Capacity）**​ = 车道数 × 最大安全车速
- **可扩展性（Scalability）**​ = 增加车辆和车道时，交通速度不下降的程度

一个系统可以性能不高但具有可扩展性（慢但能线性扩容），也可以性能很高但不可扩展（快但加到极限就崩）。

---

## 读限制与写限制工作负载

### 理解工作负载

书中开宗明义："在考虑扩展数据库基础架构时，首先要审视的问题是，需要扩展读限制工作负载还是写限制工作负载。"

**读限制工作负载（Read-Bound）**：SELECT 总流量超过服务器容量

**写限制工作负载（Write-Bound）**：DML（INSERT/UPDATE/DELETE）操作超过服务器提供容量

📌 **洞见 1：工作负载的多维性**

数据库工作负载远不止 QPS 一个数字：

- **CPU 维度**：查询等待磁盘返回信息的时间计入成本
- **IO 维度**：磁盘 IOPS、吞吐量限制
- **网络维度**：网络吞吐量
- **资源维度**：CPU 数量、内存带宽

"这其中每一个都会对延迟产生影响，而延迟将直接关系到工作负载。"——这意味着扩展决策必须基于**瓶颈资源的识别**，而非笼统的"数据库压力大"。

📌 **洞见 2：社交图谱等关联数据的扩展噩梦**

书中特别指出："如果用户间存在关系，应用可能需要在整个相关用户群体间执行查询和计算，这比处理单个用户及其数据要复杂得多。社交网站经常会面临由那些人气很旺的用户组或朋友很多的用户所带来的挑战。"

这是**读扩展中最隐蔽的瓶颈**——不是"读多"，而是"读扇出大"。一个名人用户的粉丝列表查询，可能触发对百万级关联数据的扫描。这种场景下，单纯的读副本扩容无效，必须引入缓存、预计算、或图友好的存储。

### 读限制工作负载

📌 **扩展杠杆**（按成本从低到高）：

1. **查询优化与索引**：低成本、高收益，但天花板明显
2. **缓存层（Redis/Memcached）**：将热点读卸载
3. **读副本（Read Replicas）**：异步复制，分担读流量
4. **读池 + 负载均衡**：多个副本协同服务读请求

### 写限制工作负载

📌 **扩展杠杆**：

1. **队列化写入（Queuing）**：削峰填谷，使写入可预测
2. **批量写入**：减少事务开销
3. **功能分片（Functional Sharding）**：按业务模块拆分到不同实例
4. **水平分片（Sharding）**：数据行级拆分，突破单机写瓶颈
5. **Vitess**：工业级分片编排平台

> ⚠️ **关键认知**：复制扩展读，但对写毫无帮助。"When you are write-bound, you shard."——这是本章最重要的判断线。

---

## 功能拆分

**功能拆分（Functional Sharding）**：按业务模块/功能域将数据拆分到不同的 MySQL 实例。

📌 **洞见**：功能拆分是"轻量级分片"——

- **优点**：侵入性低，每个功能域独立扩展，故障域隔离
- **缺点**：跨功能查询困难，事务跨越难
- **适用场景**：业务模块边界清晰（如：用户库、订单库、商品库独立部署）

2026 年工业实践：微服务架构下，功能拆分几乎是默认的——每个微服务拥有自己的数据库实例。但这就引入了分布式事务、跨服务查询等新问题，需要 Saga 模式、CQRS 等架构模式配合。

---

## 使用读池扩展读

### 管理读池的配置

读池（Read Pool）的核心是**一组读副本 + 一个路由层**。

📌 **洞见 1：读池配置的关键维度**

**1. 权重分配**

腾讯云 2025 年的实践指南明确指出："权重分配策略应根据从库的实际硬件配置和当前负载动态调整，例如高性能从库可以分配更高比例的读请求"。

ProxySQL 支持多种负载均衡策略：

- **轮询（Round Robin）**：简单平均分配
- **最少连接（Least Connections）**：优先分配给连接数少的从库
- **权重轮询（Weighted RR）**：结合服务器性能分配权重
- **响应时间（Latency）**：基于历史响应时间动态调整

**2. 连接池管理**

"结合连接池管理（如使用 HikariCP 5.0+ 或 Druid 2.0+）减少频繁建立数据库连接的开销，能够进一步提升整体性能"。

📌 **洞见 2：读池配置的隐形陷阱**

ProxySQL 官方文档特别强调了一个关键认知——**不要使用通用的 `^SELECT` 路由规则**：

> 🔴 "Do not use in production. Generic SELECT routing rules cause problems with transactions, session state, and consistency."

这是因为：

- 事务内的 SELECT 必须读主库（避免读不到刚写入的数据）
- `SELECT ... FOR UPDATE` 必须路由到主库
- 临时表、用户变量、会话状态等都需要会话亲和性

**正确的做法**：先分析工作负载，用 `stats_mysql_query_digest` 识别具体的 SELECT 语句，然后为**特定的安全查询**创建针对性路由规则。

### 读池健康检查

📌 **洞见：TCP 检查远远不够**

书中强调健康检查的重要性。工业级实践要求多维健康检查：

**1. TCP 连通性**：端口是否可达

**2. MySQL 协议握手**：验证握手响应

**3. 查询探针**：执行 `SELECT 1` 确认服务可用

**4. 复制延迟检查**：监控 `Seconds_Behind_Source`，超过阈值剔除

ProxySQL 的配置示例：

```
UPDATE mysql_servers SET max_replication_lag=30 WHERE hostgroup_id=20;
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

📌 **洞见：复制延迟感知的路由是读池的灵魂**

一针见血地指出："Building lag-aware routing into the proxy and the data-access layer from the start is far cheaper than retrofitting it after a customer reports seeing data that 'disappeared and came back.'"

这意味着：

- **写后读一致性要求高的查询**（如余额查询、订单状态）→ 路由到主库
- **容忍短暂延迟的查询**（如用户画像、历史记录）→ 路由到副本
- **副本延迟超阈值**​ → 自动从读池剔除

### 选择负载均衡器算法

书中讨论了负载均衡算法的选择。结合 2026 年工业实践：

|算法|适用场景|优点|缺点|
|---|---|---|---|
|轮询|副本完全同构|简单、公平|忽略副本实际负载|
|加权轮询|副本异构|按能力分配|静态权重，不感知实时负载|
|最少连接|查询耗时差异大|动态均衡|短连接场景下效果有限|
|响应时间加权|对延迟敏感|自动路由到最快副本|需要持续监控|
|一致性哈希|带分片键的读|相同 key 路由到相同副本|副本扩缩容时重组|

📌 **洞见：算法选择取决于"副本是否同质"和"查询是否带分片键"**

- 同构副本 + 无状态读 → 轮询或最少连接
- 异构副本 → 加权轮询
- 多租户场景 → 按 tenant_id 一致性哈希（同一租户的读落到同一副本，提升 buffer pool 命中率）

### 排队机制

书中提出用排队机制使写入更可预测。

📌 **洞见：排队是"写扩展"的缓冲层**

**核心价值**：

1. **削峰填谷**：将突发写入平滑为恒定速率
2. **解耦应用与数据库**：应用快速返回，数据库按自身节奏消费
3. **批处理机会**：队列累积后批量写入，大幅提升吞吐量

**典型实现**：

- **应用层队列**：Kafka / RabbitMQ 承接写请求，消费者按数据库承受能力写入
- **数据库层队列**：ProxySQL 的查询排队、MySQL 的 `MAX_EXECUTION_TIME` 提示
- **合并写入**：将多个 UPDATE 合并为批量操作

> ⚠️ **排队的代价**：写延迟增加（异步写入），需要实现"至少一次"或"精确一次"的投递语义，以及消费失败的补偿机制。

---

## 使用分片扩展写

当纵向扩展和排队都无法解决写瓶颈时，**分片（Sharding）**​ 成为唯一选择。

📌 **洞见 1：分片的本质是"以复杂度换写能力"**

分片引入的根本性挑战：

1. **分片键选择**：一旦选错，跨分片查询将成为噩梦
2. **跨分片查询**：JOIN、聚合、ORDER BY 需要应用层特殊处理
3. **再平衡（Rebalancing）**：数据增长不均时需要迁移分片
4. **跨分片事务**：分布式事务的复杂性与性能损耗
5. **运维复杂度**：N 个分片 = N 倍运维成本

📌 **洞见 2：分片键的选择是"一次决定，长期痛苦"**

给出了一个极具价值的经验法则："decide your shard key before you have ten thousand tenants, because changing it afterward is among the most painful migrations in this field."

**分片键选择原则**：

- **高基数**：取值空间大，避免热点
- **均匀分布**：避免数据倾斜
- **查询亲和性**：大多数查询都带分片键（避免 scatter query）
- **业务稳定性**：不会随时间改变语义

**典型分片键**：

- 用户 ID（最常见）
- 订单 ID
- 地理位置 ID
- 时间戳范围

### 选择切分方案

书中讨论了切分方案的选择。2026 年工业界主要有两条路径：

**路径 1：应用层分片**

- 由应用程序代码直接路由到对应分片
- 优点：无中间件开销，灵活度高
- 缺点：侵入业务代码，跨分片查询难处理，不易维护

**路径 2：中间件分片**

- 在应用和数据库之间引入代理层
- 优点：对应用透明，提供连接池、读写分离等能力
- 缺点：引入中间件开销和运维复杂度
- 主流方案：Vitess、ProxySQL、Apache ShardingSphere

📌 **洞见：Vitess vs ProxySQL vs ShardingSphere 的定位差异**

|维度|ProxySQL|Vitess|ShardingSphere|
|---|---|---|---|
|核心能力|查询路由、读写分离|完整分片编排|分布式数据库中间件生态|
|分片透明度|低（需手动配置规则）|高（VSchema 声明式）|高（JDBC/Proxy 双模式）|
|运维复杂度|低|高|中|
|在线 resharding|不支持|支持（VReplication）|部分支持|
|适用规模|中小型读扩展|超大规模写扩展|中大型混合负载|

明确指出："ProxySQL 可以通过链式规则基于用户、schema、SQL 特征等进行复杂的路由，可以用于实现简单（或静态）的分片逻辑。配合外部配置脚本管理分片节点变更。适合较简单的分片需求……不如 Vitess/ShardingSphere 功能专一且强大。"

### 多个分片键

📌 **洞见：单一分片键的局限与多分片键的必要性**

真实业务场景往往需要按不同维度查询：

- 按 user_id 路由（用户中心）
- 按 order_id 查询（订单详情）
- 按 merchant_id 聚合（商家后台）

**解决方案：主分片键 + 二级索引分片**

Vitess 的 Vindex 机制：

- **Primary Vindex**：主分片键，决定数据物理位置
- **Secondary Vindex**：二级索引，通过 lookup 表映射到主键

例如：以 user_id 为主分片键，email 为二级 vindex。查询 `WHERE email = 'xxx'` 时，先查 lookup 表得到 user_id，再路由到正确分片。

> ⚠️ **代价**：lookup 表本身是 Vitess 表，可能也需要分片，形成递归。这是分片架构的固有复杂性。

### 跨分片查询

📌 **洞见：跨分片查询是"分片化的代价"**

书中讨论了跨分片查询的挑战。Vitess 将其称为 **Scatter Query**：

```
-- 不带分片键 → Scatter Query（广播到所有分片）
SELECT * FROM orders;  -- 命中每个分片，性能灾难

-- 带分片键 → 定向路由
SELECT * FROM orders WHERE user_id = 42;  -- 路由到单一分片
```

**跨分片查询的处理策略**：

1. **避免**：通过数据建模，使绝大多数查询带分片键
2. **聚合层**：应用层或中间件层合并各分片结果
3. **物化视图**：Vitess 支持跨分片的物化视图
4. **专用分析引擎**：将分析查询导向 ClickHouse 等
5. **消息/事件机制**：Vitess 提供跨分片消息特性

> 💡 的重要建议："the most durable fix is architectural: separate the operational path from the analytical path entirely, pushing heavy aggregation onto read replicas or an analytics-optimized engine. We frequently pair MySQL with ClickHouse for analytical workloads precisely so that the transactional tier never has to answer a question it was not built for."

---

## Vitess

### 第一性原理视角

Vitess 不是 MySQL 的替代品，而是**运行在 MySQL 之前的扩展层**："Vitess is not a replacement for MySQL - it is a middleware layer that runs in front of MySQL to add horizontal sharding, connection pooling, and query routing."

Vitess 起源于 YouTube 的 MySQL 集群管理需求，现已成为 CNCF 毕业项目，生产用户包括 Slack、Square、GitHub、Etsy、JD.com，以及整个 PlanetScale 托管服务。

### 核心架构

```
应用 → VTGate → VTTablet → MySQL
                ↓
            Topology (etcd/ZooKeeper)
```

**关键组件**：

- **VTGate**：无状态 SQL 路由器，对应用暴露 MySQL 协议，背后将查询路由到对应分片
- **VTTablet**：每个 MySQL 实例的 sidecar，管理本地 mysqld 生命周期、连接池、查询节流
- **Topology Service**：etcd/ZooKeeper/Consul，存储分片元数据
- **VTCTLD**：集群控制平面，负责 resharding、schema 变更等工作流编排
- **VTOrC**：可选的故障检测与自动故障切换守护进程（Orchestrator 的继任者）

### 分片模型：Keyspace、Shard、Vindex

- **Keyspace**：应用可见的"逻辑数据库"
- **Shard**：keyspace 的水平切片，由 key range 标识（如 `-80`、`80-`）
- **VSchema**：附加到 keyspace 的 JSON 文档，声明分片键和 vindex
- **Vindex**：从列值到分片 key range 的映射函数（hash、lookup、numeric、consistent_lookup 等）

### Vitess 的核心能力

**1. 内置分片与透明路由**

```
{
  "sharded": true,
  "vindexes": {
    "hash": {"type": "hash"}
  },
  "tables": {
    "orders": {
      "column_vindexes": [
        {"column": "user_id", "name": "hash"}
      ]
    }
  }
}
```

应用无需感知分片逻辑，VTGate 自动路由。

**2. 连接池与多路复用**

MySQL 的"每连接一线程"模型在数百活跃连接后即退化。VTGate 允许 10000 个应用连接，multiplexing 为每分片 100-300 个 MySQL 连接——**两层连接收敛**。

**3. 在线 Resharding（VReplication）**

这是 Vitess 最有价值的特性：

- 从 2 分片扩展到 4 分片，**无需停机**
- VReplication 通过 binlog 流复制保持数据同步
- 切换窗口以秒计
- 出错可回滚

的作者分享："I have done this resharding operation three times in production and never had a user-visible outage."

**4. 在线 Schema 变更（Online DDL）**

通过 VReplication 实现，替代 gh-ost 或 pt-online-schema-change：

- 创建影子表
- 批量拷贝数据
- 回放 binlog 变更
- 原子切换
- 全分片并行执行

**5. 查询改写与缓存**

VTGate 在转发前改写查询，规范化以优化缓存，可强制查询超时，甚至可通过 `--no-scatter` 拦截 scatter query。

### Vitess 的适用时机

|场景|建议|
|---|---|
|单 MySQL 节点，数据 < 1TB|不需要 Vitess|
|仅需连接池|ProxySQL 更简单|
|写吞吐超过单机|考虑 Vitess|
|需要水平分片|Vitess 是原生于 MySQL 的答案|
|使用 PlanetScale|Vitess 内置|

> ⚠️ **Vitess 的真实代价**：运维复杂度显著——VTGate、VTTablet、Topology Server、VTAdmin 都需部署和管理，学习曲线陡峭。PlanetScale 提供托管服务缓解这一问题。

### 分片键选择的经验教训

> "I have seen teams choose customer ID as the sharding key when they should have chosen tenant ID, because their access patterns are almost entirely within a tenant."

**关键认知**：

- 分片键必须匹配**实际查询模式**，而非直觉上的"主实体 ID"
- 多租户场景：tenant_id 通常是比 user_id 更好的分片键
- 错误的分片键 → 持续的 scatter query 或热点分片
- **分片键一旦选定，更改是最痛苦的迁移之一**

---

## ProxySQL

### 第一性原理视角

ProxySQL 的定位与 Vitess 不同——它是**轻量级的 MySQL 代理**，专注于查询路由、连接管理、读写分离。

ProxySQL 官方文档明确："Applications don't issue uniform traffic. A single service generates reads, writes, analytical queries, and bulk operations... The conventional fix is to modify the application... This couples your application code to your infrastructure decisions. The ProxySQL Approach... decouples your database topology from your application entirely."

### 核心能力

**1. 读写分离（Read/Write Splitting）**

自动将 SELECT 路由到副本，将 INSERT/UPDATE/DELETE 路由到主库。

**2. 查询改写（Query Rewriting）**

在代理层拦截并修改查询——为缺失索引提示的查询添加 hint，修正 ORM 生成的次优查询，无需修改应用代码。

**3. 流量分片（Traffic Sharding）**

基于查询中的键值（tenant ID、account range、key prefix）路由到不同分片后端。

**4. 查询规则引擎**

基于用户、schema、客户端 IP、查询文本的正则匹配，有序规则链决定路由目标。

### ProxySQL 读写分离的正确姿势

**基础配置**：

```
-- 主机组：HG 10 = 主库（写），HG 20 = 副本（读）
INSERT INTO mysql_servers (hostgroup_id, hostname, port) 
VALUES (10, '192.168.1.101', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) 
VALUES (20, '192.168.1.102', 3306), (20, '192.168.1.103', 3306);

-- 应用用户默认到写组
INSERT INTO mysql_users (username, password, default_hostgroup) 
VALUES ('appuser', 'AppPass123!', 10);

-- 查询规则：SELECT ... FOR UPDATE 走主库，其余 SELECT 走副本
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) 
VALUES (1, 1, '^SELECT .* FOR UPDATE', 10, 1),
       (2, 1, '^SELECT', 20, 1);
```

> 🔴 **生产环境的关键警告**：
> 
> "Do not use in production. Generic SELECT routing rules cause problems with transactions, session state, and consistency."

**推荐的智能路由流程**：

1. 先将所有流量路由到主库
2. 分析 `stats_mysql_query_digest`，识别昂贵的 SELECT
3. 判断哪些 SELECT 可以安全地在副本执行
4. **针对性地**为这些查询创建路由规则

### 复制延迟感知

```
-- 监控复制延迟
UPDATE global_variables SET variable_value='5000' 
WHERE variable_name='mysql-monitor_replication_lag_interval';

-- 设置单副本最大延迟阈值（秒）
UPDATE mysql_servers SET max_replication_lag=30 WHERE hostgroup_id=20;
```

延迟超阈值的副本自动从读池剔除。

### 连接多路复用（Multiplexing）

ProxySQL 的核心性能机制：

- 多个前端会话复用后端连接
- 大幅降低后端连接数和每连接 CPU 开销
- 在需要会话亲和性时（活跃事务、临时表、用户变量）自动禁用

某社交平台实践：通过连接复用，从库连接数从 1200 降至 300，内存占用减少 70%。

### ProxySQL 的集群高可用

- **无状态、水平可扩展**：可运行多个 ProxySQL 实例
- **配置同步**：core/satellite 节点 + checksum + epoch 机制，避免脑裂
- **优雅下线（Graceful Draining）**：让新连接停止接入，允许在途事务完成
- **连接风暴控制**：后端故障时避免每个应用主机都向剩余后端打开 N 个新连接

### ProxySQL vs Vitess：何时选谁

📌 **洞见：这不是二选一，而是层次问题**

|需求|选择|
|---|---|
|读写分离、读池路由|ProxySQL|
|连接池 + 查询改写 + 轻量路由|ProxySQL|
|完整分片编排 + 在线 resharding|Vitess|
|超大规模写扩展 + 自动故障切换|Vitess|
|既要读写分离又要分片|ProxySQL 前置 + Vitess 后置（VTGate 前再加 ProxySQL 层）|

明确指出："For simpler scaling needs, read replicas and ProxySQL cover most cases without Vitess's operational overhead."

---

## 小结

这一章表面讲的是"如何扩展 MySQL"，深层传递的是**SRE 视角的扩展哲学**：

**1. 扩展之前，先问"扩展什么"**

的道路隐喻揭示了一切：

- 性能 = 车速
- 容量 = 车道数 × 最大安全车速
- 可扩展性 = 加车道时不减速的能力

读受限和写受限是完全不同的瓶颈，对应完全不同的扩展杠杆。**复制扩展读，但对写毫无帮助**。

**2. 读扩展的相对低成本**

读扩展的路径清晰且风险可控：

1. 查询优化与索引（首选）
2. 缓存层卸载热点
3. 读副本 + 异步复制
4. 读池 + ProxySQL 路由
5. **关键**：复制延迟感知的路由——"Building lag-aware routing into the proxy and the data-access layer from the start is far cheaper than retrofitting it"

ProxySQL 的正确用法：

- ❌ 不要用通用的 `^SELECT` 路由规则
- ✅ 分析 `stats_mysql_query_digest`，为特定安全查询创建针对性规则
- ✅ 事务内查询、SELECT...FOR UPDATE 必须路由到主库
- ✅ 配置 `max_replication_lag` 自动剔除延迟副本

**3. 写扩展的高昂代价**

写扩展的唯一出路是分片，但分片引入的根本性挑战：

- **分片键选择**：一旦选错，跨分片查询成为噩梦
- **跨分片查询**：Scatter query 是性能杀手
- **再平衡**：数据倾斜时需要迁移分片
- **跨分片事务**：分布式事务的复杂度
- **运维复杂度**：N 个分片 = N 倍运维成本

**4. Vitess：为超大规模写扩展而生**

Vitess 的核心价值：

- **透明分片**：应用无感知，VSchema 声明式路由
- **在线 Resharding**：VReplication 实现零停机分片数调整
- **连接收敛**：VTGate 将上万应用连接 multiplexing 为每分片百级 MySQL 连接
- **在线 DDL**：通过 VReplication 替代 gh-ost
- **CNCF 毕业项目**：YouTube、Slack、Square、GitHub、PlanetScale 的生产验证

但代价是显著的运维复杂度——"operating Vitess is a genuine commitment"。

**5. ProxySQL：读池路由的轻量级利器**

ProxySQL 的定位：

- 协议感知的查询路由
- 读写分离、查询改写、流量分片
- 连接多路复用，大幅降低后端连接数
- 复制延迟感知的健康检查
- 配置集群同步，避免脑裂

**6. 功能拆分：轻量级写扩展**

按业务模块拆分到不同 MySQL 实例，是介于"单机"和"分片"之间的中间方案。微服务架构下几乎成为默认选择，但引入了分布式事务和跨服务查询的新挑战。

**7. 扩展的隐性成本**

> 💡 **贯穿全章的洞见**：扩展 MySQL 的本质成本不是硬件，而是**复杂度**。每一次扩展决策都在"性能/容量"与"复杂度/可维护性"之间做权衡。优秀的扩展架构，始于对工作负载的深刻理解（读受限还是写受限？瓶颈在哪个资源？查询模式是什么？），终于对工具（ProxySQL/Vitess/缓存/队列）的精准选型——既不过度工程，也不欠账。

正如书中所强调的："在阅读完本章时，你应该能够确定系统有哪些周期性模型，以及如何去扩展读和写。" 这意味着扩展不是一次性决策，而是**持续的过程**——随着业务从"曲棍球棒式增长"进入不同阶段，扩展策略必须相应演进：

```
阶段 1：单机 MySQL + 主从复制（读扩展）
    ↓ 读副本不够
阶段 2：读池 + ProxySQL（读路由精细化）
    ↓ 写瓶颈出现
阶段 3：功能拆分（按业务模块分实例）
    ↓ 单模块写仍瓶颈
阶段 4：Vitess 分片（水平写扩展）
    ↓ 跨分片查询频繁
阶段 5：HTAP 混合架构（MySQL + ClickHouse）
```

📌 **给读者的实践清单**：

- ✅ 扩展前先量化：是读受限还是写受限？瓶颈在哪个资源（CPU/IO/网络/连接数）？
- ✅ 读扩展优先：优化索引 → 加缓存 → 读副本 → 读池路由
- ✅ ProxySQL 配置：针对性查询规则，而非通用 `^SELECT` 路由
- ✅ 配置 `max_replication_lag`，实现延迟感知的读池
- ✅ 写扩展前先尝试：队列化、批量写入、功能拆分
- ✅ 分片键选择先于一切："decide your shard key before you have ten thousand tenants"
- ✅ 跨分片查询是性能杀手，数据建模时最大化"带分片键查询"的比例
- ✅ Vitess 适用时机：单机写到达天花板且愿意承担运维复杂度
- ✅ ProxySQL 适用时机：读写分离、连接池、查询路由——大多数场景的首选
- ✅ 分析查询导向 ClickHouse 等专用引擎，避免 MySQL 承担不擅长的聚合
- ✅ 扩展是持续过程，随业务阶段演进架构
- ❌ 不要盲目分片——分片是最后手段，不是首选
- ❌ 不要用 ProxySQL 做完整的分片方案（它不是 Vitess 的替代品）
- ❌ 不要忽略复制延迟——"lag-aware routing"必须从一开始就构建
- ❌ 不要在应用代码中硬编码分片逻辑（除非你准备好承担维护成本）
- ❌ 不要低估扩展的运维复杂度——N 个分片 = N 倍运维

**扩展是 MySQL 架构师的核心技艺，技艺精湛，系统在业务"曲棍球棒式增长"中才能平稳扩容。一个优秀的扩展架构，是用工作负载分析确定瓶颈、用读池路由扩展读、用分片突破写瓶颈、用 Vitess/ProxySQL 等工具精准落地——这四者的精妙平衡，正是本章"扩展 MySQL"的灵魂所在。理解了扩展的本质，你就理解了为什么现代 MySQL 架构（ProxySQL 读池 + Vitess 分片 + ClickHouse 分析）全都围绕着"数据有状态"这一根本约束做权衡：因为扩展 MySQL 永远不是"加机器"那么简单，而是在一致性、性能、复杂度三者之间寻找属于你业务的最优解。这正是为什么本章把"扩展 MySQL"放在"备份与恢复"之后——没有可靠的备份与复制基础，任何扩展架构都是空中楼阁；而没有清晰的扩展路径，业务的高速增长将瞬间冲垮单体 MySQL 的天花板。两者合一，方能成就真正可扩展的 MySQL 架构。**