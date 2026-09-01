
第13章"NDIS 中间层驱动"是全书网络部分的**集大成章节**——它把前几章分散的视角（第10章 TDI 过滤、第11章 NDIS 协议驱动、第12章 NDIS 小端口驱动）汇聚到一个完整的驱动类型上：**中间层驱动（Intermediate Driver）同时具备"协议驱动的下边缘"和"小端口驱动的上边缘"**。微软官方定义得非常精确：中间层驱动的上边缘导出 MiniportXxx 函数（对上层协议驱动来说，它看起来像个网卡小端口）；下边缘导出 ProtocolXxx 函数（对下层小端口驱动来说，它看起来像个协议驱动）。其内部通过"虚拟小端口（Virtual Miniport）"把这两侧绑定起来——虚拟小端口不直接控制物理设备，而是依赖下层小端口驱动与物理设备通信。

但必须 upfront 说明工业现实：**NDIS 5.x 已被微软标记为弃用**，新开发应使用 NDIS 6.x。而且，**如果你的目的仅仅是"修改网络流量或监视网络活动"，微软官方推荐的是 NDIS Filter 驱动，而不是中间层驱动**——中间层驱动的保留阵地是"实现 n 到 m 多路复用服务、负载均衡和故障转移解决方案"。

下面按你给的目录结构，逐节做读书笔记。

---

## 13.1 NDIS 中间层驱动概述

### 13.1.1 Windows 网络架构总结

**📌 书中的世界**

Windows 网络栈自底向上是：物理网卡 → NDIS 小端口驱动 → （可选的过滤驱动/中间层驱动）→ NDIS 协议驱动（如 tcpip.sys）→ AFD → Winsock → 用户态应用程序。

**🔍 第一性原理视角：中间层驱动是"栈中的栈"**

理解中间层驱动的关键，是先理解它在 NDIS 驱动栈中的**独特定位**。微软定义了四种 NDIS 驱动类型：

- **Miniport 驱动**：管理网络适配器，为上层驱动提供接口
- **Protocol 驱动**：实现网络协议，绑定到小端口适配器
- **Filter 驱动**：在协议驱动和小端口驱动之间过滤信息
- **Intermediate 驱动**：位于协议驱动和小端口驱动之间，**同时为两者提供接口**

中间层驱动的独特之处在于：**它是 NDIS 栈中唯一一种"双面间谍"**——上边缘像小端口，下边缘像协议驱动。这种双重身份让它能够在协议驱动和小端口驱动之间"插入"自己，实现媒体转换、负载均衡、故障转移、数据包过滤等功能。

**🏭 工业应用场景**：

- **媒体转换**：在老的传输驱动和新媒体类型的网卡小端口驱动之间做翻译
- **负载均衡与故障转移（LBFO）**：一个虚拟适配器对应多个物理网卡
- **数据包过滤与安全**：安全检查、内容过滤
- **网络数据统计监控**

### 13.1.2 NDIS 中间层驱动简介

**📌 书中的世界**

中间层驱动的核心是"虚拟小端口"——它在上边缘导出的不是物理网卡，而是一个**虚拟适配器**。对上层协议驱动（如 tcpip.sys）来说，这个虚拟适配器看起来就跟物理网卡一模一样：可以绑定、可以发送接收包、可以查询/设置 OID。

**🔍 第一性原理视角：虚拟小端口是"欺骗的艺术"**

中间层驱动通过 `NdisIMInitializeDeviceInstanceEx` 在 `ProtocolBindAdapterEx` 中初始化虚拟小端口。这个虚拟小端口：

- 对 tcpip.sys 呈现为一个标准的 NDIS 小端口适配器
- 拥有自己的 MAC 地址、链路速度、OID 处理能力
- 实际的数据收发都转发给下层真实小端口

> 💡 **洞见**：中间层驱动的精髓在于**"透明代理"**——上层协议驱动不知道（也不需要知道）下层是物理网卡还是被中间层驱动"拦截"过的虚拟适配器。这种透明性是 NDIS 架构分层设计的胜利：每一层都只看到下一层的抽象接口。

**两种中间层驱动**：

- **Filter Intermediate Drivers（过滤型中间驱动）**：用于数据包过滤、修改
- **MUX Intermediate Drivers（多路复用型中间驱动）**：用于 LBFO、n 到 m 多路复用

### 13.1.3 NDIS 中间层驱动的应用

**📌 书中的世界**

书中示例是一个典型的**过滤型中间层驱动**——它在协议驱动和小端口驱动之间插入自己，对收发的包进行检查、过滤、可能还有修改。

**🏭 现代工业现实**：

微软文档明确指出，如果你只需"修改网络流量或监视网络活动而不更改现有驱动"，**应当使用 Filter 驱动（Lightweight Filter），而不是中间层驱动**。中间层驱动的现代主战场是：

- **LBFO（负载均衡与故障转移）**：如 Windows 的 NIC Teaming
- **虚拟化**：Hyper-V 的虚拟交换机（vSwitch）底层
- **VPN 聚合**：多个隧道聚合为一个虚拟适配器
- **VLAN**：IEEE 802.1Q 标签处理

Windows Driver Samples 仓库中的 `network/ndis/mux` 示例就是一个典型的 MUX 中间层驱动，它在单个物理适配器上创建多个虚拟适配器（VELANs）。

> ⚠ **工业选型警示**：
> 
> - 如果只是做包过滤/监控 → 用 **NDIS 6.x Lightweight Filter**​ 或 **WFP**
> - 如果需要 n 到 m 多路复用、LBFO → 用 **NDIS 中间层驱动**
> - 如果需要感知应用进程的过滤（防火墙/EDR）→ 用 **WFP**

### 13.1.4 NDIS 包描述符结构深究

**📌 书中的世界**

书中基于 NDIS 5.x 模型，使用的是 `NDIS_PACKET` 结构来描述一个数据包。中间层驱动面对的核心难题是：**它从上层收到一个 `NDIS_PACKET`，要发给下层小端口；但下层小端口要求的是"自己的"`NDIS_PACKET`"**。

**🔍 第一性原理视角：包描述符的"身份危机"**

这是 NDIS 5.x 中间层驱动最微妙的问题。微软文档明确说明：

- **NDIS 4.0/5.0 中间层驱动**：必须至少为转发的每个包**分配一个新的 `NDIS_PACKET`**，并把原包的 OOB（Out-of-Band）数据复制到新包中
- **NDIS 5.1 支持包堆叠（Packet Stacking）的中间层驱动**：在大多数情况下可以避免这个新包描述符的分配

**包堆叠机制**：每个 `NDIS_PACKET` 包含一个"栈"数组，每个栈是一个 `NDIS_PACKET_STACK` 结构。中间层驱动调用 `NdisIMGetCurrentPacketStack` 来检查是否有剩余栈空间——如果有，就可以把自己的上下文信息存储在 `IMReserved` 成员中，**避免分配新的包描述符**。如果 `*StacksRemaining` 为 FALSE，则必须回退到 NDIS 5.0 模型（分配新包描述符）。

> 💡 **洞见**：包堆叠是微软在 NDIS 5.1 中对中间层驱动性能的"急救"——通过让多个驱动共享同一个 `NDIS_PACKET` 的不同栈帧，避免了包描述符和 OOB 数据的反复分配/拷贝。但这只是过渡方案，**真正的架构性解决是在 NDIS 6.0 中引入 `NET_BUFFER_LIST`**——它从根本上改变了包管理方式，让中间层驱动能够直接链接/解链缓冲区，而无需分配新的包描述符。

**NDIS 6.x 的包管理**：

- 中间层驱动从上层收到 `NET_BUFFER_LIST` 结构（关联一个或多个 MDL）
- 可以直接调用 `NdisSendNetBufferLists` 或 `NdisCoSendNetBufferLists` 转发
- 如果需要修改缓冲区内容或顺序，可以重新包装 `NET_BUFFER_LIST` 关联的缓冲区
- 如果拷贝数据到新缓冲区且最后一个缓冲区实际长度小于分配长度，调用 `NdisAdjustMdlLength` 调整

---

## 13.2 中间层驱动的入口与绑定

### 13.2.1 中间层驱动的入口函数

**📌 书中的世界（NDIS 5.x 模型）**

书中示例的 `DriverEntry` 调用序列：

1. `NdisMInitializeWrapper` — 初始化 NDIS 包装器
2. `NdisIMRegisterLayeredMiniport` — 注册 MiniportXxx 函数
3. `NdisRegisterProtocol` — 注册 ProtocolXxx 函数（如果需要绑定到下层驱动）
4. `NdisIMAssociateMiniport` — 告知 NDIS 该驱动的小端口上边缘和协议下边缘的关联

**🏭 现代 NDIS 6.x 模型**：

微软在 NDIS 6.0 中重构了中间层驱动的初始化：

1. `NdisMRegisterMiniportDriver` — 带 `NDIS_INTERMEDIATE_DRIVER` 标志注册 MiniportXxx 函数
2. `NdisRegisterProtocolDriver` — 注册 ProtocolXxx 函数
3. `NdisIMAssociateMiniport` — 关联上边缘和下边缘

> 💡 **关键差异**：NDIS 6.x 用 `NdisMRegisterMiniportDriver` 取代了 `NdisMInitializeWrapper` + `NdisIMRegisterLayeredMiniport` 的组合。驱动必须保存两个句柄：`NdisMiniportDriverHandle`（供后续 `NdisXxx` 函数调用）和 `NdisProtocolHandle`。

**DriverEntry 的硬约束**：

- **必须同步完成**：不能返回 `STATUS_PENDING` 或 `NDIS_STATUS_PENDING`
- 成功后从 `MiniportDriverUnload` 中调用 `NdisMDeregisterMiniportDriver`
- 如果 `NdisMRegisterMiniportDriver` 成功后发生错误，必须在 `DriverEntry` 返回前调用 `NdisMDeregisterMiniportDriver`

### 13.2.2 动态绑定 NIC 设备

**📌 书中的世界**

中间层驱动不会在 `DriverEntry` 中静态绑定到特定网卡。相反：

1. NDIS 在所有下层小端口驱动初始化完成后，调用中间层驱动的 `ProtocolBindAdapterEx`
2. 在 `ProtocolBindAdapterEx` 中，中间层驱动调用 `NdisOpenAdapterEx` 绑定到下层小端口
3. 绑定成功后，调用 `NdisIMInitializeDeviceInstanceEx` 初始化虚拟小端口
4. NDIS 随后调用中间层驱动的 `MiniportInitializeEx` 来初始化虚拟小端口

**🔍 第一性原理视角：自下而上的初始化顺序**

微软文档明确说明了这个顺序：

> "The initialization of an intermediate driver's virtual miniport occurs when the driver calls the NdisIMInitializeDeviceInstanceEx function from its ProtocolBindAdapterEx function. NDIS calls the ProtocolBindAdapterEx function after all underlying miniport drivers have initialized."

这种设计的意义：**中间层驱动能够根据它所绑定的下层小端口的特性，在虚拟小端口创建时分配相应的资源**。例如，如果下层网卡支持 TCP 卸载，虚拟小端口也可以在初始化时声明支持相应的 OID。

**动态绑定的价值**：

- 系统上可能有多个网卡，中间层驱动可以绑定到其中每一个
- 网卡可以热插拔，中间层驱动通过 PnP 事件动态绑定/解绑
- 一个中间层驱动实例可以为每个绑定的下层网卡创建一个虚拟小端口

### 13.2.3 小端口初始化（MpInitialize）

**📌 书中的世界**

`MPInitialize`（NDIS 6.x 中是 `MiniportInitializeEx`）是虚拟小端口的初始化函数。它在 `NdisIMInitializeDeviceInstanceEx` 被调用后由 NDIS 调用。

**🔍 第一性原理视角：虚拟小端口的"假初始化"**

与真实小端口不同，虚拟小端口的 `MPInitialize`：

- **不需要**映射 I/O 端口、注册中断、设置 DMA
- **需要**分配虚拟小端口的适配器上下文
- **需要**设置虚拟小端口的属性（通过 `NdisMSetMiniportAttributes`）
- **需要**初始化发送/接收的数据结构
- **可以**查询下层真实网卡的能力，并据此决定虚拟小端口暴露哪些特性

**🏭 现代工业实践**：

对于 MUX 中间层驱动（如 LBFO），`MPInitialize` 中要做的关键决策：

- 虚拟适配器的 MAC 地址（可以是下层某个物理网卡的 MAC，也可以是虚拟 MAC）
- 链路速度（通常是下层网卡中最快的那一个）
- 支持的 OID 集合（必须与下层网卡能力匹配）

> 💡 **洞见**：虚拟小端口的 `MPInitialize` 是中间层驱动"定义自己对外形象"的时刻。它决定了上层协议驱动（tcpip.sys）如何看待这个虚拟适配器——是什么 MAC 地址、多大带宽、支持哪些特性。这是中间层驱动实现"虚拟化"和"抽象"的核心环节。

---

## 13.3 中间层驱动发送数据包

### 13.3.1 发送数据包原理

**📌 书中的世界**

当上层协议驱动（如 tcpip.sys）通过虚拟小端口发送数据时，NDIS 调用中间层驱动的 `MPSendPackets`（NDIS 5.x）或 `MPSendNetBufferLists`（NDIS 6.x）。

**发送路径的全貌**：

```
tcpip.sys（协议驱动）
    ↓ NdisSend / NdisSendPackets / NdisSendNetBufferLists
中间层驱动 MPSendXxx（上边缘，看起来像小端口）
    ↓ 中间层驱动处理（过滤/修改/转发决策）
    ↓ NdisSend / NdisSendPackets / NdisSendNetBufferLists
下层真实小端口驱动 MPxxx（下边缘，看起来像协议驱动）
    ↓
物理网卡
```

**🔍 第一性原理视角：发送路径是"双向转换"**

中间层驱动在发送路径上必须完成两个方向的"身份转换"：

1. **从"被调用者"到"调用者"**：在上边缘，它是被 NDIS 调用的小端口（接收 `MPSendXxx` 调用）；在下边缘，它变成调用者，主动调用 `NdisSendXxx` 把包发给下层
2. **包描述符的转换**：NDIS 5.x 中必须处理 `NDIS_PACKET` 的替换/拷贝；NDIS 6.x 中可以更灵活地操作 `NET_BUFFER_LIST`

### 13.3.2 包描述符"重利用"

**📌 书中的世界**

"重利用"是指：中间层驱动**不分配新的包描述符**，而是通过包堆叠机制，在原有 `NDIS_PACKET` 的栈空间中存储自己的上下文。

**🔍 第一性原理视角：零分配转发**

微软文档说明：NDIS 5.1 中间层驱动调用 `NdisIMGetCurrentPacketStack` 检查是否有剩余栈空间：

- 如果有（`*StacksRemaining` 为 TRUE）→ 在 `IMReserved` 成员中存储上下文，**直接转发原包**给下层
- 如果没有 → 必须回退到 NDIS 5.0 模型，分配新包描述符

**"重利用"模式的操作步骤**（NDIS 5.x）：

1. 在 `MPSendPackets` 中收到上层的 `NDIS_PACKET` 数组
2. 对每个包调用 `NdisIMGetCurrentPacketStack` 检查栈空间
3. 如果有空间：在 `IMReserved` 中保存上下文（如上层原始包指针），直接调用 `NdisSendPackets` 转发
4. 在下层的 `ProtocolSendComplete` 中：从 `IMReserved` 取回上下文，调用 `NdisMSendComplete` 完成上层请求

> 💡 **洞见**："重利用"的本质是**用栈空间换堆分配**——通过在共享的 `NDIS_PACKET` 中预留自己的栈帧，避免了每次发送都进行昂贵的内存分配。这是 NDIS 5.1 在包堆叠机制下的性能优化关键。

### 13.3.3 包描述符"重申请"

**📌 书中的世界**

"重申请"是指：当包堆叠不可用（栈空间耗尽）或中间层驱动需要修改包内容时，**分配全新的 `NDIS_PACKET` 及相关缓冲区**。

**🔍 第一性原理视角：何时必须分配新包**

微软文档明确列出：

- NDIS 4.0/5.0 中间层驱动**总是**必须分配新包描述符
- NDIS 5.1 中间层驱动在 `*StacksRemaining` 为 FALSE 时**必须回退**到分配新包
- 当中间层驱动需要修改包内容（如加密、压缩、修改 IP 头）时，**通常需要分配新缓冲区并拷贝数据**

**"重申请"模式的操作步骤**（NDIS 5.x）：

1. 从包池中分配新的 `NDIS_PACKET`
2. 从缓冲区池中分配新的 `NDIS_BUFFER`
3. 把原包数据拷贝到新缓冲区（或修改后拷贝）
4. 把 OOB 数据从原包复制到新包（通过 `NDIS_OOB_DATA_FROM_PACKET` 宏和 `NdisMoveMemory`）
5. 调用 `NdisSendPackets` 发送新包
6. 在 `ProtocolSendComplete` 中：
    - 释放新包的缓冲区
    - 调用 `NdisReturnPackets` 把原包归还给上层
    - 调用 `NdisMSendComplete` 通知上层发送完成

> ⚠ **工业级陷阱**：
> 
> - 新分配的 `NDIS_PACKET` 必须**在发送完成后被正确释放**，否则内存泄漏
> - OOB 数据的拷贝必须完整，否则下层驱动可能行为异常
> - 包的归属权必须清晰：在 `ProtocolSendComplete` 中必须区分"哪个包描述符属于哪一层"

### 13.3.4 发送数据包的异步完成

**📌 书中的世界**

发送操作在 NDIS 中是**隐式异步**的——`NdisSendPackets` 返回 `NDIS_STATUS_PENDING` 表示发送操作将在稍后完成。

**🔍 第一性原理视角：异步完成的全链路**

```
MPSendPackets 被调用（上边缘）
    ↓
调用 NdisSendPackets 转发给下层（下边缘）
    ↓ 返回 NDIS_STATUS_PENDING
    ↓
... 时间流逝，下层真实网卡发送数据 ...
    ↓
下层小端口调用 NdisMSendComplete
    ↓
NDIS 调用中间层驱动的 ProtocolSendComplete（下边缘）
    ↓
中间层驱动完成清理（释放 TCB/恢复包描述符）
    ↓
调用 NdisMSendComplete 通知上层（上边缘）
    ↓
tcpip.sys 的 ProtocolSendComplete 被调用
```

**异步完成的关键约束**：

- 调用 `NdisSendPackets` 后，中间层驱动**放弃对包描述符及其资源的所有权**，直到发送完成
- 必须保持包的**顺序**：NDIS 总是保持传给 `NdisSendPackets` 的包描述符指针数组的顺序
- 下层小端口驱动也假设收到的包数组按顺序发送
- 如果中间层驱动修改了包的 OOB 数据或 Per-Packet 信息，可能改变顺序

> 💡 **洞见**：发送异步性的本质是"**调用与完成的时空分离**"。中间层驱动必须在 `MPSendPackets`（发起）和 `ProtocolSendComplete`（完成）这两个时空间点之间，维持完整的上下文信息——这正是"重利用"模式中 `IMReserved` 成员的价值所在。

---

## 13.4 中间层驱动接收数据包

### 13.4.1 接收数据包概述

**📌 书中的世界**

接收路径与发送路径对称，但方向相反：

```
物理网卡收到数据
    ↓
下层小端口调用 NdisMIndicateReceivePacket / NdisMIndicateReceiveNetBufferLists
    ↓
NDIS 调用中间层驱动的 ProtocolReceivePacket / ProtocolReceiveNetBufferLists（下边缘）
    ↓
中间层驱动处理（过滤/修改/转发决策）
    ↓
调用 NdisMIndicateReceivePacket / NdisMIndicateReceiveNetBufferLists（上边缘）
    ↓
tcpip.sys 的协议回调被调用
```

**🔍 第一性原理视角：接收路径的两种指示模式**

微软文档说明，无连接下层边缘的中间层驱动有两种接收模式：

- **`ProtocolReceivePacket`**：接收完整的包描述符（`NDIS_PACKET`）
- **`ProtocolReceive`**：接收数据被指示到中间层驱动提供的包中（当下层驱动不放弃资源所有权时）

**关键差异**：

- 如果下层驱动通过 `ProtocolReceivePacket` 指示完整包，**中间层驱动可以保留包所有权**，稍后通过 `NdisReturnPackets` 归还
- 如果下层驱动通过 `ProtocolReceive` 指示数据（不放弃所有权），**中间层驱动必须拷贝数据到自己的缓冲区**

### 13.4.2 用 PtReceive 接收数据包

**📌 书中的世界**

`PtReceive`（对应 NDIS 的 `ProtocolReceive`）是当**下层驱动不放弃包资源所有权**时被调用的接收处理函数。

**🔍 第一性原理视角：被动拷贝模式**

当底层驱动调用 `NdisMIndicateReceive` 而不是 `NdisMIndicateReceivePacket` 时，会发生这种情况。此时：

1. 中间层驱动的 `PtReceive` 被调用
2. 数据通过参数传递（HeaderBuffer、LookaheadBuffer、DataBuffer 等）
3. 中间层驱动**必须**把数据拷贝到自己的包中
4. 然后通过 `NdisMIndicateReceivePacket` 把新包指示给上层

**操作步骤**：

1. 从包池分配新的 `NDIS_PACKET`
2. 从缓冲区池分配 `NDIS_BUFFER`
3. 把 HeaderBuffer、LookaheadBuffer、DataBuffer 的数据拷贝到新缓冲区
4. 如果下层指示了 OOB 数据，调用 `NdisGetReceivedPacket` 和 `NDIS_GET_ORIGINAL_PACKET` 获取 Per-Packet 信息
5. 调用 `NdisMIndicateReceivePacket` 把新包指示给上层

> ⚠ **工业级陷阱**：
> 
> - `PtReceive` 中**必须**完成数据拷贝，因为原数据缓冲区在下层驱动收回后可能被重用
> - 拷贝必须完整：包括 HeaderBuffer + LookaheadBuffer + DataBuffer 的所有部分
> - OOB 数据必须正确传递，否则上层协议驱动可能无法正确处理包

### 13.4.3 用 PtReceivePacket 接收

**📌 书中的世界**

`PtReceivePacket`（对应 NDIS 的 `ProtocolReceivePacket`）是当**下层驱动放弃包资源所有权**时被调用的接收处理函数。

**🔍 第一性原理视角：零拷贝转发模式**

微软文档明确：当 `ProtocolReceivePacket` 收到完整包时，中间层驱动可以**保留包所有权**，直到数据被消费后才通过 `NdisReturnPackets` 归还。

**两种处理策略**：

1. **拷贝策略**：把数据拷贝到中间层分配的缓冲区，把原包立即归还下层，把新包指示给上层
2. **链式策略**：创建新的包描述符，把原包的缓冲区链（buffer chain）链接到新包描述符，把新包指示给上层。当上层归还新包时，**必须把缓冲区从新包描述符上解链，重新链接到原包描述符，然后归还原包给下层**

**链式策略的操作步骤**：

1. 分配新的 `NDIS_PACKET`（但**不分配新缓冲区**）
2. 把原 `NDIS_PACKET` 的缓冲区链链接到新包
3. 调用 `NdisMIndicateReceivePacket` 把新包指示给上层
4. 当上层通过 `MPReturnPackets` 归还新包时：
    - 把缓冲区从新包描述符解链
    - 把缓冲区重新链接到原包描述符
    - 调用 `NdisReturnPackets` 把原包归还下层

> 💡 **洞见**：链式策略是"零拷贝接收"的核心——通过在不同包描述符间移动缓冲区所有权，避免了数据拷贝。这是高性能网络驱动的关键优化。NDIS 6.x 的 `NET_BUFFER_LIST` 把这种"缓冲区链式管理"做得更加优雅：通过 `NET_BUFFER` 的 `CurrentMdl` 和 `DataOffset` 字段，驱动可以零拷贝地"切片"数据包。

### 13.4.4 对包进行过滤

**📌 书中的世界**

中间层驱动的核心价值之一就是**包过滤**——检查每个收发的数据包，根据规则决定是否放行、修改或丢弃。

**🔍 第一性原理视角：过滤的决策点**

在发送和接收路径上，中间层驱动有多个"决策点"：

- **发送路径**：在 `MPSendPackets` 中，可以检查每个待发送的包
- **接收路径**：在 `PtReceivePacket` 或 `PtReceive` 中，可以检查每个待接收的包

**过滤决策的三种结果**：

1. **放行（Pass-Through）**：包不作修改，直接转发
2. **修改（Modify）**：包内容被修改（如加密、压缩、NAT 改写 IP 头）
3. **丢弃（Drop）**：包被丢弃，不转发

**丢弃包的处理**：

- 发送路径上丢弃：直接调用 `NdisMSendComplete` 返回失败状态，**不调用**​ `NdisSendPackets`
- 接收路径上丢弃：不调用 `NdisMIndicateReceivePacket`，直接归还包给下层

> 💡 **洞见**：过滤的"决策点"位置决定了性能和语义。在 `MPSendPackets` 最开头就做过滤检查，可以尽早丢弃非法包，节省后续处理开销。但这要求过滤逻辑能够快速判断——复杂的深度包检测（DPI）可能更适合在 Work Item 中异步处理。

**🏭 现代工业实践**：

- **简单过滤**（基于 IP/Port/Protocol）：在 `MPSendNetBufferLists` / `ProtocolReceiveNetBufferLists` 中同步完成
- **复杂过滤**（DPI、应用层协议识别）：在 WFP 的 ALE 层或 Stream 层做，而不是在 NDIS 中间层驱动中
- **工业级防火墙/EDR**：现代方案是 **WFP callout 驱动**，而不是 NDIS 中间层驱动

---

## 13.5 中间层驱动程序查询和设置

### 13.5.1 查询请求的处理

**📌 书中的世界**

OID（Object Identifier）查询请求从上层协议驱动下发，中间层驱动在 `MPOidRequest`（NDIS 6.x）或 `MPQueryInformation`（NDIS 5.x）中处理。

**🔍 第一性原理视角：OID 是"控制平面的对话"**

微软文档明确：中间层驱动在 `MiniportOidRequest` 中收到查询请求时，有三种处理方式：

1. **直接透传**：把请求转发给下层小端口驱动
2. **本地响应**：中间层驱动自己响应请求（基于它在上边缘导出的介质类型）
3. **修改后透传**：修改请求或响应内容

**NDIS 6.x 的 OID 转发机制**：

```
// 1. 分配克隆的 OID 请求
NdisAllocateCloneOidRequest(BindingHandle, OidRequest, SourceHandle, &ClonedOidRequest);

// 2. 发送克隆的请求给下层
NdisOidRequest(BindingHandle, ClonedOidRequest);

// 3. 在 ProtocolOidRequestComplete 中：
//    - 处理响应
//    - 调用 NdisFreeCloneOidRequest 释放克隆请求
//    - 调用 NdisMOidRequestComplete 通知上层
```

**关键约束**：

- **所有 `OID_PNP_XXX` 请求必须透传给下层小端口驱动**
- NDIS 6.0 中间层驱动可以取消 OID 请求（`NdisCancelOidRequest`）
- 如果中间层驱动修改 TCP 网络数据内容导致 TCP 卸载功能无法执行，**必须响应 `OID_TCP_OFFLOAD_CURRENT_CONFIG` 查询为 `NDIS_STATUS_NOT_SUPPORTED`**，而不是透传请求

> ⚠ **工业级陷阱**：
> 
> - TCP 卸载（TCP Offload）是 NDIS 6.x 的重要特性。如果中间层驱动修改了 TCP 数据（如做深度包检测），会导致下层网卡的 TCP 卸载失效——此时必须正确响应 OID 查询，告知上层"不支持 TCP 卸载"
> - 这是中间层驱动与卸载功能共存的关键契约

### 13.5.2 设置请求的处理

**📌 书中的世界**

OID 设置请求允许上层协议驱动配置中间层驱动的虚拟小端口。

**🔍 第一性原理视角：设置请求验证**

微软文档明确：如果中间层驱动自己处理 OID 设置（而不是透传给下层），**必须验证要设置的值**。如果值超出边界，应当使设置请求失败。

**常见设置 OID**：

- `OID_GEN_CURRENT_PACKET_FILTER`：设置包过滤器（决定哪些类型的包要指示给上层）
- `OID_802_3_MULTICAST_LIST`：设置多播地址列表
- `OID_GEN_CURRENT_LOOKAHEAD`：设置前视缓冲区大小

**处理逻辑**：

1. 验证设置值的合法性
2. 如果合法：应用设置，返回 `NDIS_STATUS_SUCCESS`
3. 如果不合法：返回 `NDIS_STATUS_INVALID_DATA` 或其他错误状态
4. 某些设置可能需要转发给下层真实网卡（如包过滤器）

> 💡 **洞见**：OID 设置请求的验证是驱动稳定性的重要防线。**永远不要信任上层传来的值**——这是 Windows 内核驱动开发的黄金法则。一个未经验证的 OID 设置值可能导致缓冲区溢出、资源耗尽或系统崩溃。

---

## 13.6 NDIS 句柄

### 13.6.1 不可见的结构指针

**📌 书中的世界**

NDIS 大量使用"句柄（Handle）"——它本质上是一个不透明的指针，指向 NDIS 内部管理的结构。驱动开发者**不应该（也无法）直接解引用这些句柄**，只能通过 NDIS 提供的 API 来使用它们。

**🔍 第一性原理视角：句柄是"能力的授权"**

NDIS 句柄的本质：**它是 NDIS 授予驱动的一种"能力凭证"**。当你调用 `NdisOpenAdapterEx` 获得 `NdisBindingHandle`，这个句柄代表了"你有权限向这个适配器发送数据"；当你调用 `NdisRegisterProtocolDriver` 获得 `NdisProtocolHandle`，这个句柄代表了"你注册的协议驱动身份"。

**为什么句柄是不透明的？**

- **封装实现细节**：NDIS 内部结构的布局可以改变，只要句柄语义不变，驱动就不需要重新编译
- **安全隔离**：驱动无法直接篡改 NDIS 内部状态
- **生命周期管理**：NDIS 可以通过句柄追踪资源的所有权

### 13.6.2 常见的 NDIS 句柄

**📌 书中的世界**

中间层驱动中常见的 NDIS 句柄：

- **`NdisMiniportDriverHandle`**：`NdisMRegisterMiniportDriver` 返回，标识中间层驱动的小端口部分
- **`NdisProtocolHandle`**：`NdisRegisterProtocolDriver` 返回，标识中间层驱动的协议部分
- **`NdisBindingHandle`**：`NdisOpenAdapterEx` 返回，标识与下层小端口的绑定
- **`NdisHandle`**（MiniportAdapterHandle）：`MiniportInitializeEx` 的参数，标识虚拟小端口适配器
- **`NDIS_PACKET` / `NET_BUFFER_LIST` 相关的句柄**：包描述符本身的句柄

**🏭 现代工业实践**：

微软文档明确强调：中间层驱动**必须保存**​ `NdisMiniportDriverHandle` 和 `NdisProtocolHandle`，因为后续的 `NdisXxx` 函数调用都需要这些句柄。

### 13.6.3 NDIS 句柄误用问题

**📌 书中的世界**

句柄误用是 NDIS 驱动最常见的 bug 来源之一：

- 使用已释放的句柄
- 在错误的上下文使用句柄（如把一个绑定的句柄用到另一个绑定）
- 句柄类型混淆（把 Miniport 句柄当作 Protocol 句柄使用）

**🔍 第一性原理视角：句柄生命周期的严格约束**

每个 NDIS 句柄都有严格的生命周期：

```
分配（如 NdisOpenAdapterEx）
    ↓
使用（发送/接收/OID 请求）
    ↓
释放（如 NdisCloseAdapterEx）
    ↓
句柄失效——后续使用将导致系统崩溃
```

**常见误用场景**：

1. **Use-After-Free**：在 `NdisCloseAdapterEx` 之后继续使用 `NdisBindingHandle`
2. **句柄泄露**：忘记调用 `NdisCloseAdapterEx` 释放绑定句柄
3. **IRQL 违规**：在某些 NDIS API 要求 `PASSIVE_LEVEL` 时，却在 `DISPATCH_LEVEL` 使用句柄

> ⚠ **工业级陷阱**：
> 
> - 句柄误用导致的 bug check 通常表现为 `DRIVER_IRQL_NOT_LESS_OR_EQUAL` 或 `PAGE_FAULT_IN_NONPAGED_AREA`
> - 这类 bug 难以调试，因为崩溃点与根因可能相距甚远
> - 使用 Driver Verifier 的 NDIS 校验功能可以提前发现句柄误用

### 13.6.4 一种解决方案

**📌 书中的世界**

书中提出了一种管理 NDIS 句柄生命周期的模式：**在适配器上下文中集中存储所有句柄**，并严格定义它们的生命周期阶段。

**🔍 第一性原理视角：句柄管理的"单一事实来源"**

设计原则：

1. **集中存储**：把所有 NDIS 句柄放在适配器上下文结构中
2. **状态机驱动**：句柄的有效性由适配器状态机保证（如只有在 `Bound` 状态，`NdisBindingHandle` 才有效）
3. **配对操作**：每个句柄的分配都有明确的对应的释放点
4. **断言检查**：在调试版本中，使用断言验证句柄的有效性

**伪代码模式**：

```
typedef struct _ADAPTER_CONTEXT {
    // 句柄集中存储
    NDIS_HANDLE MiniportAdapterHandle;
    NDIS_HANDLE BindingHandle;
    // 状态机
    ADAPTER_STATE State;  // Initialized -> Bound -> Paused -> Running -> Halted
    // 句柄有效性标志
    BOOLEAN BindingHandleValid;
} ADAPTER_CONTEXT;

// 使用时验证
NT_ASSERT(Adapter->State >= ADAPTER_STATE_BOUND);
NT_ASSERT(Adapter->BindingHandleValid);
NdisSend(Adapter->BindingHandle, ...);
```

> 💡 **洞见**：句柄管理本质上是一个**生命周期管理问题**。把句柄与状态机关联，让状态机成为"句柄有效性的裁判"，可以系统性地避免句柄误用。这也是为什么 NDIS 6.x 引入了严格的适配器状态机（Halted → Initializing → Paused → Running）——状态机不仅是功能需要，更是**安全需要**。

---

## 13.7 生成普通控制设备

### 13.7.1 在中间层驱动中添加普通设备

**📌 书中的世界**

中间层驱动可能需要一个"控制设备"——用于与用户态应用程序通信，接收配置命令、传递统计数据等。

**🔍 第一性原理视角：控制设备是"用户态与内核态的桥梁"**

虽然中间层驱动主要在网络栈内部工作，但它往往需要：

- 从用户态接收过滤规则配置
- 向用户态上报统计数据和告警
- 提供 IOCTL 接口供用户态管理

**实现方式**：创建一个普通的 WDM 设备对象（Control Device Object），通过 `IoCreateDevice` 创建，并通过符号链接暴露给用户态。

**🏭 现代工业实践**：

对于使用 WDF 构建的中间层驱动，推荐使用 WDF 的 **Control Device**​ 机制：

- 使用 `WdfControlDeviceInitAllocate` 创建控制设备初始化结构
- 使用 `WdfDeviceCreate` 创建控制设备
- 使用 `WdfDeviceCreateSymbolicLink` 创建符号链接
- 通过 `WdfIoQueueCreate` + `WdfRequestForwardToIoQueue` 处理 IOCTL 请求

### 13.7.2 使用传统方法来生成控制设备

**📌 书中的世界**

书中展示的是传统的 WDM 方法：

1. 在 `DriverEntry` 中调用 `IoCreateDevice` 创建控制设备
2. 调用 `IoCreateSymbolicLink` 创建符号链接
3. 设置 `MajorFunction` 数组中的 `IRP_MJ_DEVICE_CONTROL` 处理函数
4. 在 `Unload` 例程中调用 `IoDeleteSymbolicLink` 和 `IoDeleteDevice` 清理

**🔍 第一性原理视角：控制设备的生命周期**

控制设备的生命周期**独立于虚拟小端口**：

- 控制设备在 `DriverEntry` 中创建
- 控制设备在 `Unload` 中销毁
- 它与 NDIS 的绑定/解绑无关

**关键约束**：

- 控制设备应该使用独立的设备名（如 `\Device\MyIMDriver`）
- 符号链接应该放在 ``\??\` 或``\DosDevices` 下（如 `\DosDevices\MyIMDriver`）
- 用户态通过 `CreateFile("\\\\.\\MyIMDriver", ...)` 打开设备

> ⚠ **工业级陷阱**：
> 
> - 控制设备的 IRQL：IOCTL 处理函数在 `PASSIVE_LEVEL` 调用，可以安全地访问分页内存
> - 同步问题：用户态的配置请求可能与 NDIS 回调函数并发执行，必须使用自旋锁或互斥体保护共享数据
> - 对于 NDIS 6.x 驱动，微软推荐使用 WDF 的 Control Device，因为它自动处理了 PnP 和电源管理的复杂性

> 💡 **洞见**：控制设备是中间层驱动"接地气"的部分——它让内核态的网络栈组件能够与用户态的管理程序对话。这是"内核态处理高速数据路径 + 用户态处理策略配置"这一经典架构模式的具体体现。现代 EDR 和防火墙产品都遵循这一模式：WFP callout 驱动在内核态做高速包过滤，用户态服务进程通过 IOCTL 或共享内存下发过滤规则。

---

## 🎯 本章核心洞见总结

读完第13章，我希望你带走这七个**第一性原理**：

**1. 中间层驱动是 NDIS 栈中的"双面间谍"**

上边缘是虚拟小端口（对 tcpip.sys 伪装成网卡），下边缘是协议驱动（对真实网卡伪装成协议）。内部通过虚拟小端口把两侧绑定。这种双重身份是中间层驱动所有特性的根源。

**2. 虚拟小端口是"欺骗的艺术"**

通过 `NdisIMInitializeDeviceInstanceEx` 创建虚拟适配器，让上层协议驱动无条件接受。虚拟小端口的 MAC 地址、链路速度、OID 能力都由中间层驱动自己定义——这是网络虚拟化的基石。

**3. 包描述符管理是中间层驱动的性能命脉**

- NDIS 5.0：必须分配新包描述符，拷贝 OOB 数据
- NDIS 5.1：包堆叠机制允许"重利用"，避免分配
- NDIS 6.x：`NET_BUFFER_LIST` 让缓冲区链式管理变得优雅

"重利用 vs 重申请"的本质权衡：**零分配转发 vs 内容可修改**。

**4. 发送/接收路径是"双向身份转换"**

发送：`MPSendXxx`（被调用）→ 中间层处理 → `NdisSendXxx`（主动调用）

接收：`ProtocolReceiveXxx`（被调用）→ 中间层处理 → `NdisMIndicateReceiveXxx`（主动调用）

异步完成的全链路：发送完成从下层 `ProtocolSendComplete` 冒泡到上层 `NdisMSendComplete`；接收指示从上层的 `NdisMIndicateReceiveXxx` 下沉到下层的 `ProtocolReceiveXxx`。

**5. OID 是控制平面的"结构化对话"**

- 查询/设置请求三种处理方式：透传、本地响应、修改后透传
- `OID_PNP_XXX` 必须透传
- 修改 TCP 数据导致卸载失效时，必须响应 `OID_TCP_OFFLOAD_CURRENT_CONFIG` 为 `NDIS_STATUS_NOT_SUPPORTED`
- NDIS 6.x 通过 `NdisAllocateCloneOidRequest` / `NdisFreeCloneOidRequest` 管理克隆请求

**6. NDIS 句柄是"能力的授权凭证"**

- 不透明指针，驱动不得直接解引用
- 生命周期必须严格管理
- 集中存储 + 状态机驱动 + 配对操作是避免句柄误用的系统工程方案

**7. 控制设备是"用户态与内核态的桥梁"**

- 独立于 NDIS 绑定生命周期
- 现代推荐使用 WDF Control Device
- 用户态配置 + 内核态执行的经典架构

### 工业现实导航

|场景|推荐技术|说明|
|---|---|---|
|网络流量过滤/监控|**NDIS 6.x Lightweight Filter**​ 或 **WFP**​|微软官方推荐|
|LBFO、n 到 m 多路复用|**NDIS 中间层驱动**​|中间层驱动的保留阵地|
|虚拟化、VLAN、VPN 聚合|**NDIS 中间层驱动**​|MUX 类型|
|防火墙、EDR 网络遥测|**WFP callout 驱动**​|感知应用进程，现代标准方案|
|物理网卡驱动|**NetAdapterCx**（新）/ NDIS 6.x Miniport（旧）|见第12章|
|自定义协议|**NDIS 6.x 协议驱动**​|见第11章|

> ⚠ **工业现实警示**：
> 
> - **NDIS 5.x 已弃用**，新项目必须使用 NDIS 6.x
> - **如果只是为了过滤/监控网络流量，不要用中间层驱动**——NDIS Filter 驱动或 WFP 才是正确选择
