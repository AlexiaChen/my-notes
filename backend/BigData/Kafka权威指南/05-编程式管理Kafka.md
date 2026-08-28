
# 第5章 编程式管理Kafka · 读书笔记

> 💡 这一章表面是"如何用 AdminClient 管理集群"，实质是在回答：**当一个分布式日志系统需要被"运维"时，它的管理平面应该长什么样？**​ AdminClient 不是 kafka-topics.sh 的 Java 翻译版，它是 Kafka "一切皆代码"哲学的体现——主题、分区、配置、消费者组、副本分配，都可以通过 API 程序化治理。这背后是一条主线：**Kafka 把集群管理本身也建模成了"向 Controller 发送请求 + 最终一致性传播"的异步过程**。书里中文翻译把 AdminClient 的异步语义讲得含糊，我们从第一性原理重梳一遍，并用工业界 IaC（Infrastructure as Code）实践补全。

---

## 5.1 AdminClient 概览

KafkaAdminClient 是 Admin 接口的默认实现，由 `AdminClient.create()` 工厂方法创建，**线程安全**。这意味着你可以在多个线程间共享同一个 AdminClient 实例——这与 KafkaConsumer 的"线程不安全"形成鲜明对比。

### 5.1.1 异步和最终一致性 API

这是理解 AdminClient 最关键的一点：**每一个方法调用都立即返回一个 Result 对象（内部封装了 KafkaFuture），不会阻塞等待集群操作完成**。

```
// 异步本质：调用立即返回，操作在后台进行
CreateTopicsResult result = admin.createTopics(topics);
// result 是一个 Future 容器，不是操作结果
result.all().get();  // 这里才阻塞等待完成
```

**为什么是异步？**​ 因为 AdminClient 的多数操作是"向 Controller 发送请求 → Controller 更新元数据 → 元数据异步传播到所有 broker"。从 Controller 到 broker 的元数据传播本身就是异步的，所以当 Controller 状态被完全更新时，AdminClient API 返回的 Future 才算完成。但此时**并不是每个 broker 都知道发生了状态变更**——`listTopics` 请求可能由不包含最新创建主题的 broker 负责处理。

这就是**最终一致性（Eventual Consistency）**：最终每个 broker 都会知道每一个主题的存在，但你不能保证是在什么时候。

> ⚠️ **工业界踩坑实录**：创建 topic 后立即 `listTopics()` 或让 producer 发送消息，可能遇到"topic 不存在"异常。正确的做法是调用 `result.all().get()` 阻塞等待，或在创建后稍作重试。

### 5.1.2 配置参数

AdminClient 的配置极简，核心是 `bootstrap.servers`——只需配置初始接触点，客户端会自动发现集群全貌。其他关键配置：

- **request.timeout.ms**（默认 120s）：请求超时，超过则重发或失败
- **connections.max.idle.ms**（默认 300s）：空闲连接关闭时间
- **metadata.max.age.ms**（默认 300s）：强制刷新元数据的周期
- **default.api.timeout.ms**：所有 API 的默认超时

### 5.1.3 扁平的结构

AdminClient 的所有操作方法都直接在接口上暴露，**没有复杂的层级继承**。这是一个有意的设计选择——管理操作种类有限但每个都很重要，扁平结构让 API 更易发现和使用。

主要方法族：

|方法族|作用|
|---|---|
|`createTopics` / `deleteTopics` / `describeTopics`|主题生命周期|
|`createPartitions`|分区扩容|
|`incrementalAlterConfigs`|增量修改配置|
|`describeCluster`|集群元数据|
|`describeConsumerGroups` / `listConsumerGroups`|消费者组查询|
|`alterConsumerGroupOffsets`|重置消费位移|
|`deleteRecords`|删除指定 offset 前的消息|
|`electLeaders`|触发首领选举|
|`alterPartitionReassignments`|副本重分配|

### 5.1.4 额外的话

AdminClient 在 Kafka 生态中的定位：**它不仅是运维工具，更是 Kafka 即代码（Kafka as Code）的基石**。Spring Boot 的 `KafkaAdmin`、Terraform 的 Kafka Provider、Kubernetes 的 Strimzi Operator，底层都依赖 AdminClient。这意味着你写的每一行 AdminClient 代码，都是在用程序化的方式"描述期望的集群状态"。

### 洞见：AdminClient 是 Kafka "控制平面"的 API 化

回到第一性原理——Kafka 是分布式日志。那么"管理 Kafka"本质上是在管理：

- **日志的分片（partitions）**
- **每个分片的副本分布（replicas）**
- **每个分片的保留策略（configs）**
- **谁在读哪个分片（consumer groups）**
- **读到了哪里（offsets）**

AdminClient 把这些"管理意图"全部 API 化，并且**统一向 Controller 发送请求**——因为 Controller 是集群元数据的唯一权威。这就是为什么 AdminClient 的所有写操作都是异步的：它们需要经过"客户端 → Controller → 各 broker"的异步传播链。

理解这一点，你就能理解后面所有的"管理操作"为什么都是这个模式：**发起请求 → 拿到 Future → 等待/回调**。

---

## 5.2 AdminClient 生命周期：创建、配置和关闭

### 创建与关闭

```
Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092,broker2:9092,broker3:9092");
props.put(AdminClientConfig.CLIENT_ID_CONFIG, "my-admin-client");
props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);

try (AdminClient admin = AdminClient.create(props)) {
    // 使用 admin
    DescribeClusterResult cluster = admin.describeCluster();
    System.out.println("Cluster ID: " + cluster.clusterId().get());
}  // 自动关闭
```

> ⚠️ **工业界强制规范**：AdminClient 打开了 TCP 连接并启动了后台网络线程，**必须用 try-with-resources 或显式 close() 关闭**，否则文件句柄泄露会快速拖垮 broker。

### 5.2.1 client.dns.lookup

DNS 查找策略配置。在容器化、Kubernetes 环境下尤其重要——broker 可能用服务发现域名。可选值：

- `use_all_dns_ips`（默认）：解析所有 DNS A 记录，提高可用性
- `resolve_canonical_bootstrap_servers_only`：只解析规范主机名

### 5.2.2 request.timeout.ms

默认 120s。控制客户端等待请求响应的最长时间。超过则重发（如有重试）或失败。

**调优建议**：

- 在自动化流水线中，可以调到 30-60s，快速失败
- 在大规模集群操作中（如跨上千分区的副本重分配），可能需要调到 300s+

### 洞见：生命周期管理的"第一性原理"

AdminClient 的生命周期是"创建 → 使用 → 关闭"三段式，这与 Producer/Consumer 一致。但有一个关键差异：**AdminClient 通常是"短命"的**——它在执行完一批管理操作后就被关闭。这与 Producer/Consumer 的"长命"（贯穿应用生命周期）形成对比。

这种差异源于用途不同：

- Producer/Consumer 是"数据平面的持续参与者"
- AdminClient 是"控制平面的临时操作者"

所以工业界最佳实践：**不要在应用运行时长期持有一个 AdminClient 实例**。需要管理操作时创建，用完后立即关闭。对于需要持续监控的场景（如 Cruise Control），才长期持有。

---

## 5.3 基本的主题管理操作

AdminClient 提供了完整的 CRUD：

**创建主题**：

```
NewTopic topic = new NewTopic("orders.v1", 12, (short) 3)
    .configs(Map.of(
        "cleanup.policy", "delete",
        "retention.ms", "604800000",  // 7天
        "min.insync.replicas", "2"
    ));
CreateTopicsResult result = admin.createTopics(List.of(topic));

try {
    result.all().get();  // 阻塞等待创建完成
    System.out.println("Created orders.v1");
} catch (ExecutionException e) {
    if (e.getCause() instanceof TopicExistsException) {
        System.out.println("orders.v1 already exists");  // 幂等处理
    }
}
```

> 💡 **工业界幂等模式**：在 CI/CD 流水线中，主题创建应该是幂等的——运行两次不应报错。`TopicExistsException` 应被视为成功。

**查询主题**：

```
DescribeTopicsResult describe = admin.describeTopics(List.of("orders.v1"));
TopicDescription desc = describe.allTopicNames().get().get("orders.v1");
for (TopicPartitionInfo p : desc.partitions()) {
    System.out.printf("Partition %d leader=%d replicas=%s isr=%s%n",
        p.partition(), p.leader().id(),
        p.replicas().stream().map(Node::id).toList(),
        p.isr().stream().map(Node::id).toList());
}
```

**删除主题**：

```
admin.deleteTopics(List.of("old-topic")).all().get();
```

### 洞见：主题管理即"基础设施即代码"

AdminClient 让主题管理从"运维手工操作"升级为"代码声明"。Spring Boot 的 `TopicBuilder` 就是典型例子：

```
@Bean
public NewTopic topic1() {
    return TopicBuilder.name("thing1")
        .partitions(10)
        .replicas(3)
        .compact()
        .build();
}
```

应用启动时，Spring 的 `KafkaAdmin` 自动调用 AdminClient 创建这些主题。**主题成为代码的一部分，纳入版本控制**——这是现代 Kafka 运维的标准实践。

---

## 5.4 配置管理

Kafka 的配置管理支持两种方式，但**工业界一律推荐 incrementalAlterConfigs**：

```
// ✅ 推荐：增量修改，只改指定的 key
ConfigResource resource = new ConfigResource(ConfigResource.Type.TOPIC, "orders.v1");
AlterConfigOp op = new AlterConfigOp(
    new ConfigEntry("retention.ms", "1209600000"),  // 14天
    AlterConfigOp.OpType.SET
);
admin.incrementalAlterConfigs(Map.of(resource, List.of(op))).all().get();

// ❌ 过时：alterConfigs 会覆盖整个配置
// admin.alterConfigs(configs).all().get();
```

> ⚠️ **关键差异**：`alterConfigs` 会覆盖整个配置对象，如果不小心遗漏了某些 key，那些配置会被重置为默认值。`incrementalAlterConfigs` 只修改你列出的 key，其他保持不变。

**常见动态配置**：

- `retention.ms`：消息保留时长
- `retention.bytes`：消息保留大小
- `max.message.bytes`：单条消息最大字节
- `min.insync.replicas`：最小同步副本数
- `cleanup.policy`：delete 或 compact

### 洞见：配置即"日志行为的旋钮"

每个配置项都是在调整"分布式日志"的行为：

- `retention.*` → 日志保留多久、多大
- `min.insync.replicas` → 日志写入的可靠性级别
- `cleanup.policy` → 日志是"按时间淘汰"还是"按 key 压缩"

通过 AdminClient 动态调整这些配置，意味着**你能在不重启集群的情况下改变日志系统的行为**——这是 Kafka 运维灵活性的根源。

---

## 5.5 消费者群组管理

这是 AdminClient 最强大的能力之一——程序化地 introspect 和干预消费者组。

### 5.5.1 查看消费者群组

```
// 列出所有消费者组
Collection<ConsumerGroupListing> groups = admin.listConsumerGroups().all().get();
groups.forEach(g -> System.out.println(g.groupId() + " " + g.state().orElse(null)));

// 描述特定组
DescribeConsumerGroupsResult descResult = admin.describeConsumerGroups(List.of("my-group"));
ConsumerGroupDescription description = descResult.describedGroups().get("my-group").get();

System.out.println("State: " + description.state());
System.out.println("Protocol: " + description.partitionAssignor());
System.out.println("Coordinator: " + description.coordinator());
System.out.println("Members: " + description.members().size());

// 查看组成员分配
for (MemberDescription member : description.members()) {
    System.out.printf("  %s (%s) @ %s%n",
        member.consumerId(), member.clientId(), member.host());
    System.out.println("    Partitions: " + member.assignment().topicPartitions());
}
```

**工业界诊断脚本**：

```
// 检查消费滞后
ListConsumerGroupOffsetsResult offsetsResult = admin.listConsumerGroupOffsets(groupId);
Map<TopicPartition, OffsetAndMetadata> offsets = offsetsResult.partitionsToOffsetAndMetadata().get();

// 对每个分区查询 log-end-offset，计算 lag
for (TopicPartition tp : offsets.keySet()) {
    long committed = offsets.get(tp).offset();
    long logEndOffset = admin.listOffsets(
        Map.of(tp, OffsetSpec.latest())
    ).all().get().get(tp).offset();
    long lag = logEndOffset - committed;
    System.out.printf("%s-%d: committed=%d, logEnd=%d, lag=%d%n",
        tp.topic(), tp.partition(), committed, logEndOffset, lag);
}
```

**状态机洞察**：消费者组有五种状态：

- `EMPTY`：无活跃成员
- `PREPARING_REBALANCE`：准备再均衡
- `COMPLETING_REBALANCE`：再均衡中
- `STABLE`：稳定消费
- `DEAD`：已解散

监控时如果发现组长期处于 `PREPARING_REBALANCE` 或 `COMPLETING_REBALANCE`，说明发生了**再均衡风暴**。

### 5.5.2 修改消费者群组

**重置消费位移**——这是数据重放的刚需：

```
// 重置到位移 1000
Map<TopicPartition, OffsetAndMetadata> offsets = Map.of(
    new TopicPartition("orders", 0), new OffsetAndMetadata(1000L)
);
admin.alterConsumerGroupOffsets("my-group", offsets).all().get();
```

**CLI 等价命令**：

```
# 重置到最早
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --reset-offsets --to-earliest \
  --topic orders --execute

# 重置到时间戳
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --reset-offsets \
  --to-datetime 2024-01-15T00:00:00.000 \
  --topic orders --execute

# 相对偏移
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --reset-offsets --shift-by -100 \
  --topic orders --execute
```

> ⚠️ **生产环境铁律**：重置位移前必须先用 `--dry-run` 预览，确认无误后再 `--execute`。并且重置位移**只对非活跃组有效**——如果消费者组正在运行，需要先停掉所有消费者。

**删除消费者组**：

```
admin.deleteConsumerGroups(List.of("orphan-group")).all().get();
```

> 💡 短期存在的消费者（如批处理作业）产生的孤儿组必须定期清理，否则 `__consumer_offsets` topic 会被无用数据填满。

### 洞见：消费者组管理是"数据流的可编程控制"

传统 MQ 中，查看/重置消费进度往往需要登录管理控制台。Kafka 通过 AdminClient 把这件事**完全 API 化**——这意味着：

- 数据平台可以根据 lag 自动触发重放
- A/B 测试可以程序化地创建独立消费组
- 故障恢复可以自动重置到位移 X
- 数据合规审计可以查询任意时间点的消费状态

**消费者组不再是黑盒，而是可编程的数据流控制平面**。

---

## 5.6 集群元数据

```
DescribeClusterResult cluster = admin.describeCluster();
System.out.println("Cluster ID: " + cluster.clusterId().get());
System.out.println("Controller: " + cluster.controller().get());
cluster.nodes().get().forEach(node ->
    System.out.println("Broker " + node.id() + " @ " + node.host() + ":" + node.port()));
```

虽然 Kafka 抽象了 broker 细节，但在运维场景中，显式获取集群元数据是必要的：

- **容量规划**：知道集群有多少 broker
- **机架感知验证**：确认 broker.rack 配置正确
- **Controller 健康检查**：监控 Controller 是否频繁切换
- **拓扑发现**：自动化工具需要知道集群布局

---

## 5.7 高级的管理操作

这是本章最有"运维分量"的一节。

### 5.7.1 为主题添加分区

```
Map<String, NewPartitions> increase = Map.of(
    "events", NewPartitions.increaseTo(48)
);
admin.createPartitions(increase).all().get();
```

**CLI 方式**：

```
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic events --partitions 24
```

> ⚠️ **分区扩容的残酷真相**：
> 
> 1. **不可逆**：Kafka 不允许缩减分区
> 2. **打破 key 顺序**：已有 key 继续映射到原分区，新 key 哈希到全部 48 个分区，跨边界 key 的顺序性被破坏
> 3. **触发消费者再均衡**：消费者组检测到分区数变化，强制 rebalance
> 4. **不重分布已有数据**：现有消息仍留在原分区，新分区只有新消息

**工业界安全扩容六步法**：

1. **前置评估**：确认 `max.partitions`、ISR 完整性
2. **消费者兼容性审查**：客户端版本 ≥ 1.0，支持动态分区感知
3. **生产环境灰度测试**：影子环境模拟
4. **滚动暂停消费者**：逐个停止以冻结 offset 提交
5. **执行分区扩增**：低流量窗口操作
6. **恢复并监控**：观察再平衡时间、消费延迟

**Oracle 云的最佳实践建议**：

> 扩容后必须验证：新分区是否被 producer 实际使用。如果新分区长时间为空，可能的原因：producer 显式指定分区、自定义分区器、key 高度倾斜、流量太小。

### 5.7.2 从主题中删除消息

```
// 删除 offset 1000 之前的所有消息
Map<TopicPartition, RecordsToDelete> recordsToDelete = Map.of(
    new TopicPartition("orders", 0), RecordsToDelete.beforeOffset(1000L)
);
DeleteRecordsResult result = admin.deleteRecords(recordsToDelete);
result.all().get();  // 等待完成
```

**关键语义澄清**：

- `deleteRecords` 删除的是**指定 offset 之前的消息**——本质是"推进低水位（low watermark）"
- 返回值是每个分区的 `lowWatermark`
- **这不是"按主键删除"**，Kafka 不支持任意消息删除
- 支持 broker 版本 ≥ 0.11.0.0

**典型用例**：

- 测试环境快速清理
- GDPR "被遗忘权"合规：删除某用户所有历史事件（需要按 key 找到对应 offset 范围）
- 日志压缩的补充：对 compacted topic 删除过期 key 的旧值

> 💡 工业界更常见的"清空 topic"做法：临时把 `retention.ms` 调到极小值（如 1000ms），等待清理完成后恢复。这避免了 `deleteRecords` 逐分区调用的复杂性。

### 5.7.3 首领选举

```
// 首选副本选举（Preferred Leader Election）
admin.electLeaders(ElectionType.PREFERRED, null).all().get();
```

**CLI 方式**：

```
kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED --all-topic-partitions
```

**为什么需要首领选举？**

当 broker 发生故障或重启后，分区的 leader 可能集中在少数 broker 上，造成**leader 倾斜（leader skew）**——某些 broker 承载 10 倍于其他的流量。

**Preferred Leader Election 的原理**：每个分区的 replica 列表中，第一个副本被称为"preferred replica"。正常情况下它就是 leader。当 leader 因为故障转移而变更后，通过 preferred election 可以让 leader "回家"。

> 💡 **这是最便宜的负载均衡操作**——不涉及数据移动，只是变更 leader 角色。在 full reassignment 之前，永远先尝试 preferred election。

### 5.7.4 重新分配副本

这是最重量级的运维操作。

```
// 变更分区副本分布
Map<TopicPartition, Optional<List<Integer>>> reassignments = Map.of(
    new TopicPartition("orders", 0), Optional.of(List.of(1, 2, 3))
);
admin.alterPartitionReassignments(reassignments).all().get();

// 取消正在进行的重分配
admin.alterPartitionReassignments(
    Map.of(new TopicPartition("orders", 0), Optional.empty())
).all().get();
```

**CLI 方式**：

```
# 1. 生成重分配计划
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2,3,4" --generate

# 2. 执行重分配（限流 50MB/s）
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file plan.json \
  --execute --throttle 52428800

# 3. 验证进度
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file plan.json --verify
```

**触发副本重分配的典型场景**：

- 新增 broker 后：Kafka **不会**自动把数据迁移到新 broker，必须手动重分配
- 下线 broker 前：必须把副本搬离
- Leader 倾斜修复无效时：可能需要调整副本分布
- 磁盘容量不均：把分区搬到有余量的 broker

> ⚠️ **生产环境铁律**：
> 
> 1. **永远限流**：不加 `--throttle` 的重分配会占满网络带宽，触发 ISR 收缩
> 2. **限流值留 30% 余量**：如生产流量 150MB/s，限流设 50MB/s
> 3. **分批进行**：大规模集群一次只移动少量分区
> 4. **Review 生成计划**：`--generate` 产生的计划可能过度移动数据，人工优化

**Cruise Control——自动化均衡**：

LinkedIn 开源的 Cruise Control 是大规模 Kafka 集群（>20 broker）的标配。它：

- 持续监控 broker 的磁盘、CPU、网络、分区数
- 自动生成最优重分配计划
- 按计划执行，自带限流
- 暴露 REST API 供 DevOps 流水线调用

### 洞见：高级运维操作的"第一性原理"

把这四种"高级操作"串起来看，它们都在解决同一个问题：**分布式日志的物理布局需要随业务演化**。

|操作|解决什么问题|代价|
|---|---|---|
|添加分区|消费并行度不够|打破 key 顺序，触发 rebalance|
|删除消息|合规/清理需求|只能按 offset 截断|
|首领选举|leader 倾斜|极低（仅变更角色）|
|副本重分配|broker 增减/负载不均|高（数据在 broker 间移动）|

**核心洞察**：Kafka 的"日志"抽象在单机上是简单的，但在分布式尺度上，"日志如何分布在多个 broker 上"成为了持续的运维课题。AdminClient 提供的这些 API，本质上是让运维人员能够**程序化地重塑日志的物理分布**。

---

## 5.8 测试

Kafka 提供了 `MockAdminClient` 用于单元测试——不需要真实集群就能验证管理逻辑。

```
// 使用 MockAdminClient 进行单元测试
try (MockAdminClient mockAdmin = new MockAdminClient()) {
    // 模拟集群状态
    mockAdmin.mockCreateTopics(results -> {
        // 自定义 mock 行为
    });
    
    // 测试你的管理代码
    MyTopicManager manager = new MyTopicManager(mockAdmin);
    manager.ensureTopicExists("test-topic");
}
```

**测试策略**：

- **单元测试**：用 MockAdminClient 验证业务逻辑（如幂等创建、配置增量更新）
- **集成测试**：用 EmbeddedKafka 或 Testcontainers 启动真实 Kafka，验证端到端行为
- **混沌测试**：在测试环境模拟 broker 故障、网络分区，验证 AdminClient 的容错

---

## 5.9 小结

把全章串起来，AdminClient 的设计哲学是：

> **把"管理 Kafka 集群"这件事本身，也建模为向 Controller 发送的异步请求流。**

这与 Producer 向分区 Leader 发送异步请求、Consumer 从分区 Leader 异步拉取数据，形成了完美的对称——**Kafka 集群中所有的"操作"，无论数据平面还是控制平面，都是异步的、最终一致的**。

### 工业界 AdminClient 使用模式

**模式一：基础设施即代码（GitOps）**

```
// Spring Boot 应用启动时自动确保主题存在
@Bean
public NewTopic ordersTopic() {
    return TopicBuilder.name("orders.v1")
        .partitions(24)
        .replicas(3)
        .config(TopicConfig.RETENTION_MS_CONFIG, "604800000")
        .config(TopicConfig.MIN_INSYNC_REPLICAS_CONFIG, "2")
        .build();
}
```

**模式二：自动化运维流水线**

```
// CI/CD 中创建主题，幂等处理
public void ensureTopic(String topic, int partitions, short replication) {
    NewTopic newTopic = new NewTopic(topic, partitions, replication);
    try {
        admin.createTopics(List.of(newTopic)).all().get();
    } catch (ExecutionException e) {
        if (e.getCause() instanceof TopicExistsException) {
            log.info("Topic {} already exists", topic);
        } else {
            throw new RuntimeException(e);
        }
    }
}
```

**模式三：消费者组诊断与自愈**

```
// 监控 lag，自动触发数据重放
public void autoRemediateConsumerGroup(String groupId) {
    // 1. 检查 lag
    long totalLag = calculateTotalLag(groupId);
    if (totalLag > ALERT_THRESHOLD) {
        // 2. 通知告警
        alertService.notify("Consumer group " + groupId + " lag: " + totalLag);
        // 3. 可选：自动重置到最早（需业务确认）
        if (autoRemediationEnabled) {
            resetToEarliest(groupId);
        }
    }
}
```

### 生产环境红线清单

基于全章内容，总结 AdminClient 使用的七条铁律：

```
□ 1. 永远用 try-with-resources 或显式 close() 管理生命周期
□ 2. 创建主题时捕获 TopicExistsException 实现幂等
□ 3. 修改配置用 incrementalAlterConfigs，绝不用 alterConfigs
□ 4. 重置消费位移前先用 --dry-run 预览
□ 5. 分区扩容不可逆，必须评估 key 顺序影响
□ 6. 副本重分配必须限流，留 30% 网络余量
□ 7. deleteRecords 只能按 offset 截断，不能按主键删除
```

### 洞见：AdminClient 是 Kafka 运维文明的基石

回到第一性原理——Kafka 是分布式日志。那么"管理 Kafka"就是在管理"分布式日志的生命周期"：

- **创建/删除主题**​ = 创建/销毁日志
- **添加分区**​ = 扩展日志的并行度
- **修改配置**​ = 调整日志的行为策略
- **副本重分配**​ = 重塑日志的物理分布
- **首领选举**​ = 调整日志的读写入口
- **消费者组管理**​ = 管理日志读取游标的进度
- **删除消息**​ = 推进日志的低水位

**所有这一切，AdminClient 都用统一的异步 API 暴露出来。**​ 这就是为什么 Kafka 能够在工业界实现真正的"数据平台"——不仅是数据存储和传输的平台，更是**可通过代码完全治理的平台**。

> 📌 **全章最核心的一句话**：AdminClient 不是 kafka-topics.sh 的 Java 翻译，而是 Kafka "控制平面"的 API 化——它让"管理分布式日志"这件事本身，也变成了可编程、可自动化、可版本控制的代码。理解这一点，你就能理解为什么现代 Kafka 运维能够支撑起成千上万个主题、数百万个分区的超大规模集群：因为管理平面和数据平面一样，都是异步的、最终一致的、分布式的。

下一章我们将进入 Kafka 的"存储与副本"内部原理——你会看到 AdminClient 触发的这些管理操作（创建分区、副本重分配、首领选举），在 broker 内部是如何通过 Controller、ISR、HW（High Watermark）等机制实现的。带着"控制平面与数据平面的对称性"这条主线读下去，Kafka 架构的完整图景会彻底清晰。