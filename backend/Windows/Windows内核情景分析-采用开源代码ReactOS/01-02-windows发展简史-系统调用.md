
这两章其实是整本书的"元章节"——第1章告诉你**为什么要有一个内核**、**内核住在哪个地址空间**、**内核代码怎么命名**；第2章告诉你**用户态和内核态之间那道门是怎么开的**。我的笔记不会照本宣科，而是把这两章揉成一条主线：**Windows 内核的本质是一个"受控的特权入口 + 一张间接分发表"**，后面所有章节（内存、对象、进程线程、I/O）全是这条主线的展开。

> 💡 先说明一点：毛德操的书基于 ReactOS 0.3.3 时代源码，对应的是 x86 平台的 NT 内核思路。现代 Windows 早已切到 x64，系统调用入口、陷阱帧布局、SSDT 保护机制都变了，但**"特权级隔离 + 系统调用号分发"这一第一性原理没变**。下面我会在每节末尾补一段"现代映照"，让你拿着书也能对接今天的 Windows。

---

## 第1章 概述

### 1.1 Windows 操作系统发展简史

**书的脉络**：从 DOS → Windows 9x → Windows NT 三条线讲起。9x 系列依赖 DOS 引导、没有真正的特权级隔离，驱动崩溃就整机蓝屏，关键 API（VMM/IFS/CC）从未公开，2000 年随 ME 被废弃。**NT 架构才是研究主线**——它既有公开文档、又有 ReactOS 这种开源实现、又被现代 Windows 全部继承。

**我的洞见**：很多人学 Windows 内核喜欢从 9x 切入，因为资料多、能跑在虚拟机里随便折腾。但这是条死路。NT 架构的设计哲学从 1993 年 NT 3.1 起就定了调：**微内核思想的折衷版**——保留一个"执行体（Executive）"承载大多数服务，下方是"内核层（Kernel）"只管调度和陷阱，再下方是 HAL 屏蔽硬件。这个分层不是技术必然，而是微软当年想同时拿下服务器市场和个人用户市场的商业决策——一颗内核，两种 SKU。理解了这点，你后面看到 `ntoskrnl.exe` 既包含 Ke* 又包含 Ex* 就不会觉得违和。

**现代映照**：今天的 Windows 10/11/Server 2022 依然是 NT 架构，版本号已经走到 NT 10.x。ReactOS 至今仍在追赶 NT 4.0 ~ XP 的语义，所以书里对 NT 内核机制的讲解，**对今天依然成立**。

### 1.2 用户空间和系统空间

**书的脉络**：32 位 x86 下 4GB 虚拟地址空间一切为二——低 2GB 给用户（ring 3），高 2GB 给系统（ring 0）。好处有三：隔离（应用不能直接碰内核数据）、保护（硬件特权级）、共享（所有进程共用一份内核代码）。

**我的洞见**——这一节其实在回答一个第一性问题：**为什么操作系统非要把地址空间劈成两半？**

根本原因在于：CPU 执行一条指令时，它访问哪个地址、能不能执行特权指令，必须由硬件强制约束，不能靠"应用自觉"。所以"用户空间 vs 系统空间"不是个软件约定，而是 **x86 分段 + 分页 + CPL（Current Privilege Level）三位一体的硬件契约**：

- 用户态 CPL=3，访问高 2GB 时分页机制直接拒绝
- 要从 ring 3 进 ring 0，**只有 CPU 认可的几条路径**（中断、自陷指令、快速系统调用），这就是第2章的主题
- 每个进程有独立的用户空间映射，但通过"分页表切换"共享同一份内核映射

> ⚠️ 书里默认 2GB/2GB 划分。实际可以用 `/3GB` 启动参数调成 3GB 用户/1GB 系统，64 位系统则是 128TB/128TB。但**"用户态 vs 内核态"这个二分思想不变**——这是理解后续所有章节的钥匙。

**现代映照**：x64 下用户空间 128TB、系统空间 128TB，但 PML4 页表顶层的高半部分内核独占，低半部分每进程独立。思想上和 32 位完全一致，只是位数宽了。

### 1.3 Windows 内核

**书的脉络**：内核是"一层一层叠加的子系统"：

|层|前缀|职责|ReactOS 目录|
|---|---|---|---|
|HAL|-|屏蔽硬件差异（中断控制器、ACPI、DMA）|`hal/`|
|Kernel|Ke*|线程调度、中断/异常、IRQL、自旋锁、DPC/APC|`ntoskrnl/ke/`|
|Executive|Ex*, Ob*, Mm*, Io*|对象管理、内存管理、I/O 管理、进程线程|`ntoskrnl/` 各子目录|
|Win32k|-|窗口与图形子系统内核态部分|`win32ss/`|

**我的洞见**：这张表里最容易被忽略的一行是 **Ke 层只提供机制、不制定策略**。什么叫机制？线程切换、自旋锁、DPC 这些是机制——它们是"怎么做"。什么叫策略？哪个线程该跑多久、优先级怎么算，这是执行体的事——"做什么"。

这个分离是 NT 内核最优雅的设计决策之一。它意味着 Ke 层可以被实时扩展、可以被多处理器扩展，而不用改上层策略。你看 ReactOS 的 `ntoskrnl/ke/i386/`、`amd64/`、`arm/` 三个子目录就能明白——同一套 Ke API，三套架构实现。

**现代映照**：今天的 Windows 内核依然严格遵循这个分层。驱动开发者日常打交道的 `WDM`（Windows Driver Model）和 `WDF`（Windows Driver Foundation）都建立在 Io* 和 Ke* 之上。书里讲的 Executive 概念，在工业界就是你现在用 `dt nt!_EPROCESS`、`dt nt!_OBJECT_HEADER` 能在 WinDbg 里看到的那些结构体。

### 1.4 开源项目 ReactOS 及其代码

**书的脉络**：ReactOS 目标是"开源的 Windows"，忠实实现 NT 的系统调用语义。作者选它做参照系，是因为 NT 内核源码不公开，而 ReactOS 的 `KiSystemService`、`NtCreateFile` 等函数几乎一一对应微软同名函数。

**我的洞见**：这里有个**学习方法论**的创新点值得你记住——

> 📌 读 ReactOS 源码不是在读"Windows 源码"，而是在读"对一个二进制接口的逆向工程式还原"。ReactOS 的贡献者们当年是看着 NT 的 PDB、反汇编 `ntoskrnl.exe`、比对系统调用行为，一行一行写出 `Nt*` 函数的。所以 ReactOS 代码的价值不在"它和 Windows 一模一样"，而在"它揭示了 NT 内核设计的**意图**"。

这意味着：当你在书里看到某个 ReactOS 函数的实现细节，你应该问的不是"Windows 是不是真的这么写"，而是"Windows 为什么**必须**这么写"——前者你可能记错，后者才是第一性原理。

**现代映照**：ReactOS 今天依然活跃（最新版本 0.4.x），但它追的是 XP/2003 时代的 NT。现代 Windows 10/11 的 `ntoskrnl.exe` 已经膨胀到几十 MB，引入了 PatchGuard、HVCI、VBS 等安全机制，ReactOS 并没有完全追上。所以**书+ReactOS 给你的是"骨架"，现代 Windows 给你的是"骨架+肌肉+装甲"**。

### 1.5 Windows 内核函数的命名

**书的脉络**：以一张"函数命名词典"收尾：

- **Ke***：内核层（Kernel）——调度、中断、同步原语
- **Ex***：执行体（Executive）——通用的执行体服务
- **Ob***：对象管理（Object）
- **Mm***：内存管理（Memory）
- **Io***：I/O 管理
- **Ps***：进程（Process）
- **Nt***：系统调用（Native API），用户态经 ntdll 进入
- **Zw***：内核态发起的系统调用包装，与 Nt* 指向同一实现但 PreviousMode 不同
- **Hal***：硬件抽象层
- **Ki***：内核内部（Kernel Internal），如 `KiSystemService`

**我的洞见**：这套命名法不是装饰，而是**一种类型系统**——你看到函数前缀，就能猜出它在哪一层、调用它需不需要特定 IRQL、能不能分页。比如：

- 带 `KeAcquireSpinLock` 的，调用时 IRQL <= DISPATCH_LEVEL
- 带 `MmProbe` 系列的，必须在 `<= APC_LEVEL` 且 Probe 用户地址
- `Zw*` 和 `Nt*` 的区别**只在 PreviousMode**——`Zw*` 让内核认为调用者来自内核模式，跳过参数 Probe；`Nt*` 让用户态参数被强制校验

这个差异是第2章"从内核中发起系统调用"那一节的伏笔。

**现代映照**：今天的 Windows 内核符号（可通过 Microsoft Symbol Server 获取 PDB）依然严格遵循这套命名。你用 WinDbg 看到的 `nt!NtCreateFile`、`nt!KiSystemCall64`、`nt!MmProbe` 就是书里那些函数的"现代直系后代"。

---

## 第2章 系统调用

这一章是全书的"开关"——理解了系统调用，你就拿到了进入内核的钥匙。

### 2.1 内核与系统调用

**书的脉络**：用户态程序不能直接执行特权指令，必须通过"自陷"进入内核。Windows 提供了两条经典路径：

1. **`int 0x2E`**——传统中断门方式，CPU 自动压栈、查 IDT、跳到 `KiSystemService`
2. **`sysenter`/`syscall`**——快速系统调用（见 2.5）

典型调用链：

```
CreateFileW (kernel32.dll)
  → CreateFileInternal (kernel32 内部)
  → NtCreateFile (ntdll.dll, stub)
  → sysenter / int 0x2E
  → NtCreateFile (ntoskrnl 实现)
  → I/O 管理器派发 IRP
  → 文件系统驱动
```

**我的洞见**——这一节揭示了一个**第一性原理：系统调用是"受控的间接跳转"**。

用户态想要内核服务，但不能让用户态直接 `call ntoskrnl!NtCreateFile`——那样就没有特权级隔离了。所以 CPU 提供了一条"专用通道"：用户态只能跳到 CPU 预先设定的入口（`int 0x2E` 对应 IDT[0x2E]，`sysenter` 对应 MSR 指定的地址），入口之后内核才根据 `eax` 里的系统调用号去查表分发。

这个设计有三重含义：

- **入口唯一**：所有系统调用都必须从同一个门进，便于审计和 Hook
- **号而非址**：用户态给的是"号"（系统调用号），不是"地址"，内核掌握绝对寻址权
- **PreviousMode**：内核通过 CS 的最低位知道调用者来自用户态还是内核态，决定是否 Probe 参数

**现代映照**：x64 Windows 用 `syscall` 指令，入口是 MSR `LSTAR` 指向的 `KiSystemCall64`。调用链仍然是 `kernel32 → ntdll!Nt* → syscall → KiSystemCall64 → SSDT 查表`。安全领域流行的"直接系统调用（Direct Syscall）"技术，就是绕开 ntdll 的 stub，自己把系统调用号塞进 rax 然后执行 syscall——本质是把 2.1 节讲的路径从"用户态 DLL 代劳"变成"攻击者手动复刻"。

### 2.2 系统调用的内核入口 KiSystemService()

**书的脉络**：`KiSystemService` 是系统调用的总入口，核心工作流：

1. 从 `eax` 取系统调用号
2. 取当前线程，保存 `PreviousMode`，设为 `UserMode`
3. 解码调用号（Offset + Id）
4. 取 SSDT 描述符，边界检查
5. `ProbeForRead` 校验用户参数
6. 从 `SSDT[Id]` 取处理函数地址
7. 通过 `KiSystemCallTrampoline` 调用
8. `KiServiceExit` 返回用户态

汇编层先构建 `KTRAP_FRAME`（伪错误码 + 段寄存器 + 通用寄存器），然后跳到 C 层的 `KiSystemServiceHandler`。

**我的洞见**——`KiSystemService` 的本质是一个**解释器**：

> 💡 把系统调用号看成字节码，SSDT 看成指令表，`KiSystemService` 就是 CPU 的解释器循环。每次系统调用都是一次"取号→译码→执行→返回"。

这个视角下，你会发现 NT 内核的设计非常"脚本语言化"：用户态给一个 opcode，内核解释执行对应的 handler。这种间接性带来了巨大的灵活性——微软可以在不打断用户态 ABI 的情况下，随时更换 `Nt*` 的实现，只要系统调用号不变。

`KTRAP_FRAME` 是整个机制里最精妙的数据结构：它保存了用户态的完整上下文，使得中断/系统调用/异常三种入口能复用同一套返回逻辑。书里讲的"构建陷阱帧"不是细节，是**整个内核可重入性的基石**。

**现代映照**：x64 下 `KiSystemCall64` 做同样的事，但陷阱帧布局因 `syscall` 不自动压栈而需要软件模拟。ReactOS 的 `KiSystemServiceHandler` 函数逻辑基本反映了微软同名函数 。现代 PatchGuard（KPP）会定期检查 SSDT 是否被篡改，所以 SSDT Hook 在 x64 上基本绝迹，取而代之的是 ETW 事件追踪、VT-x 虚拟化 Hook 等更底层的玩法 。

### 2.3 系统调用的函数跳转

**书的脉络**：拿到系统调用号后，从 `KeServiceDescriptorTable` 取出对应的 `Nt*` 函数地址。`KiSystemService` 通过 `rep movsb` 把用户栈参数复制到内核栈，然后 `call ebx` 跳过去执行。

**我的洞见**：这一步揭示了 **SSDT（System Service Descriptor Table）的本质是一张跳转表**。它把"系统调用号"和"内核函数地址"解耦——用户态永远不知道 `NtCreateFile` 的真实地址，只知道它的编号是 0x??。这种解耦带来两个工程收益：

- **版本兼容**：Windows 升级时 `NtCreateFile` 的内部实现可以重写，只要编号不变，用户态程序无需重新编译
- **安全隔离**：用户态无法伪造一个内核地址去跳，必须经过这张表

但代价是：每次系统调用都要查一次表 + 复制参数，这就是 2.5 节"快速系统调用"要优化的瓶颈。

**现代映照**：现代 Windows 有 **两张**​ SSDT——普通系统调用用 `KeServiceDescriptorTable`，GUI 相关的（win32k）用 `KeServiceDescriptorTableShadow`。Shadow SSDT 是 Vista 之后引入的，把图形子系统从 `ntoskrnl` 里进一步隔离出来 。

### 2.4 系统调用的返回

**书的脉络**：`KiServiceExit` 负责返回。它会：

1. 判断 `PreviousMode`——如果是 `UserMode`，先遍历执行本线程的内核 APC 和用户 APC 队列
2. 清理陷阱帧，恢复寄存器现场
3. `iret`（中断方式）或 `sysexit`/`sysret`（快速调用方式）回到用户态

**我的洞见**：返回路径里最容易被忽视的细节是 **APC 投递点**。为什么系统调用返回前要扫一遍 APC 队列？因为 APC（异步过程调用）是一种"延迟执行"机制——内核某处可能标记了"这个线程下次回到用户态前，先跑这段代码"。系统调用返回是天然的同步点。

这揭示了一个更深的设计哲学：**NT 内核是异步的**。几乎所有 I/O 都是异步的，APC 是异步完成通知的载体。系统调用看起来是同步的（用户态 block 住等结果），但内核内部全是异步流水线，APC 就是连接"内核异步世界"和"用户态同步假象"的桥。

> ⚠️ 这一点是理解后续第5章（进程与线程）、第6章（进程间通信）、第8章（结构化异常处理）的关键伏笔。APC 不仅用于 I/O 完成，还用于跨线程操作（如 `NtQueueApcThread`）、异常处理分发等。

**现代映照**：x64 返回用 `sysret`，APC 投递逻辑不变。现代 Windows 的"IOCP（完成端口）"机制就是建立在 APC 之上的工业级抽象——你写的 `GetQueuedCompletionStatus` 背后，就是线程在等待 APC 队列里出现 I/O 完成通知。

### 2.5 快速系统调用

**书的脉络**：`int 0x2E` 走完整中断流程，开销大（两次查表：IDT + SSDT；完整陷阱帧；栈切换）。CPU 厂商后来引入了专用指令：

- **Intel `sysenter`**：依赖三个 MSR——
    - `IA32_SYSENTER_CS`（0x174）：内核代码段
    - `IA32_SYSENTER_ESP`（0x175）：内核栈指针
    - `IA32_SYSENTER_EIP`（0x176）：内核入口（`KiFastCallEntry`）
- **AMD `syscall`**：x64 统一使用，MSR `LSTAR`（0xC0000082）存内核入口

`sysenter` 执行时 CPU **不会自动压栈**，所以 `KiFastCallEntry` 要手动模拟出和 `int 0x2E` 一致的陷阱帧。返回时用 `sysexit`（32 位）或 `sysret`（64 位）。

**我的洞见**——快速系统调用的本质是 **用"专用硬件 + 软件约定"替换"通用中断机制"**。

`int 0x2E` 走的是"异常/中断"这套通用管道，它要处理一切可能：错误码、嵌套、特权级切换、栈切换……但系统调用是高频路径，99% 的情况都是"用户态→内核态→用户态"的简单往返。所以 CPU 厂商专门为这个场景造了指令：

- 不用查 IDT（直接从 MSR 拿入口）
- 不用 CPU 自动压栈（软件按需构建精简陷阱帧）
- 专用返回指令 `sysexit/sysret`

实测快速系统调用能省 **30%~50%**​ 的模式切换开销 。对于网络服务、文件 I/O 密集应用，这是量级提升。

但代价是复杂度转移到了软件：`KiFastCallEntry` 必须手工"伪造"出和中断方式兼容的陷阱帧，否则后续的 `KiSystemService` 共用逻辑就无法工作。**这是典型的"硬件简化、软件补位"工程权衡**。

**现代映照**：x64 Windows 已经**全面使用 `syscall`**，0x2E 中断门被保留但基本不走。`LSTAR` MSR 指向 `KiSystemCall64`。安全研究领域流行的"Hell's Gate"等技术，就是通过解析 ntdll 的 export 动态获取 `syscall` 指令地址，绕过 EDR 的 Hook——本质是把 2.5 节讲的"快速系统调用"路径武器化 。

### 2.6 从内核中发起系统调用

**书的脉络**：内核态代码（比如驱动程序）也可以"发起系统调用"，方式是调用 `Zw*` 系列函数。`Zw*` 和 `Nt*` 指向同一个实现，但 `Zw*` 会通过 `mov eax, xxx; mov edx, esp; int 0x2E`（或模拟自陷）进入 `KiSystemService`，此时 CS 最低位表明 CPU 运行于系统态，于是 **PreviousMode 被设为 KernelMode**。

这意味着 `KiSystemService` **可以嵌套**——内核中已经在内核态运行的代码再次发起系统调用，PreviousMode 链被保存在栈上，保证嵌套返回时能逐层恢复。

**我的洞见**：这一节揭示了一个反直觉的事实——**系统调用不是用户态专属，内核态也可以"系统调用自己"**。

为什么内核要这么做？因为如果内核代码直接 `call NtCreateFile`，那么 `NtCreateFile` 内部做参数 Probe 时会发现调用者来自内核模式，于是**跳过 Probe 直接信任参数**——这很危险，万一内核代码传了个用户态地址呢？所以 NT 内核规定：**任何经过 `KiSystemService` 入口的调用，统一走分发逻辑**，由 PreviousMode 决定要不要 Probe。

`Zw*` 的存在意义就在这里：它告诉内核"我是内核代码，但我希望这次调用**看起来像用户态调用**"——于是 `Nt*` 实现里的 Probe 逻辑会生效，参数被严格校验。这是 NT 内核"一切皆系统调用"哲学的极致体现：**内核内部也通过系统调用机制互相调用**，保证统一的安全检查。

> 📌 对比 Linux：Linux 内核态直接调用 `sys_xxx` 函数，没有 PreviousMode 概念。Windows 的这个设计更"统一"但也更"重"——每次内核内部调用都要走一遍 `KiSystemService` 的分发逻辑。这是 NT 内核和 Linux 内核的一个深层差异。

**现代映照**：现代 Windows 驱动开发中，`Zw*` 系列仍是内核态发起系统调用的标准方式（如 `ZwCreateFile`、`ZwQuerySystemInformation`）。安全软件常 Hook `Zw*` 来监控内核行为，但因为 PatchGuard 的存在，这种 Hook 在 x64 上极易触发蓝屏，所以工业界转向了**注册回调（ObRegisterCallbacks、PsSetCreateProcessNotifyRoutine 等）**——这是微软官方推荐的"合法 Hook"机制，本质上是对 `Zw*` 调用路径的官方埋点。

---

## 两章串联：一张图看懂全书骨架

把第1章和第2章合起来，你能得到这张"全书的地图"：

```
用户态 (ring 3)                          内核态 (ring 0)
─────────────────────────────────────────────────────────────
kernel32.dll                              ntoskrnl.exe
  ↓                                        ↑
  ↓ CreateFileW                             │ 系统调用分发
  ↓                                        │
ntdll.dll                                  │
  ↓ NtCreateFile (stub)                    │
  ↓ eax =  syscall_id                      │
  ↓ edx = &args                            │
═══════════════════════════════════════════════════════════
  ↓ syscall / int 0x2E                     │
  ↓───────────────→ KiSystemCall64/KiSystemService
  ↓                            ↓            │
  ↓                  ① 构建 KTRAP_FRAME    │
  ↓                  ② 解码 syscall_id     │
  ↓                  ③ 查 SSDT              │
  ↓                  ④ Probe 参数           │
  ↓                  ⑤ call NtCreateFile   │
  ↓                  ⑥ I/O 管理器派发 IRP   │
  ↓                  ⑦ 文件系统驱动         │
  ↓                  ⑧ KiServiceExit        │
  ↓                            ↓            │
  ↓←──────────────── sysexit/sysret        │
  ↓                                        │
用户态继续                                 │
```

**第3章及以后所有内容，都是这张图里 `Nt*` 函数内部的展开**：

- 第3章"内存管理" → `NtAllocateVirtualMemory` 内部怎么走
- 第4章"对象管理" → `NtOpenFile` 怎么把文件名转成对象句柄
- 第5章"进程与线程" → `NtCreateProcess/NtCreateThread` 的全身
- 第9章"设备驱动" → I/O 管理器怎么把 IRP 派发到驱动栈

所以你读书的正确姿势是：**每遇到一个 `Nt*` 函数，先在脑子里回到第2章的 SSDT 查表那一刻，再顺着第1章的分层往下钻**。这样整本书就不是 14 个孤立章节，而是一棵从第2章 `KiSystemService` 长出来的树。

---

## 工业界实际应用补充

光读书不够，你需要知道今天工业界怎么用这些知识：

**🔐 安全领域**

- **EDR（端点检测响应）**：现代 EDR 不再 Hook SSDT（会被 PatchGuard 杀），而是通过 ETW（Event Tracing for Windows）事件、注册 ObCallback/PsCallback 来监控 `Nt*` 调用
- **Direct Syscall / Hell's Gate**：红队绕开 ntdll 的 Hook，自己实现 `syscall` 指令，本质是把 2.5 节"快速系统调用"武器化
- **VT-x Hook**：利用 CPU 虚拟化，让 `syscall` 触发 VM-Exit 被 VMM 拦截，这是目前最底层的系统调用监控方式

**🛡️ 微软自身的演进**

- **PatchGuard（KPP）**：x64 上定期检查 SSDT、IDT、关键内核结构体是否被篡改，触发即 BSOD
- **HVCI（基于虚拟化的代码完整性）**：把内核代码完整性检查放到 VTL1 里，VTL0 的内核无法篡改 VTL1 的检查逻辑
- **VBS（基于虚拟化的安全）**：把敏感操作（如凭证）放到 VTL1 隔离环境

**⚙️ 驱动开发**

- WDM/WDF 驱动调用 `Zw*` 发起内核态系统调用，是 2.6 节内容的直接应用
- 现代驱动尽量避免 `Zw*` 直调，改用 WDF 框架提供的对象方法，减少直接进入 `KiSystemService` 的路径

---

## 给你的读书建议

1. **把 ReactOS 源码 clone 下来**，对照书里的函数名去 `ntoskrnl/` 目录找实现。比如读到 2.2 节，就去 `ntoskrnl/traphdlr.c` 看 `KiSystemServiceHandler` 的真实代码
2. **装一个 WinDbg + Windows 10/11 虚拟机**，用 `uf nt!KiSystemCall64` 反汇编现代 Windows 的系统调用入口，和书里 `KiSystemService` 对比——你会直观看到"哪些变了、哪些没变"
3. **关注 syscall ID 的版本差异**：同一个 `NtCreateFile`，Win10 和 Win11 的系统调用号可能不同，这是逆向工程里的常识
4. **下一章（内存管理）是全书最难的一章**，但只要你抓住了"系统调用是入口、SSDT 是分发、PreviousMode 是信任边界"这条主线，再去看 `NtAllocateVirtualMemory` 就会轻松很多

书是 2009 年写的，基于 x86 和 ReactOS 0.3.3，但它的**设计思想完全不过时**——过时的只是某些偏移量、某些 MSR 编号、某些结构体字段布局。抓住第一性原理（特权级隔离 + 受控入口 + 间接分发），你就能穿透任何版本的 Windows 内核。
