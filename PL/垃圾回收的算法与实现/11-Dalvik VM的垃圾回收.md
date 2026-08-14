
# 《垃圾回收的算法与实现》读书笔记 · 第11章 Dalvik VM 的垃圾回收

> 延续前九章建立的"第一性原理"视角，以及第10章 Python GC 的工业预热：如果说 Python 是"引用计数 + 分代循环 GC"的混合典范，那么 Dalvik VM（Android 的运行时）则是**"并发标记-清除 + 位图标记 + 卡片表写屏障"**​ 的工业典范。它把本书第2章（标记-清除）、第5章（位图标记）、第8章（增量/并发 GC + 写屏障）的理论，在**移动端嵌入式设备的严苛约束下**（内存小、CPU 弱、实时性要求高、电池敏感）做到了极致工程化。本章值得我们逐层拆开——因为它展示了"理论算法"如何被"工程约束"重塑。

---

## 一、11.1 本章前言：Dalvik VM 的工程约束

Dalvik VM 是 Android 4.x 及更早版本的运行时（Android 5.0 起被 ART 取代）。它的 GC 设计面临一组独特的工程约束：

- **内存受限**：早期 Android 设备 RAM 仅 256~512MB，堆上限通常被限制在 16~48MB
    
- **CPU 弱**：ARM 处理器主频低、缓存小
    
- **实时性要求**：UI 线程必须 16ms 内完成一帧渲染，GC 停顿直接导致掉帧、卡顿
    
- **单线程 GC**：Dalvik 的 GC 是单线程的，回收期间必须暂停所有其他线程（Stop-The-World）
    

> 💡 **第一性原理洞察**：Dalvik GC 的设计，本质上是在"单线程 STW 的硬约束"下，尽力把标记-清除算法改造成"并发化"形态。它不能像 JVM 那样用多线程并行标记来压缩 STW 时间（CPU 核数少、内存带宽小），所以选择了另一条路——**把标记阶段拆解为"短 STW 根标记 + 长并发标记 + 短 STW 重标记"**，用 Card Table 记录并发期间的引用变更。这是"工程约束塑造算法形态"的典型案例。

---

## 二、11.2 重新学习 mmap：Dalvik 堆的底层机制

Dalvik 的堆不是用普通 `malloc` 分配的，而是通过 `mmap()` 系统调用直接向 OS 申请大块匿名内存页 。

**为什么用 mmap 而非 malloc**：

- **精确控制**：mmap 可以申请一块连续的虚拟地址空间，便于位图管理和边界检查
    
- **延迟分配**：mmap 申请的页是"惰性"的——只有真正写入时才分配物理页，避免预分配浪费
    
- **可与 OS 协作**：当系统内存紧张时，OS 可以通过 `munmap` 或 `madvise` 回收 Dalvik 堆的空闲页
    

Dalvik 将 mmap 而来的堆划分为两个逻辑堆 ：

- **Zygote 堆**：Zygote 进程预加载的类和对象（所有 App 进程 fork 自 Zygote，共享这部分内存）
    
- **Active 堆**：App 自己运行时分配的对象
    

**GC 只回收 Active 堆**——Zygote 堆在所有 App 间共享，不能被单个 App 的 GC 回收 。这个设计深刻影响了 Dalvik GC 的算法结构——**Card Table 的核心职责，就是记录 Zygote 堆中的对象对 Active 堆中对象的引用变更**。

> 💡 **洞见**：Zygote 堆 + Active 堆的划分，本质上是 Dalvik 版的"分代"——Zygote 堆类比老年代（长寿命、共享），Active 堆类比新生代（短寿命、私有）。但 Dalvik 没有像 JVM 那样做复制算法，而是用 Card Table 处理跨堆引用。这是嵌入式设备约束下的务实选择。

---

## 三、11.3 Dalvik VM 的源代码：关键数据结构

Dalvik GC 的正确性建立在四个核心数据结构之上 ：

|数据结构|作用|
|---|---|
|**Live Bitmap**​|记录上一次 GC 时存活的对象|
|**Mark Bitmap**​|记录本次 GC 时存活的对象|
|**Mark Stack**​|标记阶段使用的栈，用于 DFS 遍历对象图|
|**Card Table**​|记录并发标记期间被修改的对象（写屏障的目标）|

**Live Bitmap 与 Mark Bitmap 的配合**：

- 当 `Live Bitmap = 1` 且 `Mark Bitmap = 0` 时，该对象需要被回收
    
- 当 `Live Bitmap = 1` 且 `Mark Bitmap = 1` 时，该对象存活
    
- 当 `Live Bitmap = 0` 且 `Mark Bitmap = 1` 时，该对象是新增的存活对象
    
- 当 `Live Bitmap = 0` 且 `Mark Bitmap = 0` 时，该对象是不可达垃圾（本次 GC 前已死）
    

**GC 完成后**：Mark Bitmap 的内容会复制为下一次 GC 的 Live Bitmap 。

> 💡 **洞见**：Live Bitmap + Mark Bitmap 的双位图设计，是本书第2章"位图标记"思想的工业级进化。它让 Dalvik 的清除阶段变得极简——**不需要再扫描整个堆来查找垃圾**，只需对比两张位图：Live 中为 1 而 Mark 中为 0 的对象，就是本次要回收的垃圾。清除阶段的复杂度从 O(堆大小) 降到 O(垃圾数量)，这是 Dalvik 能在移动端跑通标记-清除的关键优化。

---

## 四、11.4~11.5 Dalvik VM 的 GC 算法与对象管理

### GC 触发时机

Dalvik 的 GC 由以下四种情况触发 ：

|触发原因|含义|STW 特征|
|---|---|---|
|**GC_CONCURRENT**​|Active 堆分配量达到并发 GC 阈值|并发标记-清除，短时 STW|
|**GC_FOR_MALLOC**​|堆内存已满，分配失败|全程 STW，长停顿|
|**GC_HPROF_DUMP_HEAP**​|创建 HPROF 堆转储文件|特殊场景|
|**GC_EXPLICIT**​|应用主动调用 `System.gc()`|取决于实现|

### 堆参数调优

Dalvik 堆的行为由三个关键参数控制 ：

- **heapTargetUtilization**：堆目标利用率（默认 0.5）
    
- **heapMinFree**：堆最小空闲空间
    
- **heapMaxFree**：堆最大空闲空间
    

当 `Target` 超过 `[heapMinFree, heapMaxFree]` 区间时，GC 会选择 `[bytesAllocated + heapMaxFree]` 作为堆的最大可用内存，并减去 200KB~500KB 作为 GC_CONCURRENT 的触发阈值。

**调优规律**​ ：

- `heapTargetUtilization / heapMinFree / heapMaxFree` 越小 → GC_CONCURRENT 触发越频繁，单次 GC 时间短，卡顿少
    
- 反之 → GC 触发频率低，但单次 GC 时间长，前台 Activity 卡顿明显
    

> 💡 **洞见**：Dalvik 的堆调优参数，本质是在"GC 频率"和"单次 GC 停顿"之间做权衡——这正是本书第1章提出的"GC 评价标准不可能三角"在移动端的具象化。嵌入式设备的特殊性在于：**卡顿（UI 掉帧）比吞吐量下降更不可接受**，所以 Dalvik 默认偏向"高频、短停顿"的策略。

---

## 五、11.6 标记阶段：并发化的三步走

这是本章最精彩的部分。Dalvik 把标记阶段拆解为三个子阶段，以兼顾"正确性"与"短停顿" ：

### 子阶段一：根集标记（短 STW）

```
STW 开始
  → 暂停所有非 GC 线程
  → 从 GC Roots（全局变量、栈变量、寄存器）出发
  → 标记所有根集对象到 Mark Bitmap
  → 将这些对象压入 Mark Stack
STW 结束
```

这个阶段必须 STW，因为**根集是并发标记的起点，如果在标记根集的同时 mutator 修改了根集，会导致漏标**​ 。但根集通常很小（几十到几百个对象），所以这个 STW 非常短暂。

### 子阶段二：并发标记（无 STW）

```
GC 线程与 mutator 并发运行
  → GC 线程从 Mark Stack 弹出对象，DFS 遍历对象图
  → 每遇到一个未标记对象，标记它并压栈
  → mutator 线程继续运行，分配新对象、修改引用
  → 每次 mutator 修改堆中对象的引用时，写屏障触发
  → 写屏障将对应 Card 标记为 DIRTY
```

**写屏障的实现**​ ：

Dalvik 的 Card 大小为 `GC_CARD_SIZE = 128` 字节（即 `1 << 7`） 。Card Table 中每个 Card 对应一个字节，值为 `GC_CARD_CLEAN` 或 `GC_CARD_DIRTY`。

当执行 `iput-object` 这类写入对象引用的指令时，会插入写屏障调用 `dvmSetFieldObject` → `dvmMarkCard`，将该引用所在的 Card 置为 DIRTY 。

```
// dvmSetFieldObject 中的写屏障逻辑
void dvmSetFieldObject(Object* obj, Field* field, Object* value) {
    *field = value;
    if (value != NULL) {
        dvmMarkCard(value);  // 写屏障：将 value 所在 Card 置 DIRTY
    }
}
```

### 子阶段三：重标记（短 STW）

```
STW 开始
  → 再次暂停所有非 GC 线程
  → 扫描 Card Table 中所有 DIRTY 的 Card
  → 对这些 Card 中的对象重新进行标记（Remark）
  → 因为 DIRTY Card 数量很少（并发期间被修改的对象有限）
  → 所以这次 STW 非常短暂
STW 结束
```

> 💡 **第一性原理洞察**：Dalvik 标记阶段的三步走，是第8章"增量式 GC + 写屏障"理论在嵌入式设备上的完美落地：
> 
> - **子阶段一**​ ≈ 初始快照（类似汤浅算法的 SATB 起点）
>     
> - **子阶段二**​ ≈ 并发标记 + 写屏障记录变更（类似 Dijkstra/Steele 写屏障）
>     
> - **子阶段三**​ ≈ Remark（类似 CMS 的最终标记）
>     
> 
> 但 Dalvik 做了嵌入式设备专属的优化——**它没有选择在标记结束时重扫所有 mutator 根**（那是 Steele 类写屏障的代价），也没有选择完整的 SATB 快照（那是汤浅写屏障的代价），而是**用 Card Table 精确地记录"哪些 Card 被修改过"**，Remark 阶段只需要重扫这些 DIRTY Card。这是"精确记录"优于"全局重扫"的工程智慧——Card Table 的粗粒度（128 字节/Card）换来了写屏障的极低开销，而 DIRTY Card 的数量之少又保证了 Remark 的 STW 极短。

---

## 六、11.7 清除阶段：位图对比的极简回收

Dalvik 的清除阶段极其简洁——**不需要再扫描整个堆**​ ：

```
Sweep 开始（与 mutator 并发）
  → 遍历 Live Bitmap 和 Mark Bitmap
  → 对于满足 Live=1 且 Mark=0 的对象
  → 调用对象的 finalize（如有）
  → 将对象内存回收，链入 dlmalloc 的空闲链表
  → 在 Live Bitmap 中清除该位
Sweep 结束
  → 将 Mark Bitmap 复制为下一次 GC 的 Live Bitmap
```

**关键特征**：

- **并发清除**：清除阶段与 mutator 并发执行，不需要 STW
    
- **无碎片整理**：Dalvik 的清除只是把垃圾对象链入空闲链表，**不做压缩**——这是 Dalvik GC 最大的短板
    
- **浮动垃圾**：并发清除期间 mutator 新产生的垃圾，要等到下一次 GC 才能回收
    

**为什么不压缩？**

- 压缩需要移动对象 → 移动对象必须更新所有引用 → 在并发环境下需要复杂的写屏障保证正确性
    
- Dalvik 的 Card Table 是为"标记-清除"设计的，不支持"移动对象后的引用更新"
    
- 嵌入式设备 CPU 弱，压缩的计算成本过高
    
- **替代方案**：Dalvik 使用 `dlmalloc` 来管理空闲链表，通过 best-fit 策略尽量减少碎片
    

> ⚠️ **核心洞见**：Dalvik 选择"标记-清除 + 不压缩"，是嵌入式设备约束下的**务实妥协**：
> 
> - 优势：清除阶段简单、并发、无 STW；实现复杂度低；CPU 开销小
>     
> - 代价：堆碎片严重；前台 App 的堆中出现大量无法利用的小碎片；新对象需要大块连续内存时分配失败
>     
> 
> 这就是为什么 Android 开发有一条铁律——**"尽可能不要分配内存"**。因为 Dalvik GC 不压缩，碎片累积到一定程度后，即使总空闲内存充足，连续分配也会失败，触发 `GC_FOR_MALLOC`（长 STW）。这是 Dalvik 时代 Android 卡顿的根本原因之一。

---

## 七、11.8 Q&A：Dalvik GC 的工程权衡总结

**Q1：为什么 Dalvik 选择标记-清除而非复制算法？**

- 复制算法需要预留 To 空间（堆利用率 50%），在内存受限的移动设备上不可接受
    
- 复制算法需要移动对象，在并发环境下需要复杂机制保证引用一致性
    
- 标记-清除不移动对象，与 Dalvik 的"Zygote 堆共享"架构天然兼容
    

**Q2：为什么 Dalvik 的 GC 是单线程的？**

- 早期移动设备 CPU 核数少（单核或双核），多线程 GC 的同步开销可能超过并行收益
    
- 单线程 GC 实现简单，易于在资源受限环境中保证正确性
    

**Q3：Card Table 为什么是 128 字节/Card？**

- Card 太小 → Card Table 体积大，写屏障查表开销高
    
- Card 太大 → DIRTY Card 粒度粗，Remark 阶段需要重扫的对象多，STW 长
    
- 128 字节是经验最优值——在"写屏障开销"与"Remark STW 时间"之间取得平衡
    

**Q4：为什么 Dalvik 不做分代 GC？**

- Dalvik 的 Zygote/Active 堆划分已经是一种粗粒度分代
    
- 但真正的分代 GC 需要复制算法（新生代）+ 写屏障维护记忆集
    
- 在 Dalvik 的工程约束下（单线程、不移动对象、CPU 弱），实现完整分代 GC 的代价过高
    
- **这个遗憾在 ART 中被弥补**——ART 引入了完整的分代并发复制 GC
    

---

## 八、章节脉络：Dalvik GC 与前九章理论的全映射

```
Dalvik VM GC 架构
│
├─ 堆管理（11.2~11.5）
│   ├─ mmap 申请大块匿名内存 ← OS 层接口
│   ├─ Zygote 堆 + Active 堆 ← 粗粒度"分代"
│   ├─ 四大数据结构：Live Bitmap / Mark Bitmap / Mark Stack / Card Table
│   └─ dlmalloc 管理空闲链表 ← 第2章"多个空闲链表"思想
│
├─ 标记阶段（11.6）—— 第8章增量式 GC 的工业落地
│   ├─ 子阶段一：根集标记（短 STW）← 初始快照
│   ├─ 子阶段二：并发标记 + Card Table 写屏障 ← Dijkstra/Steele 写屏障思想
│   └─ 子阶段三：重标记（短 STW）← Remark
│
└─ 清除阶段（11.7）—— 第2章标记-清除 + 第5章位图标记
    ├─ Live Bitmap vs Mark Bitmap 对比 ← 位图标记的工业进化
    ├─ 并发清除，不压缩 ← 标记-清除的"就地回收"特性
    └─ 浮动垃圾 ← 并发 GC 的固有代价
```

**更深层的洞见**：Dalvik GC 是本书算法篇理论在**嵌入式设备约束下的"工程重塑"**：

|本书理论|在 Dalvik 中的体现|工程约束下的变形|
|---|---|---|
|第2章 标记-清除|主力算法|清除阶段不压缩（CPU 弱、不能移动对象）|
|第5章 位图标记|Live Bitmap + Mark Bitmap|双位图对比让清除极简|
|第8章 增量式 GC|标记三步走 + Card Table 写屏障|用 Card Table 精确记录变更，避免全局重扫|
|第7章 分代 GC|Zygote 堆 + Active 堆|粗粒度分代，未做完整分代复制|
|第4章 BiBOP|dlmalloc 的空闲链表管理|用 best-fit 策略缓解碎片|

**Dalvik GC 证明了：同样的算法理论，在不同的工程约束下会演化出完全不同的工业形态**。JVM 可以用多线程并行标记来压缩 STW，Dalvik 不行；JVM 可以用复制算法做新生代，Dalvik 不敢（堆利用率和对象移动的代价）；JVM 可以做分代 GC，Dalvik 只能做粗粒度分代。**工程约束不是算法的敌人，而是算法演化的驱动力**。

---

## 九、工业演进：从 Dalvik 到 ART 的 GC 革命

要真正理解 Dalvik GC 的历史地位，必须看它的继任者——ART（Android Runtime）。ART 从 Android 5.0 开始取代 Dalvik，其 GC 架构发生了质的飞跃 ：

### ART 的 GC 演进路线

|阶段|默认 GC|核心算法|关键特征|
|---|---|---|---|
|**Android 5.0~7.0**​|CMS（并发标记-清除）|标记-清除 + 并发|与 Dalvik 类似，但做了分代|
|**Android 8.0（Oreo）**​|**CC（并发复制）**​|复制算法 + 读屏障|默认方案变更为并发复制|
|**Android 10+**​|**Generational CC**​|分代 + 并发复制 + Region|CC 扩展为分代 GC|

### ART CC（Concurrent Copying）的核心创新

**读屏障（Read Barrier）取代写屏障**：

```
ART CC 的工作流程：
1. 初始标记（极短 STW）：枚举 GC Roots
2. 并发复制：
   - GC 线程将 From-Space 的存活对象复制到 To-Space
   - 应用线程恢复运行
   - 当应用线程读取一个尚未复制的对象引用时
   - 读屏障触发：立即由 GC 线程完成该对象的复制和引用更新
   - 然后应用线程继续（读到的是新地址）
3. 清理：整个 From-Space 直接回收（里面全是垃圾）
```

**ART CC 的优势**​ ：

- **GC 只有一次很短的暂停**，且该次暂停对于堆大小而言是**常量时间**
    
- **自动碎片整理**：复制过程中存活对象被紧凑排列，自然完成压缩
    
- **RegionTLAB 分配器**：每个线程有本地 TLAB，分配时只需触碰指针，无锁竞争
    
- **Android 10+ 分代化**：堆分为新生代 Region 和老年代 Region，Minor GC 只回收新生代
    

> 💡 **洞见**：ART CC 的演进，是本书算法篇理论在工业界的一次"集大成爆发"：
> 
> - **并发复制**​ = 第4章复制算法 + 第8章并发 GC + 读屏障
>     
> - **RegionTLAB**​ = 第5章 Immix 的 Block/Line 思想 + 第7章分代思想
>     
> - **分代 CC**​ = 第7章分代 GC + 第4章复制算法
>     
> - **Remember Set**​ = 第7章写屏障 + 记忆集的工业化身
>     
> 
> **Dalvik 是"受制于嵌入式约束的标记-清除"；ART CC 是"释放了约束的复制算法"**——随着移动设备 CPU 核数增多、内存增大，ART 终于能够采用本书第4章复制算法作为默认 GC，并通过读屏障实现真正并发的复制。这是 GC 算法理论与硬件演进共同进步的典型案例。

### Dalvik → ART 的传承与断裂

**传承**：

- Card Table 的思想演进为 Remember Set
    
- 位图标记的思想保留（Live Bitmap / Mark Bitmap）
    
- 分代的思想从 Zygote/Active 的粗粒度，演进为 Region 级的细粒度
    

**断裂**：

- 标记-清除 → 并发复制（算法内核根本改变）
    
- 写屏障 → 读屏障（屏障类型改变）
    
- 不压缩 → 自动压缩（碎片问题彻底解决）
    
- 单线程 → 多线程并发（硬件进步的红利）
    

---

## 十、工业实践印证与深度洞察

### 1. Dalvik GC 对 Android 应用开发的影响

Dalvik GC 的特性直接塑造了 Android 开发的"最佳实践"：

- **避免频繁分配**：因为 GC_FOR_MALLOC 会触发长 STW，所以 Android 开发强调"对象复用"（如 ViewHolder 模式、对象池）
    
- **避免在 onDraw 中分配**：16ms 渲染窗口内分配内存可能触发 GC_CONCURRENT，导致掉帧
    
- **谨慎使用 System.gc()**：主动 GC 会触发 STW，可能造成卡顿
    
- **注意大对象分配**：Dalvik 不压缩，大对象需要连续内存，碎片可能导致 OOM
    

### 2. 从 Dalvik 到 ART 的性能飞跃

实测数据对比 ：

|指标|Dalvik GC|ART CC（Android 8+）|
|---|---|---|
|GC 停顿|单次可达 10~50ms|常量级 < 1ms|
|堆碎片|严重|自动压缩，无碎片|
|分配速度|需加锁|TLAB 无锁分配|
|内存利用率|低（碎片 + 不压缩）|高（复制 + 压缩）|

这就是为什么 Android 5.0 升级到 ART 后，应用流畅度显著提升——**不仅是 AOT 编译的功劳，GC 从"标记-清除"跃迁到"并发复制"同样是关键**。

### 3. 与本书其他工业案例的横向对比

|运行时|主力算法|并发性|压缩|分代|写屏障类型|
|---|---|---|---|---|---|
|**CPython**​|引用计数 + 分代循环 GC|增量（3.14+）|否|是（3代）|引用计数增减|
|**Dalvik VM**​|并发标记-清除|标记/清除并发|否|粗粒度（Zygote/Active）|写屏障（Card Table）|
|**ART CC**​|并发复制|复制并发|是|是（Region 级）|读屏障|
|**JVM G1**​|分代 + 标记-整理|标记并发|是|是（Region 级）|写屏障（Card Table） + SATB|
|**Go**​|并发三色标记-清除|标记并发|否|否|混合写屏障|

**核心观察**：Dalvik VM 在本书的四个工业案例中处于**承上启下的位置**：

- **承上**：它是标记-清除 + 位图标记 + 写屏障思想的工业典范，与第2/5/8章理论直接对应
    
- **启下**：它的局限性（不压缩、单线程、无真正分代）直接催生了 ART 的革命性演进，而 ART CC 的设计思想与第4章复制算法、第5章 Immix、第7章分代 GC 深度呼应
    

---

## 十一、承上启下：Dalvik VM GC 在全书"实现篇"的位置

至此，我们分析了两个工业案例：

|案例|主力算法|核心特征|工程约束|
|---|---|---|---|
|**第10章 Python**​|引用计数 + 分代循环 GC|确定性回收、即时性|动态语言语义、资源敏感性|
|**第11章 Dalvik VM**​|并发标记-清除|短停顿、不压缩|嵌入式设备、CPU/RAM 受限|

**关键洞察**：两个案例展示了 GC 工业实现的两种根本不同的哲学：

- **Python**：以"引用计数"为主力，因为动态语言需要确定性回收
    
- **Dalvik VM**：以"追踪式 GC"为主力，因为移动端需要低延迟且无法承受引用计数的计数器开销
    

**而 Dalvik VM 到 ART 的演进，揭示了另一条规律**：**当硬件约束放松时，GC 算法会自然向"复制算法 + 分代 + 并发"这个现代 GC 的"黄金三角"收敛**。ART CC 的设计，与 JVM G1、Go 的并发 GC、ZGC/Shenandoah 的并发复制，本质上是同一方向的演进——**用复制算法解决碎片，用分代解决"统计规律"，用并发（读/写屏障）解决 STW**。

**带着这个视角进入后续章节**：

- **第12章 Rubinius**：Ruby 的 VM，同样是引用计数 + 追踪式 GC 的混合（与 Python 类似但不同）
    
- **第13章 V8**：JavaScript 引擎，分代式（Scavenge 复制 + Mark-Sweep-Compact）+ 增量 + 并发
    

你会发现：**所有现代语言运行时的 GC，都在向"分代 + 复制/压缩 + 并发"这个范式收敛**。Dalvik VM 作为"嵌入式约束下的标记-清除"代表，ART 作为"释放约束后的并发复制"代表，正好展示了这一收敛过程的两端。

---

📌 **本章核心收获**：

- Dalvik VM 的 GC 是"并发标记-清除"在嵌入式设备约束下的工业典范——用"短 STW 根标记 + 长并发标记 + 短 STW 重标记"三步走，把标记-清除并发化
    
- 四大核心数据结构：Live Bitmap + Mark Bitmap（双位图对比让清除极简，是"位图标记"思想的工业进化）、Mark Stack（DFS 遍历）、Card Table（写屏障目标，128 字节/Card）
    
- Zygote 堆 + Active 堆的划分是粗粒度"分代"，Card Table 专门记录 Zygote 堆对 Active 堆的引用变更
    
- 清除阶段与 mutator 并发，**但不压缩**——这是 Dalvik 最大的短板，导致碎片严重、前台 App 卡顿
    
- 堆调优三参数（heapTargetUtilization / heapMinFree / heapMaxFree）体现了"GC 频率 vs 单次停顿"的权衡，是"GC 不可能三角"在移动端的具象化
    
- **Dalvik → ART 的演进是 GC 算法理论的工业爆发**：ART CC 从 Android 8 开始成为默认，用**读屏障 + 并发复制**取代写屏障 + 标记-清除；Android 10+ 扩展为分代 CC；GC 停顿降到常量级 < 1ms，自动压缩解决碎片
    
- Dalvik GC 证明了"工程约束塑造算法形态"——同样的标记-清除理论，在嵌入式设备上演化出与 JVM 完全不同的工业形态
    
- 与 Python 形成对照：Python 以引用计数为主力（动态语言需要确定性回收），Dalvik 以追踪式 GC 为主力（移动端需要低延迟且无法承受引用计数的计数器开销）
    
- **所有现代 GC 都在向"分代 + 复制/压缩 + 并发"收敛**：Dalvik（标记-清除） → ART CC（并发复制）的演进，与 JVM G1、Go、ZGC/Shenandoah 的演进方向一致
    

