# 第7章 OpenFlow 1.1 — 读书笔记

如果 OpenFlow 1.0 是"能用"，那么 1.1（2011 年 2 月发布）就是"好用"的起点。**这一章的每一个改动，都是在为"把传统网络设备的复杂处理链搬进 OpenFlow"扫清障碍**。

作者在这一章真正想传达的是：**OpenFlow 1.1 不再是一个"单流表匹配引擎"，而是一个"可编程的包处理流水线"**。这个转变是整个 OpenFlow 协议史上最重要的一次架构升级。

---

## 7.1 OpenFlow 1.1 中的变更要点

从 1.0 到 1.1 不是小修小补，而是**架构级别的重构**：

```
OpenFlow 1.0 架构:
┌────────────────────────────────────┐
│  单流表                             │
│  Match: 12元组(固定)               │
│  Action: 输出/丢弃/修改(4种)        │
│  一张表搞定所有事情                  │
└────────────────────────────────────┘

OpenFlow 1.1 架构:
┌────────────────────────────────────────────┐
│  多级流表(流水线)                            │
│  Table 0 → Table 1 → ... → Table N         │
│  匹配字段: 扩展(加入 Metadata/MPLS/VLAN)    │
│  指令(Instructions)代替动作(Actions)         │
│  行动集(Action Set)累积跨表动作              │
│  组表(Group Table)                          │
│  TTL 操作(Copy/Decrement/Set)               │
│  虚拟端口扩展                                │
└────────────────────────────────────────────┘
```

> 💡 **第一性原理**：1.0 的单流表模型本质上是"一张巨大的二维表"——行是匹配规则，列是动作。**它的根本缺陷是"叉乘爆炸"**：当你需要组合 N 种匹配条件和 M 种动作时，需要 N×M 条流表项。1.1 的多级流表把这种"叉乘"拆解成"流水线串联"——每个阶段只处理一个维度，表项数量从 O(N×M) 降到 O(N+M)。

---

## 7.2 匹配字段的变更

1.1 在 1.0 的 12 元组基础上做了扩展：

```
OpenFlow 1.0 的匹配字段(12元组):
┌─────────────────────────────────────────┐
│  Ingress Port                            │
│  Ethernet: src/dst/type                  │
│  VLAN ID / VLAN PCP                      │
│  IPv4: src/dst/proto/ToS                 │
│  TCP/UDP: src_port/dst_port              │
│  ICMP: type/code                         │
└─────────────────────────────────────────┘

OpenFlow 1.1 新增:
┌─────────────────────────────────────────┐
│  Metadata (元数据字段，跨表传递)           │
│  MPLS: label / traffic class             │
│  SCTP 端口                               │
│  ECN 字段                                │
│  Tunnel ID (逻辑端口标识)                 │
└─────────────────────────────────────────┘
```

**关键认知**：Metadata 字段的引入是 1.1 最微妙但最重要的变更之一。它为后续的多级流表流水线提供了"跨表信息传递"的通道——Table 0 写入的 Metadata，Table 1+ 可以读取并作为匹配条件。

---

## 7.3 多流表规范的变更（流水线处理）

### 7.3.1 流水线处理

这是 1.1 最核心的架构创新。包处理不再是"匹配一张表→执行动作"，而是"依次经过多张表，逐步累积动作，最后统一执行"：

```
┌──────────────────────────────┐
                    │   Action Set (初始为空)        │
                    │   Metadata   (初始为空)        │
                    └──────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│  Table 0                                                 │
│  Match: in_port=1, eth_type=0x0800                      │
│  Instruction: Write-Metadata(tenant=100),                │
│               Write-Actions(push_vlan),                  │
│               Goto-Table 1                              │
└──────────────────────────────────┬─────────────────────┘
                                   │ Goto-Table 1
                                   ▼
┌─────────────────────────────────────────────────────────┐
│  Table 1                                                 │
│  Match: metadata=tenant=100, ip_dst=10.0.0.0/8          │
│  Instruction: Write-Actions(set_dl_dst=GW_MAC),         │
│               Goto-Table 2                              │
└──────────────────────────────────┬─────────────────────┘
                                   │ Goto-Table 2
                                   ▼
┌─────────────────────────────────────────────────────────┐
│  Table 2                                                 │
│  Match: ip_dst=10.0.0.0/8                               │
│  Instruction: Write-Actions(output:port_5)              │
│               (无 Goto-Table → 流水线结束)               │
└──────────────────────────────────┬─────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  执行 Action Set:             │
                    │  1. Copy TTL inwards          │
                    │  2. Pop tags                  │
                    │  3. Push VLAN tag             │
                    │  4. Copy TTL outwards         │
                    │  5. Decrement TTL             │
                    │  6. Set fields                │
                    │  7. QoS (set_queue)           │
                    │  8. Group (若有)              │
                    │  9. Output: port_5            │
                    └──────────────────────────────┘
```

**防环路机制**：Goto-Table 指令只能跳到**编号更大**的表。这保证了流水线必然终止——不能跳回去形成环。

> 💡 **洞见**：流水线处理的本质是把"网络功能"建模为一个**有限状态机**——每个流表是一组状态转移规则，Metadata 是状态，Goto-Table 是状态转移。这种建模方式的威力在于：**任意复杂的包处理逻辑都可以分解为多级简单匹配的串联**。

**与 1.0 的对比**：

|维度|OpenFlow 1.0|OpenFlow 1.1+|
|---|---|---|
|流表数量|1 张|最多 256 张|
|匹配结果|直接执行动作列表|累积到 Action Set，流水线结束时执行|
|动作执行时机|匹配后立即执行|可立即执行(Apply-Actions)或累积(Write-Actions)|
|跨表信息|无|Metadata 字段传递|
|表项数量|O(N×M) 叉乘|O(N+M) 流水线|

### 7.3.2 元数据（Metadata）

Metadata 是一个 64 位的字段，专门用于在流水线各阶段之间传递信息：

```
Metadata 的典型用途:
┌──────────────────────────────────────────────┐
│  Bits 0-15:   租户 ID (Tenant ID)             │
│  Bits 16-31:  输入逻辑端口 (Input Logic Port)  │
│  Bits 32-47:  服务链 ID (Service Chain ID)     │
│  Bits 48-63:  自定义标签                       │
└──────────────────────────────────────────────┘

Table 0: 写入 Metadata (tenant=100)
Table 1: 匹配 Metadata.tenant=100 → 执行租户特定逻辑
Table 2: 匹配 Metadata.service_chain=5 → 执行服务链转发
```

**工业价值**：Metadata 让"网络虚拟化中的切片标识"、"服务链中的路径标识"等抽象可以在数据面原生表达，而不需要额外的隧道封装。这是后来 Network Service Header (NSH) 等标准的思想雏形。

### 7.3.3 OpenFlow 1.1 中的自学习桥接器的实现手法

在第4章我们用 1.0 实现了自学习桥，1.1 的多级流表让这个实现更加优雅：

```
1.0 的自学习桥: 单流表，MAC 学习需要控制器介入每条新流

1.1 的自学习桥(流水线优化):
┌─────────────────────────────────────────────┐
│ Table 0: 入口分类                             │
│   Match: in_port=1                           │
│   Instruction: Write-Metadata(ingress=1),    │
│                Goto-Table 1                  │
├─────────────────────────────────────────────┤
│ Table 1: MAC 学习                             │
│   Match: eth_src=XX:XX:XX:XX:XX:XX           │
│   Instruction: Write-Metadata(learned_mac), │
│                Goto-Table 2                  │
│   (若未匹配 → Packet-In 上送控制器学习)        │
├─────────────────────────────────────────────┤
│ Table 2: MAC 查找 + 转发                       │
│   Match: eth_dst=YY:YY:YY:YY:YY:YY           │
│   Instruction: Write-Actions(output:port_X) │
└─────────────────────────────────────────────┘
```

**核心优势**：MAC 学习逻辑（Table 1）与转发逻辑（Table 2）解耦。控制器只需要下发"MAC→端口"的映射流表项到 Table 2，而 Table 0/1 的分类和学习逻辑可以预先静态配置。这大大减少了控制器与交换机的交互次数。

---

## 7.4 指令（Instructions）

### 7.4.1 何谓指令

1.1 引入了"指令"概念，**取代 1.0 中流表项直接关联"动作列表"的做法**。指令是流表项的执行单位，它决定了"匹配成功后做什么"：

```
1.0 流表项: Match → Action List (立即执行)
1.1 流表项: Match → Instructions (可能是跳表/累积动作/立即执行)
```

### 7.4.2 行动、行动集、行动列表、指令的区别

这是 1.1 最容易混淆的概念群，必须厘清：

```
┌──────────────────────────────────────────────────────┐
│  指令(Instruction)                                    │
│  ── 流表项的执行单元，决定"做什么类型的事"              │
│  ── 类型: Goto-Table / Write-Actions /                │
│          Apply-Actions / Clear-Actions /              │
│          Write-Metadata / Meter                       │
├──────────────────────────────────────────────────────┤
│  行动列表(Action List)                                │
│  ── Apply-Actions 指令携带的"立即执行的动作序列"        │
│  ── 语义同 1.0 的 Action List                         │
│  ── 执行时机: 流水线仍在处理中，立即执行                │
├──────────────────────────────────────────────────────┤
│  行动集(Action Set)                                   │
│  ── 跨流表累积的动作集合                               │
│  ── Write-Actions 指令向其中合并动作                   │
│  ── Clear-Actions 指令清空它                          │
│  ── 执行时机: 流水线结束时按固定顺序执行                │
│  ── 每类动作最多一个(后写的覆盖先写的)                 │
├──────────────────────────────────────────────────────┤
│  行动(Action)                                         │
│  ── 最小执行单元: output / set_field / push_tag 等     │
└──────────────────────────────────────────────────────┘
```

**Action Set 的执行顺序**（这是规范强制的，与写入顺序无关）：

```
1. Copy TTL inwards
2. Pop tags (VLAN/MPLS)
3. Push tags (VLAN/MPLS)
4. Copy TTL outwards
5. Decrement TTL
6. Set fields
7. QoS (set_queue)
8. Group (若指定)
9. Output (若指定，且无 Group)
```

> ⚠️ **关键规则**：如果 Action Set 中同时有 Group 和 Output，Group 优先，Output 被忽略。如果两者都没有，包被丢弃。

### 7.4.3 对行动的变更

1.1 对行动做了重要调整：

- **Set-Field 成为通用的字段修改机制**（为 1.2 的 OXM 铺垫）
- **QoS 的 set_queue 从 output 动作中解耦**——成为一个独立动作
- **TTL 操作成为一等公民**：Copy TTL inwards/outwards, Decrement TTL, Set TTL
- **Push/Pop VLAN 和 MPLS 标签**

```
1.0 的 output 动作:
  Action: output:port, queue:qid  (队列捆绑在输出里)

1.1 的 Action Set:
  Write-Actions: set_queue:qid, output:port  (队列独立)
```

这种解耦让"QoS 队列选择"和"输出端口选择"可以独立决策，分别在不同的流表项中指定。

---

## 7.5 组（Group）

### 7.5.1 组表

组表是 1.1 的第二大架构创新。**它解决的问题是"多个流表项共享同一组动作"**——避免重复下发相同的动作桶到多个流表项。

```
没有组表时(1.0):
  Flow 1 → Action: output:port1, output:port2, output:port3
  Flow 2 → Action: output:port1, output:port2, output:port3  (重复!)
  Flow 3 → Action: output:port1, output:port2, output:port3  (重复!)

有组表时(1.1+):
  Group 100:
    Bucket 1: output:port1
    Bucket 2: output:port2
    Bucket 3: output:port3
  
  Flow 1 → Group:100
  Flow 2 → Group:100
  Flow 3 → Group:100
  (共享同一组动作)
```

### 7.5.2 组表项

每个组表项的结构：

```
┌──────────────────────────────────────────┐
│  Group Identifier (组ID)                   │
│  Group Type (组类型，见 7.5.3)              │
│  Counters (统计)                           │
│  Action Buckets[] (动作桶数组):              │
│    Bucket 1: { actions: [...] }            │
│    Bucket 2: { actions: [...] }            │
│    ...                                     │
└──────────────────────────────────────────┘
```

每个 Action Bucket 可以包含：

- 独立的 Watch Port / Watch Group（用于 Fast Failover 的健康检查）
- 一组动作（output, set_field, push/pop 等）

### 7.5.3 组类型

1.1 定义了四种组类型，分别对应不同的工业场景：

**① ALL 组（广播/多播）**

```
Group 100, Type=ALL
┌─────────────────────────────────────────┐
│  Bucket 1: output:port1 (→ Host A)      │
│  Bucket 2: output:port2 (→ Host B)      │
│  Bucket 3: output:port3 (→ Host C)      │
└─────────────────────────────────────────┘
语义: 包复制到所有 Bucket，分别输出
用途: L2 广播、多播、端口镜像
```

**② SELECT 组（负载均衡/ECMP）**

```
Group 200, Type=SELECT
┌─────────────────────────────────────────┐
│  Bucket 1: weight=1, output:spine1      │
│  Bucket 2: weight=1, output:spine2      │
│  Bucket 3: weight=2, output:spine3      │
└─────────────────────────────────────────┘
语义: 选择一个 Bucket 执行(基于哈希或权重)
用途: ECMP、LACP、服务负载均衡
```

**③ INDIRECT 组（间接引用）**

```
Group 300, Type=INDIRECT
┌─────────────────────────────────────────┐
│  Bucket 1: output:next_hop_port         │
└─────────────────────────────────────────┘
语义: 只有一个 Bucket，所有引用此组的流表项共享
用途: 多路径环境下，多个目标共享同一个下一跳
```

**④ FAST_FAILOVER 组（主备切换）**

```
Group 400, Type=FAST_FAILOVER
┌─────────────────────────────────────────┐
│  Bucket 1: watch_port=port1, output:port1│
│  Bucket 2: watch_port=port2, output:port2│
│  Bucket 3: watch_port=port3, output:port3│
└─────────────────────────────────────────┘
语义: 执行第一个 Watch Port 存活的 Bucket
用途: 主备切换、链路冗余(替代 VRRP/HSRP)
切换时间: 毫秒级，无需协议报文交互
```

> 💡 **工业洞见**：FAST_FAILOVER 组是 OpenFlow 对"高可用性"问题的优雅回答。传统的主备切换依赖 VRRP/HSRP 等协议，需要选举、心跳、超时检测——整个过程秒级。FAST_FAILOVER 组把"链路存活检测"下沉到交换机硬件（端口状态机），切换直接在数据面完成，控制器完全不需要介入。

### 7.5.4 组的组

组可以嵌套引用——一个组的 Bucket 可以指向另一个组：

```
Group 500 (Type=SELECT)
├── Bucket 1: Group:100  (ALL 组，多播到 A/B/C)
├── Bucket 2: Group:200  (SELECT 组，ECMP 到 Spine)
└── Bucket 3: Group:400  (FAST_FAILOVER 组，主备)

语义: 选择 Bucket 1 → 执行 Group 100 的所有 Buckets
      (递归展开为 output:port1, output:port2, output:port3)
```

**注意**：组嵌套的深度受限于交换机实现，规范未定义上限。工业实践中通常限制在 2-3 层以内，以避免流水线过长。

---

## 7.6 虚拟端口的扩展

1.1 对端口概念做了重要扩展——把端口分为三类：

```
┌──────────────────────────────────────────────┐
│  物理端口(Physical Port)                       │
│  ── 交换机硬件端口                              │
├──────────────────────────────────────────────┤
│  逻辑端口(Logical Port)                        │
│  ── 隧道端点(GRE/VXLAN/Geneve)                 │
│  ── 链路聚合(LAG)                              │
│  ── 与物理端口一一对应，但具有虚拟抽象           │
├──────────────────────────────────────────────┤
│  保留端口(Reserved Port)                       │
│  ── ALL: 转发到所有端口(除入端口)               │
│  ── CONTROLLER: 上送控制器                     │
│  ── TABLE: 重新进入流水线                       │
│  ── IN_PORT: 从入端口返回                       │
│  ── ANY/LOCAL: 本地CPU端口                     │
│  ── NORMAL: 按传统L2/L3处理(混合交换机)         │
│  ── FLOOD: 除入端口和阻塞端口外的所有端口        │
└──────────────────────────────────────────────┘
```

**工业价值**：逻辑端口的引入使得"隧道"成为 OpenFlow 的一等公民。VXLAN/GRE 隧道不再需要外部封装/解封装设备，交换机可以直接把流量引向"逻辑端口"完成隧道处理。

---

## 7.7 TTL 字段操作

### 7.7.1 Copy TTL inwards / Copy TTL outwards

**这是为 MPLS 和 IPv6 隧道设计的 TTL 处理**：

```
Copy TTL inwards (进入隧道时):
  MPLS label 推送时，把 IP TTL 复制到 MPLS TTL
  语义: 外层标签的 TTL 继承内层 IP 的 TTL

Copy TTL outwards (离开隧道时):
  MPLS label 弹出时，把 MPLS TTL 复制回 IP TTL
  语义: 内层 IP 的 TTL 继承外层标签的 TTL

典型场景:
┌────────────────────────────────────────────┐
│  IP Packet (TTL=64)                         │
│    │                                        │
│    │ Push MPLS label                        │
│    │ Copy TTL inwards → MPLS TTL=64          │
│    ▼                                        │
│  MPLS Packet (TTL=64)                       │
│    │ (经过 2 跳 MPLS 转发，每跳 TTL-1)        │
│    │ MPLS TTL=62                            │
│    │                                        │
│    │ Pop MPLS label                         │
│    │ Copy TTL outwards → IP TTL=62          │
│    ▼                                        │
│  IP Packet (TTL=62)  ← TTL 正确递减了 2      │
└────────────────────────────────────────────┘
```

### 7.7.2 接收到包含非法 TTL 值的数据包时的处理

规范定义：如果收到 TTL=0 或 TTL=1 的包尝试转发，交换机必须：

- **Decrement TTL 动作会导致 TTL 变为 0**​ → 包被丢弃，可以上送控制器（类似传统路由器的 ICMP Time Exceeded）

```
处理流程:
  1. 匹配流表项，发现 DECREMENT_TTL 动作
  2. 执行前检查: TTL 是否为 0 或 1?
  3. 若是 → 丢弃包，可选上送控制器(理由: TTL exceeded)
  4. 若否 → TTL 减 1，继续执行后续动作
```

### 7.7.3 不能实施 TTL 的匹配

**TTL 字段不能作为匹配条件**——这是 1.1 的明确限制。原因：

- TTL 是"易变字段"，每个包经过一跳就变化
- 基于 TTL 匹配会导致流表项爆炸（每个 TTL 值一条规则）
- 语义上，TTL 是"网络层防环机制"，不应参与策略匹配

> 💡 **洞见**：TTL 操作的完整引入（Copy/Decrement/Set），标志着 OpenFlow 从"L2/L3 转发器"正式升级为"运营商级路由器"。MPLS 网络的 TTL 处理是运营商网络的基础需求——没有它，MPLS 快速重路由（FRR）等机制无法实现。

---

## 7.8 OpenFlow 1.1 中其他的变更

### 7.8.1 支持 MPLS 标签和 VLAN 标签的 Push/Pop

```
Push VLAN: 在报文头前插入 802.1Q 标签
Pop VLAN:  移除最外层 802.1Q 标签
Push MPLS: 在报文头前插入 MPLS 标签
Pop MPLS:  移除最外层 MPLS 标签

工业价值:
  - VLAN Push/Pop → 运营商以太网服务(E-Line/E-LAN)
  - MPLS Push/Pop → MPLS LSP 转发、MPLS VPN
```

### 7.8.2 OpenFlow 混合交换机

1.1 引入了"混合交换机"概念——同时支持 OpenFlow 和传统 L2/L3 处理：

```
混合交换机的端口处理:
┌─────────────────────────────────────────┐
│  包到达                                  │
│    │                                    │
│    ├── 匹配 OpenFlow 流表?               │
│    │     Yes → 按 OpenFlow 流水线处理     │
│    │     No  → 按传统 L2/L3 处理         │
│    │           (NORMAL 保留端口)         │
│    ▼                                    │
│  输出                                    │
└─────────────────────────────────────────┘
```

**工业意义**：混合模式是 OpenFlow 商业化落地的关键——它允许运营商在现有网络上"渐进式"部署 SDN，而不需要一夜之间替换所有设备。

### 7.8.3 支持 SCTP

流表可以匹配 SCTP 端口号，并支持 SCTP 相关的字段修改。SCTP 是电信信令网（如 SS7 over IP）的核心传输协议。

### 7.8.4 支持 ECN

Explicit Congestion Notification——允许 OpenFlow 匹配和修改 IP ECN 字段。这是数据中心拥塞控制（如 DCTCP）的基础。

### 7.8.5 OpenFlow 交换机和控制器之间连接名称的变更

术语调整：

- 1.0: "TCP 连接"
- 1.1: "OpenFlow Channel"（OpenFlow 通道）

这个术语变更反映了架构认知的成熟——OpenFlow 通道不再局限于 TCP，可以为任何可靠的传输层。

### 7.8.6 紧急事态流缓存的取消

1.0 的"Emergency Flow Cache"被废弃，取而代之的是两种故障模式：

- **Fail Secure Mode**：控制器断开时，交换机保留现有流表项，停止接受新流
- **Fail Standalone Mode**：控制器断开时，交换机回退到传统 L2 交换模式

> 💡 **洞见**：紧急流缓存的取消，体现了 SDN 哲学的转变——"应急方案应该由控制器统一编排，而不是硬编码在交换机里"。Fail Secure/Standalone 两种模式给了运营商明确的故障行为预期。

### 7.8.7 Vendor 消息名称的变更

`Vendor` 更名为 `Experimenter`——为厂商扩展提供官方命名空间。这个变更在 1.2 中进一步发展为 OXM Experimenter 类（0xFFFF）。

---

# 第8章 OpenFlow 1.2 — 读书笔记

2011 年 12 月发布的 1.2，距离 1.1 仅 10 个月。**这一章的核心使命是"可扩展性"**——1.1 的流水线架构已经搭好，但匹配字段仍是"固定长度结构"，无法适应 IPv6 和未来新协议的需求。

作者在这一章想传达的是：**OpenFlow 必须从一个"专用协议"演变为"可编程协议的框架"**。

---

## 8.1 OpenFlow 1.2 中的变更点

三大核心变更：

1. **OXM（OpenFlow Extensible Match）**——可扩展匹配
2. **IPv6 支持**——完整的 IPv6 匹配与改写
3. **多控制器支持**——故障转移与负载均衡

---

## 8.2 OpenFlow eXtensible Match（OXM）

### 8.2.1 OXM TLV 的基本结构

**为什么需要 OXM？**​ 一个直观的数据：如果 1.1 的固定匹配结构要加入 IPv6 地址（带掩码），匹配字段长度会从 88 字节暴增到 150+ 字节。更要命的是——每要支持一种新的协议字段，就要修改整个 `ofp_match` 结构体，导致所有消息格式连锁变化。

OXM 的解决方案：**用 TLV（Type-Length-Value）结构替代固定结构**。

```
OXM TLV 格式:
┌──────────────────────────────────────────────┐
│  oxm_class: 16 bits  (匹配类)                  │
│  oxm_field: 7 bits    (类内字段)               │
│  oxm_hasmask: 1 bit   (是否有掩码)             │
│  oxm_length: 8 bits    (值的长度)              │
│  Value: 可变长度                                 │
│  Mask: 可选，若 hasmask=1                       │
└──────────────────────────────────────────────┘
```

**oxm_class 的取值**：

```
0x0000: OFPXMC_NXM_0    (兼容 Nicira NXM)
0x0001: OFPXMC_NXM_1    (兼容 Nicira NXM)
0x0002-0x7FFF: ONF 成员保留
0x8000: OFPXMC_OPENFLOW_BASIC  (标准 OpenFlow 字段)
0x8001: OFPXMC_PACKET_REGS  (OpenFlow 1.5+ 包寄存器)
0x8002-0xFFFE: OpenFlow 规范保留
0xFFFF: OFPXMC_EXPERIMENTER  (厂商实验性)
```

### 8.2.2 匹配字段解析规范的取消和 Pre-requisite

1.2 取消了"匹配字段解析规范"——原来诸如"要匹配 TCP 端口必须先匹配 IP 协议号"之类的硬性规定被移除。取而代之的是 **Pre-requisite（前置条件）**​ 机制：

```
Pre-requisite 示例:
  匹配 TCP dst_port 的前提:
    - 必须匹配 eth_type=0x0800 (IPv4) 或 0x86DD (IPv6)
    - 必须匹配 ip_proto=6 (TCP)
    
  匹配 VLAN PCP 的前提:
    - 必须匹配 VLAN 存在(即 eth_type=0x8100)
```

**工业价值**：Pre-requisite 让"协议字段的依赖关系"变得声明式——交换机可以动态验证流表项的合法性，而不是依赖硬编码的解析逻辑。这使得新协议字段的添加不需要修改交换机核心代码。

### 8.2.3 OXM 匹配字段

1.2 通过 OXM 新增的匹配字段：

```
L2/L3 扩展:
  ARP SHA/THA (ARP 硬件地址)
  IPv4 ECN
  
IPv6 完整支持:
  IPv6 src/dst (带 128 位掩码)
  IPv6 Flow Label (20 bits)
  IPv6 DSCP/ECN
  IPv6 Protocol
  ICMPv6 type/code
  ICMPv6 Neighbor Discovery target address
  ICMPv6 ND source/target Ethernet address
  TCP/UDP/SCTP ports over IPv6
```

### 8.2.4 OXM 中的通配符

OXM 通过 `oxm_hasmask` 位支持任意位掩码：

```
精确匹配:
  oxm_hasmask=0
  Value: 10.0.0.1
  含义: 精确匹配 IP=10.0.0.1

前缀匹配:
  oxm_hasmask=1, Mask: 255.255.255.0
  Value: 10.0.0.0
  含义: 匹配 10.0.0.0/24

任意位掩码:
  oxm_hasmask=1, Mask: 0xFF0F
  Value: 0x4500
  含义: 匹配 IP ToS 字段的特定位
```

> 💡 **洞见**：OXM 的 TLV 结构本质上是"自描述"的——每个匹配字段自带类型和长度，接收方不需要预先知道协议的所有字段定义。这意味着**老版本实现可以忽略不认识的 OXM TLV**，从而实现真正的向后兼容。这是协议设计中的"可扩展性原则"的经典案例。

### 8.2.5 OXM TLV 示例

```
示例 1: 匹配 IPv4 目的地址 10.0.0.0/8
┌─────────────────────────────────────────┐
│  oxm_class = 0x8000 (OPENFLOW_BASIC)     │
│  oxm_field = 14 (IPv4_DST)               │
│  oxm_hasmask = 1                         │
│  oxm_length = 8                          │
│  Value: 10.0.0.0                         │
│  Mask:  255.0.0.0                        │
└─────────────────────────────────────────┘

示例 2: 匹配 TCP 目的端口 80
┌─────────────────────────────────────────┐
│  oxm_class = 0x8000                      │
│  oxm_field = 27 (TCP_DST)                │
│  oxm_hasmask = 0                         │
│  oxm_length = 2                          │
│  Value: 80                               │
└─────────────────────────────────────────┘

示例 3: 匹配 IPv6 源地址 2001:db8::/32
┌─────────────────────────────────────────┐
│  oxm_class = 0x8000                      │
│  oxm_field = 34 (IPV6_SRC)               │
│  oxm_hasmask = 1                         │
│  oxm_length = 32                         │
│  Value: 2001:0db8:0000:... (16 bytes)    │
│  Mask:  ffff:ffff:0000:... (16 bytes)    │
└─────────────────────────────────────────┘
```

### 8.2.6 基于 OXM 的 Set-Field

1.2 引入了通用的 **Set-Field 动作**，复用 OXM 的 TLV 结构：

```
旧方式(1.0/1.1):
  Set-DL-DST: 修改以太网目的地址
  Set-NW-DST: 修改 IP 目的地址
  Set-TP-DST: 修改 TCP/UDP 目的端口
  ... (每种字段一个独立动作)

新方式(1.2+):
  Set-Field: <OXM TLV>
  (一个通用动作，可以修改任意 OXM 支持的字段)
```

**工业价值**：Set-Field + OXM 的组合意味着——**任何可匹配的字段都可被修改**。这为实现复杂的网络功能（NAT、L3 VPN、负载均衡的地址重写）提供了统一的原语。

### 8.2.7 取消 TCP、UDP、SCTP、ICMP 重载使用相同字段

1.0/1.1 中，TCP 端口、UDP 端口、ICMP 类型/代码共用同一组匹配字段（通过重载）。这种做法导致：

```
1.0 的问题:
  tp_src 字段既可能是 TCP src_port，也可能是 UDP src_port
  控制器必须自己记住"当前 ip_proto 是什么"
  交换机无法独立验证匹配的合法性
```

1.2 中，每个协议都有**独立的匹配字段类型**：

```
TCP_SRC, TCP_DST
UDP_SRC, UDP_DST
SCTP_SRC, SCTP_DST
ICMPV4_TYPE, ICMPV4_CODE
ICMPV6_TYPE, ICMPV6_CODE
```

> 💡 **洞见**：这个变更看似微小，实则重要。它体现了"显式优于隐式"的工程哲学——把协议语义从"上下文推断"改为"字段自描述"。这对硬件实现至关重要：ASIC 可以独立解析每个字段，而不需要维护"当前 L4 协议是什么"的状态。

---

## 8.3 支持基本的 IPv6

1.2 是 OpenFlow 协议**第一个完整支持 IPv6**的版本：

```
IPv6 匹配能力:
┌─────────────────────────────────────────┐
│  IPv6 src/dst (128-bit, 支持掩码)        │
│  IPv6 Flow Label (20 bits)               │
│  IPv6 DSCP + ECN                         │
│  IPv6 Next Header (协议号)                │
│  ICMPv6 type/code                        │
│  ICMPv6 Neighbor Discovery 地址          │
│  TCP/UDP/SCTP ports over IPv6            │
└─────────────────────────────────────────┘

IPv6 改写能力:
  Set-Field: 修改任意上述字段
```

**工业背景**：2011 年 IPv6 部署进入加速期（World IPv6 Day 是 2011 年 6 月）。OpenFlow 1.2 的 IPv6 支持是 SDN 进入运营商网络的门票。

---

## 8.4 支持多台控制器（故障转移和负载均衡）

### 8.4.1 Role

1.2 首次定义了**控制器角色（Role）机制**：

```
三种角色:
┌──────────────────────────────────────────────┐
│  MASTER                                       │
│  ── 唯一的主控制器                             │
│  ── 拥有对交换机的完整写权限                     │
│  ── 可以同时存在多个，但只有一个真正生效           │
├──────────────────────────────────────────────┤
│  SLAVE                                        │
│  ── 备用控制器                                 │
│  ── 只读权限(查询统计、监听事件)                 │
│  ── 不能修改流表                               │
├──────────────────────────────────────────────┤
│  EQUAL                                        │
│  ── 对等控制器                                 │
│  ── 多个控制器拥有相同权限                       │
│  ── 适用于负载均衡场景                         │
└──────────────────────────────────────────────┘
```

### 8.4.2 Role 变更

控制器可以通过 `Role-Request` 消息主动变更自己的角色：

```
故障转移流程:
  1. 控制器 A 是 MASTER，控制器 B 是 SLAVE
  2. 控制器 A 崩溃或与交换机断连
  3. 控制器 B 检测到(通过心跳或连接超时)
  4. 控制器 B 发送 Role-Request(MASTER) 给交换机
  5. 交换机确认，控制器 B 接管
  6. 恢复时间: 秒级(取决于检测机制)
```

> 💡 **洞见**：Role 机制的本质是**把"主备选举"从分布式协议(VRRP/HSRP)简化为"控制器自主协商"**。交换机只是被动记录每个控制器的角色，真正的决策逻辑在控制器集群内部完成。这比传统网络协议更灵活——你可以用 ZooKeeper、Raft 等任意分布式协调服务来管理控制器角色。

### 8.4.3 OpenFlow 控制器之间的协作

```
典型的多控制器部署:
┌──────────────────────────────────────────────┐
│                                              │
│  Controller A (MASTER)                       │
│  ├── 负责流表编程                             │
│  ├── 负责拓扑管理                             │
│  └── 通过 East-West 协议与 B 通信              │
│         │                                  │
│         │ (ZooKeeper/Raft/gRPC)            │
│         ▼                                  │
│  Controller B (SLAVE)                       │
│  ├── 只读: 收集统计                           │
│  ├── 监听: 事件通知                           │
│  └── 准备接管: 同步流表状态                     │
│                                              │
└──────────────────────────────────────────────┘
                    │
                    │ OpenFlow Channel (主)
                    │ OpenFlow Channel (备)
                    ▼
            ┌──────────────┐
            │   Switch      │
            └──────────────┘
```

**工业实践**：ONOS 等生产级 SDN 控制器采用"主备+分布式状态存储"的架构。所有控制器共享同一个分布式数据库（如 Atomix），MASTER 角色通过 Raft 选举产生。交换机看到的是"一个逻辑控制器"，背后是 N 台物理控制器的高可用集群。

---

## 8.5 OpenFlow 1.2 中的其他变化

### 8.5.1 将虚拟端口分离为逻辑端口和保留端口

1.1 中"虚拟端口"的概念在 1.2 中进一步澄清为两类：

- **逻辑端口**：与物理端口有对应关系的抽象（隧道、LAG）
- **保留端口**：特殊用途端口（ALL/CONTROLLER/TABLE/IN_PORT/NORMAL/FLOOD）

### 8.5.2 Flow-Mod 的 MODIFY/MODIFY_STRICT 的规范变更

```
变更前(1.0/1.1):
  MODIFY: 若流表项不存在，则插入新项
  
变更后(1.2+):
  MODIFY: 仅修改已存在的流表项，不插入新项
  (插入必须使用 ADD)
```

**工业价值**：避免了"本意是修改却意外创建了新流"的 bug。这是 API 设计严谨性的提升。

### 8.5.3 对实验性扩展的支持

Experimenter 机制在 1.2 中全面落地：

- Experimenter 匹配字段（OXM class=0xFFFF）
- Experimenter 动作
- Experimenter 错误消息

这使得厂商可以在不等待 ONF 标准化的前提下，先行实现私有硬件特性。

### 8.5.4 变更历史记录的添加

1.2 规范开始**维护详细的变更历史**——每个新版本都明确记录相对上一版的 delta。这标志着 OpenFlow 从"研究原型协议"正式转变为"工业标准协议"。

---

# 第9章 OpenFlow 1.3 — 读书笔记

2012 年发布的 1.3，是 OpenFlow 协议史上**生产环境部署最广泛的版本**。它在 1.1/1.2 奠定的架构基础上，补齐了最后一个核心拼图——**QoS（服务质量）**。

作者在这一章想告诉你：**OpenFlow 1.3 是一个"完整"的协议**——它既有灵活的流水线（1.1）、可扩展的匹配（1.2），又有 QoS 能力（1.3）。这也是为什么 1.3 成为工业界的事实标准。

---

## 9.1 OpenFlow 1.3 中的变更要点

核心变更：

1. **Meter Table**——计量表，实现 QoS
2. **Table-miss 默认 Drop**——安全模型变更
3. **辅助连接**——提升控制通道吞吐量
4. **IPv6 扩展头匹配**
5. **PBB 支持**
6. **多框架（Multipart）消息重构**

---

## 9.2 计量表（Meter Table）——QoS 支持的基石

OpenFlow 1.3 最重要的新增特性是**计量表**——它让 OpenFlow 第一次具备了硬件级 QoS 能力。

### 计量表的本质定位

```
传统 QoS 的两层结构:
┌──────────────────────────────────────────────┐
│  Layer 1: Metering (测速)                       │
│          测量流量速率，决定"超速了怎么处理"        │
│          → OpenFlow 1.3 的 Meter Table          │
├──────────────────────────────────────────────┤
│  Layer 2: Queuing (排队)                        │
│          在端口出口处按队列调度                   │
│          → OpenFlow 1.0+ 的 Queue (set_queue)   │
└──────────────────────────────────────────────┘

关键区别:
  Meter  → 绑定到流表项(per-flow)，在报文处理路径中测速
  Queue  → 绑定到端口(per-port)，在出口处调度
```

> 💡 **第一性原理**：Meter 和 Queue 解决的是 QoS 的两个不同维度。**Meter 是"警察"（policer）**——测量速率，超速就罚（丢弃/标记）；**Queue 是"调度器"（scheduler）**——在端口拥塞时决定谁先走。只有两者配合，才能实现完整的 DiffServ 框架 。

### 计量表的完整结构

```
Meter Table (计量表)
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Meter Entry 1 (计量表项)                                 │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Meter Identifier: 32-bit 无符号整数 (唯一标识)       │ │
│  │  Counters: 统计计数器                                 │ │
│  │  Meter Bands[]: (无序列表，一个或多个)                │ │
│  │  ┌──────────────────────────────────────────────┐   │ │
│  │  │  Band 1:                                    │   │ │
│  │  │    Band Type: drop / dscp_remark / exp      │   │ │
│  │  │    Rate: 5000 kbps (band 生效的最低速率)     │   │ │
│  │  │    Burst: 突发容量                            │   │ │
│  │  │    Counters: 该 band 处理的包/字节计数         │   │ │
│  │  │    Type Specific Arguments: 类型相关参数       │   │ │
│  │  ├──────────────────────────────────────────────┤   │ │
│  │  │  Band 2:                                    │   │ │
│  │  │    Band Type: drop                           │ │
│  │  │    Rate: 10000 kbps                          │ │
│  │  │    ...                                       │ │
│  │  ├──────────────────────────────────────────────┤   │ │
│  │  │  Band 3:                                    │   │ │
│  │  │    Band Type: dscp_remark                    │ │
│  │  │    Rate: 2000 kbps                           │ │
│  │  │    ...                                       │ │
│  │  └──────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  Meter Entry 2                                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │  ...                                                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  Meter Entry N                                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │  ...                                                │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

这是 OpenFlow 1.3.4 规范定义的 Meter Entry 三要素：**Meter Identifier（32位无符号整数）、Meter Bands（无序列表）、Counters（统计）**​ 。

### Meter Band 的字段详解

每个 Meter Band 由 5 个字段组成 ：

```
┌──────────────────────────────────────────────────────────┐
│  Band Type (带类型)                                       │
│  ├── drop: 丢弃数据包，用于定义速率限制器                   │
│  ├── dscp_remark: 提高 IP 头中 DSCP 字段的丢弃优先级       │
│  │     (用于定义简单的 DiffServ 策略器)                    │
│  └── experimenter (0xFFFF): 厂商自定义                     │
│                                                          │
│  Rate (速率)                                              │
│  ├── 单位: kbps 或 pps (由 flag 决定)                     │
│  └── 语义: 该 band 能够生效的最低速率                      │
│                                                          │
│  Burst (突发)                                             │
│  └── 该 band 的突发容量 (kbits 或 packets)                │
│                                                          │
│  Counters (计数器)                                        │
│  └── 该 band 处理过的包数/字节数                           │
│                                                          │
│  Type Specific Arguments (类型特定参数)                     │
│  └── 某些 band 类型需要的可选参数                          │
└──────────────────────────────────────────────────────────┘
```

> ⚠️ **规范要点**：**Band Type 没有"Required"类型**——drop 和 dscp_remark 都是 Optional。控制器必须通过查询消息确认交换机支持哪些 band 类型 。这意味着工业部署中，你的控制器代码必须处理"交换机不支持 drop band"的情况。

### Meter Band 的选择算法（核心机制）

这是 Meter 最微妙的部分——**包如何被分配到具体的 band？**

```
算法伪代码:
┌──────────────────────────────────────────────────────────┐
│  current_rate = meter.measure_current_rate()              │
│                                                          │
│  selected_band = NULL                                    │
│  highest_rate_below_current = 0                          │
│                                                          │
│  for each band in meter.meter_bands:                     │
│      if band.rate <= current_rate:                       │
│          if band.rate > highest_rate_below_current:       │
│              highest_rate_below_current = band.rate      │
│              selected_band = band                         │
│                                                          │
│  if selected_band == NULL:                               │
│      # 当前速率低于所有 band 的 rate                       │
│      # 不执行任何 band 动作，包继续正常处理                 │
│  else:                                                   │
│      # 执行 selected_band 定义的动作                       │
│      execute(selected_band.band_type)                    │
└──────────────────────────────────────────────────────────┘
```

规范原文："**the meter applies the meter band with the highest configured rate that is lower than the current measured rate. If the current rate is lower than any specified meter band rate, no meter band is applied.**"

用一个具体例子说明：

```
Meter Entry:
  Band 1: rate=2000 kbps, type=dscp_remark
  Band 2: rate=5000 kbps, type=drop
  Band 3: rate=10000 kbps, type=drop

流量速率 vs Band 选择:
┌──────────────────┬─────────────────────────────────────┐
│ 当前测量速率       │ 选中的 Band                        │
├──────────────────┼─────────────────────────────────────┤
│ 1000 kbps        │ 无 (低于所有 band 的 rate)           │
│ 3000 kbps        │ Band 1 (rate=2000, dscp_remark)     │
│ 7000 kbps        │ Band 2 (rate=5000, drop)            │
│ 12000 kbps       │ Band 3 (rate=10000, drop)           │
└──────────────────┴─────────────────────────────────────┘
```

> 💡 **洞见**：这个算法的本质是**"阶梯式惩罚"**——速率越高，落入的 band 越严厉。这正好对应 DiffServ 的 AF (Assured Forwarding) 服务等级：低速流量得到保障，中速流量被标记，高速流量被丢弃。Meter Band 的"无序列表 + 最高匹配速率"设计，让我们可以用几条 band 实现一个完整的 DiffServ 策略器。

### Meter 与流表项的配合

Meter 不是独立工作的，它**通过流表项的 Instruction 被引用**：

```
流表项引用 Meter 的完整流程:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Table 0:                                               │
│  Match: ip_src=10.0.0.0/8, ip_proto=6                  │
│  Instruction:                                           │
│    Meter: meter_id=1  ← 引用计量器                       │
│    Goto-Table: 1                                      │
│       │                                                  │
│       ▼                                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Meter Table, Meter ID=1                         │   │
│  │  ┌─────────────────────────────────────────────┐ │   │
│  │  │ 测量当前速率                                    │ │   │
│  │  │ 选择匹配的 Band                                │ │   │
│  │  │ 执行 Band 动作:                               │ │   │
│  │  │   - drop: 丢弃包，流程结束                     │ │   │
│  │  │   - dscp_remark: 修改 DSCP，继续流水线         │ │   │
│  │  └─────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
│       │                                                  │
│       ▼ (未被丢弃的包继续)                                │
│  Table 1:                                               │
│  Match: ...                                             │
│  Instruction:                                           │
│    Write-Actions: set_queue:queue_id=2  ← 配合 Queue    │
│    Write-Actions: output:port_5                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**关键约束**（规范明确）：

- **同一个流表项不能同时引用两个 Meter**（"cannot set 2 meters for a flow entry"）
- 同一个流表中可以使用多个 Meter，但必须是**独占的**（即流表项之间没有交集）
- 通过在**前后相连的流表**上部署多个 Meter，可以让多个 Meter 作用于同一批数据包

### 工业实战：OVS 限速示例

SDNLab 给出的 OVS 实操非常清晰 ：

```
# 1. 设置交换机为 netdev 类型并启用 OpenFlow 1.3
ovs-vsctl set bridge s1 datapath_type=netdev
ovs-vsctl set bridge s1 protocols=OpenFlow13

# 2. 下发 Meter 表: ID=1, 速率单位 kbps, 超速则丢弃, 限速 5000 kbps
ovs-ofctl add-meter s1 meter=1,kbps,band=type=drop,rate=5000 -O OpenFlow13

# 3. 下发流表: 入端口 5 的流量先经过 meter:1 限速，再输出到端口 6
ovs-ofctl add-flow s1 in_port=5,action=meter:1,output:6 -O openflow13
```

这段配置的语义流程图：

```
入端口 5 的包
    │
    ▼
┌──────────────────┐
│  Meter ID=1       │
│  测量速率          │
│  ├── < 5000 kbps  │→ 不触发 band，继续
│  └── ≥ 5000 kbps  │→ 触发 drop band，包被丢弃
└──────────────────┘
    │ (未被丢弃)
    ▼
输出到端口 6
```

### Meter + Queue 实现完整 DiffServ

单独的 Meter 只能做"超速丢包"或"DSCP 重标记"——这只是 DiffServ 的"警察"部分。要实现完整的 DiffServ，还需要**端口队列**配合：

```
完整 DiffServ 流水线:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  步骤 1: Metering (Meter Table)                           │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Meter ID=1:                                       │ │
│  │    Band 1: rate=2Mbps, dscp_remark (AF11→AF12)    │ │
│  │    Band 2: rate=5Mbps, drop                        │ │
│  └────────────────────────────────────────────────────┘ │
│  │                                                      │
│  │ 语义:                                                │
│  │   < 2Mbps:    保持 AF11 (高优先级)                   │
│  │   2-5Mbps:    降级为 AF12 (中优先级)                 │
│  │   > 5Mbps:    丢弃                                   │
│  ▼                                                      │
│  步骤 2: Queueing (Per-Port Queue)                       │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Port 5 的出口队列:                                 │ │
│  │    Queue 0 (AF11): 严格优先级，带宽保障 2Mbps       │ │
│  │    Queue 1 (AF12): WRR 权重 2，带宽保障 3Mbps      │ │
│  │    Queue 2 (BE):   WRR 权重 1，剩余带宽             │ │
│  └────────────────────────────────────────────────────┘ │
│  │                                                      │
│  │ 流表项:                                              │
│  │  Match: ip_dscp=AF11                                │
│  │  Action: set_queue:0, output:5                      │
│  │  Match: ip_dscp=AF12                                │
│  │  Action: set_queue:1, output:5                      │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**这就是运营商级 QoS 的完整实现**：

- Meter 做"测速 + 标记/丢弃"（Policing）
- Queue 做"调度 + 带宽保障"（Scheduling）
- 两者结合 = 完整的 DiffServ AF 服务等级

> 💡 **洞见**：Meter 和 Queue 的解耦设计体现了 SDN 的模块化哲学——**每个组件只做一件事，但通过流表指令可以自由组合**。这种"乐高式"的 QoS 架构，比传统路由器"一体化 QoS 配置"灵活得多。你可以在控制器里用几行代码动态创建新的 Meter，而不需要登录每台路由器改 CBWFQ 配置。

### Meter 的工业应用场景

**场景 1：租户级带宽保障（Cloud DC）**

```
每个租户一个 Meter:
  Meter 100: 租户 A, rate=10G, drop
  Meter 200: 租户 B, rate=5G, drop
  Meter 300: 租户 C, rate=2G, drop

流表项:
  Match: tunnel_id=租户A → Meter:100 → 转发
  Match: tunnel_id=租户B → Meter:200 → 转发
  Match: tunnel_id=租户C → Meter:300 → 转发
```

**场景 2：应用级限速（企业网）**

```
Meter 1: 视频会议, rate=2M, dscp_remark (保障)
Meter 2: 文件下载, rate=5M, drop (限制)
Meter 3: P2P, rate=500k, drop (严格限制)

流表项:
  Match: tcp_dst=443, app=video → Meter:1
  Match: tcp_dst=80 → Meter:2
  Match: p2p_signature → Meter:3
```

**场景 3：DDoS 防护（运营商）**

```
Meter 999: rate=100M, drop

流表项:
  Match: ip_dst=受害服务器IP, rate>100M → Meter:999
  语义: 超过 100Mbps 的流量直接丢弃，保护后端
```

### 与 1.1/1.2 的对比

```
OpenFlow 1.0/1.1/1.2:
  QoS 能力: 仅有 per-port Queue (set_queue 动作)
  局限: 
    - 只能基于端口做队列调度
    - 无法基于 per-flow 做速率测量
    - 无法实现 DiffServ 的 Policing 部分

OpenFlow 1.3:
  QoS 能力: Meter Table + per-port Queue
  突破:
    - per-flow 速率测量
    - 阶梯式 band 惩罚
    - drop / dscp_remark 两种动作
    - 与 Queue 配合实现完整 DiffServ
```

> ⚠️ **工程现实**：虽然规范定义了 Meter，但**硬件支持程度参差不齐**。在硬件支持不完整的情况下，Meter 通常会被降级到软件实现，此时其性能可能无法达到线速。

### 第一性原理层面的理解

```
Meter Table 的本质:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"QoS = 测量(Metering) + 调度(Scheduling)"

Meter 解决 Metering:
  - 测量 per-flow 速率
  - 基于速率选择处理动作
  - 动作: drop / dscp_remark / experimenter

Queue 解决 Scheduling:
  - 在端口出口处排队
  - 基于队列优先级/权重调度
  - 动作: 严格优先级 / WRR / WFQ

Meter 的创新点:
  1. per-flow 绑定: Meter 直接挂在流表项上，不是端口
  2. 阶梯式 band: 一条 Meter Entry 可以实现多级速率策略
  3. 无序 band 列表: 硬件可以自由优化 band 匹配算法
  4. 与流水线结合: 可以在多级流表的任意一级插入 Meter

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**批判性思考**：Meter Table 的设计虽然优雅，但也有其局限：

- **Band Type 只有 Optional**：drop 和 dscp_remark 都不是 Required，导致跨厂商互通性问题
- **单流表项不能引用多个 Meter**：复杂场景必须拆分到多级流表
- **Meter 与 Queue 的协调需要控制器编程**：没有"一键 DiffServ"的原语
- **硬件实现差异大**：部分交换机的 Meter 是软件实现的，性能堪忧

这些局限催生了后续的演进：

- OpenFlow 1.4 引入了**流监控框架**（Flow Monitoring）
- OpenFlow 1.5 引入了**包寄存器**（Packet Registers），让 Meter 的处理结果可以传递到后续流表
- 工业界更多转向 **P4 + 硬件 Queue**​ 的方案，实现更灵活的 QoS

但无论如何，OpenFlow 1.3 的 Meter Table 是 SDN QoS 的**奠基性设计**——它第一次让"per-flow QoS"成为 OpenFlow 协议的一等公民。

---

## 9.3 Table-miss 的默认动作改为 Drop

### 9.3.1 Table-miss 流表项

OpenFlow 1.3 做了一个**安全模型的重大变更**：**Table-miss 的默认动作从"上送控制器"改为"丢弃"**。

```
OpenFlow 1.0/1.1/1.2:
  包不匹配任何流表项 → 默认上送控制器 (Packet-In)
  语义: "白名单"模式——未知的包交给控制器决策

OpenFlow 1.3+:
  包不匹配任何流表项 → 默认丢弃
  除非显式配置 Table-miss 流表项 (priority=0, match=通配符)
  语义: "黑名单"模式——只有显式允许的包才能通过
```

**Table-miss 流表项的显式配置**：

```
如果要恢复"上送控制器"的行为，必须显式下发:

ovs-ofctl add-flow s1 "table=0,priority=0,actions=controller:65535"

如果要"上送控制器并记录"，可以这样:
ovs-ofctl add-flow s1 "table=0,priority=0,actions=controller:65535,resubmit(,1)"
```

### 9.3.2 流表匹配流程的变更

```
OpenFlow 1.3 的匹配流程:
┌──────────────────────────────────────────────────────────┐
│                                                      │
│  包到达                                                 │
│    │                                                  │
│    ▼                                                  │
│  Table 0 匹配                                         │
│    │                                                  │
│    ├── 匹配到流表项 (priority > 0)                     │
│    │     → 执行 Instructions                           │
│    │                                                  │
│    ├── 匹配到 Table-miss 流表项 (priority=0)           │
│    │     → 执行该流表项定义的动作                        │
│    │     → 默认情况下: 该流表项不存在                    │
│    │                                                  │
│    └── 无任何匹配                                     │
│          → 默认行为: DROP (1.3 新语义)                 │
│          → 若配置了 table-miss 流表项: 执行其动作        │
│                                                      │
└──────────────────────────────────────────────────────────┘
```

> 💡 **洞见**：这个变更的深层含义是**安全立场的转变**。1.0 时代的 SDN 是"乐观主义"——默认信任控制器能处理一切；1.3 时代转为"悲观主义"——默认拒绝，显式允许。这符合工业网络安全的最佳实践：**默认拒绝（Default Deny）是安全网络的黄金法则**。

**工业价值**：

- **防 DoS**：攻击者无法用随机垃圾流量淹没控制器（1.0 模式下每个未知包都会 Packet-In）
- **减时延**：合法流量走流表快速转发，不需要每次都问控制器
- **明确语义**：Table-miss 行为是"配置出来的"，而不是"隐含的"

> ⚠️ **迁移注意**：从 1.0 升级到 1.3 时，**必须显式下发 Table-miss 流表项**，否则原本能上送控制器的包现在会被静默丢弃。这是很多早期 SDN 部署升级时"网络突然不通"的根本原因。

---

## 9.4 OpenFlow 1.3 中的其他变更

### 9.4.1 OpenFlow 控制器和 OpenFlow 交换机之间的辅助连接

1.3 引入了**辅助连接（Auxiliary Connection）**机制——一个主连接 + 多个辅助连接：

```
单 TCP 连接 (1.0/1.1/1.2):
┌─────────────────────────────────┐
│  Controller ←─── TCP ───→ Switch │
│  所有流量: Packet-In/Out,       │
│  Flow-Mod, Stats, ...           │
│  单连接 → 瓶颈!                  │
└─────────────────────────────────┘

主连接 + 辅助连接 (1.3+):
┌─────────────────────────────────────────────────┐
│  Controller                                       │
│  ├── 主连接 (TCP): 控制信令, Flow-Mod, ...        │
│  ├── 辅助连接 1 (TCP): Packet-In 高优先级流量     │
│  ├── 辅助连接 2 (TCP): Packet-In 低优先级流量     │
│  └── 辅助连接 3 (TCP): Stats 查询/响应            │
│         │      │      │      │                    │
│         ▼      ▼      ▼      ▼                    │
│         └──────┴──────┴──────┘                    │
│                Switch                              │
└─────────────────────────────────────────────────┘
```

**工业价值**：把 Packet-In 风暴、Stats 查询、Flow-Mod 下发分离到不同连接，避免相互阻塞。这对于大规模 SDN 部署至关重要——想象一下 1000 台交换机同时上报 Packet-In，单连接早就堵死了。

### 9.4.2 可以通过 UDP、DTLS 等与 OpenFlow 控制器进行通信

1.3 解除了"OpenFlow Channel 必须用 TCP"的限制：

```
传输层选项:
┌──────────────────────────────────────────────┐
│  TCP   (默认，可靠有序)                         │
│  UDP   (低时延，适合辅连接)                      │
│  TLS   (加密的 TCP)                            │
│  DTLS  (加密的 UDP)                            │
└──────────────────────────────────────────────┘
```

**工业价值**：

- **TLS/DTLS**​ 满足金融、政府等行业的合规要求
- **UDP**​ 降低 Packet-In 的时延（适合延迟敏感场景）
- **TCP**​ 仍是主连接的首选（可靠性优先）

### 9.4.3 支持 IPv6 扩展头

1.2 支持了基本 IPv6，1.3 进一步支持 **IPv6 扩展头匹配**：

```
支持的 IPv6 扩展头:
┌──────────────────────────────────────────────┐
│  Hop-by-Hop Options Header                     │
│  Routing Header                                │
│  Fragment Header                               │
│  Destination Options Header                    │
│  Authentication Header                         │
│  Encapsulating Security Payload Header         │
│  Mobility Header                               │
└──────────────────────────────────────────────┘
```

**工业价值**：使得 OpenFlow 可以精确匹配 IPv6 分段、IPsec 包、移动 IPv6 包等——这对运营商 IPv6 网络至关重要。

### 9.4.4 OXM 匹配字段的添加

1.3 在 1.2 的 OXM 基础上新增了更多匹配字段，特别是 IPv6 相关的：

```
新增 OXM 字段:
├── IPv6 Extension Headers (位掩码)
├── IPv6 ND (Neighbor Discovery) 相关字段
├── PBB (Provider Backbone Bridge) 字段
└── 各种 Tunnel ID 扩展
```

### 9.4.5 支持 PBB

**PBB (Provider Backbone Bridge)**​ —— 802.1ah 标准，运营商以太网核心技术：

```
PBB 报文结构:
┌──────────────────────────────────────────────────┐
│  Outer Ethernet Header (运营商 MAC)                 │
│    B-DA, B-SA, B-Tag (Backbone VLAN Tag)            │
│    I-Tag (Service Instance Tag, 24-bit I-SID)       │
│  Inner Ethernet Header (用户 MAC)                   │
│    C-DA, C-SA, C-Tag (用户 VLAN Tag)                │
│  Payload                                            │
└──────────────────────────────────────────────────┘

OpenFlow 1.3 支持匹配:
  - I-SID (24-bit 服务实例标识)
  - B-Tag / I-Tag 字段
  - 外层/内层 MAC 地址
```

**工业价值**：PBB 是电信运营商以太网服务的基石。OpenFlow 1.3 支持 PBB，意味着 SDN 可以直接控制运营商骨干网的设备。

### 9.4.6 多框架（Multipart）消息

1.3 引入了**Multipart 消息**——用于批量数据传输：

```
Multipart 消息类型:
├── OFPMP_DESC: 交换机描述
├── OFPMP_FLOW: 流表项统计
├── OFPMP_TABLE: 流表统计
├── OFPMP_PORT_STATS: 端口统计
├── OFPMP_QUEUE: 队列统计
├── OFPMP_METER: 计量器统计
├── OFPMP_METER_CONFIG: 计量器配置
├── OFPMP_GROUP: 组表统计
├── OFPMP_GROUP_DESC: 组表描述
├── OFPMP_TABLE_FEATURES: 流表能力
├── OFPMP_PORT_DESC: 端口描述
├── OFPMP_EXPERIMENTER: 厂商扩展
└── ...
```

**工业价值**：Multipart 让"批量查询"成为可能——控制器可以一次性拉取所有流表项的统计，而不需要逐条查询。这对大规模网络的遥测至关重要。

### 9.4.7 从握手时的 Features 响应消息中删除端口号

1.0 的 Features 响应包含端口号列表，1.3 将其**移到独立的 Multipart Port Desc 消息**中。

**原因**：端口数量可能很多（特别是虚拟交换机），塞在 Features 响应里会导致消息过大。分离后，Features 响应保持精简，端口信息按需查询。

### 9.4.8 流表项构成要素的变更

1.3 对流表项结构做了细化：

```
1.0 流表项:
  Match + Counter + Actions

1.3 流表项:
  Match + Priority + Counter + Instructions + Cookie + Flags + Timeouts
```

新增的关键字段：

- **Cookie**：控制器自定义的 64-bit 标识符，用于批量管理流表项
- **Flags**：OFPFF_SEND_FLOW_REM (流表项移除时通知), OFPFF_CHECK_OVERLAP (检查重叠)
- **Timeouts**：idle_timeout, hard_timeout

---

## 9.5 OpenFlow 1.3.1 和 1.3.2

### 9.5.1 OpenFlow 通道中版本协商的变更

1.3.1 改进了版本协商机制：

```
版本协商流程 (1.3.1+):
┌──────────────────────────────────────────────────┐
│  Switch → Controller: Hello (version_bitmap)     │
│  Controller → Switch: Hello (selected_version)   │
│  Switch → Controller: Features Request           │
│  Controller → Switch: Features Reply             │
└──────────────────────────────────────────────────┘

version_bitmap: 交换机支持的版本位图
  允许交换机一次性声明支持多个版本 (1.0/1.3/1.4/...)
  控制器从中选择双方都支持的最高版本
```

### 9.5.2 建立与 OpenFlow 控制器之间的 OpenFlow 通道

1.3.2 进一步规范了通道建立流程，特别是**辅助连接的建立**：

```
完整通道建立流程:
┌──────────────────────────────────────────────────┐
│  1. TCP 连接建立 (主连接)                            │
│  2. Hello 消息交换 (版本协商)                        │
│  3. Features 消息交换                               │
│  4. 主连接就绪                                      │
│  5. (可选) 建立辅助连接:                             │
│     - 新的 TCP 连接                                  │
│     - Hello 消息 (携带 auxiliary_id)                 │
│     - 复用主连接的版本协商结果                         │
│  6. 辅助连接就绪                                      │
└──────────────────────────────────────────────────┘
```

> 💡 **洞见**：auxiliary_id 是关键——它让辅助连接与主连接**逻辑关联**，交换机知道"这个辅助连接属于哪个主连接"。这样即使辅助连接断开重连，也不会影响主连接的控制信令。

---

## 贯穿第9章的核心洞见

```
OpenFlow 1.3 的本质:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"OpenFlow 1.3 = 完整的生产级 SDN 协议"

1.1 提供了: 多级流表 + 组表 + TTL 操作
1.2 提供了: OXM 可扩展匹配 + IPv6 + 多控制器
1.3 提供了: Meter Table (QoS) + Table-miss Drop + 辅助连接

三者叠加 = 一个工业可用的 SDN 协议栈

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Meter Table 的创新:
  - per-flow 测速 (不是 per-port)
  - 阶梯式 band 惩罚
  - drop / dscp_remark 两种动作
  - 与 Queue 配合 = 完整 DiffServ

Table-miss Drop 的安全立场:
  - 从"默认上送控制器"到"默认丢弃"
  - 安全模型: 乐观 → 悲观
  - 工业价值: 防 DoS, 降时延, 明语义

辅助连接的价值:
  - 控制信令 / Packet-In / Stats 分流
  - 避免单 TCP 连接成为瓶颈
  - 大规模部署的必备特性

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
为什么 1.3 是工业事实标准?
  1. 完整: 流水线 + 组表 + QoS + IPv6 + 多控制器
  2. 稳定: 1.3.x 系列长期维护 (1.3.0 → 1.3.5)
  3. 广泛: OVS, OpenDaylight, ONOS, Ryu 全都支持
  4. 硬件: Broadcom, Mellanox, Cavium 等 ASIC 原生支持
  5. 部署: Google B4, Microsoft SWAN 等生产网络基于 1.3+

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**六个工程层面的关键认知**：

**1. Meter 和 Queue 是 QoS 的双翼**

单独的 Meter 只能做 Policing（丢包/标记），单独的 Queue 只能做 Scheduling（调度）。只有两者配合，才能实现完整的 DiffServ。这是 OpenFlow 1.3 QoS 架构的核心设计哲学。

**2. Meter Band 的"阶梯选择算法"是精髓**

"选择低于当前速率的最高配置速率 band"——这个算法让我们可以用几条 band 实现任意复杂的速率策略。理解了这个算法，你就能用 Meter 实现 AF、EF、BE 等所有 DiffServ 服务等级。

**3. Table-miss 默认 Drop 是安全模型的进化**

从"乐观默认上送"到"悲观默认拒绝"，符合工业网络安全最佳实践。但迁移时必须显式下发 Table-miss 流表项，否则网络会"神秘不通"。

**4. 辅助连接是解决控制通道瓶颈的关键**

大规模 SDN 部署中，Packet-In 风暴是常态。辅助连接让我们可以将不同类型的流量分流到不同的 TCP 连接，避免相互阻塞。

**5. 1.3 的"完整性"使其成为分水岭**

1.0 是原型，1.1 是架构升级，1.2 是可扩展性升级，1.3 才是**第一个完整的生产级协议**。这也是为什么业界跳过 1.1/1.2 直接采用 1.3 的原因。

**6. 1.3.x 的长期维护值得信赖**

从 1.3.0 (2012) 到 1.3.5 (2015)，ONF 对这个版本做了长达 3 年的维护和改进。这种"长期支持版本"的模式，让运营商敢于在生产环境部署。

> 💡 **批判性思考**：OpenFlow 1.3 虽然是工业事实标准，但它的设计仍有历史局限性：
> 
> - **Meter 的 Band Type 只有 Optional**——跨厂商互通性差
> - **单流表项不能引用多个 Meter**——复杂 QoS 必须拆分到多级流表
> - **辅助连接增加了连接管理复杂度**——控制器需要实现完整的多连接管理
> - **PBB/MPLS/VLAN 的 Push/Pop 是"动作"而非"匹配"**——无法基于隧道内层字段做匹配
> 
> 这些局限正是 P4 等数据面可编程技术兴起的原因。P4 不再试图定义"完整的协议"，而是提供"可编程的匹配-动作引擎"——你可以用 P4 实现任意版本的 OpenFlow，甚至实现 OpenFlow 从未设想过的网络功能。
> 
> 但 OpenFlow 1.3 的历史地位不可动摇：它证明了 **"SDN 控制面与数据面分离 + 流水线化包处理"**​ 的架构是可行的、是工业的、是可扩展的。今天的 P4、今天的 VXLAN EVPN、今天的云网络 VPC，骨子里都是 OpenFlow 1.3 思想的延续和升华。

**给工程师的实践建议**：

1. **用 OVS 实操 Meter Table**
    
    ```
    # 创建限速 Meter
    ovs-ofctl add-meter s1 meter=1,kbps,band=type=drop,rate=5000 -O OpenFlow13
    
    # 创建 DiffServ Meter
    ovs-ofctl add-meter s1 meter=2,kbps,\
      band=type=dscp_remark,rate=2000,prec=1,\
      band=type=drop,rate=5000 -O OpenFlow13
    
    # 查看 Meter 统计
    ovs-ofctl meter-stats s1 -O OpenFlow13
    ```
    
2. **设计 QoS 方案时遵循"Meter + Queue"双轨**
    - Meter 做 per-flow Policing（限速/标记）
    - Queue 做 per-port Scheduling（调度）
    - 两者通过 DSCP 字段衔接
3. **迁移到 1.3 时务必配置 Table-miss**
    
    ```
    # 显式配置上送控制器
    ovs-ofctl add-flow s1 "table=0,priority=0,actions=controller:65535"
    
    # 或者显式配置丢弃 (符合 1.3 默认语义)
    ovs-ofctl add-flow s1 "table=0,priority=0,actions=drop"
    ```
    
4. **大规模部署启用辅助连接**
    
    ```
    # 在控制器侧配置多个辅助连接
    # 用于分离 Packet-In, Stats, Flow-Mod 等不同流量
    ```
    

下一章如果发过来，我们可以继续看 OpenFlow 1.4/1.5 的演进——特别是 1.5 引入的**包寄存器（Packet Registers）**和**Bundles 机制**，这些特性进一步弥合了 OpenFlow 与 P4 之间的鸿沟。第9章是 OpenFlow 的"成年礼"，而 1.4/1.5 则是它面向未来的"进化尝试"。