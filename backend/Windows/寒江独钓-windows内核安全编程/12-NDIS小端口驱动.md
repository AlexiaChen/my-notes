
第12章"NDIS 小端口驱动"是全书网络部分的**压轴**——它把视角推到了 Windows 网络栈的最底层：不再是"在链路层之上实现协议"（第11章 NDIS 协议驱动），而是"**直接管理网卡硬件、为上层协议驱动提供链路层服务**"。书中以一个巧妙的示例 `ndisedge` 展开：它并不是一个真实的物理网卡驱动，而是一个**虚拟小端口驱动**——通过打开上一章的 `ndisprot` 协议驱动设备，把"小端口"与"协议驱动"首尾相连，形成一个完整的虚拟网卡。

这个设计非常精妙：它让你在一个示例中同时看到 NDIS 栈的**两端**——小端口的下边界（面向"网卡"，这里是 ndisprot 设备）和上边界（面向协议驱动，如 tcpip.sys）。微软文档明确指出："An NDIS miniport driver has two basic functions: Managing a network interface card (NIC), including sending and receiving data through the NIC. Interfacing with higher-level drivers, such as filter drivers, intermediate drivers, and protocol drivers"——管理网卡 + 与上层驱动交互，这正是本章的核心。

但必须 upfront 说明工业现实：书中示例基于 **NDIS 5.x/6.x 经典小端口模型**。在现代 Windows 10/11 上，**新开发的小端口驱动已被微软引导向 NetAdapterCx（WDF 类扩展）模型**——"NDIS pushes many hard problems to the client driver such as data path synchronization, PnP and power handling. NetAdapterCx solves this by taking over the responsibility of synchronizing power and PnP event"。理解经典模型是读懂原理的基础，但新项目应当从 NetAdapterCx 起步。

下面按你给的目录结构，逐节做读书笔记。

---

## 12.1 小端口驱动的应用与概述

### 12.1.1 小端口驱动的应用

**📌 书中的世界**

小端口驱动是 NDIS 栈的"底座"——它直接控制网卡硬件（或虚拟设备），向上为协议驱动、过滤驱动、中间驱动提供统一的接口。微软文档明确："A miniport driver manages miniport adapters and provides an interface to the adapters for higher-level drivers. A miniport adapter is a conceptual entity that can represent either a physical device or a virtual device"。

**🔍 第一性原理视角：小端口驱动是"硬件与网络栈之间的契约"**

小端口驱动的本质价值：**屏蔽硬件差异，向 NDIS 提供统一的 MiniportXxx 入口**。无论是 Intel 千兆网卡、Realtek 无线网卡、还是一个纯软件的虚拟网卡，对上层 tcpip.sys 来说，看到的都是同一组 MiniportXxx 函数调用。

**🏭 工业应用场景**：

- **物理网卡驱动**：Intel e1000、Realtek RTL8139、Broadcom 等
- **虚拟网卡**：Hyper-V vSwitch 的虚拟 NIC、Docker 的 NAT 适配器
- **VPN 虚拟网卡**：Cisco AnyConnect、OpenVPN TAP、WireGuard 的虚拟网卡
- **移动宽带调制解调器**：通过 USB 总线的 MBB 设备（其小端口通过 non-NDIS 下边界与总线通信）
- **中间驱动的上边界**：IM 驱动向协议驱动呈现的"虚拟小端口"

> 💡 **洞见**：miniport adapter 是"概念实体"——它可以代表物理设备，也可以代表虚拟设备。这就是为什么一个"假的"小端口（如本书示例）在 NDIS 栈中与真实网卡小端口地位完全等价：tcpip.sys 不知道也不关心下层是真实硬件还是虚拟设备。

### 12.1.2 小端口驱动的实例

**📌 书中的世界**

本书的示例 `ndisedge` 是一个**虚拟小端口驱动**：

- 它**不控制真实硬件**
- 它通过打开 `ndisprot` 协议驱动创建的设备，把"网卡"抽象成对端协议驱动的以太网帧通道
- 它向 NDIS 注册为小端口，tcpip.sys 可以像绑定真实网卡一样绑定到它
- 用户态程序通过 `ndisprot` 设备注入/提取原始以太网帧

**🔍 第一性原理视角：这是一个"回环小端口"**

`ndisedge` 的架构巧妙地形成了回路：

```
tcpip.sys（协议驱动）
    ↓ 绑定
ndisedge（小端口驱动，虚拟网卡）
    ↓ 通过 ndisprot 设备转发
ndisprot（协议驱动，第11章示例）
    ↓ 绑定
真实网卡 Miniport（如 Intel e1000）
    ↓
物理网络
```

这个设计让你在一个示例中看到：

- 小端口的**上边界**：如何与 tcpip.sys 交互（MiniportXxx 回调）
- 小端口的**下边界**：如何"发送/接收"数据（这里转发给 ndisprot 设备）

> 💡 **洞见**：这是教科书级的"全栈可视化"设计——一个虚拟小端口把 NDIS 栈的两端（协议侧 + 设备侧）都暴露给开发者。理解了这个示例，你就理解了 NDIS 栈的完整数据流。

### 12.1.3 小端口驱动的运作与编程概述

**📌 书中的世界**

小端口驱动编程的核心：

1. `DriverEntry` 中调用 `NdisMRegisterMiniport` 注册小端口
2. 实现 `MiniportXxx` 系列回调（Initialize、Halt、Send、ReturnPacket、OID 处理等）
3. NDIS 在合适时机调用这些回调

**🏭 现代工业演进**：

在 NDIS 6.0+ 中，`NdisMRegisterMiniport` 被 `NdisMRegisterMiniportDriver` 取代，`MiniportInitialize` 被 `MiniportInitializeEx` 取代，`MiniportHalt` 被 `MiniportHaltEx` 取代。

更根本的变化是微软在 Windows 10/11 上的方向性引导：**新小端口驱动应当采用 NetAdapterCx 模型**——

- 传统 NDIS 小端口：驱动自己处理 PnP、电源、数据路径同步
- NetAdapterCx 小端口：驱动是 WDF 客户端，PnP/电源/队列同步由 NetAdapter 类扩展接管

> ⚠ **工业现实**：
> 
> - **物理网卡驱动**：新项目应使用 NetAdapterCx（WDF + NetAdapter Class Extension）
> - **虚拟网卡/VPN**：仍可使用 NDIS 6.x 小端口模型，或使用 WFP 的流向重定向
> - **教学/原理理解**：经典 NDIS 小端口模型仍然是最好的教材

---

## 12.2 小端口驱动的初始化

### 12.2.1 小端口驱动的 DriverEntry

**📌 书中的世界**

小端口驱动的 `DriverEntry` 与常规 WDM 驱动不同：

- 不调用 `IoCreateDevice` 创建设备
- 调用 `NdisMRegisterMiniport` 向 NDIS 注册
- 通过 `NDIS_MINIPORT_CHARACTERISTICS` 结构体声明所有 MiniportXxx 回调

**🔍 第一性原理视角：注册 = "向 NDIS 宣告我能做什么"**

`NDIS_MINIPORT_CHARACTERISTICS` 是小端口与 NDIS 之间的"契约"——你告诉 NDIS："当需要初始化适配器时调用我的 `MPInitialize`，当需要停止时调用 `MPHalt`，当有包要发送时调用 `MPSendPackets`……"

**🏭 现代工业演进**：

NDIS 6.x 中：

- `NdisMRegisterMiniport` → `NdisMRegisterMiniportDriver`
- `NDIS_MINIPORT_CHARACTERISTICS` 中 `MajorNdisVersion = 6`
- `MiniportInitialize` → `MiniportInitializeEx`
- `MiniportHalt` → `MiniportHaltEx`

而 NetAdapterCx 模型更进一步：驱动是 WDF 客户端，`DriverEntry` 中调用 `WdfDriverCreate` + `NetAdapterCx` 的初始化函数，PnP/电源由框架接管。

### 12.2.2 小端口驱动的适配器结构

**📌 书中的世界**

小端口驱动为每个"适配器"（Adapter）维护一个上下文结构（Adapter Context），保存：

- 适配器状态
- 绑定到的协议驱动信息
- 发送/接收控制块池
- OID 处理状态
- 配置信息

**🔍 第一性原理视角：适配器上下文是"每实例状态的容器"**

微软文档描述了一个严谨的**适配器状态机**：

```
Halted → Initializing → Paused ⇄ Running
                    ↓
                  Pausing → Paused
                    ↓
                  Restarting → Running
                    ↓
                  Halted（halt 后）
```

关键状态转换：

- **Halted → Initializing**：NDIS 调用 `MiniportInitializeEx`
- **Initializing → Paused**：`MiniportInitializeEx` 成功返回后，适配器进入 Paused 状态
- **Paused → Restarting → Running**：NDIS 调用 `MiniportRestart`
- **Running → Pausing → Paused**：NDIS 调用 `MiniportPause`
- **任意状态 → Halted**：NDIS 调用 `MiniportHaltEx`

> 💡 **洞见**：这个状态机是 NDIS 6.x 引入的**运行时重配置（Runtime Reconfiguration）**能力的核心。它让适配器能够在不卸载驱动的情况下暂停/恢复、重新配置——这是现代网卡热插拔、电源管理、RSS 重配置的基础。

### 12.2.3 配置信息的读取

**📌 书中的世界**

小端口驱动在 `MPInitialize` 中通过 `NdisOpenConfiguration` / `NdisReadConfiguration` 读取注册表中的配置参数（如 MAC 地址、连接速度等）。

**🔍 第一性原理视角：配置与代码分离**

配置信息（MAC 地址、链路速度、是否启用某些特性）不应硬编码在驱动中，而应从注册表读取。这是 Windows 驱动的标准做法——让同一个驱动二进制能够适应不同的部署场景。

### 12.2.4 设置小端口适配器上下文

**📌 书中的世界**

`MPInitialize` 中分配适配器上下文结构，并通过 `NdisMSetMiniportAttributes` 告知 NDIS。

**🏭 现代工业实践**：

微软文档明确指出：`MiniportInitializeEx` **必须先调用 `NdisMSetMiniportAttributes`，然后才能调用任何声明硬件资源的 NdisXxx 函数**（如 `NdisMRegisterIoPortRange` 或 `NdisMMapIoSpace`）。这是一个严格的顺序约束——**属性设置必须在资源分配之前**。

### 12.2.5 MPInitialize 的实现

**📌 书中的世界**

`MPInitialize` 是小端口的"出生证明"回调，NDIS 在适配器从 Halted 状态进入 Initializing 状态时调用。其核心任务：

1. 分配适配器上下文
2. 读取配置信息
3. 分配资源：包池、缓冲池、自旋锁、定时器
4. 调用 `NdisMSetMiniportAttributes` 设置属性
5. 如果控制真实硬件：映射 I/O 端口、注册中断、设置 DMA
6. 返回 `NDIS_STATUS_SUCCESS` → 适配器进入 Paused 状态

**🔍 第一性原理视角：MPInitialize 是"资源分配的状态机起点"**

微软文档详细描述了 `MiniportInitializeEx` 可以分配的资源：

- **Non-paged pool memory**：内核非分页池内存
- **NET_BUFFER 和 NET_BUFFER_LIST 结构池**：用于收发数据包
- **Spin locks**：自旋锁，保护共享资源
- **Timers**：定时器，用于周期性任务
- **I/O ports**：I/O 端口（真实硬件）
- **DMA**：直接内存访问
- **Shared memory**：共享内存
- **Interrupts**：中断

**关键约束**：

- `MiniportInitializeEx` 在 `PASSIVE_LEVEL` 调用
- 必须先 `NdisMSetMiniportAttributes`，再分配 DMA 等资源
- 如果支持总线主控 DMA，必须按此顺序：`NdisMSetMiniportAttributes` → `NdisMRegisterScatterGatherDma` → `NdisMAllocateSharedMemory`
- 调用 `NdisMRegisterInterruptEx` 后，可能立即开始收到中断
- 成功返回后适配器进入 **Paused 状态**，NDIS 会调用 `MiniportRestart` 将其转入 Running 状态

**书中的虚拟小端口简化**：

因为是虚拟设备，`MPInitialize` 不需要映射 I/O 端口、注册中断、设置 DMA。它主要分配：

- 适配器上下文
- 发送控制块（TCB）池
- 接收控制块（RCB）池
- 与 ndisprot 设备通信的 WDF I/O 目标

**🏭 现代工业实践**：

对于真实网卡小端口：

```
// 1. 设置适配器属性
NdisMSetMiniportAttributes(MiniportAdapterHandle, &AdapterAttributes);

// 2. 映射 I/O 端口或内存
NdisMMapIoSpace(&MMIOBase, MiniportAdapterHandle, ...);

// 3. 注册中断
NdisMRegisterInterruptEx(&InterruptHandle, MiniportAdapterHandle, ...);

// 4. 设置 DMA
NdisMRegisterScatterGatherDma(MiniportAdapterHandle, &SGDmaDescription);

// 5. 分配包池
NdisAllocateNetBufferListPool(MiniportAdapterHandle, &PoolParameters);
```

### 12.2.6 MPHalt 的实现

**📌 书中的世界**

`MPHalt` 是 `MPInitialize` 的逆操作，NDIS 在适配器需要停止时调用。其核心任务：

1. 释放所有资源
2. **逆序**调用 `NdisXxx` 函数的"反函数"

**🔍 第一性原理视角：资源释放的逆序铁律**

微软文档明确："As a general rule, a MiniportHaltEx function should call the reciprocal NdisXxx functions in reverse order to the calls the driver made from MiniportInitializeEx"。

**关键约束**：

- `MiniportHaltEx` 在 `PASSIVE_LEVEL` 调用
- NDIS **保证在调用 `MiniportHaltEx` 时没有未完成的 OID 请求或发送请求**
- 调用 `MiniportHaltEx` 后，NDIS 不会再提交新请求
- 如果网卡可能产生中断，`MiniportHaltEx` 应先禁用中断，尽快调用 `NdisMDeregisterInterruptEx`（因为在该调用返回前，仍可能被 `MiniportInterrupt` 抢占）
- 如果有挂起的定时器，应先调用 `NdisCancelTimerObject`；如果取消失败（定时器可能已经触发），必须等待定时器回调完成才能返回
- 返回后适配器回到 **Halted 状态**

> ⚠ **工业级陷阱**：
> 
> - 中断注销的抢占问题：`MiniportHaltEx` 在禁用中断前可能被 `MiniportInterrupt` 抢占——必须设计成可重入安全的
> - 定时器竞态：定时器可能在 `MiniportHaltEx` 执行期间触发，必须等待其完成
> - 资源逆序释放：严格遵守"初始化逆序"是内核驱动的金科玉律

**🏭 现代工业实践**：

`MPHalt` 与第9章 Minifilter 的 `InstanceTeardownCompleteCallback`、第11章 NDIS 协议驱动的 `ndisprotShutdownBinding` 遵循完全相同的"停止-完成-释放-移除"四步法——这是 Windows 内核资源生命周期管理的统一范式。

---

## 12.3 打开 ndisprot 设备

### 12.3.1 I/O 目标

**📌 书中的世界**

本书示例使用 **WDF（Windows Driver Framework）**​ 来构建小端口驱动——这是《寒江独钓》与时俱进的地方。书中使用 WDF 的 **I/O Target**​ 机制来与 `ndisprot` 设备进行通信。

**🔍 第一性原理视角：I/O Target 是"WDF 中的 IRP 目标抽象"**

WDF 的 I/O Target 代表了"可以接收 I/O 请求的对象"——可以是本地设备、远程设备、另一个驱动等。在小端口驱动中，`ndisprot` 设备就是一个 I/O Target，通过它发送 DeviceIoControl、WriteFile、ReadFile 请求。

**🏭 现代工业实践**：

使用 WDF 构建小端口驱动的好处（微软在 NetAdapterCx 文档中明确强调）：

- WDF 提供面向对象的、事件驱动的模型
- 智能默认的 PnP 和电源管理
- 简化数据路径同步
- 降低 bug check 风险

> 💡 **洞见**：书中示例"用 WDF 构建 NDIS 小端口"的做法，正预示了微软后来的 NetAdapterCx 方向——**WDF 管理 PnP/电源/同步，NDIS 只负责数据和控制路径**。这是 Windows 驱动框架化的统一趋势。

### 12.3.2 给 IO 目标发送 DeviceIoControl 请求

**📌 书中的世界**

小端口驱动通过 WDF I/O Target 向 `ndisprot` 发送 DeviceIoControl 请求：

- 枚举可用的绑定（网卡列表）
- 绑定到指定网卡
- 设置 Ethernet 类型过滤器

**🔍 第一性原理视角：这是"小端口下边界"的初始化**

对于真实网卡小端口，`MPInitialize` 中应该做的是映射 I/O 端口、注册中断；对于虚拟小端口 `ndisedge`，"初始化下边界"就是**打开 ndisprot 设备并发控制请求绑定到真实网卡**。

### 12.3.3 打开 ndisprot 接口并完成配置设备

**📌 书中的世界**

完整的"下边界初始化"流程：

1. 通过 WDF I/O Target 打开 `\\.\NdisProt` 设备
2. 发送 IOCTL 枚举可用绑定
3. 选择一个绑定（如真实以太网网卡）
4. 发送 IOCTL 完成绑定
5. 保存 I/O Target 句柄供后续收发使用

**🔍 第一性原理视角：虚拟小端口的"下边界"是另一个 NDIS 协议驱动**

这是一个非常巧妙的设计——`ndisedge` 的"网卡"实际上是 `ndisprot` 协议驱动，而 `ndisprot` 又绑定到真实网卡 Miniport。整个链路：

```
tcpip.sys
    ↓ 绑定
ndisedge（虚拟小端口）
    ↓ 通过 ndisprot 设备
ndisprot（协议驱动）
    ↓ 绑定
真实网卡 Miniport（如 Intel e1000）
    ↓
物理网络
```

> 💡 **洞见**：这个"俄罗斯套娃"式的设计让一个虚拟小端口能够透明地转发流量到真实网络。工业界的 **VPN 虚拟网卡**正是这种模式——虚拟小端口收到 tcpip.sys 下发的数据包，加密后通过用户态或内核态通道发送到 VPN 网关，网关解密后通过真实网卡发出。

---

## 12.4 使用 ndisprot 发送包

### 12.4.1 小端口驱动的发包接口

**📌 书中的世界**

NDIS 调用小端口的 `MPSendPackets`（NDIS 5.x）或 `MPSendNetBufferLists`（NDIS 6.x）回调来发送数据包。在小端口驱动中，这个回调的责任是：**把 NDIS 上交的数据包发送到"网卡"**——对于 `ndisedge`，就是转发给 `ndisprot` 设备。

### 12.4.2 发送控制块（TCB）

**📌 书中的世界**

TCB（Transmit Control Block）是小端口为每个发送请求维护的上下文结构，保存：

- 对应的 NET_BUFFER_LIST 指针
- 包的状态
- 与 ndisprot 设备的 I/O 请求关联
- 完成回调上下文

**🔍 第一性原理视角：TCB 是"发送操作的身份证"**

当一个发送请求从 tcpip.sys 下发到 `ndisedge` 时，小端口必须：

1. 分配一个 TCB
2. 把 NDIS 数据包的信息（如 NET_BUFFER_LIST）保存到 TCB
3. 通过 ndisprot 设备把数据包发送出去
4. 当发送完成时，通过 TCB 找到原始 NDIS 数据包并完成它

> 💡 **洞见**：TCB 模式与第11章协议驱动中的"包上下文"模式完全同构——**异步操作中必须用上下文结构把"发起请求"和"完成请求"桥接起来**。这是 Windows 内核异步模型的通用范式。

### 12.4.3 遍历包组并填写 TCB

**📌 书中的世界**

NDIS 6.x 中，`MPSendNetBufferLists` 接收的是一个 `NET_BUFFER_LIST` 链表。小端口驱动必须：

1. 遍历链表中的每个 NET_BUFFER_LIST
2. 为每个 NET_BUFFER_LIST 分配一个 TCB
3. 从 NET_BUFFER_LIST 中提取数据（通过 NET_BUFFER 的 MDL 链）
4. 将数据封装为以太网帧格式，通过 ndisprot 设备发送

**🔍 第一性原理视角：批处理是 NDIS 6.x 的第一性原理**

微软文档强调：NDIS 6.0 及以后，**发送操作总是异步的**，"NDIS always completes a send operation asynchronously by calling the ProtocolSendNetBufferListsComplete function"。

遍历 NET_BUFFER_LIST 链表并逐个处理，正是 NDIS 6.x "批处理"哲学的体现——驱动一次性收到多个包，批量处理，降低函数调用开销。

### 12.4.4 写请求的构建与发送

**📌 书中的世界**

对于每个要发送的包：

1. 从 TCB 关联的 NET_BUFFER_LIST 中提取数据缓冲区
2. 构建 WDF I/O 请求（WriteFile 等效）
3. 通过 ndisprot I/O Target 发送
4. 设置完成回调
5. 当 ndisprot 设备完成写操作时，完成回调被调用
6. 在完成回调中：释放 TCB，调用 `NdisMSendNetBufferListsComplete` 通知 NDIS 发送完成

**🔍 第一性原理视角：发送完成的全异步链条**

```
tcpip.sys 下发 NBL
    ↓
MPSendNetBufferLists（ndisedge）
    ↓ 分配 TCB，构建写请求
    ↓
WDF I/O Target 发送到 ndisprot
    ↓ 通过 \Device\NdisProt 下发
ndisprot 协议驱动
    ↓ 通过 NdisSend 发送到真实网卡
真实网卡 Miniport
    ↓ 硬件发送
    ↓
发送完成中断
    ↓
真实网卡 Miniport 调用 NdisMSendComplete
    ↓
ndisprot 的 SendCompleteHandler
    ↓ 完成 WriteFile IRP
    ↓
WDF I/O Target 完成回调（ndisedge）
    ↓ 释放 TCB
    ↓
NdisMSendNetBufferListsComplete（ndisedge）
    ↓
tcpip.sys 的 ProtocolSendNetBufferListsComplete
```

> ⚠ **工业级陷阱**：
> 
> - TCB 的生命周期必须跨越整个异步发送链
> - 在 WDF I/O Target 的完成回调中才能释放 TCB 和完成 NBL
> - 必须处理发送失败的情况（如 ndisprot 设备不可用）

---

## 12.5 使用 ndisprot 接收包

### 12.5.1 提交数据包的内核 API

**📌 书中的世界**

小端口驱动通过 `NdisMIndicateReceiveNetBufferLists` 向 NDIS（及上层协议驱动）提交接收到的数据包。这是 NDIS 6.x 的标准 API。

**🔍 第一性原理视角：接收指示是"小端口 → NDIS → 协议驱动"的方向**

与发送相反，接收的数据流是从"网卡"（这里是 ndisprot 设备）上行到 tcpip.sys：

```
ndisprot 设备收到真实网卡的数据
    ↓
ndisedge 的读完成回调被触发
    ↓ 分配 NBL，填充数据
    ↓
NdisMIndicateReceiveNetBufferLists
    ↓
NDIS 调用 tcpip.sys 的协议回调
    ↓
tcpip.sys 处理 IP/TCP/UDP 协议
    ↓
数据最终送达用户态 socket
```

### 12.5.2 从接收控制块（RCB）提交包

**📌 书中的世界**

RCB（Receive Control Block）是小端口为每个接收操作维护的上下文结构，类似于 TCB 但用于接收方向。

**🔍 第一性原理视角：RCB 是"接收操作的身份证"**

当 ndisprot 设备有数据到达时：

1. 小端口的读完成回调被调用
2. 从 RCB 中获取数据缓冲区
3. 将数据封装为 NET_BUFFER_LIST
4. 调用 `NdisMIndicateReceiveNetBufferLists` 提交给 NDIS

### 12.5.3 对 ndisprot 读请求的完成函数

**📌 书中的世界**

读请求的完成函数是接收路径的核心：

1. 当 ndisprot 设备的 ReadFile IRP 完成时，此函数被调用
2. 从 IRP 中获取接收到的数据
3. 分配 NET_BUFFER_LIST 和数据缓冲区
4. 复制数据到 NET_BUFFER
5. 调用 `NdisMIndicateReceiveNetBufferLists` 提交

**🔍 第一性原理视角：这是一个"生产者-消费者"模型的变体**

```
真实网卡收到数据
    ↓
ndisprot 的 ReceiveHandler（DISPATCH_LEVEL）
    ↓ 数据入队
    ↓ 完成 ReadFile IRP
    ↓
ndisedge 读完成回调（PASSIVE_LEVEL）
    ↓ 分配 NBL
    ↓
NdisMIndicateReceiveNetBufferLists
    ↓
tcpip.sys 处理
```

> 💡 **洞见**：这与第11章协议驱动的"ReceiveHandler 入队 + ReadFile 出队"模式完全对称——只不过方向相反。第11章是协议驱动从网卡读数据交给用户态；本章是小端口驱动从 ndisprot 读数据交给 tcpip.sys。

### 12.5.4 读请求的发送

**📌 书中的世界**

小端口驱动需要持续地从 ndisprot 设备读取数据。方法：

1. 在适配器初始化完成后，发起第一个异步读请求
2. 在读完成回调中，提交数据包给 NDIS
3. 立即发起下一个异步读请求
4. 形成"持续读取"的循环

**🔍 第一性原理视角：这是一种"主动轮询式异步读取"**

与真实网卡通过中断触发接收不同，虚拟小端口通过 ndisprot 设备的 ReadFile 异步读取来模拟接收路径。这是一种"拉"模型而非"推"模型。

### 12.5.5 用于读包的 WDF 工作任务

**📌 书中的世界**

书中使用 WDF 的 **Work Item（工作任务）**​ 来处理接收数据。这是因为：

- 读完成回调可能在 `DISPATCH_LEVEL` 执行
- 但 `NdisMIndicateReceiveNetBufferLists` 某些操作可能需要 `PASSIVE_LEVEL`
- Work Item 可以把工作推迟到系统线程上下文（PASSIVE_LEVEL）执行

**🔍 第一性原理视角：Work Item 是"高 IRQL 到低 IRQL 的桥接工具"**

Windows 内核中常用的 IRQL 桥接机制：

- **DPC（Deferred Procedure Call）**：DISPATCH_LEVEL 的延迟执行
- **Work Item**：PASSIVE_LEVEL 的延迟执行（通过系统线程）
- **IoQueueWorkItem**：将工作项入队到系统工作线程

> 💡 **洞见**：本章使用 WDF Work Item 来处理接收数据，这体现了 WDF 框架的优势——**开发者不需要手动管理 IRQL 转换的复杂性**，WDF 提供了清晰的抽象。

### 12.5.6 ndisedge 读工作任务的生成与入列

**📌 书中的世界**

完整的接收路径：

1. 读完成回调（可能在 DISPATCH_LEVEL）被调用
2. 将数据保存到 RCB
3. 入列一个 WDF Work Item
4. 系统工作线程执行 Work Item 回调（PASSIVE_LEVEL）
5. 在 Work Item 回调中：分配 NBL，调用 `NdisMIndicateReceiveNetBufferLists`
6. 发起下一个异步读请求

**🔍 第一性原理视角：这是一个完整的"中断到用户"的接收流水线**

```
硬件中断（真实网卡）
    ↓
ndisprot 的 ReceiveHandler（DISPATCH_LEVEL）
    ↓ 数据入队到 ndisprot 内部队列
    ↓ 完成 ReadFile IRP
    ↓
ndisedge 读完成回调（DISPATCH_LEVEL）
    ↓ 入列 Work Item
    ↓
系统工作线程执行 Work Item（PASSIVE_LEVEL）
    ↓ 分配 NBL，填充数据
    ↓
NdisMIndicateReceiveNetBufferLists
    ↓
NDIS 调用 tcpip.sys 的接收回调
    ↓
tcpip.sys 处理 IP/TCP 协议栈
    ↓
数据送达用户态 socket
```

> 💡 **洞见**：这个流水线的每一段都在合适的 IRQL 执行：
> 
> - DISPATCH_LEVEL：硬件中断处理、数据初步入队
> - PASSIVE_LEVEL：协议栈处理、数据交付
> 
> **IRQL 的正确管理是 Windows 内核驱动稳定性的命脉**。WDF 通过 Work Item 等机制大大简化了这一管理。

---

## 12.6 其他的特征回调函数的实现

### 12.6.1 包的归还

**📌 书中的世界**

当 tcpip.sys 处理完小端口指示的数据包后，会调用小端口的 `MPReturnNetBufferLists`（NDIS 6.x）回调，将 NBL 归还给小端口。

**🔍 第一性原理视角：包的归还是"零拷贝"模型的核心**

NDIS 6.x 使用"零拷贝"接收模型：

- 小端口分配 NBL 并指示给 NDIS
- 上层协议驱动直接在原 NBL 上操作（不拷贝）
- 处理完成后通过 `MPReturnNetBufferLists` 归还 NBL
- 小端口回收 NBL 到池中

> 💡 **洞见**：这与第11章 `ReceivePacketHandler` 中的 `NdisReturnPackets` 机制完全对应——**NBL/NB 的所有权在"指示 → 归还"模型中传递**。理解了这个模型，你就能理解为什么现代网络驱动能够实现高吞吐量：避免了不必要的数据拷贝。

### 12.6.2 OID 查询处理的直接完成

**📌 书中的世界**

OID（Object Identifier）请求是小端口与上层驱动之间的"控制平面"。上层驱动（如 tcpip.sys）通过 OID 查询/设置：

- 网卡统计信息（OID_GEN_STATISTICS）
- MAC 地址（OID_802_3_CURRENT_ADDRESS）
- 链路速度（OID_GEN_LINK_SPEED）
- 等数百种标准化对象

小端口必须实现 `MPOidRequest`（NDIS 6.x）回调来处理这些请求。

**🔍 第一性原理视角：OID 是"驱动间的结构化对话协议"**

正如第11章所述，OID 是 NDIS 中驱动间配置协商与状态查询的核心信令通道。对于虚拟小端口 `ndisedge`：

- 某些 OID 可以直接完成（如返回固定的 MAC 地址、链路速度）
- 某些 OID 需要转发给 ndisprot 设备查询真实网卡的状态
- 某些 OID 可能需要特殊处理

**🏭 现代工业实践**：

对于虚拟小端口，常见的 OID 处理：

```
NDIS_STATUS MPOidRequest(PNDIS_OID_REQUEST OidRequest) {
    switch (OidRequest->Oid) {
        case OID_802_3_CURRENT_ADDRESS:
            // 返回虚拟 MAC 地址
            return NDIS_STATUS_SUCCESS;
        case OID_GEN_LINK_SPEED:
            // 返回 100Mbps 或转发查询真实网卡
            return NDIS_STATUS_SUCCESS;
        case OID_GEN_STATISTICS:
            // 返回统计信息
            return NDIS_STATUS_SUCCESS;
        // ... 其他 OID
    }
}
```

微软文档明确指出：**NDIS 保证在调用 `MiniportHaltEx` 时没有未完成的 OID 请求**。这使得 OID 处理的资源管理变得相对简单——你不需要在 Halt 时担心悬空的 OID 请求。

### 12.6.3 OID 设置处理

**📌 书中的世界**

OID 设置请求允许上层驱动配置小端口：

- 设置数据包过滤器（OID_GEN_CURRENT_PACKET_FILTER）
- 设置多播地址列表（OID_802_3_MULTICAST_LIST）
- 等

**🔍 第一性原理视角：OID 设置是"控制平面"的写操作**

对于虚拟小端口 `ndisedge`：

- `OID_GEN_CURRENT_PACKET_FILTER`：决定哪些类型的包需要提交给 tcpip.sys（如只提交单播、广播、多播）
- 这些设置可能转发给 ndisprot 设备，以配置真实网卡的包过滤

> 💡 **洞见**：OID 的查询和设置是 NDIS 驱动"控制平面"的完整表达。理解 OID 机制，你就理解了为什么 Windows 网络栈可以在不解耦的情况下支持数百种网卡特性——所有特性都通过标准化的 OID 接口暴露。

---

## 🎯 本章核心洞见总结

读完第12章，我希望你带走的不只是"怎么写一个 NDIS 小端口驱动"，而是这六个**第一性原理**：

**1. 小端口驱动的本质是"硬件与网络栈之间的契约"**

小端口驱动的两大职责：

- 管理网卡（物理或虚拟），收发数据
- 与上层驱动（过滤驱动、中间驱动、协议驱动）交互

它向 NDIS 提供统一的 `MiniportXxx` 入口，让上层协议驱动（如 tcpip.sys）无需关心下层是真实硬件还是虚拟设备。

**2. 适配器状态机是 NDIS 6.x 运行时重配置的基础**

NDIS 6.x 引入的适配器状态机：

```
Halted → Initializing → Paused ⇄ Running
```

- `MiniportInitializeEx` 成功后进入 **Paused**（不是直接 Running）
- NDIS 调用 `MiniportRestart` 才进入 **Running**
- 可以随时 Pause/Restart 而不卸载驱动

这个状态机是现代网卡热插拔、电源管理、RSS 重配置的基础架构。

**3. 资源管理的逆序铁律**

`MiniportHaltEx` 必须**逆序**释放 `MiniportInitializeEx` 中分配的资源：

- 先 `NdisMSetMiniportAttributes`，后分配 DMA 等资源
- Halt 时按相反顺序调用 `NdisXxx` 的反函数
- NDIS 保证 Halt 时没有未完成的 OID 或发送请求

这是 Windows 内核资源生命周期管理的统一范式，与 Minifilter 实例拆卸、TDI 绑定解绑、协议驱动绑定解除遵循相同的"停止-完成-释放-移除"四步法。

**4. TCB/RCB 是异步收发的"身份证"**

发送路径：分配 TCB → 关联 NBL → 异步发送到 ndisprot → 完成回调中释放 TCB 并通知 NDIS

接收路径：ndisprot 读完成 → 入列 Work Item → 分配 NBL → `NdisMIndicateReceiveNetBufferLists` → 上层归还 NBL

**"用上下文结构桥接异步操作的发起与完成"**是 Windows 内核异步模型的通用范式。

**5. WDF 与 NDIS 的融合是未来的方向**

书中示例用 WDF 构建小端口驱动（I/O Target、Work Item），这正预示了微软后来的 NetAdapterCx 方向：

- **传统 NDIS 小端口**：驱动自己处理 PnP、电源、数据路径同步
- **NetAdapterCx 小端口**：驱动是 WDF 客户端，PnP/电源/队列同步由 NetAdapter 类扩展接管

> 💡 **洞见**：微软的方向非常明确——**"NDIS 负责数据和控制路径，WDF 负责 PnP/电源/硬件交互"**。NetAdapterCx 仍然是"NDIS 的小端口"，但它把那些容易出 bug 的硬骨头（同步、PnP、电源）交给了 WDF 框架。

**6. 从经典 NDIS 小端口到 NetAdapterCx 的演进路径**

|维度|经典 NDIS 小端口（书中模型）|NetAdapterCx（现代模型）|
|---|---|---|
|PnP/电源管理|驱动自己处理|NetAdapter 类扩展接管|
|数据路径同步|驱动自己序列化|NetAdapter 类扩展序列化队列|
|适配器状态机|`MiniportInitializeEx`/`Halt`/`Pause`/`Restart`|NetAdapter 的 `Evt` 回调|
|数据包表示|`NET_BUFFER_LIST`|`NET_BUFFER_LIST`（相同）|
|适用范围|仍完全支持|新项目的推荐方向|
|特权级别|客户端驱动运行在高特权|状态分离 + DCHU 合规|

> ⚠ **工业现实**：
> 
> - **物理网卡驱动**：新项目应使用 NetAdapterCx 模型
> - **虚拟网卡/VPN**：仍可使用 NDIS 6.x 小端口，或使用 WFP 流向重定向
> - **教学/原理理解**：经典 NDIS 小端口模型是最好的教材
> - 微软明确表示："we see NDIS 6 and NetAdapter co-existing for the foreseeable future"——两者长期共存

### 从书中环境到现代工业环境的迁移清单

|书中技术（经典 NDIS 小端口）|现代工业做法|优势|
|---|---|---|
|`NdisMRegisterMiniport`|`NdisMRegisterMiniportDriver`（NDIS 6.x）或 WDF + NetAdapterCx|框架化管理|
|`MiniportInitialize`|`MiniportInitializeEx`|适配器状态机|
|`MiniportHalt`|`MiniportHaltEx`|运行时重配置|
|`MiniportSendPackets`（`NDIS_PACKET`）|`MiniportSendNetBufferLists`（`NET_BUFFER_LIST`）|批处理 + 零拷贝|
|`MiniportReturnPacket`|`MiniportReturnNetBufferLists`|批量归还|
|`MiniportQueryInformation`|`MiniportOidRequest`|统一的 OID 处理|
|手动管理 PnP/电源|NetAdapterCx 自动管理|减少 bug check 风险|
|手动数据路径同步|NetAdapterCx 序列化队列|简化开发|

### 工业应用场景导航

|场景|推荐技术|说明|
|---|---|---|
|物理网卡驱动|**NetAdapterCx**（新）/ NDIS 6.x Miniport（旧）|微软推荐新项目用 NetAdapterCx|
|VPN 虚拟网卡|NDIS 6.x 小端口 或 WFP 流向重定向|虚拟小端口模式（本书示例）|
|Hyper-V vSwitch 虚拟 NIC|NDIS 6.x 小端口|虚拟化场景|
|移动宽带调制解调器（MBB）|NetAdapterCx（RS5+）|微软明确方向|
|网络过滤/防火墙|**WFP**（不是小端口！）|WFP 位于 NDIS 之上，感知应用进程|
|链路层协议实现|NDIS 6.x 协议驱动|如第11章所述|
|流量修改/监控|NDIS 6.x Lightweight Filter|不需要应用层上下文时使用|

> 📌 **最后一句话**：第12章"NDIS 小端口驱动"是全书网络部分的**基石章节**——它让你看见 Windows 网络栈的最底层如何与"网卡"（无论是物理硬件还是虚拟设备）对话。本书通过 `ndisedge` 这个虚拟小端口示例，巧妙地让你在一个示例中看到 NDIS 栈的两端：小端口的上边界（与 tcpip.sys 交互）和下边界（与 ndisprot 设备交互）。这种"全栈可视化"的设计是理解 NDIS 架构的最佳教材。

**但从工业现实出发**，你必须知道：

- **经典 NDIS 小端口模型并未过时**，它仍然是 Windows 网络驱动的核心
- **新项目的方向是 NetAdapterCx**——WDF 客户端 + NetAdapter 类扩展，由框架承担 PnP/电源/同步的复杂性
- **网络过滤不要用小端口**——WFP 才是正确的选择
- **虚拟网卡/VPN**​ 仍是小端口驱动的重要应用场景

**第一性原理式的学习收获**：从第10章 TDI → 第11章 NDIS 协议驱动 → 第12章 NDIS 小端口驱动，你完整地穿越了 Windows 网络栈的传输层、链路层、硬件层。**这三章共同揭示了一个统一的工程哲学**：微软不断把"容易出 bug 的复杂性"框架化——TDI 过滤被 WFP 取代，NDIS IM 过滤被 WFP 取代，经典小端口的 PnP/电源/同步被 NetAdapterCx 接管。**理解这个演进方向，你就能在 Windows 内核网络驱动的开发中做出正确的技术选型**。

