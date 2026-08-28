
# 第2章 安装Kafka · 读书笔记

> 💡 这一章表面是"怎么把 Kafka 跑起来"，实质是在回答：**一个以"分布式提交日志"为第一性原理的系统，它的物理载体应该怎样被构建？**​ 安装 Kafka 不是装软件，而是把"日志抽象"映射到"CPU、内存、磁盘、网络、机架、可用区"这些物理资源上的过程。书里第2版依然用较大篇幅讲 ZooKeeper 部署，但工业界自 Kafka 3.3 起新集群已经全面转向 KRaft 模式——这是本章笔记第一条要修正的认知。

---

## 2.1 环境配置

### 2.1.1 选择操作系统

书里明确推荐 Linux，这一点毫无疑问。但"为什么是 Linux"值得从第一性原理讲清楚：

Kafka 的高性能源于三点——**顺序 I/O、页缓存（page cache）、零拷贝（sendfile）**。这三项能力在 Linux 上最为成熟：

- 页缓存由 Linux 内核统一管理，Kafka 的堆外内存策略完全建立在"信任 OS 缓存"之上
- `sendfile()` 零拷贝系统调用在 Linux 上性能最优
- I/O 调度器、文件系统（XFS/ext4）的调优手段在 Linux 上最丰富

Windows 和 macOS 只适合开发测试。附录里讲它们在非 Linux 环境安装，仅仅是"能跑"，不是"应该跑"。

### 2.1.2 安装Java

Kafka 是 JVM 应用。第2版书里示例用 JDK 11，但站在 2025-2026 年的视角：

- **JDK 17+ 是新的生产基线**，ZGC 在低延迟场景下优势明显
- 仍然推荐 **6-8GB 的 JVM 堆**——堆不是越大越好，因为 Kafka 重度依赖 OS 页缓存而非堆内存
- 安装完整 JDK（非 JRE），便于用 `jstack`、`jmap` 等工具排查问题

### 2.1.3 安装ZooKeeper——一个正在过时的话题

书里花了相当篇幅讲 ZooKeeper 独立安装和 ensemble 集群配置，这需要放在历史语境下理解：

> ⚠️ **重要更新**：从 Kafka 3.3 开始，KRaft 模式已经 production-ready；新集群**不应该再装 ZooKeeper**。

ZooKeeper 在老架构中承担元数据管理：broker 注册、controller 选举、topic 配置等。但它带来了两个结构性问题：

1. **运维复杂度**：你要维护两个分布式系统（Kafka + ZK），版本升级需要协调
2. **故障转移延迟**：controller 变更需经 ZK 往返，failover 慢

KRaft 模式用内置的 Raft 共识协议取代 ZK，带来：

|维度|ZooKeeper 模式|KRaft 模式|
|---|---|---|
|元数据存哪里|外部 ZK 集群|Kafka 内部 quorum|
|Controller 选举|ZK 主导|Raft 共识内部选|
|故障转移|慢（需从 ZK 加载）|近瞬时（元数据已在内存）|
|运维对象|Kafka + ZK 两套|仅 Kafka|
|扩展能力|ZK 易成瓶颈|支持百万级分区|

官方文档明确：KRaft 模式下 `zookeeper.connect` 等一大批配置被移除，`broker.id` 被 `node.id` 取代，controller 与 broker 角色分离。

📌 **读书笔记的实践指引**：如果你是按书学习，照着书把 ZK 装一遍理解架构演进有必要；但如果是在生产环境部署新集群，**直接采用 KRaft**。

---

## 2.2 安装 broker

书的步骤：下载 tarball → 解压到 `/usr/local/kafka` → `kafka-server-start.sh` 启动。这个过程本身没变，但工业界实际做法已经分化：

- **裸金属/自有 IDC**：自动化部署（Ansible/Puppet/Terraform）
- **云端自管**：市场镜像 + 启动脚本
- **完全托管**：Amazon MSK、Confluent Cloud、Azure Event Hubs for Kafka

启动命令本身不变，但**真正决定集群命运的从来不是"怎么启动"，而是"用什么配置启动"**——这就引出 2.3 节。

---

## 2.3 配置 broker

这是本章最值钱的一节。书里分了"常规配置"和"主题默认配置"两类，我把工业界生产基线的关键参数整理如下。

### 2.3.1 常规配置参数

**Broker 身份与监听**

```
broker.id=1                    # 集群内唯一，KRaft 模式下改为 node.id
listeners=PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093
advertised.listeners=PLAINTEXT://broker1.example.com:9092
broker.rack=us-east-1a         # 机架感知，云上映射到 AZ
log.dirs=/data1/kafka-logs,/data2/kafka-logs,/data3/kafka-logs
```

`log.dirs` 配多个物理磁盘路径是关键——Kafka 会用 JBOD 方式把分区均衡到各磁盘，**不要用 RAID**（失去并行 I/O 能力）。`broker.rack` 是生产环境必配项，它保证分区的多个副本跨机架/跨可用区分布。

**线程与网络**

```
num.network.threads=8          # 约等于 CPU 核数
num.io.threads=16              # 约等于磁盘数的 2 倍
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
```

经验公式：**`num.network.threads` ≈ CPU 核数，`num.io.threads` ≈ 2 × 磁盘数**。

**复制与持久化（这是生产配置的灵魂）**

```
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
auto.create.topics.enable=false
```

这组配置的含义是：**允许 1 个 broker 挂掉而不丢数据，且绝不让"落后副本"成为 leader**。

> 💡 `unclean.leader.election.enable=false` 是用"可用性"换"一致性"的经典取舍。如果设为 true，当所有 ISR 副本都挂了，Kafka 会从一个非同步副本中选 leader——这可能丢数据，但集群能继续服务。生产环境金融/交易类业务一律设 false。

**内部 topic 的副本数**

```
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

> ⚠️ 必须在集群首次启动前设好！如果先用默认值 1 启动，后续再改成 3，已有 offset 数据不会自动复制，broker 故障时会导致消费者组失败。

### 2.3.2 主题的默认配置

**保留策略**

```
log.retention.hours=168        # 7 天
log.retention.bytes=-1        # 不按大小限制（-1 表示禁用）
log.segment.bytes=1073741824  # 1GB 一段
```

书里提到保留可以基于时间或大小。工业视角的洞见是：

> **保留时长不是"技术参数"，而是"成本旋钮"**。

Kafka 是传输层，不是归档层。大部分管线下游是 S3/数据仓库/数据湖，Kafka 只需要保留 24-48 小时即可覆盖"重处理"需求。**把保留从 7 天降到 2 天，磁盘需求直接砍掉 3.5 倍**。如果业务确实需要长保留（合规、审计），用 **Kafka 分层存储（Tiered Storage）**——热数据留本地 NVMe，冷数据卸到 S3，整体成本降 60-80%。

**分区数与刷盘**

```
num.partitions=12              # 新书部分场景建议更多
log.flush.interval.messages=9223372036854775807  # 极大值=从不主动 flush
```

**Kafka 故意不做应用层刷盘**——它完全依赖 OS 页缓存的回写机制。这是"信任 OS"哲学的延伸：既然 OS 的页缓存算法已经最优，何必自己在应用层再做一次？

### 洞见：配置的本质是"声明你的取舍"

翻完 2.3 的所有参数，你会发现 Kafka 的配置项不是在问"系统能做什么"，而是在逼你想清楚：

- **可用性 vs 一致性**：`unclean.leader.election.enable`
- **成本 vs 可靠性**：`default.replication.factor`
- **延迟 vs 吞吐**：`linger.ms`（生产者侧）、`fetch.min.bytes`（消费者侧）
- **本地性能 vs 长期归档**：`log.retention.hours` + 分层存储

**第一性原理视角**：Kafka 把所有"鱼与熊掌"的抉择都交给了你，因为它自己只保证一件事——**日志的 append-only 语义**。其他的，都是你的业务决策。

---

## 2.4 选择硬件

书里按磁盘吞吐量、磁盘容量、内存、网络、CPU 五个维度讲。我把工业界的真实经验融入进来。

### 2.4.1 磁盘吞吐量

Kafka 是**顺序写**密集型，不是随机写。这意味着：

- **HDD 也能跑 Kafka**，但现代生产环境一律用 SSD，高吞吐场景必须用 **NVMe SSD**
- NVMe 比 SATA SSD 吞吐量高 3-7 倍，p99 延迟 < 10ms
- 云端优先选**本地 NVMe 实例存储**（如 AWS i4i），比网络附加存储（EBS）性能更好、成本更低
- 多磁盘用 JBOD，不用 RAID

### 2.4.2 磁盘容量

容量规划的公式（这是本章最重要的一个算式）：

```
单 broker 磁盘需求 = (入口速率 MB/s × 副本因子 × 保留秒数) / broker 数 / 0.70
```

**永远留出 30% 余量**——这是工业界的血的教训。

举个真实例子：电商峰值 300 MB/s，保留 7 天，RF=3，预期年增长 50%：

- 日均原始数据：300 × 86400 = 25.3 TB
- 7 天 × 3 副本 = 531 TB
- 加 20% buffer + 50% 增长 = 956 TB
- 用 8TB NVMe × 12 = 96TB 的 X-Large broker，需要 **10 台**

如果用分层存储（本地 1 天 + S3 6 天），本地只需 3 台 Large broker + S3 存储，成本降 **71%**。

### 2.4.3 内存

Kafka 对堆内存需求不大，但对**页缓存**极度渴求：

- **JVM 堆：6-8 GB**，绝不超过 32 GB（ compressed OOPs 阈值）
- **物理内存：32-128 GB**，剩余全给 OS 页缓存
- 经验法则：页缓存应能容纳至少 **2 小时**的写入量

> 💡 这就是为什么 Kafka broker 不像传统数据库那样需要超大内存——它把"缓存管理"外包给了 Linux 内核，自己专注做日志。

### 2.4.4 网络

网络是 Kafka 集群最容易被低估的瓶颈：

- **生产环境最低 10 Gbps，高吞吐集群推荐 25 Gbps**
- 带宽需求 = 生产者流量 × (副本因子 + 消费者组倍数)
- 例：200 MB/s 入口 + RF=3 + 3 个消费者组 = 内部网络需承载约 5× 流量
- broker 间延迟 **< 1 ms**

### 2.4.5 CPU

Kafka 本身不是 CPU 密集型，但**压缩、加密会吃 CPU**：

- 生产环境 **8-16 核起步**，高吞吐 16-32 核
- 偏好**高主频**而非堆核数
- 线程配置：`num.network.threads` ≈ 核数，`num.io.threads` ≈ 2× 磁盘数

### 洞见：硬件选型的"第一性原理"

回到日志抽象——Kafka 的磁盘访问模式是**纯顺序 append + 顺序 read**。这个事实决定了：

|资源|传统 DB 的需求|Kafka 的需求|原因|
|---|---|---|---|
|磁盘|随机写，要 IOPS|顺序写，要吞吐|日志 append-only|
|内存|大堆缓存热点数据|小堆 + 大页缓存|信任 OS|
|CPU|大量计算|中等，压缩时激增|主要工作是 I/O 转发|
|网络|客户端交互|内部复制放大|副本间全量同步|

**所以 Kafka 的"理想硬件画像"是：中等 CPU + 中等内存 + 多块 NVMe + 高带宽网络。**​ 它不是一台"数据库服务器"，而是一台"日志流水线专用机"。

---

## 2.5 云端的 Kafka

书里只简要提及 Azure 和 AWS，但云端 Kafka 在 2025-2026 已经是主流形态。

### 2.5.1 微软 Azure

Azure 上有三种玩法：

1. **Azure Event Hubs for Kafka**：Azure 原生服务，兼容 Kafka 协议，但底层不是真 Kafka——这是个"Kafka 兼容端点"
2. **Azure VM 上自管**：手工部署，控制力强但运维重
3. **Confluent Cloud on Azure**：全托管真 Kafka，与 Azure 服务集成

> 📌 如果团队没有专职 Kafka 运维，Event Hubs for Kafka 或 Confluent Cloud 是更务实的选择。

### 2.5.2 AWS

AWS 上对应的是 **Amazon MSK**：

- **完全托管的开源 Kafka**，协议 100% 兼容
- AWS 负责：broker 基础设施、故障替换、补丁升级、多 AZ 复制、元数据节点（KRaft 或 ZK 都由 AWS 管）
- 你负责：topic 设计、分区策略、副本因子、客户端配置、Schema 治理、访问控制
- 两种形态：**MSK Provisioned**（自选 broker 类型/存储/版本）+ **MSK Serverless**（完全免容量规划）
- 支持 KRaft 和 ZooKeeper 两种元数据模式

**工业抉择框架**：

- 团队有 Kafka 专家 + 需要深度调优 → 自管（EC2 或 EKS 上）
- 团队想聚焦业务 + 接受托管约束 → MSK Provisioned
- 流量波动剧烈 + 不想做容量规划 → MSK Serverless 或 Confluent Cloud

> 💡 MSK 不是黑盒——"完全托管"不等于"零责任"。Topic 设计烂、分区倾斜、producer 配置错，AWS 也救不了你。

---

## 2.6 配置 Kafka 集群

### 2.6.1 需要多少个 broker

书里讨论了 broker 数量决策。工业界的判断框架是**取三个维度的最大值**：

```
broker 数 = max(
    (峰值吞吐 × 副本因子) / 单 broker 吞吐,
    总分区数 / 单 broker 最大分区数,
    3  # HA 最小要求
)
```

硬性约束：

- **KRaft 模式**：单 broker 最多 ~4000 分区，单集群最多 ~200 万分区
- **ZooKeeper 模式**：单集群最多 ~20 万分区

**生产环境最小 3 broker**——这是 `default.replication.factor=3` 的物理基础。

### 2.6.2 broker 配置

集群部署的关键补充配置（书里未充分展开）：

```
# KRaft 模式（新集群推荐）
process.roles=broker,controller   # 生产环境 controller 应独立部署
node.id=0
controller.quorum.voters=0@broker0:9093,1@broker1:9093,2@broker2:9093

# 跨可用区分布
broker.rack=us-east-1a|b|c

# 防止 unclean leader election
unclean.leader.election.enable=false
min.insync.replicas=2
```

> ⚠️ KRaft 模式下，**controller 节点与生产 broker 节点应分离部署**。`combined` 模式仅适用于开发环境。

### 2.6.3 操作系统调优

这是书里 2.6.3 的重点，也是工业界最容易忽视的"隐形性能金矿"。

**文件描述符**

```
# /etc/security/limits.conf
kafka soft nofile 128000
kafka hard nofile 128000
fs.file-max = 500000
```

每个分区、每个连接都要占 fd，高分区集群 fd 耗尽是常见故障。

**虚拟内存**

```
vm.swappiness = 1       # 极端避免 swap
vm.dirty_ratio = 80     # 提高脏页比例，批量回写
vm.dirty_background_ratio = 5
```

Kafka 依赖页缓存，swap 一旦触发性能断崖。

**网络栈**

```
net.core.wmem_max = 16777216
net.core.rmem_max = 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.core.somaxconn = 65535
net.ipv4.tcp_slow_start_after_idle = 0   # 禁空闲后慢启动
```

**磁盘 I/O 调度**

```
# SSD/NVMe 用 none 或 noop
echo none > /sys/block/nvme0n1/queue/scheduler
# 提高顺序读预取
echo 4096 > /sys/block/nvme0n1/queue/read_ahead_kb
```

**文件系统挂载**

```
# /etc/fstab
/dev/nvme0n1 /var/kafka-logs xfs defaults,noatime,nodiratime 0 2
```

`noatime` 避免每次读都更新访问时间——对 Kafka 这种读多场景省下大量元数据写。

**JVM 调优**

```
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
export KAFKA_JVM_PERFORMANCE_OPTS="-server \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent \
  -XX:+DisableExplicitGC \
  -Djava.awt.headless=true"
```

G1GC 是 Kafka 的现代标配——自适应、可预测的低停顿、对大堆友好。JDK 17+ 可考虑 ZGC 进一步压低延迟。

---

## 2.7 生产环境的注意事项

### 2.7.1 垃圾回收器选项

书里专门讨论 GC，工业界现状：

- **G1GC 是默认推荐**
- 堆大小控制在 **6-8 GB**——过大的堆会导致 GC 停顿时间变长，反而伤害延迟
- 监控 GC 日志，**单次停顿 > 100 ms 就要告警**
- JDK 17+ 生产环境可评估 **ZGC**，理论上停顿 < 10 ms
- 关键参数：`-XX:MaxGCPauseMillis=20`，`-XX:InitiatingHeapOccupancyPercent=35`

> 💡 一个反直觉的事实：**Kafka broker 的 JVM 堆不是越大越好**。因为 Kafka 主要依赖 OS 页缓存而非堆内存来服务读请求，把内存都分给堆反而挤占了页缓存，得不偿失。

### 2.7.2 数据中心布局

这是书里 2.7.2 的重点，也是生产事故的头号杀手。

**机架感知（Rack Awareness）**

```
broker.rack=us-east-1a   # 云上映射到可用区
```

Kafka 会在创建分区时将副本分散到不同 rack。但有两个坑：

> ⚠️ **坑 1**：机架感知**只在分区创建时生效**。后续手动 reassign 可能导致副本同 rack，Kafka 不会自动纠正。
> 
> ⚠️ **坑 2**：必须用 **Cruise Control**​ 等工具定期 rebalance，维持跨 rack 均衡。

**物理基础设施建议**：

- 每个 broker 尽量在不同机架
- 双电源（不同电路）
- 双网络交换机 + bonded interface
- 云上：broker 跨 3 个 AZ 均匀分布

**ISR 与 min.insync.replicas 的配合**

`min.insync.replicas=2` + 生产者 `acks=all` + RF=3：**允许 1 个 broker 挂掉不丢数据**。这是生产环境的黄金组合。

### 2.7.3 共享 ZooKeeper

书里讨论了共享 ZK 的问题。在 2025-2026 的视角下：

- **新集群直接用 KRaft**，不再需要 ZK
- 老集群迁移到 KRaft 的路径已经官方支持（KIP-833），但迁移过程中不能变更 metadata version
- 如果暂时还要用 ZK：**不要让 Kafka 与其他系统共享 ZK 集群**——ZK 是强一致性系统，争用会影响 Kafka 的 controller 选举延迟

---

## 2.8 小结

把全章串起来，Kafka 的安装与配置本质上是在回答三个问题：

**第一：用什么元数据模式？**

→ 新集群一律 KRaft，老集群规划迁移。

**第二：硬件怎样匹配"日志抽象"？**

→ 中等 CPU + 32-128GB 内存 + 多块 NVMe + 25Gbps 网络 + JBOD。

**第三：配置如何表达业务取舍？**

→ `default.replication.factor=3` + `min.insync.replicas=2` + `unclean.leader.election.enable=false` + `auto.create.topics.enable=false` 是生产基线。

### 给读者的"安装前清单"

基于全章内容，我把它提炼成一张工业界 checklist：

```
□ 操作系统：Linux（XFS 或 ext4）
□ Java：JDK 17+，G1GC 或 ZGC
□ 元数据模式：KRaft（新集群）
□ broker 数：max(吞吐需求, 分区需求, 3)
□ 每台 broker：8-16 核 / 32-64GB 内存 / 多块 NVMe / 10-25Gbps
□ broker.rack 跨可用区分布
□ 副本因子 3，min.insync.replicas 2
□ OS 调优：fd limit / vm.swappiness / tcp buffers / I/O scheduler
□ JVM：堆 6-8GB，G1GC
□ 关闭 auto.create.topics，关闭 unclean leader election
□ 保留策略：先算业务需求，长保留用分层存储
□ 监控：GC 停顿、under-replicated partitions、磁盘使用率
```

> 📌 **全章最核心的一句话**：安装 Kafka 不是"部署一个消息中间件"，而是"为分布式日志这个第一性抽象选择最合适的物理载体"。书里第2版关于 ZooKeeper 的安装步骤已经成为历史，但关于硬件选型、配置取舍、机架感知、OS 调优的洞察，依然是 2025 年工业界的最佳实践。

