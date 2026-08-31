
# 第 12 章　云端的 MySQL——读书笔记

> 💡 这一章的本质，不是教你"Aurora 是什么"或"Cloud SQL 怎么用"，而是建立一个**第一性原理**：云端 MySQL 的根本矛盾在于**你用一部分控制权，去交换运维安全边界和弹性**。当你把 MySQL 搬到云上，无论是托管服务还是虚拟机自建，你买的都不是"一台数据库"，而是**一组关于控制权、运维责任、性能天花板、成本结构的权衡**。
> 
> 正如 Silvia Botros 和 Jeremy Tinley 在第 12 章的核心判断——云端 MySQL 的决策树起点永远是："你想自己管多少？" 2026 年的工业现实进一步验证了这一点：Aurora 已成为 AWS 上 MySQL 兼容工作负载的主流选择，Cloud SQL 在 GCP 上持续进化到 MySQL 9.7，而虚拟机自建依然是那些需要深度内核调优、特殊插件、或与现有基础设施紧耦合场景的唯一出路。

---

## 🎯 本章核心脉络

这一章埋下了一条主线——**云端的 MySQL 部署，本质是"控制权与运维责任的交易"**：

1. **托管 MySQL**（Aurora / Cloud SQL / RDS）：用控制权换运维安全边界
2. **虚拟机上的 MySQL**：用运维责任换控制权和成本效率
3. **机器类型选择**：读 QPS 看内存与 buffer pool，写 TPS 看块存储 IOPS
4. **磁盘类型选择**：云端 MySQL 的瓶颈常从 CPU 转移到网络与存储配额
5. **额外建议**：透明大页、文件系统挂载选项、备份策略、监控——这些"老规矩"在云上依然成立

---

## 托管 MySQL

### 第一性原理视角

托管 MySQL 服务的本质：**云厂商接管了"数据库生命周期管理"的一切脏活——安装、补丁、备份、故障切换、版本升级、监控**——而你保留 SQL 接口和大部分参数调优能力。

📌 **洞见 1：托管服务的真正价值不是"免运维"，而是"运维责任的清晰边界"**

|责任维度|自建 MySQL|托管 MySQL|
|---|---|---|
|硬件故障|你负责|云厂商负责|
|OS 补丁|你负责|云厂商负责|
|MySQL 版本升级|你负责|云厂商负责（可配置自动升级）|
|备份与 PITR|你负责|云厂商自动执行|
|故障切换|你负责（Orchestrator/MHA）|云厂商自动切换|
|参数调优|完全控制|受托管平台白名单限制|
|插件与存储引擎|完全自由|受平台支持列表限制|
|网络拓扑|完全控制|受 VPC/私有连接约束|

📌 **洞见 2：2026 年主流托管 MySQL 服务矩阵**

|云平台|服务名|最新版本|关键能力|
|---|---|---|---|
|AWS|Amazon Aurora MySQL-Compatible|8.4（对齐社区 LTS）|计算存储分离、共享存储、最多 15 副本、Global Database、Serverless v2、Parallel Query、I/O-Optimized 定价|
|AWS|Amazon RDS for MySQL|8.4|传统主从、Multi-AZ、只读副本|
|GCP|Cloud SQL for MySQL|9.7（Enterprise Plus 默认）|向量检索、Hypergraph 优化器、JSON Duality Views、Private Service Connect、Performance Capture|
|Azure|Azure Database for MySQL|8.x|灵活服务器、只读副本|

> 💡 **选择正确的托管服务层级**：AWS 上 Aurora 与 RDS 并非简单替换——Aurora 的计算存储分离架构带来显著的性能与可用性优势，但价格也更高；RDS 是更经济的传统主从架构。GCP 上 Cloud SQL 的 Enterprise Plus 版本提供了更高的性能和企业特性。

📌 **洞见 3：托管服务的"隐性约束"**

- **参数白名单**：云厂商只暴露经过验证的参数，某些深度调优参数不可修改
- **插件限制**：无法安装自定义插件（如特定审计插件、存储引擎）
- **版本滞后**：托管平台通常滞后社区版本数月到一年（Aurora 承诺在社区 LTS 发布后 12 个月内跟进主要版本）
- **网络隔离**：必须通过 VPC、Private Service Connect 等机制接入，无法物理接触
- **出口锁定**：数据量大时，从托管服务导出数据可能产生高昂的网络费用

---

## Amazon Aurora for MySQL

### 第一性原理视角

Aurora 不是"RDS 的升级版"，而是**为云重新设计的数据库引擎**——它的核心创新是**计算与存储的彻底分离**：

> 📌 **Aurora 架构的本质**：计算层（DB 实例）与存储层（分布式、日志结构的存储服务）解耦。存储层由分布式的、日志结构的存储服务组成，数据以 10GB 为段（segment）在三个可用区各复制两份，共六副本，写入以四副本法定多数（quorum）确认。

### 核心架构特性

**1. 共享存储卷**

所有实例（1 个 writer + 最多 15 个 reader）**共享同一个底层存储卷**。这意味着：

- 副本不产生额外存储成本
- 副本滞后通常远低于传统复制（存储层即复制）
- 新增副本是"秒级"操作，无需数据拷贝

**2. 计算存储分离的红利**

- 存储自动增长：以 10GB 为增量，最大 128 TiB
- 按实际使用量计费，而非预置容量
- I/O 使用分布式系统的 quorum 技术提升性能一致性

**3. 性能数据**

AWS 官方数据：Aurora MySQL 提供**最高相当于标准 MySQL 在同等硬件上 5 倍的吞吐量**。这源于：

- 写入减少到存储系统的次数
- 最小化锁竞争
- 消除数据库进程线程造成的延迟

### 关键能力矩阵（2026 年）

**高可用与持久性**

- 默认跨三 AZ 的六副本复制
- 实例故障自动重启，无需 redo 日志回放
- 缓冲区缓存与数据库进程隔离，重启后缓存存活
- Multi-AZ 部署 + 自动故障切换至 15 个副本之一
- 使用 RDS Proxy 进一步减少故障切换时间

**读扩展**

- 最多 15 个 Aurora Replicas，共享底层存储
- Reader Endpoint 自动负载均衡
- Cross-Region 副本支持全球本地读
- 自定义端点（Custom Endpoints）将不同工作负载路由到不同配置的实例

**全球分布**

- Aurora Global Database：单一数据库跨多个 AWS 区域
- 存储层复制，区域间典型低延迟
- 次级区域可在 1 分钟内提升为主区域，用于跨区域故障转移

**Serverless**

- Aurora Serverless v2：基于应用需求自动启停和扩缩容
- 与预置实例可混用在同一集群
- 适合波动或不可预测的工作负载

**Parallel Query（并行查询）**

这是 Aurora MySQL 最具创新性的特性之一：

> 📌 **第一性原理**：传统 MySQL 查询将所有扫描数据传回单一节点处理。Aurora Parallel Query 将 I/O 密集型工作（行检索、列提取、WHERE/JOIN 条件过滤）**下推到存储层的数千个 CPU 并行处理**，头节点只接收已过滤的紧凑结果集。

带来的收益：

- 分析查询提速**最高达两个数量级**
- 减少网络流量（只传输结果列，不传输整个数据页）
- 减少头节点 CPU 消耗
- **不污染 buffer pool**（并行查询处理的数据页不进入 buffer pool，避免驱逐热数据）
- 无需修改 SQL——查询优化器自动决定是否使用

**I/O-Optimized 配置**

- 替代按 I/O 请求计费的模式
- 对于 I/O 成本超过总成本 25% 的工作负载，可显著节约成本

**DevOps Guru for RDS**

- ML 驱动的异常检测
- 自动识别性能问题的根因
- 提供修复建议，"在数分钟而非数天内"解决问题

### Aurora 的适用场景

📌 **洞见：Aurora 不是万能药**

**适合 Aurora**：

- 高并发 OLTP 工作负载（Aurora 在高并发下性能优势最显著）
- 需要快速读扩展（共享存储使新增副本秒级完成）
- 全球分布需求（Global Database）
- 读写比极高（副本不占额外存储）
- 波动/不可预测负载（Serverless v2）
- 分析查询需要近实时数据（Parallel Query）

**不适合 Aurora**：

- 成本极度敏感且负载稳定（RDS 或自建更经济）
- 需要深度参数调优或自定义插件
- 数据量小、QPS 低（Aurora 的最低成本门槛较高）
- 需要特定 MySQL 版本（Aurora 版本滞后于社区）

### Aurora 的定价结构

Aurora 按四个维度计费：

1. **实例小时数**：基于实例等级和定价模式（Standard 或 I/O-Optimized）
2. **存储**：按实际消耗 GB-月计费（非预置）
3. **I/O 请求**：Standard 模式下按百万 I/O 请求计费
4. **备份存储**：免费额度内（等于数据大小）

> ⚠️ **成本陷阱**：Aurora Standard 模式下，I/O 请求是隐藏的成本大头。对于 I/O 密集型工作负载，I/O-Optimized 模式可能反而更便宜。

---

## GCP Cloud SQL

### 第一性原理视角

Cloud SQL 是 GCP 上的托管 MySQL 服务，2026 年已发展到支持 MySQL 9.7，并且**默认创建即为 Cloud SQL Enterprise Plus 版本**。

### 最新特性（截至 2026 年 8 月）

**MySQL 9.7 支持**（2026 年 8 月 13 日 GA）：

- **Vector Search**：支持社区标准的向量存储格式 + 高级近似最近邻（ANN）向量索引
- **Hypergraph Optimizer**：面向复杂多表查询的替代连接规划框架
- **JSON Duality Views**：桥接关系型 SQL 与层次化 JSON 文档模型
- 从 Cloud SQL for MySQL 8.4 原地大版本升级
- 通过 Database Migration Service (DMS) 从 8.4 迁移

> 📌 **洞见：MySQL 9.7 在 Cloud SQL 上的默认认证插件变更**
> 
> `mysql_native_password` 在 9.7 中不再受支持，所有客户端必须改用 `caching_sha2_password`。这是迁移到 9.7 时的关键兼容性检查点。

**性能与运维能力**：

- **Performance Capture**（GA）：自动采集数据库和 OS 指标快照并路由到 Cloud Logging，支持自定义阈值自动终止长事务，新增 6 个触发维度（高 CPU、高内存、高临时文件、历史列表长度、信号量等待、事务锁等待）
- **Managed Buffer Pool**（GA）：内存使用率高时自动缩减 `innodb_buffer_pool_size`，避免 OOM
- **Resource Groups**：MySQL 资源组，防止次要工作负载消耗过多 CPU/内存
- **Parameterized Secure Views**：基于会话变量的参数化安全视图
- **CMEK 重新加密零停机**：客户托管加密密钥的重新加密现在就地完成，零停机

**网络与安全**：

- **Private Service Connect 自动化**：DNS 自动化 GA，可启用全局写端点 DNS 自动解析到当前主实例
- **Connection Reconciliation**：默认启用，从移除的项目来的连接立即关闭
- **Secret Manager 认证**：Data API 执行 SQL 时可通过 Secret Manager 认证

**AI 与自动化**：

- **Remote MCP Server**：支持 `/readonly`、`/instance_manage`、`/query_execution` 专用工具集
- **Conversational Analytics**：自然语言查询运营数据
- **Gemini Cloud Assist**：AI 辅助故障排查（Premium Support 合约要求）

### Cloud SQL 的版本策略

📌 **洞见：Cloud SQL 的版本演进路径**

```
MySQL 8.0.45（默认次版本）
    ↓
MySQL 8.4.10（LTS）
    ↓
MySQL 9.7（Enterprise Plus 默认）
```

GCP 的演进策略比 AWS Aurora 更激进——直接拥抱 MySQL 9.7 的最新特性（向量检索、Hypergraph 优化器）。

### Cloud SQL vs Aurora 的核心差异

|维度|Aurora|Cloud SQL|
|---|---|---|
|架构|计算存储分离、共享存储|传统主从 + 托管运维|
|最新版本|8.4（对齐社区 LTS）|9.7（Enterprise Plus 默认）|
|读扩展|15 副本共享存储，亚秒级滞后|只读副本|
|全球分布|Global Database，存储层复制|跨区域副本|
|分析能力|Parallel Query|向量检索、Hypergraph 优化器|
|Serverless|Serverless v2|暂无等效能力|
|成本模式|实例 + 存储 + I/O 请求|实例 + 存储 + 网络|
|AI 集成|DevOps Guru|Gemini Cloud Assist、MCP Server|

> 💡 **选型判断**：Aurora 胜在架构创新和全球分布；Cloud SQL 胜在版本前瞻性和 GCP 生态集成。

---

## 虚拟机上的 MySQL

### 第一性原理视角

虚拟机自建 MySQL 的本质：**你买的是完全的控制权，代价是承担全部的运维责任**。

📌 **洞见：什么时候选自建而不是托管？**

1. **需要深度内核调优**：修改 `innodb_*` 参数白名单之外的配置
2. **需要自定义插件**：特定审计插件、存储引擎、UDF
3. **成本极度敏感**：大规模部署下，预留实例 + 自建可能比托管节省 30-50%
4. **合规与隔离**：必须物理隔离或控制底层 OS
5. **特殊工作负载**：需要特殊文件系统、内核参数、CPU 亲和性调优
6. **混合部署**：MySQL 与强耦合的中间件共同部署

### 云上的机器类型

GCP 官方文档给出的最新建议（2026 年）：

**性能敏感工作负载**（业务关键型数据库）：

- **C4 和 C4A 实例**（最新一代，基于 Titanium + 最新 Intel/AMD/Axion 处理器）
- 提供最低且最一致的延迟
- 特性：主机维护事件提前通知、单核睿频控制、Tier_1 网络
- 备选：C3 和 C3D 实例

**成本优化工作负载**（低到中等流量）：

- **N4 实例**：使用下一代动态资源管理优化总成本
- 适合开发/测试环境或低流量数据库
- 可配合扩展内存的自定义机器类型，进一步提高 RAM/vCPU 比

📌 **洞见 1：读 QPS 与写 TPS 的硬件需求根本不同**

**读 QPS 主导**：

- 关键因素是 **RAM**——buffer pool 必须容纳工作集
- "Choose a VM size that ensures that the working set fits into the buffer pool"
- 经验法则：buffer pool 大小 ≈ 工作集大小，占用 VM RAM 的 70-80%

**写 TPS 主导**：

- 关键因素是 **块存储的 IOPS 和延迟**
- 高写 TPS 性能的主要考虑因素是块存储
- 需要低延迟、高耐久性的 SSD

📌 **洞见 2：工作集必须进内存**

GCP 官方建议原文："we strongly recommend that you use MySQL's RAM-based buffer pool to cache hot data and reduce disk accesses... Choose a VM size that ensures that the working set, or total amount of data that your database processes at once, fits into the buffer pool."

这意味着：

- 如果工作集是 100GB，那么 VM 内存应该 ≥ 128GB（留出 OS 和其他进程开销）
- buffer pool 设置为 RAM 的 70-80%（128GB RAM → `innodb_buffer_pool_size = 100G`）
- 工作集不进内存 → 性能悬崖式下降

### 选择正确的机器类型

📌 **洞见：机器类型选择的决策框架**

```
工作负载特征分析
    │
    ├── 性能敏感 + 计算密集？
    │   └── C4/C4A（Intel/AMD/Axion 最新代）
    │
    ├── 性能敏感 + 内存密集？
    │   └── C4A/C3/C3D + Local SSD（本地 NVMe）
    │
    ├── 成本敏感 + 低到中等流量？
    │   └── N4（动态资源管理）
    │
    └── 大工作集 + 低 QPS？
        └── N4 + 扩展内存自定义机型（提高 RAM/vCPU 比）
```

**特殊场景：Local SSD 的使用**

- C4A、C3、C3D 实例支持 Local SSD
- 适用于需要极低延迟的特定性能需求
- 注意：Local SSD 是临时的，需要配合持久化块存储使用

### 选择正确的磁盘类型

GCP 的明确建议：**Hyperdisk Balanced 适合绝大多数 MySQL 工作负载**。

📌 **洞见：云端磁盘选择的"性能-成本"权衡**

|磁盘类型|适用场景|IOPS|延迟|
|---|---|---|---|
|Hyperdisk Balanced|绝大多数 MySQL 工作负载|中高|低|
|Hyperdisk Extreme|高 IOPS 需求（高写 TPS）|极高|极低|
|Hyperdisk Throughput|大批量顺序读写|中|中|
|Local SSD|临时表、缓冲扩展|最高|最低|

**关键配置原则**：

- 数据盘与系统盘分离
- 数据盘使用 Hyperdisk Balanced 起步
- 高写负载切换到 Hyperdisk Extreme
- 利用 Local SSD 扩展缓冲能力（如 Aurora Optimized Reads 的思路）

### 额外的建议

📌 **洞见：云端 MySQL 的"老规矩"依然成立**

**1. 操作系统层优化**

- 关闭 Transparent Huge Pages (THP)：减少内存管理开销
- 文件系统挂载使用 `relatime` 和 `lazytime`：减少 atime 更新开销
- 使用 XFS 文件系统

**2. MySQL 配置优化**

- `innodb_buffer_pool_size`：设为 VM RAM 的 70-80%
- `innodb_buffer_pool_instances`：多实例减少锁竞争
- 使用 MySQL 8.0 或更高版本
- 根据工作负载调整 `innodb_io_capacity`

**3. 网络优化**

- 大多数情况使用默认网络带宽即可
- 性能敏感场景升级到 Tier_1 网络
- 使用 VPC 内部通信，避免公网暴露

**4. 监控与运维**

- 利用云厂商的监控服务（Cloud Monitoring、Cloud Logging）
- 配置自动备份和 PITR
- 定期演练故障切换

**5. 成本优化**

- 使用抢占式实例（preemptible）用于非生产环境
- 利用承诺使用折扣（Committed Use Discounts）
- 定期审查实例利用率，right-size 机器类型

---

## 云端 MySQL 的工业实践趋势（2026 年）

📌 **洞见 1：托管服务与自建的边界在模糊**

- AWS 的 Aurora Serverless v2 + RDS Proxy 让托管服务具备接近自建的灵活性
- GCP 的 Cloud SQL 通过 Enterprise Plus、Managed Buffer Pool、Performance Capture 缩小与自建的运维差距
- 自建阵营通过 Terraform + Ansible + Prometheus 自动化缩小与托管的运维差距

📌 **洞见 2：多云平台策略成为大型企业的标配**

- 避免厂商锁定
- 利用各云优势（AWS 的 Aurora 架构、GCP 的 AI 集成）
- Vitess 等中间件使得跨云 MySQL 部署成为可能

📌 **洞见 3：AI 驱动的数据库运维成为新常态**

- AWS DevOps Guru for RDS
- GCP Gemini Cloud Assist、Conversational Analytics
- 这些 ML 驱动的洞察将"DBA 经验"产品化

📌 **洞见 4：向量检索与 AI 工作负载融合**

- Cloud SQL for MySQL 9.7 原生支持 Vector Search
- Aurora 通过 ML 集成调用 SageMaker
- MySQL 正在从纯 OLTP 向"OLTP + 向量 + 文档"多模演进

---

## 小结

这一章表面讲的是"云端的 MySQL 怎么选"，深层传递的是**云时代的数据库架构哲学**：

**1. 云端 MySQL 的本质是控制权与运维责任的交易**

|选择|你获得|你放弃|
|---|---|---|
|托管服务（Aurora/Cloud SQL）|运维安全边界、弹性、高可用|深度控制权、参数自由度|
|虚拟机自建|完全控制权、成本效率|运维责任、高可用需自建设计|

**2. Aurora 的架构创新定义了云原生数据库的标杆**

- 计算存储分离 + 共享存储卷
- 六副本跨三 AZ 的 quorum 写入
- 最多 15 个共享存储的读副本
- Parallel Query 将计算下推到存储层
- Global Database 实现跨区域低延迟复制
- Serverless v2 实现细粒度自动扩缩
- I/O-Optimized 定价模式应对 I/O 密集型负载

**3. Cloud SQL 的版本前瞻性独树一帜**

- 率先支持 MySQL 9.7（2026 年 8 月）
- Enterprise Plus 默认启用
- Vector Search、Hypergraph Optimizer、JSON Duality Views
- Performance Capture、Managed Buffer Pool 等企业级运维能力
- Private Service Connect + DNS 自动化

**4. 虚拟机自建的硬件选择原则**

- **读 QPS 主导**​ → 内存优先，工作集必须进 buffer pool
- **写 TPS 主导**​ → 块存储 IOPS 优先，Hyperdisk Balanced 起步
- **性能敏感**​ → C4/C4A 最新代机器族
- **成本敏感**​ → N4 动态资源管理
- **大工作集低 QPS**​ → 扩展内存自定义机型

**5. 云端 MySQL 的"不变真理"**

- buffer pool 必须容纳工作集（云上依然成立）
- THP 必须关闭
- 数据盘与系统盘分离
- 监控、备份、故障切换演练缺一不可

> 💡 **贯穿全章的洞见**：云端 MySQL 的选型，本质是在**"控制权-运维责任-成本-性能"四维空间**中寻找你业务的最优解。优秀的云端 MySQL 架构，始于对工作负载的深刻理解（读主导还是写主导？工作集多大？QPS 峰值多少？），终于对部署形态的精准选择——既不盲目拥抱托管服务而丧失控制权，也不固执自建而承担不必要的运维负担。
> 
> 2026 年的工业现实是：
> 
> - **绝大多数新项目**​ → 直接选择托管服务（Aurora 或 Cloud SQL）
> - **大规模、成本敏感、需深度调优**​ → 虚拟机自建 + 自动化运维
> - **全球分布需求**​ → Aurora Global Database 或 Cloud SQL 跨区域副本
> - **AI/向量工作负载**​ → Cloud SQL for MySQL 9.7（原生向量检索）
> - **不可预测负载**​ → Aurora Serverless v2

正如书中所揭示的云端 MySQL 决策核心——**"How much do you want to manage yourself?"**​ 这个问题没有标准答案，但它引出的决策框架是永恒的：

```
1. 评估团队运维能力
2. 量化工作负载特征（读/写比例、工作集大小、QPS/TPS）
3. 明确可用性要求（SLA、RTO、RPO）
4. 计算总拥有成本（TCO）
5. 选择部署形态：
   ├── 运维能力弱 + 标准工作负载 → 托管服务
   ├── 运维能力强 + 特殊需求 → 虚拟机自建
   └── 全球分布 → Aurora Global Database / Cloud SQL 跨区域
```

📌 **给读者的实践清单**：

- ✅ 新项目优先考虑托管服务（Aurora / Cloud SQL）
- ✅ Aurora 选择依据：需要共享存储读扩展、Global Database、Parallel Query、Serverless
- ✅ Cloud SQL 选择依据：需要 MySQL 9.7 新特性、GCP 生态集成、向量检索
- ✅ 自建虚拟机选择 C4/C4A（性能）或 N4（成本）
- ✅ 读 QPS 主导 → 内存优先，工作集进 buffer pool
- ✅ 写 TPS 主导 → Hyperdisk Balanced/Extreme，关注 IOPS
- ✅ 关闭 THP，使用 `relatime`/`lazytime` 挂载选项
- ✅ `innodb_buffer_pool_size` 设为 VM RAM 的 70-80%
- ✅ 利用托管服务的自动备份和 PITR
- ✅ 生产环境使用 Multi-AZ / 跨 AZ 部署
- ✅ 使用私有网络连接（Private Service Connect / VPC Peering）
- ✅ 监控 buffer pool 命中率、复制延迟、磁盘 IOPS
- ❌ 不要用突发性能实例（t 系列）运行生产 MySQL
- ❌ 不要忽视 I/O 成本（Aurora Standard 模式的隐性成本）
- ❌ 不要将工作集远大于内存的实例依赖 buffer pool
- ❌ 不要在云端使用本地盘（Local SSD）作为唯一持久化存储
- ❌ 不要将数据库暴露在公网

**云端 MySQL 是云时代数据库架构的必修课，课业精通，系统才能在弹性、成本、控制权的三角中寻找最优平衡。一个优秀的云端 MySQL 架构，是用工作负载分析确定部署形态、用机器类型匹配性能需求、用磁盘类型优化 I/O 瓶颈、用托管服务降低运维负担——这四者的精妙平衡，正是本章"云端的 MySQL"的灵魂所在。理解了云端 MySQL 的本质，你就理解了为什么现代数据库架构（Aurora Serverless + Global Database + Parallel Query）全都围绕"计算存储分离"这一核心创新：因为云的本质是"弹性"，而传统 MySQL 的紧耦合架构无法充分利用云的弹性。Aurora 的计算存储分离不仅是技术革新，更是哲学革新——它重新定义了"什么是云原生数据库"。这正是为什么本章把"云端的 MySQL"放在"扩展 MySQL"之后——没有对云端部署形态的理解，扩展策略就无法在云上落地；而没有扩展的思维，云端 MySQL 也只是把单机数据库搬到了虚拟机里，浪费了云的真正价值。两者合一，方能成就真正云原生的 MySQL 架构。**