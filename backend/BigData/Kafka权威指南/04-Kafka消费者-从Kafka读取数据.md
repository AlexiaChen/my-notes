
# 第4章 Kafka消费者——从Kafka读取数据 · 读书笔记

> 💡 这一章表面是"怎么读消息"，实质是在回答：**当一个客户端要把"分布式日志里的某一行"变成应用程序里的对象时，它需要在哪些维度上做取舍？**​ 消费者不是简单的"拉取器"，它是客户端侧的"日志游标管理者"——管理自己读到哪了（offset）、和谁一起读（consumer group）、何时让出分区（rebalance）、如何处理失败（commit strategy）。Producer 端"写入语义"的四维权衡，到这里变成了消费者端"读取语义"的三维博弈：**交付语义（at-least-once / at-most-once / exactly-once）、消费并行度（partition 分配）、再均衡稳定性**。书里中文翻译把这些参数讲得散，我们从第一性原理重梳一遍。

---

## 4.1 Kafka消费者相关概念

### 4.1.1 消费者和消费者群组

KafkaConsumer 的本质职责只有一件：**从分配给自己的分区日志里按顺序读取记录，并把"读到哪了"这个游标（offset）管理好**。

两个核心概念必须先刻进骨头里：

**Consumer**：一个独立的读取客户端。一个 Consumer 实例可以消费多个分区，但**一个分区在同一时刻只能被同一个 group 内的一个 Consumer 消费**。

**Consumer Group**：多个 Consumer 共享同一个 `group.id`，协同消费一个或多个 topic。group 内的 Consumer 瓜分所有分区——**每个分区只分配给组内的一个 Consumer**，从而实现水平扩展。

> 💡 关键洞察：多个 Consumer Group 可以**独立地、完整地**消费同一个 topic。这意味着"订单服务"和"风控服务"可以各自是一个 group，都从 `topic.orders` 消费全量数据，互不干扰——这就是 Kafka 作为"数据平台"的 fan-out 能力根源。

**为什么必须是"一个分区一个消费者"？**​ 回到第一性原理：Kafka 只保证**单个分区内有序**。如果允许两个 Consumer 同时读同一个分区，它们各自的 offset 推进速度不同，就无法保证消息按顺序被处理。所以"分区 → 单消费者"的映射，是"分区内有序"这个物理事实的逻辑推论。

**消费者数量与分区数的关系**：

```
消费者数 < 分区数：部分消费者负责多个分区
消费者数 = 分区数：理想状态，1:1 映射
消费者数 > 分区数：多余的消费者空闲，浪费资源
```

所以**分区数是消费并行度的硬上限**。这也是为什么第2章、第3章都在强调"提前规划分区数"。

### 4.1.2 消费者群组和分区再均衡

**再均衡（rebalance）**​ 是指：分区所有权从一个消费者转移到另一个消费者的过程。触发条件：

1. 新消费者加入 group
2. 已有消费者离开 group（优雅关闭或崩溃）
3. 订阅的 topic 分区数发生变化
4. 消费者被 coordinator 踢出（心跳超时、poll 超时）

**再均衡的两种协议**：

**Eager Rebalance（急切再均衡，传统默认）**：

- 所有消费者**立即停止消费**
- 放弃所有分区所有权
- 重新加入 group
- 获取全新的分区分配
- 整个 group 在再均衡期间 **stop-the-world**

**Cooperative Rebalance（协作再均衡，Kafka 2.4+）**：

- 只重新分配**需要移动的那部分分区**
- 其他消费者**继续处理**自己未被回收的分区
- 大幅缩短 group 整体不可用窗口

> ⚠️ 工业界最佳实践：使用 `CooperativeStickyAssignor` 替代默认的 `RangeAssignor` 或 `RoundRobinAssignor`。它把"再均衡影响面"从"全体消费者 × 全部分区"缩小到"少数消费者 × 少数分区"。

**再均衡为什么是痛点？**

再均衡的成本不仅仅是"几秒钟不消费"——对于**有状态消费者**（如 Kafka Streams 的 KTable），分区迁移意味着**海量状态数据要在消费者之间搬运**。一次再均衡可能导致分钟级甚至十分钟级的停顿。

### 4.1.3 群组固定成员

这是 Kafka 2.3 引入、KIP-345 定义的**静态成员资格（Static Membership）**机制。

**动态成员资格（默认）的问题**：

- 每次 consumer 重启，broker 都会分配一个新的随机 memberId
- Coordinator 把"重连的消费者"当成"新成员"
- 触发 rebalance（离开时一次 + 重连时一次）
- 滚动部署 N 个 pod = 2N 次 rebalance

**静态成员资格的解决方案**：

- 配置 `group.instance.id` 给消费者一个**稳定身份**
- Coordinator 在 `session.timeout.ms` 时间内保留该成员的分区分配
- 消费者在超时窗口内重连（如滚动部署场景），**直接拿回原来的分区，不触发 rebalance**

```
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "order-consumer-" + System.getenv("POD_NAME"));
```

> 💡 **工业界实测**：在 Kubernetes 滚动部署场景下，启用静态成员资格后，再均衡次数从"每次部署 N 次"降到"接近 0 次"。这是生产环境**性价比最高的一个配置**。

**代价**：静态成员的优雅关闭不会发送 LeaveGroup 请求，coordinator 会一直保留其分区直到 session 超时。所以需要把 `session.timeout.ms` 调小（如 10s），让真正的故障能被快速检测到。

### 洞见：消费者群组是第一性原理的"消费端投影"

回到"Kafka 是日志"这个原点：

|日志的特性|消费者端的对应机制|
|---|---|
|分区是并行单位|Consumer Group 内部分区分配|
|分区内有序|单分区单消费者|
|offset 是游标|消费者自主管理 offset|
|多组独立消费|多个 Consumer Group|
|日志可重放|`auto.offset.reset=earliest`|

**Consumer Group 不是 Kafka 的"功能"，而是"日志抽象"在读取端的必然推论。**​ 理解了这一点，后面所有配置项都会变得理所当然。

---

## 4.2 创建 Kafka 消费者

三个强制配置：

- `bootstrap.servers`：初始接触点
- `key.deserializer`：key 的反序列化器
- `value.deserializer`：value 的反序列化器

常见可选：`group.id`（消费者组标识）。

> ⚠️ **线程安全警告**：KafkaConsumer **不是线程安全的**。一个线程一个消费者，是铁律。多线程共享一个 consumer 会导致不可预期的行为。如果需要在一个应用里跑多个消费者，每个都必须跑在自己的线程里，推荐用 `ExecutorService` 管理。

**创建开销**：与 Producer 类似，KafkaConsumer 创建代价高（要建立连接、分配缓冲区、启动后台线程）。**一个线程只创建一个，长期复用**。

---

## 4.3 订阅主题

三种订阅方式：

1. **subscribe(Collection\<String\>)**：订阅固定 topic 列表
2. **subscribe(Pattern)**：正则匹配，动态订阅新创建的匹配 topic
3. **assign(Collection\<TopicPartition\>)**：手动分配分区（绕过 group 机制）

最常用的是第一种。订阅后，消费者会自动加入对应的 consumer group，coordinator 负责分配分区。

> 💡 `subscribe(Pattern)` 配合正则是非常强大的模式——新 topic 出现时自动纳入消费，无需改代码重启。但注意：这会让 coordinator 在正则匹配变化触发 rebalance 时，把"leader 重加入"作为触发条件之一。

---

## 4.4 轮询

`consumer.poll(timeout)` 是消费者 API 的心脏。它做四件事：

1. **发送心跳**：维持与 coordinator 的连接
2. **发起 fetch 请求**：从 assigned 分区的 leader 拉取数据
3. **返回批次**：把拉到的消息封装成 `ConsumerRecords` 返回
4. **触发必要的 rebalance**：如果发生了分区变更

```
while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
}
```

> ⚠️ **致命陷阱**：`poll()` 必须被**定期调用**。如果两次 `poll()` 之间的间隔超过 `max.poll.interval.ms`（默认 5 分钟），coordinator 会认为该消费者已死，将其踢出 group 并触发 rebalance。这意味着：**如果你的消息处理逻辑可能超过 5 分钟，要么调大 `max.poll.interval.ms`，要么减小 `max.poll.records`，要么把处理逻辑异步化**。

### 洞见：poll 循环是"消费者活性"的唯一证据

心跳（heartbeat）是在 poll 循环内部自动发送的——**它不是后台线程独立运行的**。所以：

- 如果你在 `poll()` 返回的 records 上做长时间同步处理，心跳就不会被发送
- Coordinator 收不到心跳 → 认为消费者死亡 → 触发 rebalance
- 这就是为什么"慢处理"和"rebalance 风暴"经常同时出现

**正确的处理模式**：

```
poll() → 拿到一批 records
       → 快速存入内存队列
       → 立即返回下一轮 poll()
后台线程池异步处理队列中的 records
```

这样 poll 循环始终轻快，心跳正常，rebalance 不会误触发。

---

## 4.5 配置消费者

这是全章最值钱的一节。我们按"四维博弈"重新组织：可靠性、吞吐、延迟、稳定性。

### 4.5.1 fetch.min.bytes

默认 1 字节。broker 接到 fetch 请求后，**至少要累积这么多字节才返回**（除非等到了 `fetch.max.wait.ms`）。

- **调大**（如 1MB）：提高吞吐，减少网络往返，但增加延迟
- **调小**：降低延迟，但吞吐下降

### 4.5.2 fetch.max.wait.ms

默认 500ms。broker 等待 `fetch.min.bytes` 累积的最长时间。

- 若 `fetch.min.bytes=1MB`，`fetch.max.wait.ms=500ms`：broker 要么攒够 1MB 立刻返回，要么等 500ms 后有多少返回多少
- **吞吐优先**：调大到 1-2s
- **延迟优先**：调小到 100ms

### 4.5.3 fetch.max.bytes

单次 fetch 请求返回的最大字节数。默认 50MB。客户端内存占用估算：`broker数 × fetch.max.bytes`。

### 4.5.4 max.poll.records

单次 `poll()` 返回的最大记录数。默认 500。

> ⚠️ **关键调优点**：如果你的单条消息处理耗时较长（如涉及 DB 写入、外部 API 调用），必须**调小这个值**，确保能在 `max.poll.interval.ms` 内处理完。工业界经验：处理单条消息 > 100ms 时，`max.poll.records` 设 50-100 更安全。

### 4.5.5 max.partition.fetch.bytes

每个分区单次 fetch 返回的最大字节数。默认 1MB。**必须大于 broker/topic 的 `max.message.bytes`**。

### 4.5.6 session.timeout.ms 和 heartbeat.interval.ms

**session.timeout.ms**（默认 45s）：coordinator 多久没收到心跳就判定消费者死亡。

**heartbeat.interval.ms**（默认 3s）：消费者发送心跳的频率。**建议设为 session.timeout.ms 的 1/3**。

```
session.timeout.ms = 30s
heartbeat.interval.ms = 10s   # 1/3 关系
```

> 💡 调优思路：
> 
> - 调小 session.timeout.ms（如 10-15s）：故障检测更快，但 GC 暂停或网络抖动可能导致误判
> - 调大：容忍短暂故障，但故障切换慢

### 4.5.7 max.poll.interval.ms

默认 5 分钟。两次 `poll()` 的最大间隔。

**这是最容易被误用的参数**：

- 如果你的处理逻辑可能超过 5 分钟 → 调大（如 10-30 分钟）
- 如果你希望快速检测"卡死"的消费者 → 调小（如 1-2 分钟）
- **工业界推荐**：根据业务处理最慢情况设定，典型值 1-5 分钟

### 4.5.8 default.api.timeout.ms

默认 60s。所有 consumer API 调用的超时时间。

### 4.5.9 request.timeout.ms

默认 30s。broker 响应请求的超时。

### 4.5.10 auto.offset.reset

当 Kafka 没有该 group 的已提交 offset 时（如新 group、offset 已过期），从哪里开始读：

- **latest**（默认）：从最新消息开始，跳过历史
- **earliest**：从最早消息开始，重放全部历史
- **none**：没有 offset 时直接抛异常

> 💡 工业界**几乎总是用 `earliest`**——因为 `latest` 会导致新 group 启动时丢掉大量历史消息，而 Kafka 的价值恰恰在于"重放"。唯一例外：纯实时监控场景，历史数据无意义，可以用 `latest`。

### 4.5.11 enable.auto.commit

默认 true。**这是生产环境的头号陷阱**。

自动提交每 `auto.commit.interval.ms`（默认 5s）提交一次 poll 回来的 offset，**与时间进度绑定，与业务处理进度无关**。

**两种灾难场景**：

1. **处理了消息但未到提交时间 → 消费者崩溃**​ → offset 未提交 → 重启后**重复消费**
2. **自动提交了 offset 但消息还在处理中 → 消费者崩溃**​ → 已提交的 offset 超前 → **消息丢失**

### 4.5.12 partition.assignment.strategy

分区分配策略：

- **RangeAssignor**（默认）：按 topic 范围分配，可能导致分区倾斜
- **RoundRobinAssignor**：轮询分配，更均衡
- **StickyAssignor**：尽量保持原有分配，减少 rebalance 时的分区移动
- **CooperativeStickyAssignor**：协作再均衡 + 粘性分配（**Kafka 2.4+ 推荐**）

```
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

### 4.5.13 client.id

同 Producer 端，broker 日志、监控、配额都靠它识别。生产环境**务必设置为有意义的业务标识**。

### 4.5.14 client.rack

Kafka 2.4+ 引入。**告诉 broker 消费者所在的机架/可用区**，broker 会优先从就近副本读取（follower fetch）。跨可用区部署时必配。

### 4.5.15 group.instance.id

静态成员资格的身份标识（见 4.1.3）。Kubernetes 环境中用 pod name 或 StatefulSet 序号：

```
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, 
    "consumer-" + System.getenv("POD_NAME"));
```

### 4.5.16 receive.buffer.bytes 和 send.buffer.bytes

TCP 收发缓冲区大小。默认 32KB。高带宽、高 RTT 网络（跨机房）建议调大。

### 4.5.17 offsets.retention.minutes

已提交 offset 的保留时间。默认 7 天。如果一个 group 停用超过 7 天，offset 会被清除，下次启动触发 `auto.offset.reset` 逻辑。

> ⚠️ 注意：这是 broker 端配置，不是 consumer 端。但直接影响"长期停用的 group 重启时从哪里开始"。

### 洞见：消费者配置的本质是"四维博弈"

把所有配置项按"可靠性、吞吐、延迟、稳定性"分类：

|维度|关键参数|调优方向|
|---|---|---|
|**可靠性**​|enable.auto.commit, isolation.level, max.poll.interval.ms|手动提交、read_committed、合理超时|
|**吞吐**​|fetch.min.bytes, fetch.max.wait.ms, max.poll.records|大批次、长等待|
|**延迟**​|fetch.min.bytes, fetch.max.wait.ms|小批次、短等待|
|**稳定性**​|session.timeout.ms, heartbeat.interval.ms, group.instance.id|静态成员 + 合理心跳|

**工业界生产基线配置**：

```
// 可靠性优先
props.put("enable.auto.commit", "false");
props.put("auto.offset.reset", "earliest");
props.put("max.poll.records", "100");
props.put("max.poll.interval.ms", "300000");  // 5分钟

// 稳定性优先
props.put("session.timeout.ms", "30000");
props.put("heartbeat.interval.ms", "10000");
props.put("group.instance.id", "consumer-" + podName);

// 再均衡优化
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

// 跨 AZ 优化
props.put("client.rack", availabilityZone);
```

---

## 4.6 提交和偏移量

这是**全章的灵魂**。Offset 是消费者在分区日志里的"书签"——它记录了"这个 group 在这个分区上已经处理到哪了"。

Offset 提交的本质：**消费者向 Kafka 发送消息，由一个特殊的内部 topic `__consumer_offsets` 记录每个分区的已提交 offset**。

### 4.6.1 自动提交

```
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");
```

**工作机制**：每隔 5 秒，consumer 自动提交"上一次 poll 回来的一批消息中，最大的 offset"。

**致命缺陷**：

```
场景 A：处理了消息，但还没到 5 秒自动提交 → 消费者崩溃
结果：offset 未提交 → 重启后从上次提交的 offset 开始 → 重复消费

场景 B：自动提交了 offset，但消息还在处理中 → 消费者崩溃  
结果：offset 已超前 → 已提交但未处理的消息 → 永久丢失
```

**原书图 4-8 的经典图示**：

```
已提交 offset ← 这里
    ↓
正在处理的消息 ★ ← 这里
    ↓
上次 poll 返回的消息
    ↓
如果发生 rebalance，新消费者从上次已提交 offset 开始 → 中间的消息被重复处理
```

> ⚠️ **结论**：自动提交**只在"偶尔重复或丢失可接受"的场景下可用**（如日志收集、指标上报）。任何严肃的业务处理都必须**关闭自动提交，改用手动提交**。

### 4.6.2 提交当前偏移量

```
props.put("enable.auto.commit", "false");

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);  // 业务处理
    }
    consumer.commitSync();  // 同步提交
}
```

**commitSync() 的语义**：阻塞等待 broker 确认提交成功。

**优点**：简单、可靠，保证"at-least-once"语义（处理成功 → 提交 → 绝不丢失；崩溃未提交 → 重启后重复处理）。

**缺点**：每次提交都阻塞，吞吐受影响。

### 4.6.3 异步提交

```
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        // 处理提交失败
        log.error("Commit failed for offsets {}", offsets, exception);
    }
});
```

**commitAsync() 的语义**：不阻塞，立即返回，结果通过 Callback 通知。

**优点**：吞吐高，不阻塞 poll 循环。

**缺点**：

- 提交失败不会自动重试（因为可能已有更新的提交在路上）
- 如果异步提交失败且发生 rebalance，可能产生更多重复

### 4.6.4 同步和异步组合提交

工业界**推荐模式**：

```
try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            process(record);
        }
        // 日常用异步，保证吞吐
        consumer.commitAsync();
    }
} catch (Exception e) {
    log.error("Processing error", e);
} finally {
    try {
        // 关闭前用同步，确保最后提交成功
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```

**这是 Red Hat 官方推荐的模式**：日常异步提交保吞吐，关闭/rebalance 前同步提交保可靠。

### 4.6.5 提交特定的偏移量

更精细的控制——只提交处理成功的那部分 offset：

```
Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
        // 记录每个分区的最新 offset
        currentOffsets.put(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)  // +1 表示下一条要读的
        );
    }
    // 批量提交所有分区的进度
    consumer.commitAsync(currentOffsets, null);
}
```

**关键细节**：提交的 offset 应该是 `record.offset() + 1`，表示"下一条要读取的位置"。

### 洞见：offset 提交定义了"消费语义"

|提交策略|语义|重复/丢失风险|
|---|---|---|
|处理后提交（commitSync 在 process 之后）|**At-Least-Once**​|可能重复，绝不丢失|
|处理前提交（commitSync 在 process 之前）|**At-Most-Once**​|可能丢失，绝不重复|
|事务性提交（配合 Producer 事务）|**Exactly-Once**​|既不重复也不丢失|

**第一性原理**：Kafka 本身只提供"at-least-once"语义——消息不丢，但可能重复。要实现"exactly-once"，必须：

1. **Producer 端**：开启幂等或事务
2. **Consumer 端**：`enable.auto.commit=false` + 手动提交 + 业务处理幂等
3. **端到端**：Consumer 的业务处理逻辑必须能容忍重复（按唯一键去重、用数据库唯一索引等）

> 💡 **工业界事实**：99% 的 Kafka 消费者用的是"at-least-once + 业务幂等"模式。真正的 exactly-once 只在金融交易等极端场景用事务实现，代价是显著的延迟和复杂度。

---

## 4.7 再均衡监听器

`ConsumerRebalanceListener` 是处理 rebalance 的钩子接口：

```
consumer.subscribe(Arrays.asList("orders"), new ConsumerRebalanceListener() {
    
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 分区被回收前调用
        log.info("Partitions revoked: {}", partitions);
        // 关键：在失去分区前提交 offset
        consumer.commitSync(currentOffsets);
        // 清理这些分区的本地状态
        partitions.forEach(p -> localState.remove(p));
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 获得新分区后调用
        log.info("Partitions assigned: {}", partitions);
        // 可以从特定 offset 开始（如从数据库恢复状态）
        partitions.forEach(p -> {
            Long savedOffset = stateStore.getOffset(p);
            if (savedOffset != null) {
                consumer.seek(p, savedOffset);
            }
        });
    }
});
```

**为什么必须实现 rebalance listener？**

原书图 4-8 揭示的真相：如果 rebalance 发生时没有提交 offset，**已处理但未提交的消息会被重复消费**。在 `onPartitionsRevoked` 里调用 `commitSync()` 是避免重复消费的**最后防线**。

> ⚠️ 特别是在 **eager rebalance**​ 模式下，rebalance 发生时所有分区都被回收，如果没有在 `onPartitionsRevoked` 中提交 offset，所有已处理但未提交的进度都会丢失，导致大规模重复消费。

**Cooperative rebalance 的优势**：只有被回收的那部分分区需要提交，其他分区继续消费，所以 `onPartitionsRevoked` 的提交压力小得多。

---

## 4.8 从特定偏移量位置读取记录

有时候你需要绕过 group 的 offset 管理，直接从指定位置开始读：

```
// 1. 手动分配分区（绕过 group）
TopicPartition partition = new TopicPartition("orders", 0);
consumer.assign(Arrays.asList(partition));

// 2. 定位到特定 offset
consumer.seek(partition, 12345L);  // 从 offset 12345 开始
consumer.seekToBeginning(Arrays.asList(partition));  // 从最早开始
consumer.seekToEnd(Arrays.asList(partition));  // 从最新开始

// 3. 根据时间戳定位
Map<TopicPartition, Long> timestamps = new HashMap<>();
timestamps.put(partition, System.currentTimeMillis() - 3600_000);  // 1小时前
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer.offsetsForTimes(timestamps);
offsets.forEach((tp, offsetAndTime) -> {
    consumer.seek(tp, offsetAndTime.offset());
});
```

**典型用途**：

- 数据补录：从指定时间点重新处理
- 消费者状态恢复：从外部存储（如数据库）恢复 offset
- 数据校验：重新消费历史数据做对账

---

## 4.9 如何退出

优雅关闭消费者：

```
// 1. 设置一个 volatile 标志
private volatile boolean running = true;

// 2. 在另一个线程（如 shutdown hook）中设置 running = false

// 3. poll 循环检查标志
while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    // 处理 records
}

// 4. 关闭前最后的同步提交
try {
    consumer.commitSync();
} finally {
    consumer.close();  // 这会触发 LeaveGroup，coordinator 回收分区
}
```

> 💡 关键点：`consumer.close()` 会发送 LeaveGroup 请求，coordinator 立即回收该消费者的分区并触发 rebalance。如果不调用 close() 直接 kill 进程，coordinator 要等 `session.timeout.ms` 才会判定死亡——这段时间内分区无人消费。

**静态成员资格的微妙之处**：配置了 `group.instance.id` 的消费者，**默认不会在 close() 时发送 LeaveGroup**——它希望保留分区分配以便快速重连。所以如果你确实想"永久退出"（如缩容），需要显式调用 `consumer.close()` 并确保 coordinator 在超时后回收，或者临时去掉 `group.instance.id`。

---

## 4.10 反序列化器

这是中文译本最薄弱的一节。我们用 Confluent 官方资料补齐。

### 4.10.1 自定义反序列化器

实现 `Deserializer<T>` 接口。但工业界**强烈不推荐手写**——难以处理 Schema 演进。

### 4.10.2 在消费者里使用 Avro 反序列化器

这是工业界的事实标准。消息的线格式是：

```
┌──────────┬────────────┬──────────────────────┐
│ Magic Byte│ Schema ID  │   Avro 编码的载荷     │
│  (1 byte) │ (4 bytes)  │                      │
└──────────┴────────────┴──────────────────────┘
   0x00        全局唯一ID      紧凑二进制数据
```

**消费者配置**：

```
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
    "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
    "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://schema-registry:8081");
props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);
```

**KafkaAvroDeserializer 的工作流**：

1. 从消息头部读取 5 字节：magic byte + schema id
2. 用 schema id 向 Schema Registry 查询对应的 Avro Schema
3. 用该 Schema 反序列化后续的二进制载荷
4. 返回 `SpecificRecord`（如果 `SPECIFIC_AVRO_READER_CONFIG=true`）或 `GenericRecord`

**两种返回类型**：

- **SpecificRecord**：返回代码生成的 Java 类（类型安全，推荐）
- **GenericRecord**：返回通用的 `GenericRecord`（无类型安全，但灵活，适合多 Schema 共存场景）

**Schema 演进的威力**：

由于 Schema 存储在 Registry 中且消息只包含 schema id，**消费者可以独立于生产者演进**：

- 生产者升级 Schema（如新增可选字段）→ 注册新版本 → 获得新 schema id
- 老消费者用老 schema id 反序列化老消息 → 正常工作
- 新消费者用新 schema id 反序列化新消息 → 正常工作
- 新老消息在同一个 topic 中共存 → 完全兼容

**兼容性模式**（Registry 级别配置）：

- **BACKWARD**（默认）：新 schema 可以读旧数据。消费者先升级
- **FORWARD**：旧 schema 可以读新数据。生产者先升级
- **FULL**：双向兼容

> 📌 **工业界最佳实践**：使用 BACKWARD 兼容 + 新增字段带默认值 + 永不删除/修改现有字段类型。这样消费者可以先升级，生产者后升级，实现零停机 Schema 演进。

---

## 4.11 独立的消费者：为什么以及怎样使用不属于任何群组的消费者

有些场景**故意不用 consumer group**：

```
// 手动分配分区，不指定 group.id
consumer.assign(Arrays.asList(
    new TopicPartition("orders", 0),
    new TopicPartition("orders", 1)
));
consumer.seek(new TopicPartition("orders", 0), 12345L);
```

**典型用例**：

1. **数据补录/重放**：从指定 offset 精确读取，不走 group 的 offset 管理
2. **数据校验**：独立消费者扫描 topic 全量数据做对账
3. **监控探针**：只读最新消息检查管道健康度
4. **流处理状态恢复**：Kafka Streams 内部用独立消费者恢复状态

> 💡 **关键区别**：独立消费者**不需要 coordinator**，不触发 rebalance，offset 也不提交到 `__consumer_offsets`。它完全自主管理读取位置。

---

## 4.12 小结

把全章串起来，Kafka Consumer 的设计哲学是：

> **把"读取日志"这件事的所有权衡，都交给客户端来决定。**

Broker 端只负责一件事：按 offset 顺序返回消息。而 Consumer 端要决定：

1. **和谁一起读**（group.id）—— 决定消费并行度
2. **何时让出分区**（rebalance protocol）—— 决定稳定性
3. **读到哪了**（offset commit）—— 决定交付语义
4. **如何反序列化**（deserializer + Schema Registry）—— 决定 Schema 治理能力
5. **如何高效拉取**（fetch.* 参数）—— 决定吞吐与延迟

### 工业界生产基线配置

```
Properties props = new Properties();

// 基础配置
props.put("bootstrap.servers", "broker1:9092,broker2:9092,broker3:9092");
props.put("group.id", "order-processor");
props.put("client.id", "order-processor-" + podName);
props.put("client.rack", availabilityZone);

// 反序列化（Avro + Schema Registry）
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://schema-registry:8081");
props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);

// 可靠性：手动提交
props.put("enable.auto.commit", "false");
props.put("auto.offset.reset", "earliest");

// 稳定性：静态成员 + 协作再均衡
props.put("group.instance.id", "order-consumer-" + podName);
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
props.put("session.timeout.ms", "30000");
props.put("heartbeat.interval.ms", "10000");
props.put("max.poll.interval.ms", "300000");

// 吞吐优化
props.put("fetch.min.bytes", "1048576");  // 1MB
props.put("fetch.max.wait.ms", "500");
props.put("max.poll.records", "100");
props.put("max.partition.fetch.bytes", "1048576");

// 事务支持（如需 exactly-once）
props.put("isolation.level", "read_committed");
```

### 给读者的"消费者调优决策树"

```
遇到什么问题？
├─ 重复消费严重
│  ├─ 关闭 enable.auto.commit
│  ├─ 处理后 commitSync
│  └─ 实现 ConsumerRebalanceListener 在 onPartitionsRevoked 中提交
├─ Rebalance 风暴
│  ├─ 配置 group.instance.id（静态成员）
│  ├─ 使用 CooperativeStickyAssignor
│  └─ 调优 session.timeout.ms / heartbeat.interval.ms
├─ 消费滞后（lag 增长）
│  ├─ 增加分区数
│  ├─ 增加消费者实例数（≤ 分区数）
│  ├─ 调大 fetch.min.bytes + max.poll.records
│  └─ 优化业务逻辑处理速度
├─ 消费者被踢出 group
│  ├─ 调大 max.poll.interval.ms
│  ├─ 减小 max.poll.records
│  └─ 异步化处理逻辑，让 poll 循环轻快
└─ 消息丢失
   ├─ 设 auto.offset.reset=earliest
   ├─ 业务处理后提交 offset
   └─ 用事务 + isolation.level=read_committed
```

### 洞见：消费者是"日志游标"的分布式管理者

回到第一性原理——Kafka 是日志。那么"读取日志"意味着什么？

**单进程读日志**：维护一个文件指针（offset），读一条进一步。

**分布式读日志**：

- 日志被切成多个分区 → 需要多个游标
- 多个消费者协同 → 游标所有权需要动态分配
- 消费者可能崩溃 → 游标状态需要持久化（这就是 `__consumer_offsets` topic 的作用）
- 游标推进需要与业务处理进度一致 → 这就是"offset 提交时机"决定的交付语义

**所以 Consumer Group 不是 Kafka 的"功能"，而是"分布式日志读取"这一第一性原理的必然推论。**​ 所有配置项、所有 API、所有 rebalance 协议，都是在为"如何在分布式环境下正确、高效、稳定地推进日志游标"这个问题服务。

> 📌 **全章最核心的一句话**：Kafka Consumer 不是一个"拉取消息的客户端"，而是一个"在可靠性、吞吐、稳定性三维空间中管理日志游标的分布式协程"。理解这一点，所有消费者配置都会变得理所当然；反之，如果只把消费者看作"消息接收器"，就会陷入"为什么又 rebalance 了"、"为什么又重复消费了"的无尽 debugging 中。

