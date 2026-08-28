
# 第12章 管理 Kafka · 读书笔记

> 💡 这一章表面是"Kafka 的运维命令大全"，实质是在回答：**当一个 Kafka 集群从"能跑"变成"生产级"时，运维者拿什么工具去塑造它？**​ 原书第2版有大量基于 ZooKeeper 的命令（如 `--zookeeper localhost:2181`），但 Kafka 4.0 已彻底移除 ZooKeeper。所以这份笔记的做法是：**以原书目录为骨架，用现代 Kafka（KRaft）的 `--bootstrap-server` 视角重写所有命令**，并贯穿一条运维第一性原理——**每一个管理操作都要回答四个问题：是否在线？是否可逆？是否影响 key 顺序？是否影响复制带宽？**​ 这四个问题的答案，决定了你是"优雅调整"还是"制造事故"。

---

## 12.1 主题操作

主题是 Kafka 管理的最基本单元。所有主题操作都应使用 `--bootstrap-server`（KRaft 和现代 ZooKeeper 模式都支持），原书基于 `--zookeeper` 的旧命令在新版本中已废弃。

### 12.1.1 创建新主题

```
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2
```

**分区数规划——这是全章最重要的决策**：

工业界共识公式（来自 Jun Rao 的经典帖子，仍是 Confluent 官方文档基础）：

```
partitions = max(T/P, T/C)
```

其中 T=目标吞吐，P=单分区生产吞吐，C=单分区消费吞吐。

但 LinkedIn、Confluent、Shopify、Cloudflare 等超大规模部署的共同经验是：**按未来 2 年的峰值吞吐规划分区数，而不是今天**。原因是：

> ⚠️ **增加分区是不可逆操作，且对已按 key 路由的 topic 会永久破坏跨 key 的顺序保证**。新消息会 hash 到新旧分区全集，导致同一 key 的历史消息和新消息落在不同分区——任何依赖"key 级顺序"的消费者都会出错。

**分区数规划的最佳实践**：

```
低吞吐场景：3-6 个分区（最小化开销）
中等吞吐：12-24 个分区（平衡并行度）
高吞吐：50-100 个分区
极高吞吐：100+ 个分区
```

生产环境建议：

- **预留 1.5-2 倍增长空间**：`P = max(P_writes, P_consume, P_consumers) × 1.5~2`
- **避免质数分区数**：质数不能均匀分配到 broker，会导致 leader 倾斜。常用 12、24、30、60 等除数多的数字
- **基准测试单分区吞吐**：用 `kafka-producer-perf-test.sh` 在实际集群测出你的单分区安全吞吐（通常为 10-100 MB/s），取 60-70% 作为规划常数
- **在 topic 上记录规划假设**：用 `kafka-configs.sh` 把分区数的规划依据（峰值 MB/s、消费者数、保留时长）作为文档写入

**副本因子决策矩阵**：

|RF|min.insync.replicas|耐久性|可用性|适用场景|
|---|---|---|---|---|
|1|1|无|单 broker|开发环境|
|2|1|基础|容忍 1 broker 故障|低风险生产数据|
|3|2|强|容忍 1 broker 故障|**标准生产配置**​|
|5+|3+|极强|容忍 4 broker 故障|金融/医疗关键数据|

> 💡 **工业铁律**：生产环境 RF=3 + min.insync.replicas=2 + acks=all，这是与第7章可靠性黄金组合的配套。

### 12.1.2 列出集群中的所有主题

```
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 12.1.3 列出主题详情

```
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
```

输出包括：分区数、副本因子、每个分区的 Leader、Replicas、ISR 列表、配置覆盖。

**关键诊断信息**：

- **Leader 分布是否均衡**：如果某 broker 上的 leader 远多于其他 broker，说明需要做首选 leader 选举
- **ISR 是否等于 Replicas**：如果 ISR < Replicas，说明有副本掉队，需要排查网络/磁盘
- **OfflinePartition**：如果 Leader 显示为 none，分区不可用

### 12.1.4 增加分区

```
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic orders --partitions 24
```

**⚠️ 关键限制（必须刻进肌肉记忆）**：

1. **Kafka 永远不允许减少分区**——这是设计决定，因为"如何合并两个分区的日志"在语义上无解
2. **增加分区不会重新分布已有数据**——旧消息仍在原分区，只有新消息使用新分区
3. **对已按 key 路由的 topic，增加分区会破坏 key 的顺序保证**——同一 key 的新消息可能落到与新分区，旧消息还在老分区
4. **操作是在线的**——不影响现有生产消费，但可能在瞬间影响消费者 rebalance

**工业界的应对模式**：

```
模式1：创建时过度规划（推荐）
  创建 topic 时直接给 24 或 48 分区，避免后续 --alter
  
模式2：新建 topic + 双写迁移
  创建 orders-v2（更多分区）
  生产者双写到 orders 和 orders-v2
  消费者逐步迁移到 orders-v2
  确认无误后停写 orders，最后删除
```

### 12.1.5 减少分区

**Kafka 不支持**。原书提到这是设计限制，至今未变。

如果需要"减少"分区，唯一安全的做法是：

```
# 1. 创建新 topic 带目标分区数
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders-new --partitions 6 --replication-factor 3

# 2. 用 MirrorMaker 或客户端双写迁移

# 3. 验证数据一致性后删除旧 topic
kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic orders
```

### 12.1.6 删除主题

```
kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic orders
```

**前置条件**：

```
delete.topic.enable=true  # broker 配置，默认 true
```

**删除主题的工业级检查清单**：

> ⚠️ **删除主题是高风险的不可逆操作**——它可能删除现有主题和数据。

1. **确认 `delete.topic.enable=true`**
2. **审计下游依赖**：用 Confluent Control Center 或 Kafka Connect 监控确认没有其他应用、流处理作业、ETL 管道在消费
3. **优雅停止消费者**：先停消费应用，或把消费者重新分配给其他 topic
4. **检查 ACL 权限**：用户必须有该 topic 的 ALTER 和 DELETE 权限
5. **考虑软删除替代硬删除**：

```
# 软删除：设置极短保留时间，让日志自然清空
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders \
  --alter --add-config retention.ms=1
```

1. **删除关联的消费者组**（否则 offsets.retention.minutes 默认 7 天内旧 offset 仍在，重建 topic 时可能造成重复消费）：

```
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --delete --group orders-consumers
```

1. **僵尸 topic 处理**：如果 topic 标记为待删除但迟迟不消失，可能是 broker 缓存或元数据清理延迟。KRaft 模式下用 `kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status` 检查 controller 元数据；必要时重启 broker

### 洞见：主题操作的"第一性原理"

主题是"日志"这一 Kafka 核心抽象的容器。所有主题操作都在与"日志的物理约束"博弈：

- **分区是并行度的基本单位**——增加分区=增加并行度，但破坏 key 顺序
- **副本是可靠性的基本单位**——增加副本=增加耐久性，但增加存储和网络开销
- **日志是不可变的有序序列**——所以"减少分区"（需要合并日志）在语义上无解

**这就是为什么工业界在创建 topic 时就要"过度规划"**——因为后续调整的成本远高于初始规划的成本。一个设计良好的 topic 创建流程应该是：

```
1. 估算 2 年后的峰值吞吐
2. 基准测试得出单分区安全吞吐
3. 计算分区数（取除数多的数字，预留 1.5-2 倍）
4. RF=3, min.insync.replicas=2
5. 设置 retention.ms 和 retention.bytes
6. 用文档记录规划假设
```

---

## 12.2 消费者群组

### 12.2.1 列出并描述消费者群组信息

```
# 列出所有消费者组
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 描述特定消费者组
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

**输出关键列**：

- **TOPIC**：消费的 topic
- **PARTITION**：分区号
- **CURRENT-OFFSET**：消费者组当前已提交的 offset
- **LOG-END-OFFSET**：分区最新 offset
- **LAG**：滞后消息数（LOG-END-OFFSET - CURRENT-OFFSET）
- **CONSUMER-ID**：消费者实例 ID
- **HOST**：消费者主机
- **CLIENT-ID**：客户端 ID

**LAG 监控是 Kafka 运维的核心 SLI**——它直接反映了管道的健康度。

### 12.2.2 删除消费者群组

```
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --delete --group order-processors
```

**前置条件**：

> ⚠️ **消费者组必须处于 Empty 状态才能删除**——即没有任何活跃消费者。如果有消费者在运行，删除会失败。

```
# 先确认组状态
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
# STATE 列应为 Empty

# 再删除
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --delete --group order-processors
```

**典型场景**：

- 应用下线，清理遗留的消费者组
- topic 重建前清理旧的 offset 记录
- 切换消费者组命名策略

### 12.2.3 偏移量管理

这是运维中最常用的操作之一：

**查看偏移量**：

```
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

**重置偏移量（谨慎操作）**：

```
# 重置到最早
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --reset-offsets --group order-processors \
  --topic orders --to-earliest --execute

# 重置到最新
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --reset-offsets --group order-processors \
  --topic orders --to-latest --execute

# 重置到指定偏移量
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --reset-offsets --group order-processors \
  --topic orders --to-offset 12345 --execute

# 相对当前偏移量向前/向后跳过
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --reset-offsets --group order-processors \
  --topic orders --shift-by -1000 --execute  # 重放最近 1000 条
```

**⚠️ 偏移量重置的风险**：

> 重置偏移量是"时间旅行"操作——它会改变消费者对"已经处理过什么"的认知。如果下游系统（数据库、外部 API）不具备幂等性，重置偏移量可能导致：
> 
> - 重复处理（向前重置）
> - 数据丢失（向后重置）
> 
> 第8章讲过的 EOS 和幂等消费在此至关重要。

**偏移量管理的最佳实践**：

1. **先用 `--reset-offsets` 不加 `--execute` 预览**——会显示将要设置的偏移量但不实际执行
2. **确认消费者组已停止**——活跃消费者会与重置操作冲突
3. **下游系统必须幂等**——否则重置后重复处理会破坏业务数据
4. **记录操作日志**——偏移量重置是可审计事件

**导出/导入偏移量**（集群迁移场景）：

```
# 导出
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --export --group order-processors --topic orders \
  > offsets.txt

# 导入到新集群
kafka-consumer-groups.sh --bootstrap-server new-cluster:9092 \
  --import --group order-processors --topic orders \
  --from-file offsets.txt
```

---

## 12.3 动态配置变更

Kafka 的配置有三个层级（从具体到通用）：

```
Topic 级配置 > Client/User 级配置 > Broker 级默认配置
```

动态配置的优点：**无需重启 broker 即可生效**（部分配置例外）。

### 12.3.1 覆盖主题的默认配置

```
# 设置 topic 级配置
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders \
  --alter \
  --add-config retention.ms=2592000000,min.insync.replicas=2

# 常用 topic 配置
retention.ms=604800000        # 7 天保留
retention.bytes=107374182400  # 100GB 上限
segment.bytes=1073741824      # 1GB 段文件
segment.ms=3600000            # 1 小时滚动
cleanup.policy=compact        # 压缩策略
min.insync.replicas=2         # 最小同步副本
max.message.bytes=10485760    # 最大消息 10MB
```

### 12.3.2 覆盖客户端和用户的默认配置

```
# 为用户设置配额
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type users --entity-name producer-app \
  --alter \
  --add-config producer_byte_rate=10485760

# 为客户端 ID 设置配额
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type clients --entity-name consumer-app \
  --alter \
  --add-config consumer_byte_rate=5242880

# 常用配额配置
producer_byte_rate=10485760    # 10MB/s 生产限速
consumer_byte_rate=5242880     # 5MB/s 消费限速
request_percentage=75          # 请求队列使用率上限 75%
```

**配额的作用**：

- 防止"吵闹的邻居"效应——单个客户端耗尽集群资源
- 多租户环境下的公平保障
- 突发流量时的保护

### 12.3.3 覆盖 broker 的默认配置

```
# 动态设置 broker 级配置
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 1 \
  --alter \
  --add-config log.retention.hours=168

# 或设置集群默认（对所有 broker 生效）
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-default \
  --alter \
  --add-config log.retention.hours=168
```

**KRaft 模式的重要变化**：

> ⚠️ 某些原本动态的 broker 配置在 KRaft 模式下改为**静态配置，需要重启 broker/controller 才能生效**：
> 
> - `advertised.listeners`

这是因为 KRaft 模式下，元数据由 Raft 共识管理，某些配置的生命周期与 ZooKeeper 时代不同。

### 12.3.4 查看被覆盖的配置

```
# 查看 topic 的所有配置（包括默认值）
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders \
  --describe --all

# 查看 broker 的动态配置
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 1 \
  --describe
```

### 12.3.5 移除被覆盖的配置

```
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders \
  --alter \
  --delete-config retention.ms
```

移除后，配置回退到 broker 级默认值或集群级默认值。

### 洞见：动态配置的"第一性原理"

Kafka 的配置体系是一个**三层继承链**：

```
Topic 级 (最具体) → Client/User 级 → Broker 级默认 (最通用)
```

查找配置时自底向上：先看 topic 有没有覆盖，再看 client/user 有没有覆盖，最后用 broker 默认。

**为什么这个设计精妙？**

1. **细粒度控制**：可以为特定 topic 设置特定配置，而不影响其他 topic
2. **动态调整**：大多数配置可以在线修改，无需重启
3. **继承与覆盖**：通用配置在 broker 级设一次，特殊配置在 topic 级覆盖
4. **多租户友好**：不同用户/客户端可以有不同配额

> 💡 **工业实践**：把"集群级默认配置"作为基线（如 retention.ms=7天），把"业务级特殊配置"在 topic 级覆盖（如审计日志 retention.ms=365天）。这样既有统一性又有灵活性。

---

## 12.4 生产和消费

控制台生产者和消费者是调试、验证、快速测试的利器。

### 12.4.1 控制台生产者

```
# 基本生产
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders

# 带 key 的生产
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --property parse.key=true \
  --property key.separator=:

# 从文件批量生产
cat messages.txt | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --property parse.key=true \
  --property key.separator=:

# 带 SASL_SSL 认证的生产
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --producer.config client.properties
```

**client.properties 示例**：

```
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="producer-app" \
  password="producer-secret";
ssl.truststore.location=/var/private/ssl/kafka.truststore.p12
ssl.truststore.password=truststore-password
```

**性能测试生产者**：

```
kafka-producer-perf-test.sh \
  --topic orders \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput 10000 \
  --producer-props bootstrap.servers=localhost:9092
```

### 12.4.2 控制台消费者

```
# 从最新位置消费
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders

# 从头消费
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --from-beginning

# 用消费者组消费
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --group debug-consumers

# 显示 key
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --property print.key=true \
  --property key.separator=:

# 消费 __consumer_offsets 内部 topic（调试消费者偏移量）
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic __consumer_offsets \
  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" \
  --consumer.config client.properties

# 消费 __transaction_state 内部 topic（调试事务）
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic __transaction_state \
  --formatter "kafka.coordinator.transaction.TransactionLog\$TransactionLogMessageFormatter"

# 性能测试消费者
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --messages 1000000
```

### 洞见：控制台工具的真正价值

控制台生产者/消费者看似简单，但它们是**分布式系统调试的"显微镜"**：

- **隔离故障**：当整个管道出现问题时，用控制台消费者直接消费 topic，判断是生产者问题、topic 问题、还是下游消费者问题
- **验证配置**：用控制台生产者以特定 key 发送消息，验证分区路由是否符合预期
- **检查数据格式**：直接消费观察消息的 key/value 格式，排查序列化问题
- **应急手动操作**：在自动化系统故障时，用手动生产和消费作为应急手段

> 💡 **工业铁律**：生产环境必须能通过控制台工具消费任何 topic（在 ACL 允许范围内）。如果某个 topic "只能被应用程序消费，不能被控制台消费"，说明你的管道有黑盒——这是运维噩梦的开始。

---

## 12.5 分区管理

### 12.5.1 首选首领选举

Kafka 的每个分区有一个"首选副本"（preferred replica）——即 replica 列表中的第一个。当集群平衡时，首选副本应该是 leader。

**为什么需要手动触发**：

当 broker 重启、网络分区恢复、或副本重分配完成后，leader 可能不在首选副本上，导致 leader 分布不均。

```
# 触发全集群首选 leader 选举
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions

# 触发特定 topic 的首选 leader 选举
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --topic orders
```

**首选 leader 选举的特性**：

- **安全操作**：只在 ISR 内部迁移 leader，不丢数据
- **快速**：毫秒级完成
- **自动触发**：`auto.leader.rebalance.enable=true`（默认）会定期自动平衡，但周期较长（默认 300 秒检查一次）

**Unclean leader 选举（最后手段）**：

```
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type UNCLEAN \
  --topic orders --partition 3
```

> ⚠️ **Unclean 选举会选择一个非 ISR 副本作为 leader，直接丢弃未同步的数据**。生产环境必须设置 `unclean.leader.election.enable=false`，只在紧急情况下手动触发。这是第7章可靠性契约的底线。

### 12.5.2 修改分区的副本

这是 Kafka 运维中最常见的操作——在以下场景触发：

- 扩容集群后，重新平衡副本分布
- 缩容集群前，把副本从待下线 broker 移走
- 提高副本因子以增强可靠性
- 修复副本倾斜

**步骤1：生成重分配计划**

```
# 定义要移动的 topic
cat > topics-to-move.json <<EOF
{"version":1,"topics":[{"topic":"orders"}]}
EOF

# 生成候选计划（目标 broker 列表：0,1,2,3,4）
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics-to-move.json \
  --broker-list "0,1,2,3,4" \
  --generate > reassignment-plan.json
```

**步骤2：执行重分配（带限流）**

```
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment-plan.json \
  --execute \
  --throttle 52428800  # 50 MB/s 限流
```

**⚠️ 限流是强制性的工业实践**：

> 不限流的副本重分配会打满网络带宽，导致生产消费延迟飙升。50 MB/s 是一个常见的起点，应根据集群网络容量调整，留出至少 30% 余量给正常业务流量。

**步骤3：验证进度**

```
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment-plan.json \
  --verify
```

`--verify` 输出每个分区的状态：in progress / completed successfully / failed。重分配完成后，`--verify` 会自动移除限流配置。

**步骤4：提高副本因子**

```
// increase-rf.json
{"version":1,"partitions":[{"topic":"orders","partition":0,"replicas":[0,1,2]}]}
```

```
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file increase-rf.json \
  --execute --throttle 52428800
```

**取消正在进行的重分配**：

```
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment-plan.json \
  --cancel
```

**修改副本分配的工业级最佳实践**：

1. **始终使用 --throttle**：50 MB/s 起步，根据网络容量调整
2. **在低流量窗口操作**：虽然在线操作，但大量数据移动会影响性能
3. **review 生成的计划**：有时生成器会选择移动更多数据的计划，手动编辑 JSON 可以减少跨 broker 流量
4. **监控重分配进度**：用 --verify 定期检查，超时未完成要排查慢 follower（磁盘 IO 或网络瓶颈）
5. **完成后确认限流已移除**：否则正常复制流量会被限流

### 12.5.3 转储日志片段

`kafka-dump-log.sh` 是离线解析日志段文件的工具：

```
# 解析数据日志段
kafka-dump-log.sh --files 00000000000000000000.log --print-data-log

# 检查索引完整性
kafka-dump-log.sh --files 00000000000000000000.log --index-sanity-check

# 仅验证索引而不打印
kafka-dump-log.sh --files 00000000000000000000.log --verify-index-only

# 解码 __consumer_offsets topic 的日志
kafka-dump-log.sh --files 00000000000000000000.log --offsets-decoder

# 解码 __transaction_state topic 的日志
kafka-dump-log.sh --files 00000000000000000000.log --transaction-log-decoder

# KRaft 模式下解析元数据日志
kafka-dump-log.sh --cluster-metadata-decoder \
  --files tmp/kraft-combined-logs/_cluster_metadata-0/00000000000000023946.log
```

**使用场景**：

- **诊断数据损坏**：当 Kafka 集群出现异常时，快速定位问题
- **消息审计**：显示消息的偏移量、时间戳、键值大小，解析消息头和事务信息
- **验证日志完整性**：确保数据一致性
- **性能分析**：检查消息批量大小，分析压缩效率，评估存储利用率

### 12.5.4 副本验证

验证跨 broker 的副本数据一致性：

```
kafka-replica-verification.sh --bootstrap-server localhost:9092 \
  --topic-white-list "orders.*"
```

该工具检查所有副本是否包含相同的消息集合，用于诊断副本不一致问题。

### 洞见：分区管理的"第一性原理"

分区管理本质上是在操作"日志副本的物理布局"。每一次副本移动都是一次数据迁移，涉及：

```
1. 在新副本上追上 Leader 的日志（高网络 IO）
2. 切换 Leader（毫秒级不可用）
3. 删除旧副本（磁盘 IO）
```

这就是为什么：

- **必须限流**：否则正常业务流量被饿死
- **必须选低峰期**：虽然在线操作，但数据移动会影响性能
- **KRaft 让这事更简单**：Controller 用 Raft 协议管理元数据，broker 关闭过程由 quorum-based controller 自动管理 leader 转移和元数据更新，比 ZooKeeper 模式更可靠高效

> 💡 **工业界真相**：在健康的 Kafka 集群上，副本重平衡是最频繁的运维任务——增加 broker 后要重平衡、发现 leader 倾斜后要重平衡、broker 硬件老化要重平衡。LinkedIn 的 Cruise Control 就是为自动化这个任务而生——它监控磁盘、CPU、网络、分区数等指标，自动生成最优重分配计划。在 Kafka 3.x + Kubernetes (Strimzi) 环境中，Cruise Control 已是一等公民。

---

## 12.6 其他工具

Kafka 还提供了一系列辅助工具：

|工具|用途|
|---|---|
|`kafka-metadata-quorum.sh`|KRaft 模式下查看元数据仲裁状态|
|`kafka-storage.sh`|格式化存储目录（KRaft 模式首次启动）|
|`kafka-features.sh`|管理 KRaft 的 metadata.version|
|`kafka-get-offsets.sh`|获取 topic 的偏移量信息|
|`kafka-jmx.sh`|读取 JMX 指标|
|`kafka-delegation-tokens.sh`|管理委派令牌|
|`kafka-log-dirs.sh`|查询日志目录信息|
|`kafka-producer-perf-test.sh`|生产者性能测试|
|`kafka-consumer-perf-test.sh`|消费者性能测试|
|`kafka-verifiable-producer.sh`|可验证生产者（用于测试）|
|`kafka-verifiable-consumer.sh`|可验证消费者（用于测试）|

**工业界用法**：

- **性能基准**：用 perf-test 工具在集群部署前测出实际吞吐
- **存储规划**：用 `kafka-log-dirs.sh` 监控磁盘使用情况
- **KRaft 运维**：用 `kafka-metadata-quorum.sh` 检查 controller 仲裁健康度

---

## 12.7 不安全的操作

> ⚠️ 这一节的所有操作都是"紧急逃生舱"，正常情况下**永远不应该执行**。每次执行前都要三思，最好有第二个人 review。

### 12.7.1 移动集群控制器

KRaft 模式下，controller 是 quorum 的一部分，手动移动 controller 概念已不存在。但在 ZooKeeper 模式下：

```
# 优雅关闭当前 controller broker，触发新 controller 选举
kafka-server-stop.sh
```

**现代做法**：KRaft 的 controller 故障转移是自动的、近实时的（毫秒级），无需人工干预。

### 12.7.2 移除待删除的主题

当 topic 被标记为待删除但迟迟不消失时：

**KRaft 模式**：

```
# 检查 controller 元数据
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status

# 如果元数据中没有该 topic，说明 broker 缓存问题，重启 broker
kafka-server-stop.sh && kafka-server-start.sh
```

**ZooKeeper 模式（旧版）**：

```
# 手动删除 ZooKeeper 中的 topic 节点
zookeeper-shell localhost:2181
rmr /brokers/topics/orders
rmr /admin/delete_topics/orders
```

### 12.7.3 手动删除主题

**⚠️ 绝对不要在 broker 运行时手动删除数据目录**——这可能导致 broker 实例故障。

如果必须手动清理（如磁盘已满，broker 已停）：

```
# 1. 完全停止 broker
kafka-server-stop.sh

# 2. 删除该 topic 的数据目录
rm -rf /kafka/data/orders-*/

# 3. 如果是 KRaft 模式，还需清理元数据日志中的残留
# （通常需要重启 controller quorum 来重新同步）

# 4. 重启 broker
kafka-server-start.sh
```

> ⚠️ **华为云运维手册明确警告**：手动删除数据目录可能导致服务信息丢失，broker 实例故障。**不要手动创建或修改数据目录中的文件或文件夹**。

### 洞见：不安全操作的"第一性原理"

Kafka 的所有"不安全操作"本质上都是在绕过其分布式共识机制：

- 手动删 ZooKeeper 节点 = 绕过 controller 的元数据管理
- 手动删数据目录 = 绕过 broker 的日志管理
- 手动移动 controller = 绕过 Raft 选举协议

**这些操作的共同特征**：它们破坏了 Kafka "元数据与数据一致性"的核心 invariant。一旦执行，集群可能进入不可用状态，且难以恢复。

> 📌 **工业铁律**：任何时候你"必须"执行 12.7 节的操作，都意味着集群已经处于异常状态。正确的做法是：
> 
> 1. 先尝试安全的恢复路径（重启 broker、用工具清理元数据）
> 2. 如果必须手动介入，先在测试环境演练
> 3. 执行前备份所有相关数据
> 4. 执行后全面验证集群状态
> 
> KRaft 模式的出现大幅减少了这类不安全操作的需求——controller 故障转移是自动的，元数据由 Raft 共识保证一致性，broker 关闭由 quorum controller 自动管理。这也是为什么 Kafka 4.0 彻底移除 ZooKeeper。

---

## 12.8 小结

把全章串起来，Kafka 管理是一个**四层运维体系**：

```
┌─────────────────────────────────────────────────────────┐
│ 第一层：主题与分区管理                                     │
│  • 创建主题：按 2 年峰值规划分区数，RF=3                  │
│  • 增加分区：不可逆，破坏 key 顺序                        │
│  • 减少分区：Kafka 不支持，只能新建 topic 迁移             │
│  • 删除主题：高风险，先审计下游依赖                        │
├─────────────────────────────────────────────────────────┤
│ 第二层：消费者组与偏移量管理                                │
│  • 监控 LAG 作为核心 SLI                                 │
│  • 重置偏移量前必须确保下游幂等                            │
│  • 删除消费者组前必须停止所有消费者                        │
├─────────────────────────────────────────────────────────┤
│ 第三层：动态配置管理                                       │
│  • 三层继承：Topic > Client/User > Broker                │
│  • 大多数配置在线生效，无需重启                            │
│  • KRaft 模式下某些配置改为静态，需重启                     │
├─────────────────────────────────────────────────────────┤
│ 第四层：分区与副本管理                                     │
│  • 首选 leader 选举：安全操作，毫秒级                     │
│  • 副本重分配：必须限流，低峰期执行                        │
│  • 日志转储：离线诊断数据损坏                              │
│  • KRaft 让 controller 故障转移自动化                      │
└─────────────────────────────────────────────────────────┘
```

### 工业界 Kafka 管理决策框架

```
需要管理 Kafka 集群？
├─ 主题操作
│  ├─ 新建 → 按 2 年峰值规划分区，RF=3，min.insync.replicas=2
│  ├─ 扩分区 → 确认无 key 顺序依赖，否则新建 topic 迁移
│  └─ 删 topic → 审计下游，软删除优先，备份 offset
│
├─ 消费者组
│  ├─ LAG 告警 → 检查消费者是否掉线，broker 是否过载
│  ├─ 重置偏移 → 预览(--reset-offsets 不加 --execute)
│  └─ 删组 → 确认 Empty 状态
│
├─ 配置变更
│  ├─ Topic 级 → kafka-configs --entity-type topics
│  ├─ 用户级配额 → kafka-configs --entity-type users
│  └─ Broker 级 → kafka-configs --entity-type brokers
│
└─ 分区管理
   ├─ Leader 倾斜 → 首选 leader 选举（安全）
   ├─ 扩容/缩容 → 副本重分配 + 限流 50MB/s
   └─ 数据损坏 → kafka-dump-log 离线诊断
```

### 生产环境管理基线

```
# ========== 主题创建基线 ==========
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders \
  --partitions 24 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2 \
  --config cleanup.policy=delete

# ========== 消费者组监控 ==========
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
# 关注 LAG 列，设置告警阈值

# ========== 副本重分配（带限流） ==========
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file plan.json \
  --execute --throttle 52428800

kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file plan.json \
  --verify
# verify 通过后限流自动解除

# ========== 首选 Leader 选举 ==========
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions

# ========== KRaft 元数据检查 ==========
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
```

### 给读者的"Kafka 管理心智模型"

> 📌 **全章最核心的一句话**：Kafka 管理的本质是"在分布式日志的物理约束下塑造集群"——分区是并行度的原子单位（增加不可逆、破坏 key 顺序），副本是可靠性的原子单位（移动必须限流），配置是行为的调节旋钮（三层继承、动态生效）。理解每一次管理操作的"四问"（是否在线？是否可逆？是否影响 key 顺序？是否影响复制带宽？），你就能区分"优雅调整"和"制造事故"。而 KRaft 模式的出现，把原本需要 ZooKeeper 介入的不安全操作，变成了 controller quorum 自动处理的内部机制——这正是 Kafka 向"自管理"演化的方向。

**三个必须刻进肌肉记忆的认知**：

1. **分区数在创建时就要过度规划**。增加分区不可逆且破坏 key 顺序，这是 Kafka 最容易被低估的设计约束。工业界的正确做法是：按 2 年峰值规划，取除数多的数字（12/24/60），预留 1.5-2 倍增长空间。如果事后发现分区不够，正确的姿势是新建 topic + 双写迁移，而不是 `--alter --partitions`。
2. **副本重分配必须限流**。50 MB/s 是起点，留出 30% 网络余量给正常业务。不限流的重分配会打满网络，导致生产消费延迟飙升——这是 Kafka 运维最常见的"自残"操作。
3. **KRaft 改变了管理的边界**。ZooKeeper 模式下需要手动干预的很多操作（controller 故障转移、元数据清理、broker 优雅关闭），在 KRaft 模式下由 quorum controller 自动处理。Kafka 4.0 彻底移除 ZooKeeper 不是简单的"换组件"，而是把"集群管理"从外部系统收编进 Kafka 自身——这意味着 12.7 节的"不安全操作"在现代 Kafka 中大多已无必要。

**工业界真相**：

> 绝大多数 Kafka 管理事故的根源，不是工具不好用，而是违反了三条原则：
> 
> - **低估分区数规划的长期影响**：创建时 3 个分区，半年后业务增长 10 倍，不得不痛苦迁移
> - **副本重分配不限流**：半夜被报警叫醒，发现整个集群生产消费延迟飙升
> - **滥用不安全操作**：手动删 ZooKeeper 节点或数据目录，导致集群进入无法自动恢复的状态
> 
> 优秀 Kafka 运维的标志，不是"会使用所有命令"，而是**知道哪些操作不要做**。一个设计良好的 Kafka 集群，90% 的管理操作应该是：创建 topic、监控 LAG、偶尔做首选 leader 选举、扩缩容时限流重分配。其余的"高级操作"越是很少用，说明你的集群越健康。

