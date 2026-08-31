第11章"NDIS协议驱动"把视角从上一章的传输层（TDI）再往下推一层，直达**链路层**——你将看到一个协议驱动如何绑定网卡、收发原始以太网帧、并与用户态程序协同工作。这一章在全书网络部分中的地位非常特殊：第10章 TDI 是在"传输层语义"上做过滤（看得见 socket 连接），而第11章 NDIS 协议驱动是在"链路层"自己实现协议（看得见原始以太网帧）。

但要先讲清楚一个**工业现实**：这一章的示例代码基于经典 NDIS 模型（NDIS 5.x 风格），其中 `ReceiveHandler` / `TransferDataCompleteHandler` / `ReceivePacketHandler` 这套回调、`NdisRegisterProtocol` / `NdisOpenAdapter` / `NdisRegisterProtocol` 这套 API，在现代 Windows（Win10/Win11）上已经演进到 NDIS 6.x 的 `NET_BUFFER_LIST` 模型。而且，**如果你的目标是"网络过滤/防火墙/EDR 网络遥测"，现代工业界的标准答案是 WFP（Windows Filtering Platform）**——WFP 从 Vista 引入，统一替代了老的 TDI 过滤和 NDIS 过滤技术。

但**这不代表第11章过时无用**。相反，要真正理解 Windows 网络栈，你必须理解 NDIS 协议驱动——它是 Windows 网络栈的"链路层契约"，TCP/IP 协议栈（tcpip.sys）本身就是作为一个 NDIS 协议驱动挂到网卡上的。VPN 虚拟网卡、自定义工业协议、原始以太网帧捕获，今天仍然用 NDIS 协议驱动或 NDIS 6.x 的等价模型实现。

下面按你给的目录结构，逐节做读书笔记。

---

## 11.1 以太网包和网络驱动架构

### 11.1.1 以太网包和协议驱动

**📌 书中的世界**

书中从以太网帧结构讲起：目的 MAC + 源 MAC + EtherType + Payload + FCS。协议驱动的本质，就是**绑定到网卡上、直接收发这种原始以太网帧的内核模块**。

它与 tcpip.sys 是"平级"的关系——tcpip.sys 是微软实现的"TCP/IP 协议驱动"，你的协议驱动是"自定义协议驱动"，两者都通过 NDIS 绑定到同一个网卡 Miniport 上。

**🔍 第一性原理视角：协议驱动 = "链路层之上的协议实体"**

Windows 网络栈的分层（来自现代工业剖析）：

```
应用层 (Winsock API)
    ↓
afd.sys (内核 Socket 抽象层)
    ↓
tcpip.sys (TCP/IP 协议栈——它本身就是一个 NDIS 协议驱动)
    ↓
NDIS (链路层抽象)
    ↓
Miniport 驱动 (Intel e1000 / Realtek RTL8139 / Hyper-V vSwitch)
    ↓
物理网卡
```

关键的洞见是：**tcpip.sys 在 NDIS 看来，和你写的协议驱动没有任何区别**——都是"向 NDIS 注册一组 ProtocolXxx 回调、绑定到 Miniport、收发以太网帧"的内核模块。

> 💡 **洞见**：当你写一个 NDIS 协议驱动时，你不是"过滤网络"，而是"在链路层之上实现自己的协议"。这与第10章 TDI 过滤的本质区别是：
> 
> - TDI 过滤：在现有 TCP/IP 协议栈旁边"偷看"连接请求
> - NDIS 协议驱动：与 TCP/IP 协议栈"并列"，自己处理原始帧

这就是为什么自定义 VPN 协议、工业私有协议、Raw Socket 类应用要用 NDIS 协议驱动——它们需要**绕过 TCP/IP 协议栈**，直接处理以太网帧。

### 11.1.2 NDIS 网络驱动

**📌 书中的世界**

书中梳理 NDIS 的三类驱动：

- **Miniport 驱动**（底层）：直接操作网卡，发送/接收以太网帧，处理中断和 DMA
- **协议驱动**（顶层）：实现具体协议逻辑，绑定 Miniport，收发帧
- **中间层驱动 IM**（中间）：对上下层都呈现双重身份，可拦截/修改/过滤流量

**🔍 第一性原理视角：NDIS 是"链路层的 HAL（硬件抽象层）"**

NDIS 的核心价值：**统一屏蔽异构网卡差异，向上提供标准化的 MAC 层收发原语**。

Miniport 驱动不知道上层用什么协议（TCP/IP？你的自定义协议？），它只搬运以太网帧；协议驱动不知道下层网卡型号（Intel？Realtek？虚拟网卡？），它只通过 NDIS 提供的标准接口收发帧。

这种对称设计让 IM 驱动可以串联成链：防火墙 → QoS → 流量监控，每层都可以修改或拒绝流量，而不需要修改上层或下层的代码。Windows 自带的 IM 驱动包括 WFP 子系统（fwpkclnt.sys）、WAN 微型端口、桥接驱动。

> 💡 **洞见**：NDIS 的"三层模型"是软件工程"分层架构"的教科书案例——每一层只与相邻层通信，Winsock 不知道 TCP 三次握手，TCP/IP 不知道网卡型号，网卡不知道数据含义。**模块化使得 Windows 可以同时支持 IPv4/IPv6/Teredo 隧道，可以在不修改应用程序的情况下更换网卡驱动，可以在同一台机器上运行数十个网络协议而互不干扰**。

**🏭 现代工业现实**：

- 如果你要做**网络过滤/防火墙/EDR 网络遥测**：用 WFP，不用 NDIS IM 驱动。WFP 是 Vista 引入的统一过滤框架，已经替代了老的 NDIS 过滤技术。
- 如果你要做**网卡硬件驱动**：用 NDIS Miniport 驱动（现代是 NDIS 6.x 的 Miniport 模型）
- 如果你要做**自定义链路层协议/VPN/虚拟网卡**：用 NDIS 协议驱动或 NDIS 6.x 的等价模型

---

## 11.2 协议驱动的 DriverEntry

### 11.2.1 生成控制设备

**📌 书中的世界**

协议驱动的 `DriverEntry` 与常规 WDM 驱动类似：

1. 创建控制设备对象（CDO），用于与用户态通信
2. 设置分发函数
3. 调用 `NdisRegisterProtocol` 向 NDIS 注册协议

控制设备通常通过 `IoCreateDevice` 创建，用户态通过 `CreateFile("\\.\NdisProt")` 打开它，再通过 `DeviceIoControl` / `ReadFile` / `WriteFile` 与驱动交互。

**🔍 第一性原理视角：控制设备是"用户态与内核协议实体的桥梁"**

协议驱动通常有两种工作模式：

- **内核态自主处理**：协议驱动自己解析以太网帧、实现协议逻辑（如自定义 VPN 协议）
- **转发给用户态处理**：协议驱动只负责收发包，把帧通过控制设备交给用户态程序处理

第二种模式是第11.5/11.6节的核心——让用户态程序"像操作文件一样"读写原始以太网帧。这是 WinPcap/Npcap 的核心思想之一。

### 11.2.2 注册协议

**📌 书中的世界**

`NdisRegisterProtocol` 是协议驱动的"出生证明"：

```
NDIS_PROTOCOL_CHARACTERISTICS protChar;
NdisZeroMemory(&protChar, sizeof(protChar));
protChar.MajorNdisVersion = 5;
protChar.MinorNdisVersion = 0;
protChar.Name = protoName;
protChar.OpenAdapterCompleteHandler = NdisProtOpenAdapterComplete;
protChar.CloseAdapterCompleteHandler = NdisProtCloseAdapterComplete;
protChar.SendCompleteHandler = NdisProtSendComplete;
protChar.TransferDataCompleteHandler = NdisProtTransferDataComplete;
protChar.ResetCompleteHandler = NdisProtResetComplete;
protChar.RequestCompleteHandler = NdisProtRequestComplete;
protChar.ReceiveHandler = NdisProtReceive;
protChar.ReceiveCompleteHandler = NdisProtReceiveComplete;
protChar.StatusHandler = NdisProtStatus;
protChar.StatusCompleteHandler = NdisProtStatusComplete;
protChar.BindAdapterHandler = NdisProtBindAdapter;
protChar.UnbindAdapterHandler = NdisProtUnbindAdapter;
protChar.UnloadHandler = NdisProtUnload;
protChar.ReceivePacketHandler = NdisProtReceivePacket;
protChar.PnPEventHandler = NdisProtPnPEvent;

NdisRegisterProtocol(&gNdisProtocolHandle, &protChar);
```

**🔍 第一性原理视角：注册 = "向 NDIS 宣告我能处理什么事件"**

`NDIS_PROTOCOL_CHARACTERISTICS` 结构体是协议驱动与 NDIS 之间的"契约"——你告诉 NDIS："当网卡有数据到达时调用我的 `ReceiveHandler`，当发送完成时调用我的 `SendCompleteHandler`，当有网卡可用时调用我的 `BindAdapterHandler`……"

**🏭 现代工业演进**：

在 NDIS 6.x 中，`NdisRegisterProtocol` 被 `NdisRegisterProtocolDriver` 取代，回调结构体改为 `NDIS_PROTOCOL_DRIVER_CHARACTERISTICS`，接收路径从 `NDIS_PACKET` 改为 `NET_BUFFER_LIST`。但**核心思想完全一样**——"注册回调，让 NDIS 在合适时机调用"。

> ⚠ **重要区分**：
> 
> - 如果你开发**现代网络过滤**：用 WFP，不用 NDIS 协议驱动
> - 如果你开发**自定义协议/VPN/虚拟网卡**：仍用 NDIS 协议驱动模型（NDIS 6.x），这是唯一选择
> - 书中示例基于 NDIS 5.x 风格，理解原理后迁移到 NDIS 6.x 是直接的

---

## 11.3 协议与网卡的绑定

### 11.3.1 协议与网卡的绑定概念

**📌 书中的世界**

当系统中有网卡可用时，NDIS 调用协议驱动的 `BindAdapterHandler`，参数中包含网卡设备名。协议驱动在 `BindAdapterHandler` 中调用 `NdisOpenAdapter` 完成绑定。

**🔍 第一性原理视角：绑定 = "协议实体与网卡的逻辑连接"**

一台机器可能有多个网卡（有线、无线、虚拟网卡、VPN 网卡），一个协议驱动可以绑定到多个网卡上。**绑定是动态的**——网卡热插拔、启用/禁用，都会触发 NDIS 调用 `BindAdapterHandler` / `UnbindAdapterHandler`。

> 💡 **洞见**：`BindAdapterHandler` / `UnbindAdapterHandler` 是支持即插即用的关键。正如 DDK 文档所述："支持即插即用功能的协议驱动程序通过提供 ProtocolBindAdapter 函数和 ProtocolUnbindAdapter 函数就可以支持对低层 NIC 的动态绑定……只要低层 NIC 可用，NDIS 将调用能够将自己绑定到该适配器的任何协议驱动的 ProtocolBindAdapter 函数"。

### 11.3.2 绑定回调处理的实现

**📌 书中的世界**

`NdisProtBindAdapter` 的核心流程：

1. 为这个绑定分配上下文结构（BindingContext）
2. 调用 `NdisOpenAdapter` 打开网卡
3. 如果返回 `NDIS_STATUS_PENDING`，等待 `OpenAdapterCompleteHandler` 回调
4. 在 `OpenAdapterCompleteHandler` 中完成绑定后的初始化（分配包池、缓冲池、OID 查询）

**🔍 第一性原理视角：异步绑定是 NDIS 的"第一性原则"**

`NdisOpenAdapter` 可能同步完成，也可能返回 `NDIS_STATUS_PENDING` 异步完成——这是 NDIS 的"异步第一性"：任何可能耗时的操作都不阻塞，而是通过"调用 + 完成回调"的模式。

这与第10章 TDI 过滤的 IRP 异步模型如出一辙——**Windows 内核的哲学就是"异步优先"**。理解这一点，你就能理解为什么 NDIS 协议驱动有如此多的"CompleteHandler"回调：

- `OpenAdapterCompleteHandler`
- `CloseAdapterCompleteHandler`
- `SendCompleteHandler`
- `TransferDataCompleteHandler`
- `RequestCompleteHandler`

每个"发起操作"都对应一个"完成回调"。**这是 Windows 内核异步模型的统一范式**。

### 11.3.3 协议绑定网卡的 API

**📌 书中的世界**

书中详细介绍了 `NdisOpenAdapter` 的参数：

- `BindingContext`：绑定的上下文句柄
- `MacAddr`：本地 MAC 地址
- `DeviceName`：网卡设备名（从 `BindAdapterHandler` 参数获得）
- `PhysicalDeviceObject`：NULL
- `OpenOptions`：打开选项
- `MediumArray` / `MediumArraySize`：期望的介质类型数组（如 NdisMedium802_3 表示以太网）

**🏭 工业实践**：

`MediumArray` 告诉 NDIS"我期望的链路层介质类型"。对于以太网，`NdisMedium802_3` 是标准值。如果你的协议驱动要支持 Wi-Fi，需要 `NdisMedium802_11`；支持令牌环，需要 `NdisMediumTokenRing`……

现代 NDIS 6.x 中，介质类型通过 `NET_BUFFER_LIST` 的 `NetBufferListHeader` 中的 `MediaType` 字段传递，但语义一致。

### 11.3.4 解决绑定竞争问题

**📌 书中的世界**

绑定过程中存在竞争：NDIS 可能同时调用 `BindAdapterHandler` 绑定多个网卡，而你的驱动可能还没完成上一个绑定的初始化。书中给出解决方案：**用一个全局锁保护绑定列表，在 `BindAdapterHandler` 中先创建上下文结构并插入列表，再调用 `NdisOpenAdapter`**。

**🔍 第一性原理视角：多核并发下的状态一致性**

这是 Windows 内核驱动开发的通用难题——**多核并行 + 异步完成 = 竞态条件**。绑定列表必须用自旋锁（Spinlock）保护，因为 `BindAdapterHandler` 可能在 `DISPATCH_LEVEL` 调用。

> ⚠ **工业级陷阱**：
> 
> - 绑定上下文结构的生命周期必须跨越 `NdisOpenAdapter` 的异步完成
> - 如果 `NdisOpenAdapter` 返回 PENDING，上下文结构不能被释放
> - 必须在 `OpenAdapterCompleteHandler` 中检查绑定状态，处理"绑定已被取消"的情况
> - 解绑时也要处理"正在绑定中"的竞态

### 11.3.5 分配接收和发送的包池与缓冲池

**📌 书中的世界**

协议中必须分配两类资源：

- **包池（Packet Pool）**：通过 `NdisAllocatePacketPool` 分配 `NDIS_PACKET` 对象的池
- **缓冲池（Buffer Pool）**：通过 `NdisAllocateBufferPool` 分配 `NDIS_BUFFER` 对象的池

接收时，协议驱动从包池取一个 Packet，从缓冲池取一个 Buffer，把网卡收到的数据填入 Buffer，再把 Buffer 链入 Packet，最后交给上层处理。

**🔍 第一性原理视角：池化分配是"内核性能的第一性原理"**

内核中频繁分配/释放内存会导致：

- 内存碎片
- 分配延迟不确定
- 高 IRQL 下无法使用分页内存

**池化（Pool）分配解决了这些问题**：预先分配一批固定大小的对象，使用时从池中取，用完归还池中。这是 Windows 内核中**​ everywhere 的模式**——Lookaside List、NPaged Pool、Packet Pool 都是这个思想。

> 💡 **洞见**：理解包池/缓冲池，你就理解了 Windows 内核内存管理的核心思想——**"预分配 + 复用"优于"按需分配"**。这在高性能网络驱动中至关重要：每秒几十万包的收发，如果每次都 `ExAllocatePool`，系统会瞬间崩溃。

**🏭 现代 NDIS 6.x 演进**：

NDIS 6.x 用 `NET_BUFFER` / `NET_BUFFER_LIST` 取代了 `NDIS_PACKET` / `NDIS_BUFFER`，分配函数变为：

- `NdisAllocateNetBufferListPool` 代替 `NdisAllocatePacketPool`
- `NdisAllocateNetBuffer` 代替 `NdisAllocatePacket`

但**池化思想完全一样**。

### 11.3.6 OID 请求的发送和请求完成回调

**📌 书中的世界**

OID（Object Identifier）是 NDIS 中用于驱动间**配置协商与状态查询**的核心信令通道。协议驱动通过 `NdisRequest` 向 Miniport 发起 OID 查询，涵盖：

- MIB-II 统计信息获取
- 电源管理控制
- QoS 参数设置
- 802.1X 认证触发
- MAC 地址查询（OID_802_3_CURRENT_ADDRESS）
- 链路速度查询（OID_GEN_LINK_SPEED）
- 等数百种标准化对象标识符

`NdisRequest` 异步完成，通过 `RequestCompleteHandler` 回调通知结果。

**🔍 第一性原理视角：OID 是"驱动间的结构化对话协议"**

OID 的设计哲学：**用"对象标识符 + 标准结构体"的方式，让协议驱动与 Miniport 驱动解耦**。协议驱动不需要知道网卡型号，只需要查询 `OID_802_3_CURRENT_ADDRESS` 就能拿到 MAC 地址。

正如 CSDN 文库的技术文章所述："OID 机制是 NDIS 中用于驱动间配置协商与状态查询的核心信令通道，涵盖 MIB-II 统计信息获取、电源管理控制、QoS 参数设置、802.1X 认证触发等数百种标准化对象标识符，协议驱动通过 NdisRequest 向 Miniport 发起同步/异步 OID 查询"。

**🏭 现代工业演进**：

NDIS 6.x 中 OID 请求通过 `NdisOidRequest` + `NDIS_OID_REQUEST` 结构发起，但语义一致。在 NDIS Filter 驱动中，OID 请求的处理更加灵活——"如果不应将请求转发到基础驱动程序，Filter 驱动程序可以立即完成请求"，也可以"通过调用 NdisAllocateCloneOidRequest 将请求信息传输到新结构"后再转发。

> 💡 **洞见**：OID 是 NDIS 的"ioctl 等价物"——它是驱动间标准化的"控制平面"。理解 OID，你就能理解为什么 Windows 网络栈可以在不解耦的情况下支持数百种网卡特性。

### 11.3.7 ndisprotCreateBinding 的最终实现

**📌 书中的世界**

书中给出了 `ndisprotCreateBinding` 的完整实现，整合了前面几节的所有要素：

1. 分配绑定上下文
2. 调用 `NdisOpenAdapter` 绑定网卡
3. 在 `OpenAdapterCompleteHandler` 中完成后续初始化：
    - 分配包池和缓冲池
    - 查询 OID 获取 MAC 地址和链路速度
    - 将绑定标记为"已就绪"状态
4. 处理异步竞态

**🔍 第一性原理视角：这是一个完整的"异步状态机"**

`ndisprotCreateBinding` 实际上实现了一个小型状态机：

```
状态1: 初始化中（分配上下文，调用 NdisOpenAdapter）
    ↓ 同步完成 → 直接进入状态3
    ↓ 异步完成 → 进入状态2
状态2: 等待 OpenAdapterComplete
    ↓ 收到回调 → 进入状态3
状态3: 绑定就绪（分配资源，OID 查询，标记 Ready）
    ↓
绑定可用，开始收发包
```

> ⚠ **工业级陷阱**：状态机的每个状态转换都必须考虑"解绑请求中途到达"的情况。如果用户在 `NdisOpenAdapter` 返回 PENDING 期间禁用了网卡，NDIS 会调用 `UnbindAdapterHandler`——此时绑定还在"初始化中"状态，必须正确处理这种竞态。

---

## 11.4 绑定的解除

### 11.4.1 解除绑定使用的 API

**📌 书中的世界**

`UnbindAdapterHandler` 是 `BindAdapterHandler` 的逆操作：

1. 将绑定标记为"正在解绑"
2. 调用 `NdisCloseAdapter` 关闭与网卡的绑定
3. 如果返回 PENDING，等待 `CloseAdapterCompleteHandler`
4. 在 `CloseAdapterCompleteHandler` 中释放所有资源（包池、缓冲池、绑定上下文）

**🔍 第一性原理视角：解绑比绑定更难**

绑定是"从无到有"构建状态，解绑是"从有到无"销毁状态。**销毁状态时必须保证**：

- 没有正在进行的发送/接收操作
- 没有未完成的 OID 请求
- 所有pending的包都已返回
- 用户态没有打开的句柄（否则拒绝解绑或强制关闭）

这正是 Windows 内核资源管理的"黄金法则"——**资源释放顺序必须与分配顺序严格相反，且必须处理异步竞态**。

### 11.4.2 ndisprotShutdownBinding 的实现

**📌 书中的世界**

书中给出 `ndisprotShutdownBinding` 的完整实现，它：

1. 取消所有 pending 的 OID 请求
2. 等待所有发送完成
3. 调用 `NdisCloseAdapter` 关闭绑定
4. 在 `CloseAdapterCompleteHandler` 中释放包池、缓冲池、绑定上下文
5. 从全局绑定列表中移除

**🏭 工业实践**：

这是内核驱动"优雅停机"的标准范式。对比第9章 Minifilter 的 `InstanceTeardownStartCallback` / `InstanceTeardownCompleteCallback`——**两者的设计哲学完全一致**：

- 先标记"不再接受新请求"
- 完成所有进行中的操作
- 释放资源
- 从全局结构中移除

> 💡 **洞见**：Windows 内核中"优雅停机"的模式是统一的——无论是 Minifilter 实例拆卸、TDI 地址对象清理、还是 NDIS 绑定解绑，都遵循相同的"停止-完成-释放-移除"四步法。理解了这个模式，你就能处理任何内核资源的生命周期管理。

---

## 11.5 在用户态操作协议驱动

### 11.5.1 协议的收包与发包

**📌 书中的世界**

书中设计了一个用户态与协议驱动交互的模型：

- 用户态通过 `CreateFile` 打开控制设备
- 通过 `WriteFile` 发送数据包（驱动将用户数据封装为以太网帧发出）
- 通过 `ReadFile` 读取数据包（驱动将收到的以太网帧交给用户态）
- 通过 `DeviceIoControl` 发送控制请求（如查询绑定状态、设置过滤条件）

**🔍 第一性原理视角：把"网卡"抽象成"文件"**

这是 Unix "一切皆文件"思想在 Windows 上的呼应——用户态程序通过文件 I/O API 操作原始以太网帧，就像操作一个字符设备文件。

WinPcap/Npcap 的核心思想正是如此——把网卡抽象成一个文件句柄，用户态程序通过 `ReadFile` 读取数据包，通过 `WriteFile` 发送数据包。

### 11.5.2 在用户态编程打开设备

**📌 书中的世界**

用户态程序通过 `CreateFile("\\\\.\\NdisProt", ...)` 打开控制设备。驱动在 `IRP_MJ_CREATE` 分发函数中：

- 验证调用者权限
- 分配文件对象上下文
- 记录用户态进程信息

**🔍 第一性原理视角：控制设备是"内核协议实体的用户态门户"**

每个 `CreateFile` 打开的句柄都对应一个"用户态会话"。驱动需要跟踪：

- 哪个进程打开了句柄
- 绑定到哪个网卡
- 设置了什么过滤条件
- 是否有 pending 的读请求

### 11.5.3 用 DeviceIoControl 发送控制请求

**📌 书中的世界**

`DeviceIoControl` 用于发送控制命令，如：

- 枚举所有绑定（网卡列表）
- 绑定到指定网卡
- 设置 Ethernet 类型过滤器（只接收特定 EtherType 的帧）
- 查询统计信息

**🏭 工业实践**：

这是 WFP 之前时代"用户态与网络驱动通信"的标准方式。现代 WFP 仍然保留了类似的模式——用户态通过 `Fwpm*` API 与 WFP 内核引擎通信，设置过滤规则。

### 11.5.4 用 WriteFile 发送数据包

**📌 书中的世界**

用户态通过 `WriteFile` 发送数据包的流程：

1. 用户态填充以太网帧数据（目的 MAC + 源 MAC + EtherType + Payload）
2. 调用 `WriteFile` 将数据发送到驱动
3. 驱动在 `IRP_MJ_WRITE` 分发函数中：
    - 从包池分配 `NDIS_PACKET`
    - 从缓冲池分配 `NDIS_BUFFER`
    - 将用户态数据复制到内核缓冲区
    - 调用 `NdisSend` 发送
4. 发送完成后，`SendCompleteHandler` 被调用，完成 IRP

**🔍 第一性原理视角：WriteFile → NDIS_PACKET 的转换**

这是"用户态语义"到"内核 NDIS 语义"的翻译。关键挑战：

- 用户态缓冲区可能被换页，必须在 `PASSIVE_LEVEL` 复制
- `NdisSend` 是异步的，必须跟踪 IRP 与 Packet 的对应关系
- 发送完成后才能完成 IRP

> 💡 **洞见**：这个"WriteFile → NdisSend"的转换，本质上是"同步文件 I/O 语义"到"异步 NDIS 发送"的桥接。驱动的 `IRP_MJ_WRITE` 处理必须：
> 
> 1. 保存 IRP 指针到 Packet 的上下文
> 2. 调用 `NdisSend`
> 3. 如果同步完成，立即完成 IRP
> 4. 如果异步完成（返回 PENDING），在 `SendCompleteHandler` 中取出 IRP 并完成

### 11.5.5 用 ReadFile 发送数据包

**📌 书中的世界**

用户态通过 `ReadFile` 读取数据包的流程：

1. 用户态调用 `ReadFile` 等待数据包
2. 驱动在 `IRP_MJ_READ` 分发函数中：
    - 检查是否有已接收的数据包
    - 如果有，复制到用户缓冲区并完成 IRP
    - 如果没有，将 IRP 标记为 Pending，入队到"待处理读请求"队列
3. 当网卡收到数据，`ReceiveHandler` / `ReceivePacketHandler` 被调用：
    - 将收到的数据包入队到"接收包队列"
    - 检查是否有 Pending 的读 IRP
    - 如果有，出队一个读 IRP，复制数据并完成

**🔍 第一性原理视角：这是一个经典的"生产者-消费者"模型**

```
网卡中断 → ReceiveHandler（生产者）
                ↓ 数据包入队
        接收包队列
                ↓ 出队
        ReadFile IRP（消费者）
```

这个模型是内核中"中断上下文与用户态上下文桥接"的标准范式：

- 生产者运行在 `DISPATCH_LEVEL`（网卡中断上下文）
- 消费者运行在 `PASSIVE_LEVEL`（用户态线程上下文）
- 两者通过"队列 + 自旋锁"解耦

> ⚠ **工业级陷阱**：
> 
> - 队列操作必须在 `DISPATCH_LEVEL` 使用自旋锁
> - 数据包复制到用户缓冲区必须在 `PASSIVE_LEVEL`
> - 必须处理"用户态取消 ReadFile"的情况（IRP 被取消）
> - 必须处理"驱动卸载时还有 Pending 读 IRP"的情况

---

## 11.6 在内核态完成功能的实现

### 11.6.1 请求的分发与实现

**📌 书中的世界**

协议驱动的分发函数处理用户态通过 `WriteFile` / `ReadFile` / `DeviceIoControl` 发来的请求。核心分发函数：

- `IRP_MJ_CREATE`：打开设备
- `IRP_MJ_CLOSE`：关闭设备
- `IRP_MJ_WRITE`：发送数据包
- `IRP_MJ_READ`：读取数据包
- `IRP_MJ_DEVICE_CONTROL`：控制请求

**🔍 第一性原理视角：WDM 分发范式的内核协议驱动应用**

这些分发函数与第10章 TDI 过滤驱动的分发函数高度同构——都是 WDM 驱动的标准范式。区别在于：

- TDI 过滤驱动主要处理 `IRP_MJ_INTERNAL_DEVICE_CONTROL`（内核组件间通信）
- NDIS 协议驱动主要处理 `IRP_MJ_WRITE` / `IRP_MJ_READ`（用户态文件 I/O 语义）

### 11.6.2 等待设备绑定完成与指定设备名

**📌 书中的世界**

用户态程序在 `DeviceIoControl` 中指定要绑定的网卡名（如 `"\\Device\\{GUID}"`）。驱动在绑定前必须：

1. 等待该网卡可用（可能尚未绑定）
2. 在绑定列表中查找匹配的绑定
3. 如果找到，关联到当前文件句柄
4. 如果没找到，返回错误

**🔍 第一性原理视角：异步绑定与同步用户请求的桥接**

用户态的 `DeviceIoControl` 是同步调用，但网卡的绑定可能是异步的（PENDING）。驱动必须处理这种时间差：

- 如果绑定已完成，直接关联
- 如果绑定正在进行，等待完成（使用事件对象）
- 如果绑定失败，返回错误

### 11.6.3 指派设备的完成

**📌 书中的世界**

书中详细介绍了如何在 `DeviceIoControl` 处理中完成"设备指派"——将用户态句柄与一个具体的 NDIS 绑定关联起来。

**🏭 工业实践**：

这是 Npcap/WinPcap 的核心逻辑之一——用户态程序 `pcap_open` 时，内核驱动将文件句柄与指定网卡绑定关联。后续该句柄的所有 `ReadFile` / `WriteFile` 都针对这个绑定。

### 11.6.4 处理读请求

**📌 书中的世界**

`IRP_MJ_READ` 的处理逻辑：

1. 检查接收包队列是否有数据
2. 如果有，复制到用户缓冲区，完成 IRP
3. 如果没有，将 IRP 标记为 Pending，入队到"Pending 读请求"队列
4. 返回 `STATUS_PENDING`

**🔍 第一性原理视角：这是内核中"异步 I/O"的标准实现**

用户态 `ReadFile` 时，如果驱动没有数据，必须：

- 调用 `IoMarkIrpPending` 标记 IRP 为 Pending
- 将 IRP 入队
- 返回 `STATUS_PENDING`
- 当数据到达时，出队 IRP 并完成

> ⚠ **工业级陷阱**：
> 
> - IRP 取消处理：用户态可能随时取消 `ReadFile`，驱动必须能够取消 Pending 的 IRP
> - 超时处理：用户态可能设置超时，驱动需要处理
> - 缓冲区溢出：用户态缓冲区可能小于数据包大小，必须返回错误

### 11.6.5 处理写请求

**📌 书中的世界**

`IRP_MJ_WRITE` 的处理逻辑：

1. 从用户缓冲区复制数据到内核分配的缓冲区
2. 从包池分配 `NDIS_PACKET`，从缓冲池分配 `NDIS_BUFFER`
3. 将内核缓冲区链入 Packet
4. 调用 `NdisSend` 发送
5. 如果同步完成，完成 IRP
6. 如果异步完成（PENDING），在 `SendCompleteHandler` 中完成 IRP

**🔍 第一性原理视角：SendCompleteHandler 是"发送生命周期的终点"**

`NdisSend` 返回 PENDING 后，协议驱动必须等待 `SendCompleteHandler` 回调才能释放 Packet 和完成 IRP。这是 NDIS 异步模型的铁律——**Packet 在 SendComplete 之前不能被释放**。

> 💡 **洞见**：这里有一个关键设计——IRP 指针必须保存在 Packet 的上下文中（通过 `NdisAllocatePacket` 时分配的"Packet 上下文"区域）。当 `SendCompleteHandler` 被调用时，从 Packet 上下文取出 IRP 指针，复制发送状态，调用 `IoCompleteRequest` 完成 IRP，然后释放 Packet 到包池。

**🏭 现代 NDIS 6.x 演进**：

NDIS 6.x 中，`NdisSend` 被 `NdisSendNetBufferList` 取代，`NDIS_PACKET` 被 `NET_BUFFER_LIST` 取代。但**异步完成模型完全一样**——`SendCompleteHandler` 对应 `SendNetBufferListCompleteHandler`。

---

## 11.7 协议驱动的接收回调

### 11.7.1 和接收包有关的回调函数

**📌 书中的世界**

NDIS 协议驱动有三种接收回调：

1. **`ReceiveHandler`**：老式接收回调，以前视缓冲区（Lookahead Buffer）为参数。如果前视缓冲区不包含完整数据包，需调用 `NdisTransferData` 获取剩余部分，在 `TransferDataCompleteHandler` 中完成
2. **`ReceivePacketHandler`**：新式接收回调，以完整的 `NDIS_PACKET` 为参数，适用于不使用 NDIS 接收队列的驱动
3. **`ReceiveCompleteHandler`**：指示一批接收完成

**🔍 第一性原理视角：前视缓冲区是"性能与完整性的权衡"**

网卡收到帧时，NDIS 只把帧的前 N 字节（前视缓冲区）交给 `ReceiveHandler`，这是为了：

- **性能**：大多数协议只需要检查头部（MAC 地址、EtherType）就能决定如何处理
- **效率**：如果协议驱动对这类帧不感兴趣，无需复制整个帧

只有当协议驱动需要完整帧时，才调用 `NdisTransferData` 获取剩余部分——这会触发一次额外的数据拷贝，在 `TransferDataCompleteHandler` 中完成。

> 💡 **洞见**：`ReceiveHandler` + `NdisTransferData` + `TransferDataCompleteHandler` 的三段式，是"惰性评估"思想在内核网络栈的体现——"不到必要时不付出代价"。这与现代编程中的 Lazy Loading 模式异曲同工。

### 11.7.2 ReceiveHandler 的实现

**📌 书中的世界**

`NdisProtReceive` 的实现逻辑：

1. 从前视缓冲区提取目的 MAC、源 MAC、EtherType
2. 检查是否匹配用户态设置的过滤条件
3. 如果不匹配，返回 `NDIS_STATUS_NOT_ACCEPTED`
4. 如果匹配，调用 `NdisTransferData` 获取完整帧
5. 在 `TransferDataCompleteHandler` 中将完整帧入队

**🏭 工业实践**：

这正是 Npcap 的内核驱动 `npf.sys` 的核心逻辑——检查 EtherType 过滤，匹配则将帧提交给用户态。

> ⚠ **关键细节**：`ReceiveHandler` 的返回值非常重要：
> 
> - `NDIS_STATUS_SUCCESS`：协议驱动接受了这个帧
> - `NDIS_STATUS_NOT_ACCEPTED`：协议驱动拒绝了这个帧（但其他协议驱动可能接受）
> - 这意味着**多个协议驱动可以绑定到同一个网卡，各自独立过滤**

这是 NDIS 协议驱动模型的精妙之处——TCP/IP 协议栈和你的自定义协议驱动可以**同时**绑定到同一个网卡，各自处理自己感兴趣的帧。

### 11.7.3 TransferDataCompleteHandler 的实现

**📌 书中的世界**

`NdisProtTransferDataComplete` 在 `NdisTransferData` 异步完成后被调用：

1. 从 Packet 上下文取出相关信息
2. 将完整帧入队到接收包队列
3. 检查是否有 Pending 的读 IRP
4. 如果有，出队一个读 IRP，复制数据并完成

**🔍 第一性原理视角：这是"异步操作完成"的标准范式**

`TransferDataCompleteHandler` 是 `NdisTransferData` 的"完成回调"。整个流程：

```
ReceiveHandler（前视缓冲区）
    ↓ 调用 NdisTransferData
    ↓ 返回 NDIS_STATUS_PENDING
    （网卡 DMA 拷贝剩余数据）
    ↓
TransferDataCompleteHandler（完整帧已就绪）
    ↓ 入队
    ↓ 出队读 IRP
    ↓
ReadFile 完成
```

> 💡 **洞见**：这个三段式流程（Receive → TransferData → TransferDataComplete）是 NDIS 5.x 时代的经典模式。现代 NDIS 6.x 推荐使用 `ReceivePacketHandler` + `NET_BUFFER_LIST`，避免了 `NdisTransferData` 的额外拷贝，性能更好。

### 11.7.4 ReceivePacketHandler 的实现

**📌 书中的世界**

`NdisProtReceivePacket` 是更现代的接收回调，它以完整的 `NDIS_PACKET` 为参数：

1. 检查过滤条件
2. 如果匹配，调用 `NdisReferencePacket` 增加引用计数
3. 将 Packet 入队到接收包队列
4. 返回 `NDIS_STATUS_SUCCESS`
5. 稍后在读 IRP 处理中，从 Packet 中复制数据，调用 `NdisReturnPackets` 归还 Packet

**🔍 第一性原理视角：ReceivePacketHandler 是"零拷贝"思想的应用**

与 `ReceiveHandler` 不同，`ReceivePacketHandler` 直接拿到完整的 `NDIS_PACKET`——无需调用 `NdisTransferData`。协议驱动通过"增加引用计数"的方式"借用"这个 Packet，稍后处理完再归还。

> 💡 **洞见**：`ReceivePacketHandler` 模型体现了"零拷贝"的性能哲学——数据在网卡 DMA 缓冲区中只存在一份，协议驱动通过引用计数"借用"，避免了不必要的内存拷贝。这是高性能网络驱动的标准做法。

**🏭 现代工业演进**：

NDIS 6.x 的 `ReceiveNetBufferListsHandler` 以 `NET_BUFFER_LIST` 为参数，语义与 `ReceivePacketHandler` 完全一致，但是基于更现代的 `NET_BUFFER` 结构。

### 11.7.5 接收数据包的入队

**📌 书中的世界**

无论通过哪种接收回调，协议驱动都需要将收到的数据包入队：

```
接收包队列（Spinlock 保护）
    ↓ 入队（ReceiveHandler / ReceivePacketHandler 上下文，DISPATCH_LEVEL）
    ↓ 出队（ReadFile IRP 处理上下文，PASSIVE_LEVEL）
```

**🔍 第一性原理视角：这是一个"中断上下文到用户上下文"的桥接队列**

关键设计点：

- 入队操作在 `DISPATCH_LEVEL` 执行（网卡中断上下文），必须使用自旋锁
- 出队操作在 `PASSIVE_LEVEL` 执行（用户态线程上下文）
- 队列必须支持并发入队和出队

> ⚠ **工业级陷阱**：
> 
> - 自旋锁保护的临界区必须极短（只做入队/出队操作）
> - 不能在自旋锁保护的临界区内调用可能导致页错误的函数
> - 必须处理队列满的情况（丢弃数据包或阻塞接收）

### 11.7.6 接收数据包的出队和读请求的完成

**📌 书中的世界**

当 ReadFile IRP 到达或数据包到达时，驱动执行出队操作：

1. 检查接收包队列是否有数据
2. 如果有，出队一个数据包
3. 从数据包中复制数据到用户缓冲区
4. 调用 `NdisReturnPackets` 归还 Packet（如果是 ReceivePacketHandler 收到的）
5. 完成 IRP

**🔍 第一性原理视角：生产者-消费者模型的闭环**

```
生产者：ReceiveHandler / ReceivePacketHandler（DISPATCH_LEVEL）
    ↓ 数据包入队
接收包队列
    ↓ 数据包出队
消费者：ReadFile IRP 处理（PASSIVE_LEVEL）
    ↓ 复制数据到用户缓冲区
    ↓ 完成 IRP
用户态 ReadFile 返回
```

> 💡 **洞见**：这个模型是 Windows 内核中"中断上下文与用户态上下文解耦"的标准范式。类似的模式也出现在：
> 
> - 第9章 Minifilter 的 Post-operation 回调（DISPATCH_LEVEL）将工作项入队，用户态服务线程处理
> - 第10章 TDI 过滤的完成例程（DISPATCH_LEVEL）将事件通知入队，用户态服务线程处理
> - 第11章 NDIS 协议驱动的 ReceiveHandler（DISPATCH_LEVEL）将数据包入队，用户态 ReadFile 线程处理

**理解了这个"中断上下文入队 + 用户态上下文出队"的范式，你就掌握了 Windows 内核网络驱动的核心架构模式**。

---

## 🎯 本章核心洞见总结

读完第11章，我希望你带走的不只是"怎么写一个 NDIS 协议驱动"，而是这五个**第一性原理**：

**1. NDIS 协议驱动的本质是"在链路层之上实现协议实体"**

与第10章 TDI 过滤（"偷看"传输层连接）不同，NDIS 协议驱动是与 tcpip.sys **并列**的协议实体——它绑定到 Miniport，直接收发原始以太网帧。

这正是 Windows 网络栈分层架构的体现：

- 应用层不知道 TCP 三次握手
- TCP/IP 不知道网卡型号
- 网卡不知道数据含义

**每一层只与相邻层通信**——这是模块化设计的威力。

**2. 异步优先是 NDIS（乃至整个 Windows 内核）的第一性哲学**

NDIS 协议驱动充满了"调用 + CompleteHandler"的异步对：

- `NdisOpenAdapter` ↔ `OpenAdapterCompleteHandler`
- `NdisCloseAdapter` ↔ `CloseAdapterCompleteHandler`
- `NdisSend` ↔ `SendCompleteHandler`
- `NdisTransferData` ↔ `TransferDataCompleteHandler`
- `NdisRequest` (OID) ↔ `RequestCompleteHandler`

**理解了这个异步范式，你就能理解 Windows 内核中所有"可能耗时操作"的设计**——从 IRP 完成例程到 Work Item 到 DPC，无一不是这个思想的变体。

**3. 池化分配是内核性能的第一性原理**

包池、缓冲池、Lookaside List、NPaged Pool——**"预分配 + 复用"优于"按需分配"**。在高性能网络驱动中，每秒几十万包的收发，如果每次都 `ExAllocatePool`，系统会瞬间崩溃。

现代 NDIS 6.x 用 `NET_BUFFER_LIST` 池取代了 `NDIS_PACKET` 池，但池化思想完全一样。

**4. 生产者-消费者队列是中断上下文与用户态上下文解耦的标准范式**

```
网卡中断（DISPATCH_LEVEL）→ 接收回调（生产者）
                              ↓ 数据包入队
                          接收包队列
                              ↓ 数据包出队
用户态 ReadFile（PASSIVE_LEVEL）→ 完成 IRP（消费者）
```

这个范式在 Windows 内核中无处不在——Minifilter 的 Post 回调、TDI 过滤的完成例程、NDIS 协议驱动的 ReceiveHandler，都用这个模式解耦"高 IRQL 生产者"与"低 IRQL 消费者"。

接着上一节的五个第一性原理，我们把第5点完整展开——这是理解 Windows 网络驱动三十年演进的"地图"，也是你读了《寒江独钓》第10章（TDI）和第11章（NDIS 协议驱动）之后，必须建立的现代视角。

## 5. 从 NDIS 5.x 到 NDIS 6.x 再到 WFP 的演进路径

### 5.1 NDIS 5.x：书中示例所处的时代

书中第11章的示例代码基于 **NDIS 5.x 模型**——这是 Windows 2000/XP/Server 2003 时代的 NDIS 版本。它的核心特征：

- **数据结构**：`NDIS_PACKET` + `NDIS_BUFFER` 描述一个数据包
- **接收模型**：`ReceiveHandler`（前视缓冲区）+ `NdisTransferData` + `TransferDataCompleteHandler` 三段式
- **发送模型**：`NdisSendPackets` 发送 `NDIS_PACKET` 数组，通过 `NdisMSendComplete` 完成
- **池化**：`NdisAllocatePacketPool` / `NdisAllocateBufferPool`
- **包堆叠（Packet Stacking）**：NDIS 5.1 引入的优化，避免 IM 驱动转发时拷贝 OOB 数据

这套模型在当时的硬件条件下是合理的——百兆/千兆网卡，CPU 单核或少量多核，数据包是"一个个处理"的。

### 5.2 NDIS 6.x：Vista 带来的架构级重构

Windows Vista 引入了 **NDIS 6.0**，随后演进到今天的 **NDIS 6.89**（Windows 11 / Server 2025）。这是 NDIS 历史上最大的一次架构升级，核心变化有三个：

#### 变化一：数据结构从 `NDIS_PACKET` 到 `NET_BUFFER_LIST` / `NET_BUFFER`

这是最深刻的变革。NDIS 6.0 用 `NET_BUFFER_LIST` 结构取代了 `NDIS_PACKET` 结构——`NET_BUFFER` 是 `NDIS_PACKET` 的功能对等物，但它可以链接在 `NET_BUFFER_LIST` 描述的链表中。

带来的工程收益（微软官方明确列出的）：

- **单次调用处理多个包**：驱动通过链表形式的 `NET_BUFFER` 一次性发送/接收多个包，无需预先知道包数量
- **统一的发送/接收完成状态**：完成状态写在 `NET_BUFFER_LIST` 的 `Status` 成员中，而不是通过函数返回值或 `NdisMSendComplete` 的参数传递
- **消除包堆叠**：NDIS 6.0 通过 `NET_BUFFER_LIST` 提供上下文信息，消除了对 packet stacking 的需求
- **支持克隆、分片、重组**：`NET_BUFFER_LIST` 可以被派生、克隆、分片和重组

> 💡 **洞见**：`NET_BUFFER_LIST` 的设计哲学是"**批处理 + 零拷贝**"。它把多个包组织成链表一次性递交，减少了函数调用次数；整个驱动栈使用统一的数据封装，"不需要重新封装数据、简化数据处理，并减少函数调用数目"。这对 10G/40G/100G+ 网卡至关重要——单个包的处理开销必须被摊销到包链表上。

#### 变化二：发送/接收全异步化

NDIS 6.0 中：

- `NdisSendPackets` → `NdisSendNetBufferLists`
- `MiniportSendPackets` → `MiniportSendNetBufferLists`
- `NdisMIndicateReceivePacket` → `NdisMIndicateReceiveNetBufferLists`
- `NdisMSendComplete` → `NdisMSendNetBufferListsComplete`
- `MiniportReturnPacket` → `MiniportReturnNetBufferLists`

**所有发送和接收操作都是异步的**。"NDIS always completes a send operation asynchronously by calling the ProtocolSendNetBufferListsComplete function"——这是 NDIS 6.x 的硬性规定，不再有同步完成的选项。

#### 变化三：Backfill 机制

NDIS 6.0 引入了 backfill（数据回填）需求传播机制：

- 中间驱动从下层驱动接收 backfill 需求
- 在 `NDIS_MINIPORT_ADAPTER_GENERAL_ATTRIBUTES` 或重启属性中为虚拟 miniport 指定自己的 backfill 大小需求
- 驱动将自己的 backfill 需求加到下层驱动报告的大小上

这解决了 NDIS 5.x 时代一个微妙的问题：当 IM 驱动需要在包头前插入数据时（如 VPN 封装），必须有预留空间。NDIS 6.x 通过 backfill 机制让整个驱动栈"自上而下"协商预留空间，**避免了 NDIS 5.x 中"分配新包+拷贝 OOB 数据"的开销**。

#### 5.x → 6.x 的映射表

|NDIS 5.x|NDIS 6.x|本质变化|
|---|---|---|
|`NDIS_PACKET`|`NET_BUFFER_LIST`|单包 → 包链表|
|`NDIS_BUFFER`|`NET_BUFFER`|功能对等|
|`NdisSendPackets`|`NdisSendNetBufferLists`|数组 → 链表|
|`NdisMIndicateReceivePacket`|`NdisMIndicateReceiveNetBufferLists`|数组 → 链表|
|`NdisMSendComplete`|`NdisMSendNetBufferListsComplete`|完成状态在 NBL.Status|
|`MiniportReturnPacket`|`MiniportReturnNetBufferLists`|批量返回|
|`NdisTransferData` + `TransferDataCompleteHandler`|`ReceivePacketHandler` + `NET_BUFFER_LIST`|零拷贝接收|
|包堆叠（Packet Stacking）|`NET_BUFFER_LIST` 上下文|消除额外数据处理|
|同步/异步混合|全异步|统一异步模型|

> 💡 **洞见**：NDIS 6.x 不是"打补丁"，而是**对数据包抽象的整体重构**。它解决的核心问题是：NDIS 5.x 的"单包处理模型"在 10G+ 网卡时代成为了性能瓶颈。`NET_BUFFER_LIST` 把"批处理"做成了数据结构的一等公民——这不是简单的 API 改名，而是**网络驱动架构哲学的转变**：从"处理包"到"处理包的批次"。

### 5.3 WFP：网络过滤的"Minifilter 时刻"

如果说 NDIS 6.x 解决了"数据包如何高效表示和传输"，那么 **WFP（Windows Filtering Platform）**​ 解决的是另一个问题："**网络过滤如何框架化**"。

微软在 WFP 官方文档中明确宣布："**WFP 旨在取代以前的数据包筛选技术，例如 TDI 过滤器、NDIS 过滤器，以及 Winsock 分层服务提供商（LSP）**"。从 Windows Server 2008 和 Windows Vista 开始，防火墙钩子和过滤钩子驱动程序已不可用，使用这些驱动程序的应用程序应改用 WFP。

#### WFP 解决的根本问题

在 WFP 之前，开发者面临一个两难：

- **TDI 过滤**：能看到进程上下文（"哪个进程发起连接"），但看不到原始包，且只能处理 TCP/IP 流量，无法感知应用和用户
- **NDIS 过滤**：能看到原始包，但看不到进程身份，需要自己实现连接跟踪，且"一个 bug 就可能杀死整个网络栈"

WFP 的解法是：**在网络栈的多个层插入"过滤层（Layer）"**，每层都能提供丰富的上下文：

- **ALE 层（Application Layer Enforcement）**：提供进程 ID、用户 ID、应用路径——这是"按应用过滤"的基础
- **传输层（Transport Layer）**：提供 TCP/UDP 端口、IP 地址
- **网络层（Network Layer）**：提供 IP 规则过滤
- **流层（Stream Layer）**：提供重组后的流数据，可做 DPI

WFP 的开发者只需**声明过滤规则**，而不是**挂钩（hook）某个驱动**：

```
FWPM_FILTER filter = {0};
filter.layerKey = FWPM_LAYER_ALE_AUTH_CONNECT_V4;
filter.action.type = FWP_ACTION_BLOCK;
filter.filterCondition[0].fieldKey = FWPM_CONDITION_ALE_APP_ID;
filter.filterCondition[0].matchType = FWP_MATCH_EQUAL;
filter.filterCondition[0].conditionValue.byteBlob = AppIdBlob; // app.exe
FwpmFilterAdd(engineHandle, &filter, NULL, NULL);
```

就这么简单——**"No packet parsing. No PID tracking. No race handling."**

#### WFP 的工业地位

微软文档明确指出："**Windows Vista、Windows Server 2008 及更高版本操作系统中内置的 Windows 防火墙高级安全（WFAS）是使用 WFP 实现的**"。这意味着：

- Windows 自带的高级安全防火墙就是 WFP 应用
- 使用 WFP API 或 WFAS API 开发的应用程序使用 WFP 内置的通用过滤仲裁逻辑
- **CrowdStrike、Microsoft Defender for Endpoint、Carbon Black、几乎所有商业 EDR**​ 的网络遥测核心都是 WFP callout 驱动

#### 从 TDI/NDIS 过滤到 WFP 的迁移映射

|书中技术（TDI/NDIS 过滤）|现代 WFP 等价物|优势|
|---|---|---|
|TDI 过滤驱动挂钩 `\Device\Tcp`|ALE Connect/Accept 层 callout|WFP 原生理解进程身份|
|解析 `TDI_ADDRESS_IP` 结构|`FWPS_INCOMING_VALUES` 直接提供 IP:Port|无需二进制解析|
|NDIS IM 驱动挂钩发送/接收|传输层/网络层 callout|自动提供流上下文|
|手动维护连接跟踪表|ALE 层自动维护连接状态|无内存泄漏风险|
|多个 TDI/NDIS 驱动冲突、排序不确定|WFP 多层过滤 + 通用仲裁逻辑|多产品共存|
|`FILTER_ALLOW/DENY` 返回值|`FWP_ACTION_BLOCK/PERMIT`|语义一致，但框架化|
|内核态复杂逻辑|用户态 `Fwpm*` API + 内核态 `Fwps*` callout|策略与执行分离|

> 💡 **洞见**：WFP 之于 TDI/NDIS 过滤，正如 Minifilter 之于 legacy 文件系统过滤驱动——**把"不变的过滤基础设施"框架化，把"变的业务决策"通过回调/规则注入**。这是 Windows 内核架构演进的统一范式。

### 5.4 NDIS 协议驱动在现代 Windows 中的定位

这里必须澄清一个常见误解：**"WFP 取代 NDIS"是错误的说法**。准确的说法是——**"WFP 取代了 NDIS 过滤驱动在网络过滤场景中的地位，但 NDIS 协议驱动、NDIS 过滤驱动、NDIS 中间驱动在各自的领域仍然活跃"**。

微软文档明确说明，Windows Vista 及更高版本仍然支持四种 NDIS 驱动类型，且**最新 NDIS 版本是 6.89**（集成在 Windows 11 和 Windows Server 2025 中）：

|驱动类型|现代适用场景|工业实例|
|---|---|---|
|**Miniport 驱动**​|控制物理网卡或虚拟设备|Intel e1000、Realtek 驱动、Hyper-V vSwitch|
|**协议驱动**​|实现网络协议或特定于应用的网络接口|tcpip.sys 本身就是协议驱动；自定义 VPN 协议、工业私有协议、原始以太网捕获|
|**过滤驱动（Lightweight Filter）**​|在不改变现有驱动的情况下修改或监视网络活动|收集网络统计信息、实现修改或监控筛选器|
|**中间驱动**​|实现负载均衡或故障转移等多路复用服务|VPN 虚拟网卡、网卡团队化（NIC Teaming）|

**关键判断**：

- **如果你的目标是"网络过滤/防火墙/EDR 网络遥测"**​ → 用 WFP，不要用 NDIS 过滤驱动或 TDI 过滤
- **如果你的目标是"实现自定义链路层协议/VPN/虚拟网卡/原始帧捕获"**​ → 用 NDIS 协议驱动（NDIS 6.x 模型）
- **如果你的目标是"控制物理网卡"**​ → 用 NDIS Miniport 驱动（NDIS 6.x 模型，且 NDIS 6.89 强制新 Miniport 使用 WDF）
- **如果你的目标是"流量修改/监控且不需要应用层上下文"**​ → 用 NDIS Lightweight Filter 驱动

### 5.5 完整的演进路径图

把 NDIS 5.x → 6.x → WFP 的演进放到 Windows 网络栈的历史中看：

```
Windows 2000/XP 时代：
    ┌─────────────────────────────────────────┐
    │ 应用层 (Winsock)                         │
    ├─────────────────────────────────────────┤
    │ afd.sys                                  │
    ├─────────────────────────────────────────┤
    │ tcpip.sys (NDIS 5.x 协议驱动)            │
    ├─────────────────────────────────────────┤
    │ TDI 过滤驱动（挂钩 \Device\Tcp）          │ ← 网络过滤方案①
    ├─────────────────────────────────────────┤
    │ NDIS 5.x IM 驱动（中间层过滤）            │ ← 网络过滤方案②
    ├─────────────────────────────────────────┤
    │ NDIS 5.x Miniport 驱动                   │
    ├─────────────────────────────────────────┤
    │ 物理网卡                                 │
    └─────────────────────────────────────────┘
    过滤方案：TDI 挂钩 + NDIS IM，多个驱动排序冲突

Windows Vista/Server 2008 时代（分水岭）：
    ┌─────────────────────────────────────────┐
    │ 应用层 (Winsock)                         │
    ├─────────────────────────────────────────┤
    │ afd.sys ──TLNPI──┐                       │
    ├──────────────────┼──────────────────────┤
    │                  ↓                       │
    │         tcpip.sys (NDIS 6.x 协议驱动)    │
    │              ↓                            │
    │    ┌──────────────────────────────────┐  │
    │    │  WFP 引擎（多层过滤框架）          │  │ ← 网络过滤的统一方案
    │    │  ALE 层 / 传输层 / 网络层 / 流层   │  │
    │    └──────────────────────────────────┘  │
    │              ↓                            │
    │ NDIS 6.x Lightweight Filter 驱动         │
    ├─────────────────────────────────────────┤
    │ NDIS 6.x Miniport 驱动                   │
    ├─────────────────────────────────────────┤
    │ 物理网卡                                 │
    └─────────────────────────────────────────┘
    TDI 和 LSP 弃用，NDIS IM 过滤被 WFP 取代
    
Windows 10/11/Server 2025 时代（今天）：
    - NDIS 版本：6.89
    - 网络过滤：WFP（Windows 高级安全防火墙基于 WFP）
    - NDIS 协议驱动：仍用于自定义协议、VPN、虚拟网卡、原始帧捕获
    - NDIS Miniport：强制使用 WDF（Windows Driver Framework）
```

### 5.6 对《寒江独钓》读者的实践指导

读完第10章（TDI）和第11章（NDIS 协议驱动），你应该建立这样的现代视角：

**📌 书中的知识并不过时，但必须知道"什么时候用"**

1. **理解原理的价值永不过时**：
    - TDI 过滤让你理解"传输层语义如何在内核中表示"
    - NDIS 协议驱动让你理解"链路层契约如何工作"
    - 这些底层知识让你在 WFP 出现诡异问题时能快速定位根因
2. **新项目的正确技术选型**：
    - 网络防火墙/EDR 网络遥测 → **WFP callout 驱动**
    - 自定义 VPN 协议 → **NDIS 6.x 协议驱动**​ 或 **WFP 流向重定向**
    - 虚拟网卡 → **NDIS 6.x Miniport 驱动**（用 WDF）
    - 流量统计/监控（无应用上下文需求）→ **NDIS 6.x Lightweight Filter**
    - 网卡硬件驱动 → **NDIS 6.x Miniport 驱动**（WDF 强制）
3. **从书中到现代的迁移路径**：
    - 第10章 TDI 过滤 → 学 WFP ALE 层 callout（`FWPM_LAYER_ALE_AUTH_CONNECT_V4` 等）
    - 第11章 NDIS 5.x 协议驱动 → 学 NDIS 6.x 协议驱动（`NET_BUFFER_LIST` 模型）
    - 书中 `ReceiveHandler` + `NdisTransferData` → NDIS 6.x 的 `ProtocolReceiveNetBufferLists`
    - 书中 `NdisSendPackets` → NDIS 6.x 的 `NdisSendNetBufferLists`
4. **遗留代码维护**：
    - Windows 10/11 仍然兼容 NDIS 5.x 驱动（NDIS 会在内部翻译 `NDIS_PACKET` ↔ `NET_BUFFER_LIST`）
    - TDI 过滤驱动在 Win10+ 上通过 TDX.sys 兼容层仍可运行，但微软已明确"will be removed in future versions"
    - **不要为新项目写 TDI 过滤驱动**

> ⚠️ **工业现实警示**：
> 
> 截至 2026 年，如果你在一个 EDR 或防火墙项目中从零开始写网络过滤驱动，**不二之选是 WFP**。TDI 过滤和 NDIS IM 过滤驱动只应在维护老代码时有意义。而在最新的 NDIS 6.89 中，新开发的 Miniport 驱动已经被强制要求使用 WDF——这是微软推动驱动开发框架化的又一强硬手段。

### 5.7 第一性原理的终极洞察

从 NDIS 5.x 到 NDIS 6.x 再到 WFP 的演进，揭示了 Windows 内核架构演进的统一第一性原理：

**💡 第一性原理：把"不变的复杂性"框架化，把"变的业务逻辑"通过声明式接口注入**

- **NDIS 6.x 对数据包抽象的重构**：把"批处理"做成了数据结构的一等公民（`NET_BUFFER_LIST`），消除了 NDIS 5.x 中包堆叠和 OOB 数据拷贝的开销
- **WFP 对网络过滤的重构**：把"多层过滤、多产品仲裁、连接跟踪"框架化，开发者只需声明规则或注册 callout
- **Minifilter 对文件系统过滤的重构**（第9章）：把"设备栈绑定、IRP 路由、上下文生命周期"框架化

这三次重构是同一个工程哲学的三种表达。**理解了这个哲学，你就能预测 Windows 内核的未来演进方向**——任何"过滤/拦截/监控"场景，最终都会被框架化，开发者从"过程式挂钩"解放出来，转向"声明式注册"。

---

## 🎯 第11章读书笔记完整总结

读完第11章"NDIS 协议驱动"，结合现代工业实践，我希望你带走这六个**第一性原理**：

1. **NDIS 协议驱动的本质是"在链路层之上实现协议实体"**——与 tcpip.sys 并列，直接收发原始以太网帧
2. **异步优先是 NDIS（乃至整个 Windows 内核）的第一性哲学**——"调用 + CompleteHandler"模式无处不在
3. **池化分配是内核性能的第一性原理**——"预分配 + 复用"优于"按需分配"
4. **生产者-消费者队列是中断上下文与用户态上下文解耦的标准范式**——`ReceiveHandler`（DISPATCH_LEVEL）入队，ReadFile（PASSIVE_LEVEL）出队
5. **从 NDIS 5.x 到 6.x 的演进是"批处理化 + 零拷贝 + 全异步"**——`NET_BUFFER_LIST` 把"批处理"做成了一等公民
6. **WFP 是网络过滤的"Minifilter 时刻"**——框架化"多层过滤、多产品仲裁、连接跟踪"，开发者只需声明规则

**工业现实导航**：

- 学 NDIS 协议驱动 → 理解链路层契约和 Windows 网络栈原理（本书第11章的价值）
- 写现代网络过滤 → 用 WFP
- 写自定义协议/VPN/虚拟网卡 → 用 NDIS 6.x 协议驱动
- 写网卡驱动 → 用 NDIS 6.x Miniport + WDF
- **绝对不要为新项目写 TDI 过滤驱动**——TDI 已弃用，且"将从未来版本的 Windows 中移除"

> 📌 **最后一句话**：第10章（TDI）和第11章（NDIS 协议驱动）在今天的工业界，**作为"网络过滤方案"已经退役**，但**作为"理解 Windows 网络栈底层原理的教材"仍然鲜活**。你读这两章的正确姿势是：通过它们理解"网络栈过滤的底层机制与历史演进"，然后知道在现代 Windows 上如何用 WFP 更优雅地解决同样的问题。这正符合你要求的"第一性原理式学习"——不只学"怎么做"，更要学"为什么这么做"以及"为什么不这么做"。

