
# 第13章 监控 Kafka · 读书笔记

> 💡 这一章表面是"Kafka 有哪些 JMX 指标"，实质是在回答一个更根本的问题：**在一个分布式日志系统里，如何用最少的健康信号，证明集群"可用、耐久、及时"？**​ 原书第2版写于 ZooKeeper 时代，指标清单里还混着 ZooKeeper 相关 MBean，但 Kafka 4.0 已彻底移除 ZooKeeper，监控重心转移到 KRaft controller 元数据健康。所以这份笔记的做法是：**以原书 13.1—13.7 目录为骨架，用"五类不可妥协信号"（可用性、耐久性、时延、吞吐、滞后）重构监控体系**，并给出工业界可直接落地的 Prometheus + JMX Exporter + Grafana 告警基线。Kafka 官方文档明确指出：它暴露的指标非常多，"很容易让人混淆到底什么重要、什么可以放一边"——这正是本章要解决的核心痛点。

---

## 13.1 指标基础

### 13.1.1 指标来自哪里

所有 Kafka 指标都通过 **JMX（Java Management Extensions）**​ 暴露，以 MBean 形式组织。

**工业界采集架构**：

```
Kafka Broker/Client
    ↓ JMX
JMX Exporter (Sidecar)
    ↓ /metrics (HTTP)
Prometheus (拉取)
    ↓
Grafana (可视化) + AlertManager (告警)
```

**关键认知**：

- Kafka 所有 rate 类指标都有对应的 `-total` 累加计数
- 远程 JMX 默认禁用认证，**生产环境必须启用安全配置**——用 `KAFKA_JMX_OPTS` 覆盖默认安全配置，否则任何人都能监控甚至控制你的 broker
- 客户端指标同样通过 JMX 暴露，需要在应用侧部署 JMX Exporter 或用 Micrometer 绑定

### 13.1.2 需要哪些指标

**第一性原理：监控不是"收集所有指标"，而是"用最少的信号覆盖最关键的不变式"**。

Kafka 集群的健康可以归约为五类不可妥协的信号：

|信号类别|核心问题|代表指标|
|---|---|---|
|**可用性**​|集群能不能读写？|ActiveControllerCount、OfflinePartitionsCount|
|**耐久性**​|数据会不会丢？|UnderReplicatedPartitions、UnderMinIsrPartitionCount|
|**时延**​|请求快不快？|Produce/Fetch 的 TotalTimeMs P99、RequestHandlerAvgIdlePercent|
|**吞吐**​|管道饱没饱和？|BytesInPerSec、BytesOutPerSec、MessagesInPerSec|
|**滞后**​|消费跟不跟得上？|records-lag-max、消费者组 LAG|

> 💡 **洞见**：这五类信号构成了一个"监控金字塔"——底层是基础设施（CPU/内存/磁盘/网络），中间是 broker 指标，顶层是端到端业务指标。多数 Kafka 监控的失败，不是因为指标少，而是因为倒金字塔：收集了几百个 broker 指标，却没有任何端到端探针。

### 13.1.3 应用程序健康检测

除了 Kafka 自身指标，还需要：

- **基础设施指标**：CPU、内存、磁盘 I/O、网络吞吐（node_exporter）
- **JVM 指标**：GC 时间、堆内存、线程数（JMX Exporter 自带）
- **黑盒探针**：从生产者发一个带 timestamp 的消息，消费者收到后计算端到端延迟
- **主动健康检查**：用 AdminClient 周期性验证集群元数据可达性

**工业级健康脚本伪代码**：

```
def cluster_health_check():
    assert ActiveControllerCount.sum() == 1        # 恰好 1 个 controller
    assert OfflinePartitionsCount.sum() == 0       # 无离线分区
    assert UnderReplicatedPartitions.sum() == 0   # 无 URP
    assert UnderMinIsrPartitionCount.sum() == 0   # 无低于 min.insync 的分区
    assert IsrShrinksPerSec < threshold           # ISR 收缩率在阈值内
    assert leader_skew < 20%                       # leader 分布倾斜 < 20%
```

这个六条件检查适合做 cron job 或 CI 流水线，输出 pass/fail 到告警通道。

---

## 13.2 服务级别目标（SLO）

### 13.2.1 服务级别定义

SLO 不是"监控所有指标"，而是**为业务定义可量化的目标**：

```
例：订单管道 SLO
- 可用性：99.95%（每月停机 < 22 分钟）
- 端到端延迟：P99 < 500ms
- 消息持久性：99.999999999%（11 个 9）
- 消费者滞后：< 10,000 条消息
```

### 13.2.2 哪些指标是好的 SLI

**SLI（Service Level Indicator）必须是可测量的代理变量**：

|SLO|推荐 SLI|数据来源|
|---|---|---|
|可用性|生产请求成功率|ErrorsPerSec{error!=NONE} / RequestsPerSec|
|时延|Produce/Fetch 的 P99|TotalTimeMs{quantile="0.99"}|
|耐久性|URP = 0 的持续时间占比|UnderReplicatedPartitions|
|及时性|消费者组最大滞后|records-lag-max|
|吞吐|集群入站字节率|BytesInPerSec|

> ⚠️ **百分位的重要性**：原书特别强调，"需要查看平均值和 99% 或 99.9% 的值，这样就可以知道平均请求处理情况和异常值"。平均值掩盖尾部延迟——一个 P99 为 2s 的集群，平均值可能只有 20ms，但用户感知到的是"偶尔卡死"。

### 13.2.3 将 SLO 用于告警

**告警哲学：对人或对自动化？**

- **对人告警**：SLO 即将被破坏（如 URP > 0 持续 5 分钟）
- **对自动化告警**：可自愈的事件（如自动触发首选 leader 选举）

**Prometheus 告警规则示例**：

```
groups:
- name: kafka-cluster-health
  rules:
  - alert: NoActiveController
    expr: sum(kafka_controller_kafkacontroller_activecontrollercount) != 1
    for: 1m
    severity: critical
    
  - alert: UnderReplicatedPartitions
    expr: sum(kafka_server_replicamanager_underreplicatedpartitions) > 0
    for: 5m
    severity: warning
    
  - alert: OfflinePartitions
    expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
    for: 0m
    severity: critical
    
  - alert: HighProduceLatencyP99
    expr: kafka_network_requestmetrics_totaltimems{request="Produce",quantile="0.99"} > 100
    for: 5m
    severity: warning
```

> 💡 **洞见**：告警阈值不应该是拍脑袋的数字，而应该来自 SLO 的反推。如果 SLO 是"P99 < 500ms"，那么告警阈值应设在 400ms（留 20% 缓冲），而不是 1000ms。

---

## 13.3 Broker 的指标

### 13.3.1 诊断集群问题

broker 指标诊断遵循"由外而内"的路径：

```
1. 集群级：ActiveControllerCount、OfflinePartitionsCount、URP
2. 主机级：CPU、内存、磁盘 I/O、网络
3. Broker 级：RequestHandlerAvgIdlePercent、BytesIn/Out
4. 主题/分区级：per-topic BytesIn、per-partition lag
```

### 13.3.2 非同步分区的艺术（Under-Replicated Partitions）

这是**全章最重要的指标**，没有之一。

**URP 的本质**：`|ISR| < |所有副本|`——即有副本掉队，不再是同步副本集合的一员。

**MBean**：

```
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

**正常值为 0**。但"URP > 0"本身不说明问题，关键是**解读它的形态**：

|URP 形态|含义|处置|
|---|---|---|
|**瞬态波动**（几秒到几分钟）|副本正在追赶、网络抖动、broker GC|观察，通常自动恢复|
|**稳态持续**（多 broker 同时报告）|某个 broker 离线或硬件故障|立即排查离线 broker|
|**伴随高请求延迟**​|集群性能问题（磁盘慢、CPU 饱和）|排查主机级指标|

**配套指标**：

```
UnderMinIsrPartitionCount  # |ISR| < min.insync.replicas，生产者 acks=all 会开始报错
IsrShrinksPerSec           # ISR 收缩率，正常为 0
IsrExpandsPerSec           # ISR 扩张率，副本追上后扩张
```

**Red Hat 的诊断智慧**：

> 如果 URP 数量在波动，或多个 broker 显示高请求延迟 → 通常是集群性能问题，需要调查。
> 
> 如果 URP 数量是稳态的（不变的），且集群中多个 broker 都报告 → 通常意味着某个 broker 离线。

**命令行验证**：

```
kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions
```

输出示例：

```
Topic: topic-1 Partition: 0 Leader: 4 Replicas: 4,2,3 Isr: 4,3
                                          ↑副本3缺失        ↑ISR只有4,3
```

这说明 broker 2 上的副本掉队了。

**UnderMinIsrPartitionCount 的特殊严重性**：

- URP > 0：集群仍可用（只要 ISR ≥ min.insync.replicas）
- UnderMinIsr > 0：**acks=all 的生产者开始收到 NotEnoughReplicasException**，数据写入被拒

**Prometheus 告警**：

```
- alert: KafkaUnderMinIsr
    expr: kafka_server_replicamanager_underminisrpartitioncount > 0
    for: 0m
    severity: critical
```

> ⚠️ **工业铁律**：URP 是"数据耐久性降级"的早期信号，UnderMinIsr 是"写入开始失败"的紧急信号。前者 warning 后者 critical，二者必须分开告警。

### 13.3.3 Broker 指标

**🎯 集群级关键指标**

**ActiveControllerCount**：

```
kafka.controller:type=KafkaController,name=ActiveControllerCount
```

- 正常：集群中**恰好 1 个 broker 为 1**
- = 0：集群无法处理元数据变更（创建 topic、分区重分配等全部卡死）
- > 1：脑裂（KRaft 模式下理论上不可能，但网络分区时可能出现）
    
- **Alert**: `sum(...) != 1`

**KRaft 模式新增 controller 指标**：

```
kafka.controller:type=ControllerEventManager,name=EventQueueSize
kafka.controller:type=ControllerEventManager,name=EventQueueTimeMs
kafka.controller:type=KafkaRaftServer,name=MetadataLogEndOffset
```

- EventQueueSize > 100：controller 过载，分区选举和 topic 操作会延迟
- EventQueueTimeMs P99 > 1000ms：controller 事件处理变慢

**OfflinePartitionsCount**：

```
kafka.controller:type=KafkaController,name=OfflinePartitionsCount
```

- 正常：0
- > 0：有分区没有 leader，生产消费该分区完全阻塞
    
- **Alert**: `> 0` for 0m（立即告警）

**🔥 性能核心指标**

**RequestHandlerAvgIdlePercent**：

```
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
```

- 值域 [0, 1]，**理想值 > 0.3**
- < 0.2：broker 接近饱和
- < 0.1：broker 严重过载，请求排队

**NetworkProcessorAvgIdlePercent**：

```
kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent
```

- 值域 [0, 1]，**理想值 > 0.3**
- < 0.3：网络线程瓶颈

**📊 吞吐指标**

```
BytesInPerSec              # 客户端入站字节率
BytesOutPerSec             # 客户端出站字节率  
ReplicationBytesInPerSec   # 其他 broker 入站（副本追赶）
ReplicationBytesOutPerSec  # 向其他 broker 出站（作为 leader 时）
MessagesInPerSec           # 消息入站速率
```

**⏱️ 请求时延指标**

对每种请求类型（Produce、FetchConsumer、FetchFollower），关注：

```
TotalTimeMs               # 总耗时，重点看 P99/P99.9
RequestQueueTimeMs        # 在请求队列中等待的时间
LocalTimeMs               # Leader 本地处理时间
RemoteTimeMs              # 等待 follower 响应的时间（仅 acks=all 时非零）
ResponseQueueTimeMs       # 在响应队列中等待的时间
ResponseSendTimeMs        # 发送响应时间
```

**原书洞见**：

> 对每种请求类型，至少需要收集总时间指标的平均值和较高的百分位（99% 或 99.9%），以及每秒请求次数。为总时间指标设定告警阈值有一定难度——一般来说，可以为总时间指标设定 99.9 百分位基线并设置告警，对 Produce 请求类型来说更是如此。与非同步分区类似，Produce 请求的 99.9 百分位快速增长说明集群出现了大范围的性能问题。

**🛡️ 可靠性指标**

```
UnderReplicatedPartitions      # URP，正常 0
UnderMinIsrPartitionCount      # 低于 min.insync 的分区数，正常 0
OfflineLogDirectoryCount       # 离线日志目录数，正常 0
PartitionCount                 # 分区总数，broker 间应均衡
LeaderCount                    # Leader 分区数，broker 间应均衡
IsrShrinksPerSec               # ISR 收缩率，正常为 0
IsrExpandsPerSec               # ISR 扩张率
UncleanLeaderElectionsPerSec   # 不洁 leader 选举率，必须为 0
```

**💧 Purgatory（炼狱）指标**

```
DelayedOperationPurgatory,PurgatorySize,delayedOperation=Produce  # acks=all 时非零
DelayedOperationPurgatory,PurgatorySize,delayedOperation=Fetch   # 取决于 fetch.wait.max.ms
```

- Purgatory 是"等待条件满足"的请求挂起区
- acks=all 时，生产者请求会在 Purgatory 等待所有 ISR 确认
- 数值持续增长 → 副本跟不上或网络延迟高

### 13.3.4 主题的指标和分区的指标

**per-topic 指标**：

```
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=([-.\w]+)
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=([-.\w]+)
kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec,topic=([-.\w]+)
```

**用途**：

- 识别造成集群流量大量增长的主题
- 定位异常 topic（如消息过大、错误率突增）
- 调试与客户端相关的问题

**per-partition 指标**：

- 在较大集群中，per-partition 指标数量巨大，不可能全部收集到监控系统
- 但调试时非常有用：识别热点分区、副本分配不均

> 💡 **工业实践**：日常监控只收集集群级和 broker 级指标；只有在调试特定 topic 问题时，才临时开启 per-topic/per-partition 指标收集。

### 13.3.5 Java 虚拟机监控

Kafka 是 JVM 应用，JVM 健康直接影响 broker 性能。

**核心 JVM 指标**：

```
GC 时间占比           # Young GC + Full GC 总时间
堆内存使用率          # 老年代使用率 > 80% 告警
线程数                # 线程泄漏检测
Class Loader 数量     # 类加载器泄漏检测
```

**GC 调优基线**（工业界常见配置）：

```
G1GC + 堆内存 = 4-8GB（broker 堆不宜过大）
MaxGCPauseMillis=20ms
避免 Full GC（如出现，说明堆内存不足或内存泄漏）
```

> ⚠️ **关键认知**：Kafka broker 的堆内存主要用于**页缓存索引、请求处理、副本管理**，而非消息存储（消息直接写文件系统页缓存）。因此 broker 堆通常配置 4-8GB 即可，过大的堆反而导致 GC 停顿变长。

### 13.3.6 操作系统监控

**四类基础设施指标**：

**CPU**：

- 用户态 CPU%：broker 处理逻辑消耗
- 系统态 CPU%：上下文切换、系统调用
- 负载均值（load average）：应 < CPU 核数
- 高 CPU + 高 Load → 线程饥饿或资源争用

**内存**：

- 物理内存使用率
- Swap 使用率：**必须接近 0**（vm.swappiness 设为 1）
- 页缓存命中率：Kafka 依赖页缓存加速读写

**磁盘 I/O**：

- 磁盘吞吐量（MB/s）
- 磁盘延迟（await、svctm）
- **这是 Kafka 最常见的瓶颈**——日志段滚动、副本同步都依赖磁盘

**网络**：

- 网络吞吐（MB/s）：应 < 网卡容量的 70%
- 网络错误包数
- TCP 重传率：高重传率说明网络不稳定，直接导致 ISR 收缩

**Red Hat 的诊断经验**：集群指标出现尖峰往往指向 broker 问题，通常与慢速或故障的存储设备有关，或者与其他进程的计算资源争用有关。

### 13.3.7 日志

Kafka broker 日志是最后的诊断手段：

**关键日志模式**：

```
ERROR [KafkaApi-1] ... NotEnoughReplicasException  # ISR 不足
WARN  [ReplicaManager] ... ISR shrink for partition  # ISR 收缩
ERROR [Controller] ... Failed to elect leader         # Leader 选举失败
GC pause 时间过长                                     # JVM 问题
```

**日志监控要点**：

- ERROR 日志速率突增 → 立即告警
- 特定异常模式（NotEnoughReplicas、TimeoutException）→ 定向告警
- 日志转发到 ELK/Splunk 做聚合分析

---

## 13.4 客户端监控

### 13.4.1 生产者指标

**全局生产者指标**：

```
record-send-rate              # 发送速率
record-error-rate             # 错误率，> 0 即告警
record-retry-rate             # 重试率，> 10/s 告警
request-latency-avg           # 平均请求延迟
request-latency-max           # 最大请求延迟
batch-size-avg                # 平均批大小
bufferpool-wait-time          # 缓冲区等待时间
compression-rate-avg          # 压缩率
```

**关键生产者指标解读**：

|指标|健康值|异常信号|
|---|---|---|
|`request-latency-avg`|< 100ms|> 500ms 说明 broker 或网络问题|
|`record-error-rate`|0|任何非零值都需要调查|
|`record-retry-rate`|偶发|持续 > 10/s 说明集群不健康|
|`bufferpool-wait-time`|接近 0|显著 > 0 说明 buffer.memory 不足|
|`batch-size-avg`|接近 batch.size|过小说明 linger.ms 太短或吞吐低|

**一级银行超低延迟交易调优案例**：

> 某顶级银行将 Kafka 调优用于超低延迟交易，关键生产者配置：
> 
> - `linger.ms=1`：增加 1ms 延迟换取更大批次
> - `batch.size=32000`：32KB 批次
> - `compression.type=lz4`：快速压缩
> - `enable.idempotence=true`：保证 EOS
> - `acks=all`：最高持久性
> 
> 通过监控 `request-latency-avg/max`、`batch-size-avg`、`bufferpool-wait-time`、`record-error-rate`、`record-retry-rate`，将端到端延迟优化到微秒级。

**per-broker 和 per-topic 指标**：

```
kafka.producer:type=producer-metrics,client-id={clientId},topic={topic}
```

用于定位"发送到哪个 broker/topic 特别慢"的问题。

### 13.4.2 消费者指标

**Fetch Manager 指标**：

```
records-lag-max              # 所有分区中最大滞后
records-lag                  # 每分区滞后
records-consumed-rate        # 消费速率
bytes-consumed-rate         # 消费字节率
fetch-latency-avg           # 拉取请求平均延迟
fetch-latency-max           # 拉取请求最大延迟
fetch-rate                  # 拉取请求速率
```

**消费者协调器指标**：

```
commit-latency-avg          # 偏移量提交延迟
rebalance-latency-max       # 重平衡延迟（rare high p99 延迟的来源）
```

**records-lag-max 的陷阱**：

> ⚠️ **records-lag-max 只在消费者实例存活时才被发布**。如果消费者完全崩溃，这个指标从 JMX 消失。因此，**仅靠客户端 JMX 监控是不够的**——必须结合 broker 侧的 `kafka-consumer-groups.sh` 或专用 lag exporter 作为离线消费者组的权威数据源。

**per-partition lag 的价值**：

消费者组平均 lag 可能为 5000，但某个分区可能高达 200,000 而其他分区接近 0。这种模式通常指示：

- 分区分配不均
- 热分区（hot partition）
- 特定分区上的消息导致处理失败和重试

**消费速率 vs 生产速率**：

```
ratio = records-consumed-rate / record-send-rate
```

- ratio ≈ 1：消费者跟上生产者
- ratio < 1：消费者落后，lag 会持续增长
- 这是 lag 累积的**领先指标**——可以在 lag 达到临界值前告警

### 13.4.3 配额

**配额指标**：

```
kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec
```

当客户端超过配置的 `producer_byte_rate` 或 `consumer_byte_rate` 配额时，broker 会拒绝请求并记录到 BytesRejectedPerSec。

**监控要点**：

- BytesRejectedPerSec > 0：说明有客户端触发了配额限制
- 可能是"吵闹的邻居"效应，也可能是配额配置过低
- 配合 `kafka-configs.sh --entity-type users --describe` 查看当前配额

---

## 13.5 滞后监控

**Consumer Lag 是应用团队最关心的指标**。

**监控层次**：

```
1. 客户端 JMX：records-lag-max（实时，但消费者崩溃即消失）
2. Broker 侧：kafka-consumer-groups.sh --describe --group xxx
3. 专用工具：Burrow（LinkedIn）、Kafka Exporter、Confluent Control Center
```

**Prometheus + Kafka Exporter 告警**：

```
- alert: ConsumerLagTooHigh
    expr: kafka_consumergroup_lag > 10000
    for: 10m
    severity: warning
    
  - alert: ConsumerLagGrowing
    expr: delta(kafka_consumergroup_group_sum_lag[5m]) > 100
    for: 15m
    severity: warning
```

**Lag 阈值的设定哲学**：

- **绝对阈值**（如 > 10,000）：适合稳态消费
- **相对阈值**（如 lag / 消费速率 > 5分钟）：更符合业务语义
- **趋势告警**（如 5分钟内 lag 增长率 > X）：能在 lag 达到临界值前预警

**工业工具选型**：

|工具|特点|
|---|---|
|**Burrow**​|LinkedIn 开源，专用 lag 监控，支持复杂告警规则|
|**Kafka Exporter**​|轻量级，Prometheus 生态，社区活跃|
|**Confluent Control Center**​|商业，UI 完善，内置 lag 仪表板|
|**Datadog / Dynatrace**​|云原生 APM，端到端追踪|

---

## 13.6 端到端监控

**第一性原理：broker 健康 ≠ 管道健康**。

即使所有 broker 指标完美，端到端管道仍可能出问题：

- 生产者到消费者的实际延迟
- 消息是否按预期被处理
- 下游系统的健康度

**端到端探针设计**：

```
1. 生产者发送带 timestamp 和 correlationId 的探测消息到 probe-topic
2. 消费者（或专门的探针服务）消费该消息
3. 计算 end-to-end-latency = consume_timestamp - produce_timestamp
4. 将延迟上报到监控系统
```

**关键端到端指标**：

```
end-to-end-latency-p99     # 端到端 P99 延迟
end-to-end-success-rate    # 端到端成功率（消息是否被正确处理）
processing-error-rate      # 下游处理错误率
```

**一级银行案例的端到端验证**：

> 通过同时监控 broker 侧指标（NetworkProcessorAvgIdlePercent / RequestHandlerAvgIdlePercent）、生产者指标（request-latency-avg/max、batch-size-avg、record-error-rate）和消费者指标（records-consumed-rate、fetch-latency-avg/max、records-lag-max、commit-latency-avg），该银行验证了端到端延迟目标不仅在生产侧达成，在消费侧也同样满足——避免了"上游优化被下游 lag 掩盖"的常见陷阱。

> 💡 **洞见**：端到端监控的真正价值，是暴露"broker 看起来健康，但业务实际上受损"的盲区。Kafka 的分布式特性意味着：broker 正常 + 生产者正常 + 消费者正常 ≠ 管道正常。只有端到端探针能回答"消息从生产到被业务处理，到底花了多长时间"。

---

## 13.7 小结

把全章串起来，Kafka 监控是一个**五层金字塔体系**：

```
┌─────────────────────────────────────────────────────────┐
│ 第五层：端到端监控                                        │
│  • 端到端延迟探针                                        │
│  • 业务成功率                                           │
│  • 下游处理错误率                                        │
├─────────────────────────────────────────────────────────┤
│ 第四层：客户端监控                                        │
│  • 生产者：record-error-rate, request-latency, batch-size│
│  • 消费者：records-lag-max, fetch-latency, commit-latency│
│  • 配额：BytesRejectedPerSec                            │
├─────────────────────────────────────────────────────────┤
│ 第三层：Broker 监控（核心）                               │
│  • 可用性：ActiveControllerCount, OfflinePartitionsCount │
│  • 耐久性：UnderReplicatedPartitions, UnderMinIsr        │
│  • 时延：TotalTimeMs P99, RequestHandlerAvgIdlePercent   │
│  • 吞吐：BytesIn/Out, MessagesIn                         │
├─────────────────────────────────────────────────────────┤
│ 第二层：JVM 监控                                          │
│  • GC 时间、堆内存、线程数                                │
├─────────────────────────────────────────────────────────┤
│ 第一层：基础设施监控                                      │
│  • CPU、内存、磁盘 I/O、网络                             │
└─────────────────────────────────────────────────────────┘
```

### 工业界 Kafka 监控告警基线

**🔴 Critical 告警（立即呼叫）**：

```
- OfflinePartitionsCount > 0
- sum(ActiveControllerCount) != 1
- UnderMinIsrPartitionCount > 0
- OfflineLogDirectoryCount > 0
- RequestHandlerAvgIdlePercent < 0.1
- UncleanLeaderElectionsPerSec > 0
```

**🟡 Warning 告警（团队频道，1小时内处理）**：

```
- sum(UnderReplicatedPartitions) > 0 for 5m
- IsrShrinksPerSec 持续 > 0
- Produce 请求 TotalTimeMs P99 > 100ms for 5m
- Consumer Lag > 10,000 for 10m
- NetworkProcessorAvgIdlePercent < 0.3
- 生产者 record-error-rate > 0
- 磁盘使用率 > 80%
```

### 生产环境监控栈推荐

```
采集：JMX Exporter (Sidecar) + Kafka Exporter + node_exporter
存储：Prometheus
可视化：Grafana（Kafka 官方 dashboard + 自定义）
告警：AlertManager + 告警分级路由
日志：Filebeat → ELK / Splunk
端到端：自定义探针 + APM（Datadog/Dynatrace）
```

### 给读者的"Kafka 监控心智模型"

> 📌 **全章最核心的一句话**：Kafka 监控的本质不是"收集指标"，而是**用五类不可妥协的信号（可用性、耐久性、时延、吞吐、滞后）去证明集群满足了业务 SLO**——URP 是数据耐久性的早期信号，UnderMinIsr 是写入失败的紧急信号，RequestHandlerAvgIdlePercent 是 broker 饱和度的直接反映，records-lag-max 是消费健康度的核心指标，而端到端探针是唯一能回答"业务真健康吗"的手段。原书第2版写于 ZooKeeper 时代，但 Kafka 4.0 已彻底移除 ZooKeeper——现代 Kafka 监控必须把 KRaft controller 的 `EventQueueSize`、`EventQueueTimeMs`、`MetadataLogEndOffset` 纳入核心指标集，同时保留 URP、ISR、请求时延这些永恒的真理指标。

**三个必须刻进肌肉记忆的认知**：

1. **URP=0 是健康基线，但不是唯一基线**。URP 只回答了"副本是否同步"，还需要 UnderMinIsr 回答"是否还能写入"、OfflinePartitions 回答"是否还有 leader"、ActiveControllerCount 回答"controller 是否正常"。这五个指标构成 broker 可用性+耐久性的最小完备集。
2. **百分位比平均值重要 100 倍**。一个 P99 为 2s 的集群，平均值可能只有 20ms，但用户感知到的是"偶尔卡死"。原书特别强调要关注 99% 或 99.9% 的百分位——这是 Kafka 监控区别于传统监控的核心思维转变。
3. **客户端监控不能只靠 broker 侧**。records-lag-max 只在消费者存活时存在，消费者崩溃后这个指标消失。因此生产环境必须同时部署：客户端 JMX Exporter（实时细粒度）+ Kafka Exporter/Burrow（离线消费者组的权威 lag 源）+ 端到端探针（业务视角的健康度）。三层互补，缺一不可。

**工业界真相**：

> 绝大多数 Kafka 监控的失败，不是因为工具不好，而是因为两个反模式：
> 
> - **指标堆积症**：收集了几百个 broker 指标，却没有定义任何 SLO，告警阈值全是拍脑袋的数字。结果要么告警疲劳（全部 ignore），要么关键事件被淹没。
> - **倒金字塔监控**：底层基础设施指标齐全，顶层端到端探针缺失。broker 看起来都健康，但业务已经中断 3 小时没人知道。
> 
> 优秀 Kafka 监控的标志，是**能用 6 个核心指标回答"集群是否健康"**（ActiveControllerCount、OfflinePartitionsCount、URP、UnderMinIsr、RequestHandlerAvgIdlePercent、端到端延迟 P99），其余几百个指标都是为调试这 6 个核心信号服务的。LinkedIn 的 Kafka 部署用 Burrow 专门做 lag 监控，一级银行的超低延迟交易用 request-latency/bufferpool-wait-time/records-lag-max 这组小指标集完成微秒级优化——他们都证明了：**少而准的指标 >> 多而杂的指标**。

