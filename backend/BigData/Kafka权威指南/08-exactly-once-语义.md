
# 第8章 精确一次性语义 · 读书笔记

> 💡 这一章表面是"如何让 Kafka 做到 exactly-once"，实质是在回答：**在分布式日志这个"追加即真理"的抽象之上，能不能构筑一层比 at-least-once 更强的保证？**​ 答案是"能，但有边界"——Kafka 的 EOS（Exactly-Once Semantics）分为两层：幂等生产者解决"单分区重试产生的重复"，事务解决"跨分区/跨主题的原子写入 + 消费位移的原子提交"。但 Kafka 的 EOS **只在 Kafka 内部有效**，一旦涉及数据库、REST API 等外部系统，保证就破了。这一章的价值，就是帮你理清"EOS 能覆盖到哪里、不能覆盖到哪里、什么时候值得付出那个性能代价"。

---

## 8.1 幂等生产者

### 8.1.1 幂等生产者的工作原理

幂等生产者的核心是**PID（Producer ID）+ 序列号（sequence number）**这对组合：

- 每个新生产者实例初始化时被分配一个集群内唯一的 PID（对用户透明）
- 对于每个 topic-partition，序列号从 0 单调递增
- 生产者每发送一条消息，序列号 +1
- Broker 在内存中维护每个 PID/partition 收到的最后序列号
- 收到 produce 请求时做校验：
    - **序列号正好大 1**​ → 正常写入
    - **序列号小于等于上次**​ → 重复，broker 返回成功（幂等生效）
    - **序列号大于上次 +1**​ → 乱序错误（OutOfOrderSequenceException），致命

这套机制保证了：**即使生产者因网络抖动而重试，每条消息在日志中也只持久化一次**。

Kafka 3.0 起，`enable.idempotence` 默认为 true；启用后 `retries` 自动设为 `Integer.MAX_VALUE`，`acks` 自动设为 `all`。也就是说，**现代 Kafka 的幂等性是"开箱半默认"的**——你只要不显式关掉它，单分区重试去重就一直在工作。

### 8.1.2 幂等生产者的局限性

这是中文版翻译讲得含糊、但工业界极其关键的一点：

> ⚠️ **PID 不跨会话生存**。生产者进程重启后，会获得一个新的 PID。

这意味着幂等性的保证范围被严格限定在：

```
✅ 同一生产者实例 + 同一会话 + 同一 topic-partition
❌ 跨生产者重启
❌ 跨 topic-partition（单条消息级别，不跨分区）
❌ 跨系统（如数据库写入）
```

更微妙的边界：

- **应用层重发破坏幂等**：如果你在应用代码里 catch 异常后手动重发，broker 无法去重——因为两条消息会带不同的序列号。Kafka 文档明确警告："it is imperative to avoid application level re-sends since these cannot be de-duplicated"
- **缓冲区过期**：如果消息在缓冲区中超时过期（delivery.timeout.ms），即使无限重试也无法发送，此时需要关闭生产者并检查最后一条消息是否重复

### 8.1.3 如何使用幂等生产者

```
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092,broker3:9092");
props.put("enable.idempotence", "true");  // Kafka 3.0+ 默认开启
props.put("acks", "all");                  // 幂等要求 acks=all
props.put("retries", Integer.MAX_VALUE);   // 默认已是这个值
// 注意：max.in.flight.requests.per.connection 必须 <= 5 以保证有序性

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
// 无需 initTransactions()，无需 transactional.id
// 直接 send() 即可享受幂等保护
```

### 洞见：幂等是第一性原理的"最小契约"

回到分布式系统的第一性原理——**"至少一次"是容易的，"至多一次"也是容易的，但"精确一次"需要在两者间架桥**。

幂等生产者架的这座桥非常精巧：**它不改变 Kafka 的"追加日志"本质，只是在追加协议上加了一个去重维度**。PID 标识"谁在写"，序列号标识"写了第几条"。broker 用这两个维度构成唯一键，重复的键直接丢弃。

这是工程上的极致优雅——**没有引入两阶段提交，没有引入分布式事务协调器，仅靠"唯一键 + 去重"就消灭了重试产生的重复**。代价仅仅是 broker 端为每个活跃 PID/partition 维护一个序列号计数器，内存开销极小。

但正是这种"轻量"决定了它的边界：**它只能解决"同一根 TCP 连接、同一个生产者实例内的重试去重"**。跨实例、跨会话、跨系统，它就无能为力了——这些场景需要下一节的"事务"。

---

## 8.2 事务

### 8.2.1 事务的应用场景

事务要解决的不是"单条消息重复"，而是**"一组操作要么全成功、要么全失败"**的原子性问题。

最经典的例子——**转账**：

```
Alice 账户 -100 元
Bob 账户 +100 元
```

如果没有事务：

1. 消费者从 transfers topic 读到转账事件
2. 向 balances topic 发送 Alice 的扣款事件
3. **应用崩溃**
4. 新实例从最后提交的 offset 重新开始
5. Alice 被扣款两次，Bob 只收到一次

这就是"读-处理-写"管道中的典型原子性缺失。事务要解决的就是这个。

### 8.2.2 事务可以解决哪些问题

根据 KIP-98 的官方定义，事务提供以下保证：

1. **跨多 topic-partition 的原子写入**：一个事务内 produce 到多个分区的消息，要么全部可见，要么全部不可见
2. **消费位移与产出消息的原子绑定**：通过 `sendOffsetsToTransaction()` 将消费者的 offset 提交也纳入事务——这意味着"输出消息"和"输入 offset"要么一起生效，要么一起回滚
3. **跨会话的事务恢复**：通过 TransactionalId 识别生产者身份，旧实例未完成的事务由新实例继续完成（提交或中止）

### 8.2.3 事务是如何保证精确一次性的

这是事务机制的精华。官方设计文档定义了几个新概念：

|概念|作用|
|---|---|
|**Transaction Coordinator**​|每个事务生产者被分配一个事务协调器（类似消费者组协调器），负责管理事务生命周期|
|**Transaction Log（`__transaction_state`）**​|内部的 compacted topic，持久化记录每个事务的状态转换|
|**Control Messages**​|写入用户 topic 的特殊消息（COMMIT/ABORT 标记），对用户透明|
|**TransactionalId**​|用户提供的稳定 ID，跨会话标识同一个生产者逻辑实例|
|**Producer Epoch**​|每次新实例用相同 TransactionalId 初始化时递增，用于 fencing 掉僵尸实例|

**两阶段提交流程**：

```
1. 生产者 → 协调器：initTransactions()（携带 TransactionalId）
   协调器：分配 PID，递增 epoch（fencing 旧实例），恢复/中止未完成事务
   
2. 生产者 → 协调器：beginTransaction()
   协调器：在 __transaction_state 中记录事务开始
   
3. 生产者 → 各分区 leader：发送消息（对用户不可见，对 read_committed 消费者阻塞）
   
4. 生产者 → 协调器：AddPartitionsToTxn（声明涉及哪些分区）
   
5. 生产者 → 协调器：sendOffsetsToTransaction()（绑定消费位移）
   
6. 生产者 → 协调器：commitTransaction()
   协调器：
   a. 在 __transaction_state 写入 COMMIT 决策（point of no return）
   b. 向所有涉及的分区写入 COMMIT 控制消息
   c. 提交消费位移
   
7. 分区 leader 上的消息对 read_committed 消费者变为可见
```

关键点：**"PrepareCommit"写入 `__transaction_state` 那一刻是不可逆的点**——此后即使协调器崩溃，新协调器也能从事务日志中看到决策并完成提交。

### 8.2.4 事务不能解决哪些问题

这是全章最重要的"反常识"部分，也是工业界踩坑的重灾区：

> ⚠️ **Kafka 事务的 EOS 保证仅限于 Kafka 内部**

具体来说：

**1. 外部系统写入不在事务边界内**

```
producer.beginTransaction();
producer.send(new ProducerRecord("orders", key, order));
httpClient.post("https://payments.example.com/charge", order);  // 外部调用
producer.commitTransaction();  // 如果 HTTP 成功但 commit 失败：客户被扣款，Kafka 无记录
```

HTTP 调用、数据库写入、Redis 操作——**这些都不受 Kafka 事务保护**。

**2. 消费者侧保证更弱**

KIP-98 明确指出：

- compacted topic 可能覆盖事务中的部分消息
- 事务可能跨越段文件，旧段被删除时会丢失事务头部
- 消费者可能 seek 到事务中间任意位置
- 消费者可能不消费事务涉及的所有分区

因此，**不能保证一个已提交事务的所有消息被某个消费者完整消费**。

**3. 跨集群不保证原子性**

> Ⓘ 一个事务由一个集群中的一个协调器协调。MirrorMaker2 等跨集群复制工具复制的是消息本身，**不保留事务原子性**。跨集群交付应视为 at-least-once，消费端必须做幂等。

**4. read_committed 消费者的阻塞代价**

`read_committed` 消费者受 **LSO（Last Stable Offset）**​ 限制——它不能读到 LSO 之后的消息。如果一个事务长时间未提交（默认 `transaction.timeout.ms=60s`），**所有 read_committed 消费者在该分区上都会被阻塞长达 60 秒**。

如果一个分区上有多个生产者的事务交错，**一个卡住的生产者会阻塞所有消费者**——这是多小时级故障的真实案例。

### 8.2.5 如何使用事务

**生产者配置**：

```
Properties props = new Properties();
props.put("bootstrap.servers", "brokers:9092");
props.put("transactional.id", "order-processor-1");  // 必须，集群内唯一
props.put("enable.idempotence", "true");  // 设置 transactional.id 后自动开启
props.put("acks", "all");                 // 事务强制要求 acks=all
// retries 自动为 Integer.MAX_VALUE
// max.in.flight.requests.per.connection 自动为 5

KafkaProducer<String, String> producer = new KafkaProducer<>(props, 
    new StringSerializer(), new StringSerializer());
producer.initTransactions();  // 必须调用一次
```

**消费者的 "读-处理-写" 事务模式**：

```
// 消费者配置
Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "brokers:9092");
consumerProps.put("group.id", "order-processor-group");
consumerProps.put("isolation.level", "read_committed");  // 关键！
consumerProps.put("enable.auto.commit", "false");        // 关键！

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
consumer.subscribe(Collections.singleton("incoming-orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    producer.beginTransaction();
    try {
        for (ConsumerRecord<String, String> record : records) {
            Order order = parse(record.value());
            OrderResult result = process(order);
            producer.send(new ProducerRecord<>("order-results", record.key(), result.toJson()));
        }
        
        // 关键：将消费位移提交纳入事务
        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        for (ConsumerRecord<String, String> record : records) {
            offsets.put(
                new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1)
            );
        }
        producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
        
        producer.commitTransaction();
    } catch (Exception e) {
        producer.abortTransaction();
        // 事务中止后，消费者会从最后提交的位移重新消费
        // 因此消费端处理逻辑必须是幂等的
    }
}
```

**关键配置说明**：

- `transactional.id` 必须在集群内唯一，且**跨生产者重启保持稳定**
- 每个消费者实例配一个生产者实例（1:1 映射），这是官方推荐模式
- 事务主题（参与事务的 topic）必须满足：`replication.factor >= 3`，`min.insync.replicas >= 2`
- `transaction.timeout.ms` 生产环境建议 ≤ 15 秒，避免阻塞消费者太久

### 8.2.6 事务 ID 和隔离

**TransactionalId 的本质**：

TransactionalId 是用户提供的、跨会话稳定的标识符。它与内部 PID 的关系是 1:1 映射，但区别在于：**PID 由 broker 分配且不跨会话，TransactionalId 由用户配置且跨会话持久**。

它的两个核心作用：

1. **​ fencing（ fencing off old generations）**：当新实例用相同 TransactionalId 调用 `initTransactions()` 时，PID 的 epoch 递增，旧实例的所有后续操作都会被拒绝（抛出 `ProducerFencedException`）
2. **事务恢复**：新实例可以安全地接续/中止旧实例遗留的未完成事务

**隔离级别（isolation.level）**：

|隔离级别|行为|
|---|---|
|`read_uncommitted`（默认）|读取所有消息，包括未提交/已中止事务的消息|
|`read_committed`|只读取非事务消息和已提交事务的消息；遇到未提交事务时在 LSO 处等待|

> 💡 **关键警示**：如果消费者没有设置 `isolation.level=read_committed`，那么即使生产者用了事务，消费者仍然会看到未提交/已中止的消息。这是工业界最常见的"EOS 失效"配置错误。

**LSO（Last Stable Offset）机制**：

```
分区日志：
Offset 10: 非事务消息   ← read_committed 可读
Offset 11: TX-1 消息    ← read_committed 可读（TX-1 已提交）
Offset 12: TX-1 消息    ← read_committed 可读
Offset 13: TX-2 消息    ← TX-2 未提交，LSO=13，read_committed 在此阻塞
Offset 14: 后续消息
```

read_committed 消费者只能读到 LSO 之前的消息。LSO 是"未提交事务的最小 offset"。

### 8.2.7 事务的工作原理

把 8.2.3 的流程再深化一层，看几个工程细节：

**1. 僵尸 fencing 机制**

```
场景：实例1 拿到 transactional.id=X，开始事务
实例1 进入长时间 GC 停顿
Kafka 触发 rebalance，实例2 接管分区
实例2 用相同 transactional.id=X 调用 initTransactions()
  → 协调器递增 epoch
  → 实例1 变为僵尸（zombie）
实例2 开始新事务，正常提交
实例1 GC 恢复，尝试继续旧事务
  → broker 检查 epoch，拒绝（stale epoch）
  → 实例1 收到 ProducerFencedException，关闭
结果：A, B（正确，无重复）
```

**2. 事务标记的异步写入**

`commitTransaction()` 调用返回后，COMMIT 控制消息的写入是异步进行的。这意味着事务提交决策（写入 `__transaction_state`）与标记广播（写入各分区）是两个阶段——前者是 point of no return，后者可能重试。

**3. 事务的常量开销特性**

- **每个事务的额外开销是恒定的**，与消息数量无关——包括：若干次协调器往返 + 每个参与分区一个控制消息
- 因此**"事务越大（消息越多），摊销成本越低"**
- 但代价是：**事务越长（时间越久），read_committed 消费者的端到端延迟越高**

这构成了核心调优矛盾：

```
大事务 → 高吞吐、低开销，但高延迟
小事务 → 低延迟，但高开销、低吞吐
```

### 洞见：事务是"日志抽象的原子性扩展"

回到第一性原理——Kafka 的日志是"只追加、按 offset 寻址"的。事务并没有改变这个本质，而是在日志协议之上叠加了一层"原子性边界"：

- **写入阶段**：消息正常追加到各分区，但对 read_committed 消费者不可见
- **提交阶段**：协调器在 `__transaction_state` 写入决策，然后向各分区追加 COMMIT 控制消息
- **可见性阶段**：read_committed 消费者看到控制消息后，将 LSO 推过这些消息

**这是一次极致的"日志复用"**——事务状态本身也存在 Kafka 内部 topic 里，事务标记也是普通的日志追加（只是带特殊标记）。整个事务机制没有引入任何"外部"组件，完全运行在 Kafka 自身的日志抽象之上。

这也解释了为什么 KRaft 模式下事务性能更好：事务协调器不再依赖 ZooKeeper，而是通过 Raft 协议管理 `__transaction_state`，故障转移从秒级降到毫秒级，提交延迟降低 5-10%。

> 📌 **工业界铁律**：TransactionalId 必须从应用的"分片标识符"派生（如 `order-processor-{shardId}`），且**每个逻辑生产者实例全局唯一**。如果用主机名或 pod 名派生，滚动重启时新实例获得新 ID，旧实例的僵尸事务会一直阻塞 LSO 直到超时——这是生产环境经典的"幽灵阻塞"故障。

---

## 8.3 事务的性能

这是工业界决策 EOS 与否的关键依据。综合多个基准测试：

|模式|吞吐量|p50 延迟|p99 延迟|
|---|---|---|---|
|At-least-once (acks=all)|100K msg/s|15ms|-|
|幂等生产者|98K msg/s|16ms|-|
|**事务**​|**60K msg/s**​|**45ms**​|**+30ms**​|

**开销来源**：

1. **额外 RPC**：每次事务提交需要多次与事务协调器交互
2. **控制消息写入**：每个参与分区都需要写入 COMMIT/ABORT 标记
3. **消费者缓冲**：read_committed 消费者必须缓冲消息直到看到事务标记
4. **LSO 推进延迟**：read_committed 消费者的端到端延迟至少为"提交间隔 + 协调器往返 + 标记写入 + fetch 延迟"

**Conduktor 的真实案例**：

> "我们为了'安全'启用了事务。吞吐量从 100K 降到 60K msg/s。后来意识到我们的消费者反正要写 PostgreSQL——EOS 帮不上忙。切回幂等消费者后，性能回来了。"

**MinervaDB 的工程建议**：

```
三条规则控制事务开销：
1. 事务时间短（time-wise）但宽（record-wise）—— 即时间跨度短，但批量大
2. 每个事务涉及的分区数尽量少 —— 因为每个分区都需要写控制消息
3. 事务打开期间绝不等待外部系统 —— 否则会阻塞所有 read_committed 消费者
```

### 洞见：EOS 是"正确性工具"，不是"默认选项"

综合工业界实践，EOS 值得启用的条件是**同时满足**：

```
✅ 纯 Kafka-to-Kafka 处理（无外部系统写入）
✅ 需要跨分区/跨主题的原子写入
✅ 重复消息会破坏业务逻辑（且无法用幂等消费解决）
✅ 延迟预算允许承受 10-50ms 的提交开销
```

**典型合法场景**：

- Kafka Streams 有状态处理（`processing.guarantee=exactly_once_v2`）
- 金融领域的"读-变换-写"管道（如风控规则引擎）
- 事件溯源中"一个业务事件必须原子地反映到多个投影"

**不应该用 EOS 的场景**：

```
❌ 高吞吐遥测/点击流/metrics（重复统计上无关紧要）
❌ 消费者最终写入数据库/REST/Redis（EOS 帮不上忙）
❌ 单 topic 单分区写入（幂等生产者足矣）
```

对于"消费者写数据库"这种最常见场景，**正确的模式是**：

1. **At-least-once + 幂等消费者**：用数据库唯一键 / UPSERT / 去重表处理重复
2. **Transactional Outbox**：数据库写入和 Kafka 消息在同一个本地事务中写入 outbox 表，由 Debezium CDC 轮询发布到 Kafka

> 💡 **核心洞察**：对于写外部系统的场景，Kafka EOS 是"错误层的正确性"——它解决了 Kafka 内部的原子性，但没解决"Kafka 与外部系统之间的原子性"。后者需要 outbox 模式或消费端幂等，Kafka EOS 对此无能为力。

---

## 8.4 小结

把全章串起来，Kafka 的 EOS 是一个**分层的保证体系**：

```
┌─────────────────────────────────────────────────────────┐
│ 第一层：幂等生产者                                        │
│  PID + 序列号 → 单分区内重试去重                          │
│  范围：单生产者会话内、单 topic-partition                 │
│  开销：+2~5% 吞吐量                                      │
├─────────────────────────────────────────────────────────┤
│ 第二层：事务                                              │
│  TransactionalId + 事务协调器 + 两阶段提交                │
│  范围：跨多分区/跨主题的原子写入 + 消费位移原子提交         │
│  开销：-30~40% 吞吐量，+30ms p99 延迟                     │
│  边界：仅在 Kafka 内部有效                                │
├─────────────────────────────────────────────────────────┤
│ 第三层：端到端 EOS（应用层）                               │
│  Kafka EOS + 幂等消费 / Transactional Outbox              │
│  范围：Kafka → 外部系统                                  │
│  实现：消费端幂等 + 去重键 / Outbox 模式                   │
└─────────────────────────────────────────────────────────┘
```

### 工业界决策框架

```
你的场景是 Kafka-to-Kafka 的纯流处理吗？
├─ 否 → 用 at-least-once + 幂等消费（数据库唯一键/UPSERT）
└─ 是 → 需要跨分区原子性吗？
       ├─ 否 → 幂等生产者足矣
       └─ 是 → 延迟预算允许 10-50ms 开销吗？
              ├─ 否 → 重新评估业务需求
              └─ 是 → 启用事务，并严格遵守：
                     • transactional.id 全局唯一且稳定
                     • isolation.level=read_committed
                     • enable.auto.commit=false
                     • transaction.timeout.ms ≤ 15s
                     • 每个消费实例配一个生产实例
                     • 事务主题 RF≥3, min.insync.replicas≥2
```

### 生产环境配置基线

```
// 幂等生产者（Kafka 3.0+ 默认开启，显式写出以便清晰）
props.put("enable.idempotence", "true");
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
props.put("max.in.flight.requests.per.connection", "5");

// 事务生产者（如启用）
props.put("transactional.id", "order-processor-" + shardId);  // 全局唯一且稳定
props.put("transaction.timeout.ms", "15000");  // ≤ 15s

// 事务消费者
consumerProps.put("isolation.level", "read_committed");
consumerProps.put("enable.auto.commit", "false");

// 事务主题（topic 级别）
configs.put("min.insync.replicas", "2");
// replication.factor=3（创建 topic 时指定）
```

### 给读者的"EOS 心智模型"

> 📌 **全章最核心的一句话**：Kafka 的精确一次性不是"跨系统的分布式事务"，而是"Kafka 内部日志抽象的原子性扩展"——幂等生产者解决了"单分区重试去重"（PID+序列号），事务解决了"跨分区原子写入 + 消费位移原子绑定"（TransactionalId+协调器+两阶段提交）。**任何超出 Kafka 边界的"一次性"诉求，都必须由应用层通过幂等消费或 Outbox 模式自行实现**。

**三个必须刻进肌肉记忆的认知**：

1. **幂等 ≠ 事务**：幂等只解决单分区重试重复，事务才解决跨分区原子性。两者正交，事务自动启用幂等
2. **EOS 有边界**：Kafka 事务只对 Kafka-to-Kafka 有效。消费者写数据库的场景，EOS 帮不上忙——需要幂等消费或 Outbox
3. **EOS 有代价**：吞吐量降 30-40%，p99 延迟增 30ms，且 read_committed 消费者会被卡住的事务阻塞。不要"因为重复不好"就盲目启用

**工业界真相**：

> 大多数应用不需要 EOS。At-least-once + 幂等消费者更简单、更快、且覆盖了几乎所有真实场景。Kafka 事务是为"金融转账、库存管理、精确的事件溯源投影"这类"重复即业务事故"的场景准备的。在 99% 的场景下，投资幂等消费者比投资 Kafka 事务回报更高。

