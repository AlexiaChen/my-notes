
## 开篇：第 16 章的主线——"Python 怎么管内存？为什么有时候内存降不下来？"

前面几章我们搞清楚了：

- **PVM 启动**时建立了"运行时-解释器-线程"三级状态 [第 13 章]
- **GIL**​ 让同一时刻只有一个线程能执行 Python 字节码 [第 15 章]
- **import**​ 是一套可扩展的查找-加载协议 [第 14 章]

但还有一个根本问题没回答：**每当你写 `x = []` 或 `obj = MyClass()` 时，这块内存在哪里、由谁分配、什么时候释放、为什么有时候 `del` 之后内存没降下来？**​ 第 16 章要揭开的，是 CPython "内存管理"的全景——它其实是一套**三层协作体系**：

- **底层**：pymalloc 小对象分配器（≤512 字节）
- **中层**：引用计数（实时、即时析构）
- **顶层**：分代循环垃圾收集器（GC，专门处理引用环）

**本章与书里的最大时差**：

- **书里（Python 2.5）**：内存管理就是"内存池（pymalloc）+ 引用计数 + 标记清除"。pymalloc 是唯一的专用分配器，三代 GC 默认阈值 (700, 10, 10)。
- **现代（3.14）**：
    - **3.13 起引入 mimalloc 分配器**，在 free-threaded build 中作为默认的 PYMEM_DOMAIN_MEM 和 PYMEM_DOMAIN_OBJ 分配器，且不能禁用；默认（带 GIL）构建中也可用 `PYTHONMALLOC=mimalloc` 启用
    - **3.13 起 GC 有两套实现**：默认构建用经典 `Python/gc.c`（分代、标记-清除）；`--disable-gil` 自由线程构建用 `Python/gc_free_threading.c`（并行实现）
    - **3.14 中 `threshold2` 曾被忽略，3.14.5 又恢复**——这说明 GC 的分代模型仍在演化
    - arena 大小：32 位 256 KiB，64 位 **1 MiB**（书里只提 256 KiB，那是 32 位平台的值）
    - pymalloc 仍然专为 **≤512 字节小对象**优化

**本章双轨阅读法**：

- **轨道一（机制，跨版本稳定）**：Python 内存 = 对象分配器（pymalloc/mimalloc）→ 引用计数（主要机制）→ 分代 GC（补充机制，专治循环引用）
- **轨道二（关键演化）**：
    - 分配器：pymalloc（默认 GIL 构建）→ mimalloc（3.13+ free-threaded 构建默认）
    - GC 实现：单一种类 → 两套实现（GIL 构建 vs 自由线程构建）
    - 分代模型：三代 (700,10,10) → 3.14 中二代模型（但 3.14.5 又调整回来）
    - 容器对象参与 GC 的前提：必须设置 `Py_TPFLAGS_HAVE_GC` 并提供 `tp_traverse`

---

## 16.1 内存管理架构

**书里（Python 2.5）的视角**：

- Python 的内存管理分为「内存池」和「垃圾回收」两部分
- 内存池处理小块内存分配（pymalloc），垃圾回收处理循环引用
- 整体是「引用计数为主，标记清除为辅」

**现代（3.14）的视角**：

CPython 的内存管理是一个**分层架构**，从底到顶依次是：

```
┌─────────────────────────────────────────────────────┐
│  Python 对象层 (PyObject_Malloc / PyObject_Free)    │  ← PYMEM_DOMAIN_OBJ
├─────────────────────────────────────────────────────┤
│  Python 内存域 (PyMem_Malloc / PyMem_Free)          │  ← PYMEM_DOMAIN_MEM
├─────────────────────────────────────────────────────┤
│  原始内存域 (PyMem_RawMalloc / PyMem_RawFree)        │  ← PYMEM_DOMAIN_RAW
├─────────────────────────────────────────────────────┤
│  系统分配器 (malloc / free / mmap / VirtualAlloc)    │
└─────────────────────────────────────────────────────┘
```

CPython 在系统 malloc 之上做了一层抽象，有三个"域"（domain）：

- **PYMEM_DOMAIN_RAW**：`PyMem_RawMalloc` 等，直接使用系统 malloc
- **PYMEM_DOMAIN_MEM**：`PyMem_Malloc` 等，使用 pymalloc（或 mimalloc）
- **PYMEM_DOMAIN_OBJ**：`PyObject_Malloc` 等，专为 Python 对象优化，使用 pymalloc（或 mimalloc）

**pymalloc 的定位**：

> _"Python has a pymalloc allocator optimized for small objects (smaller or equal to 512 bytes) with a short lifetime... It falls back to PyMem_RawMalloc() and PyMem_RawRealloc() for allocations larger than 512 bytes."_

也就是说：

- **≤512 字节**的小对象 → pymalloc 处理
- **>512 字节**的大对象 → 直接走系统 malloc
- pymalloc 是 PYMEM_DOMAIN_MEM 和 PYMEM_DOMAIN_OBJ 域的默认分配器

**mimalloc 的引入（3.13+）**：

> _"Unlike pymalloc, which is optimized for small objects (512 bytes or fewer), mimalloc handles allocations of any size. In the free-threaded build, mimalloc is the default and required allocator... It cannot be disabled in free-threaded builds."_

关键差异：

- **pymalloc**：专为 ≤512 字节小对象优化，大对象回落到 malloc
- **mimalloc**：通用分配器，处理任意大小；在 free-threaded build 中使用**每线程堆（per-thread heaps）**，大多数情况下分配/释放无需加锁
- 默认（带 GIL）构建中，mimalloc 可通过 `PYTHONMALLOC=mimalloc` 启用，但不是默认

**Python 堆的特性**：

> _"CPython has a specialized allocator that handles allocation of small objects and defers to the system allocator for large ones."_

Python 对象全部位于 Python 私有的堆（heap）中，与系统堆隔离。这意味着：

- Python 堆中的内存释放**不一定立即归还给操作系统**
- pymalloc 的 arena 只有在完全空闲时才会还给系统（这很少发生）
- 这就是为什么 Python 进程常驻内存高——释放的小对象内存留在 arena 中复用

**第一性原理**：

- **Python 内存管理 = "分层分配器 + 引用计数 + 分代 GC" 的三层协作**。分配器解决"高效分配/释放小对象"；引用计数解决"绝大多数对象的即时析构"；分代 GC 解决"引用计数搞不定的循环引用"。
- **为什么需要 pymalloc？**​ 因为 Python 程序会产生海量的、生命周期很短的小对象（整数、元组、小列表、函数调用栈帧等）。如果每次都调用系统 malloc/free，开销太大且会产生内存碎片。pymalloc 预先向系统申请大块内存（arena），然后在里面切分成小块（pool/block）自己管理——相当于"批发转零售"。
- **为什么 3.13+ 引入 mimalloc？**​ 因为 pymalloc 的并发性能差——它在分配时隐式依赖 GIL 做串行化。在 free-threaded build 中，pymalloc 无法高效地工作，所以引入了 mimalloc，它使用**每线程堆 + 原子操作**实现无锁分配，更适合多线程并行场景 。

**洞见——为什么 Python 进程的内存占用"只增不减"？**

```
import os, psutil, gc

def process_memory_mb():
    return psutil.Process(os.getpid()).memory_info().rss / 1024 / 1024

# 分配大量小对象
initial = process_memory_mb()
data = [list(range(100)) for _ in range(100000)]
after_alloc = process_memory_mb()

# 删除引用
del data
gc.collect()
after_free = process_memory_mb()

print(f"Initial: {initial:.1f} MB")
print(f"After alloc: {after_alloc:.1f} MB")
print(f"After free: {after_free:.1f} MB")
# 你会发现：after_free 可能仍然明显高于 initial
# 因为 pymalloc 的 arena 不会立即还给 OS，留着复用
```

这就是 Python 的"内存粘性"——pymalloc 释放的 block 会标记为"空闲"留在 pool 中，pool 空闲了留在 arena 中，**只有整个 arena 完全空闲时才会通过 munmap/malloc_free 还给系统**（这很少发生）。

**工业实际**：

```
import pymalloc  # 伪代码示意——实际上通过 PYTHONMALLOC 环境变量控制

# 1. 查看当前内存分配器
import sys
print(sys._allocator)  # 取决于 PYTHONMALLOC 设置

# 2. 切换分配器（通过环境变量）
# PYTHONMALLOC=malloc python script.py   # 完全禁用 pymalloc，用系统 malloc
# PYTHONMALLOC=mimalloc python script.py  # 使用 mimalloc（3.13+）
# PYTHONMALLOC=pymalloc python script.py   # 强制使用 pymalloc（默认）

# 3. 调试内存问题时的配置
# 使用 PYTHONMALLOC=debug 可以在每次分配/释放时检查内存越界
# 使用 PYTHONMALLOC=malloc + AddressSanitizer 可以检测 C 层内存错误

# 4. tracemalloc —— Python 层的内存追踪
import tracemalloc

tracemalloc.start()
# ... 执行一些代码 ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:10]:
    print(stat)
# 可以看到每一行代码分配了多少内存
```

---

## 16.2 小块空间的内存池

**这是 pymalloc 的核心机制**——三级层次结构。

**书里（Python 2.5）的描述**：

- **Arena**：256 KB 大块内存，从 OS 申请
- **Pool**：4 KB 页面，每个 pool 只分配一种大小
- **Block**：具体分配给对象的小块

**现代（3.14）的精确数据**（依据官方文档）：

> _"It uses memory mappings called 'arenas' with a fixed size of either 256 KiB on 32-bit platforms or 1 MiB on 64-bit platforms."_

也就是说：

- **32 位平台**：arena = **256 KiB**
- **64 位平台**：arena = **1 MiB**（书里说的 256 KB 是 32 位平台的值）

**完整的三级结构**：

**第一级：Arena（竞技场）**

- 从 OS 通过 `mmap()`/`VirtualAlloc()`/`malloc()` 申请的大块连续内存
- 64 位平台：1 MiB
- Arena 是内存池管理的**最大单位**
- 只有当 Arena 中所有 pool 都空闲时，Arena 才会还给 OS（很少发生）

**第二级：Pool（内存池）**

- 每个 Arena 被切分为多个 Pool
- 每个 Pool 固定 **4 KiB**（一页）
- **关键约束**：每个 Pool 只服务于**单一尺寸等级（size class）**——要么全是 16 字节的 block，要么全是 32 字节的 block，不能混用
- 这样设计的好处：没有内部碎片——因为所有 block 大小相同，任何释放的 block 都能立即服务下一次同尺寸的分配

**第三级：Block（块）**

- Pool 被切分为多个相同大小的 Block
- Block 大小遵循 **8 字节对齐**，从 8 到 512 字节共 **64 个尺寸等级**​

**尺寸等级映射表**（关键数据）：

|请求字节数|实际分配 Block 大小|尺寸等级索引|
|---|---|---|
|1-8|8|0|
|9-16|16|1|
|17-24|24|2|
|25-32|32|3|
|...|...|...|
|497-504|504|62|
|505-512|512|63|

数据来源：

**为什么是 8 字节对齐？**

- 64 位平台上指针是 8 字节，对齐到 8 字节边界有利于 CPU 访问
- 64 个尺寸等级 × 8 字节步长 = 完美覆盖 1-512 字节范围

**pymalloc 的速度优势**：

1. **无系统调用开销**：对象分配完全在已申请的 arena 内进行，不需要每次都调用 mmap/malloc
2. **无碎片**：每个 pool 内 block 大小相同，释放的 block 立即可用
3. **缓存友好**：相似大小的对象集中在同一 pool 中，提升 CPU 缓存命中率
4. **O(1) 分配**：找到对应尺寸等级的 pool，取一个空闲 block 即可

**内存释放的真相**：

> _"When the user code is done with a block, it is marked as unused. When all blocks in a pool have been marked as unused, the pool is marked as unused. Only when all pools in an arena have been marked as unused can the arena be returned to the system allocator. This rarely happens."_

也就是说：

- Block 释放 → 标记为未使用（**不归还 OS**）
- Pool 中所有 Block 都空闲 → Pool 标记为未使用（**不归还 OS**）
- Arena 中所有 Pool 都空闲 → Arena 归还 OS（**很少发生**）

这就是为什么 Python 进程的内存占用**具有粘性**——释放的内存留在 pymalloc 的 arena 中复用，不会立即还给系统。

**第一性原理**：

- **pymalloc 的本质是"批发转零售"**：它一次性从 OS 批发大块内存（arena），然后零售成小块（block）给 Python 对象使用。这避免了频繁的系统调用，是 Python 处理海量小对象的关键优化。
- **"每个 Pool 只服务单一尺寸等级"是设计的精髓**：它消除了内部碎片——因为任何释放的 block 都能立即满足下一次同尺寸的分配请求，无需搜索或合并。代价是可能有"外部碎片"（不同 pool 之间无法共享空间），但对于 Python 的工作负载（大量小对象、生命周期短），这个权衡是值得的。
- **512 字节的边界是精心选择的**：大于 512 字节的对象（如大列表、大字典、大字符串）生命周期往往较长，数量相对较少，直接用系统 malloc 更高效。小于等于 512 字节的对象（如小整数、短字符串、元组、栈帧）数量庞大且频繁创建销毁，必须用小对象分配器优化。

**洞见——为什么 `int` 和 `float` 不用 pymalloc？**

CPython 源码中，整数和浮点数使用了**专属的空闲列表（free list）**机制：

```
// 整数对象分配（简化示意）
#define NSMALLPOSINTS 257
#define NSMALLNEGINTS 5
static PyLongObject* small_ints[NSMALLPOSINTS + NSMALLNEGINTS];  // 小整数对象池

// 浮点数对象分配
// 使用专属的 free list，避免 pymalloc 的 8 字节对齐浪费
// 因为 float 对象大小约 24 字节，但 pymalloc 会向上取整到 24 字节
// 而 float 专用的分配器直接用约 1KB 的块，自己管理
```

为什么要这样？

- **小整数（-5 到 256）是单例**：Python 预先创建这些整数对象并复用，根本不需要分配器
- **float 对象大小约 24 字节**：如果用 pymalloc，会分配到 24 字节的 block，但 float 专用分配器用约 1KB 的块自己管理，避免 pymalloc 的取整浪费

**工业实际**：

```
import sys, tracemalloc, os

# 1. 验证小对象的尺寸等级对齐
import pymalloc  # 伪代码——实际上通过内部 API 观察
def alloc_size(requested):
    """返回 pymalloc 实际分配的 block 大小"""
    # 1-8 → 8, 9-16 → 16, 17-24 → 24, ...
    size_class = (requested + 7) // 8  # 向上取整到 8 的倍数
    return size_class * 8

for req in [1, 8, 9, 16, 17, 24, 100, 512, 513]:
    if req <= 512:
        print(f"Request {req:3d} → Block {alloc_size(req):3d} bytes")
    else:
        print(f"Request {req:3d} → System malloc (pymalloc bypass)")

# 2. 使用 tracemalloc 观察 pymalloc 行为
import tracemalloc

tracemalloc.start(1)  # 保留 1 帧的栈

# 分配大量小对象
data = [i for i in range(10000)]  # 小整数（已缓存，不分配）
big_data = [f"string_{i}" for i in range(10000)]  # 字符串对象

snapshot = tracemalloc.take_snapshot()
stats = snapshot.statistics('filename')
for stat in stats[:5]:
    print(f"{stat.count} objects, {stat.size/1024:.1f} KiB: {stat.filename}")

# 3. 感受 pymalloc 的"内存粘性"
import psutil
def rss_mb():
    return psutil.Process(os.getpid()).memory_info().rss / 1024 / 1024

before = rss_mb()
hold = [bytearray(256) for _ in range(100000)]  # 分配大量 256 字节对象
peak = rss_mb()
del hold
import gc; gc.collect()
after = rss_mb()
print(f"Before: {before:.1f} MB, Peak: {peak:.1f} MB, After free: {after:.1f} MB")
# After free 可能仍然接近 Peak——因为 pymalloc 保留了 arena

# 4. 禁用 pymalloc（通过 PYTHONMALLOC=malloc 启动）可以看到不同行为
# 这时所有分配都直接走系统 malloc，没有内存池优化
# 适合用 AddressSanitizer 调试 C 层内存错误
```

---

## 16.3 循环引用的垃圾收集

**这是 Python GC 最核心的职责**——解决引用计数无法处理的循环引用问题。

**书里（Python 2.5）的视角**：

- 引用计数无法处理循环引用（如 `a.ref = b; b.ref = a`）
- 需要一个专门的循环垃圾收集器
- 使用"标记-清除"（Mark-Sweep）算法

**现代（3.14）的精确机制**：

**问题定义**：

```
import sys
container = []
container.append(container)  # 创建循环引用
print(sys.getrefcount(container))  # 3: 变量 + 自引用 + getrefcount 临时引用
del container  # 变量引用删除，但对象内部仍有自引用
# 引用计数 = 1（来自自引用），永远不会降到 0，对象永远无法被引用计数回收！
```

**GC 的参与前提**（关键约束）：

> _"Types which do not store references to other objects, or which only store references to atomic types (such as numbers or strings), do not need to provide any explicit support for garbage collection."_

也就是说，**只有"容器类型"才参与 GC**：

- **参与 GC**：list、dict、set、自定义类的实例（因为它们可以引用其他对象）
- **不参与 GC**：int、float、str、bytes（原子类型，不引用其他对象）

**容器类型要参与 GC，必须满足**：

1. 类型对象的 `tp_flags` 中包含 `Py_TPFLAGS_HAVE_GC`
2. 提供 `tp_traverse` 函数——遍历对象引用的所有其他容器对象
3. 如果是可变容器，还需提供 `tp_clear` 函数——打破内部引用

**CPython GC 算法（现代描述）**：

CPython 的循环 GC **不是经典的标记-清除**，而是**基于引用计数减法的变体**（有时称为 "trial deletion"）：

**Phase 1：初始标记（快照引用计数）**

- 复制每个被跟踪容器的引用计数到临时字段 `gc_ref`
- 对于集合中的每个对象，递减它引用的其他对象的 `gc_ref`
- 这相当于"减去集合内部的引用"——只保留"来自集合外部的引用"

```
# 概念示意
# 假设有循环：A ↔ B（互相引用）
# A 的引用计数 = 2（外部变量 + B 的引用）
# B 的引用计数 = 2（外部变量 + A 的引用）

# 复制引用计数
A.gc_ref = 2
B.gc_ref = 2

# 减去内部引用
A.gc_ref -= 1  # B 引用了 A，所以 A 的 gc_ref 减 1
B.gc_ref -= 1  # A 引用了 B，所以 B 的 gc_ref 减 1

# 结果：A.gc_ref = 1, B.gc_ref = 1
# 仍然 > 0，说明 A 和 B 都有来自外部的引用（通过外部变量）
```

**Phase 2：传播标记（找可达对象）**

- `gc_ref > 0` 的对象是"从外部可达的根"
- 从这些根开始，通过 `tp_traverse` 遍历所有引用的对象
- 被遍历到的对象标记为"可达"

**Phase 3：收集不可达对象**

- 仍然标记为"暂不可达"的对象 → 真正不可达 → 可以被回收
- 调用 `tp_finalize`（PEP 442 安全终结）处理有 `__del__` 方法的对象
- 如果 `tp_finalize` 复活了对象（在 `__del__` 中重新建立引用），对象回到存活集
- 调用 `tp_clear` 打破循环引用，触发引用计数递减
- 引用计数归零 → 对象被真正释放

**3.13+ 的两套 GC 实现**：

> _"Since 3.13, CPython has two implementations of the cycle GC, selected at build time: Default (with GIL): Python/gc.c — The classic, generational, mark-and-sweep cycle collector. --disable-gil: Python/gc_free_threading.c — A separate implementation tuned for free-threaded execution."_

**自由线程构建的 GC 特点**：

- 收集时需要"stop-the-world"——所有线程暂停
- 但每个线程通过**每线程工作队列**参与标记
- 引用计数使用偏向技术——每个对象由一个线程"拥有"，跨线程的 INCREF/DECREF 使用原子操作
- 与 QSBR（Quiescent-State Based Reclamation）协作安全回收内存

**PEP 442：安全终结（Python 3.4+）**：

在 Python 3.4 之前，带有 `__del__` 方法的循环引用对象被认为是"不可收集的"——GC 无法确定安全的终结顺序，只能放到 `gc.garbage` 中让用户手动处理。

3.4+ 通过 PEP 442 引入了安全的终结排序：

- 对引用图进行拓扑排序，确定终结顺序
- 如果不存在安全的顺序，仍然收集对象，但 `__del__` 的调用顺序可能任意
- 如果 `__del__` 复活了对象，重新运行循环检测

**第一性原理**：

- **GC 的唯一职责是处理循环引用**。所有非循环的垃圾都由引用计数即时处理。GC 不参与普通对象的析构，它只在引用计数无法解决问题时才出手。
- **"基于引用计数减法的 trial deletion"是 CPython GC 的核心创新**。它不是从根对象开始标记存活对象，而是反过来——先假设所有对象都不可达，然后通过"减去内部引用"找出真正有外部引用的根，再从这些根标记可达对象。剩下的就是垃圾。
- **为什么只有容器对象参与 GC？**​ 因为循环引用**只能发生在容器对象之间**——只有容器才能"持有"对其他对象的引用。原子类型（int/float/str）不引用其他对象，不可能形成循环。这是 GC 的性能优化——它不必扫描所有对象，只需扫描容器。

**洞见——为什么 `gc.garbage` 列表会有对象？**

```
import gc

class Node:
    def __init__(self, name):
        self.name = name
        self.parent = None
        self.children = []
    
    def __del__(self):
        print(f"Finalizing {self.name}")
        # 如果在 __del__ 中重新建立引用，对象会被复活
        # global _revived
        # _revived = self  # 这会复活对象

# 创建循环引用
parent = Node("parent")
child = Node("child")
parent.children.append(child)
child.parent = parent

# 删除外部引用
del parent, child

# 强制 GC
collected = gc.collect()
print(f"Collected {collected} objects")

# 查看是否有不可收集的对象
if gc.garbage:
    print(f"Uncollectable: {len(gc.garbage)}")
    # 在 Python 3.4+ 中，这种情况很少见
    # 因为 PEP 442 引入了安全的终结排序
```

**工业实际**：

```
import gc, weakref

# 1. 创建循环引用并观察 GC 行为
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.parent = None
        self.children = []
    
    def add_child(self, child):
        self.children.append(child)
        child.parent = self  # 创建反向引用 → 潜在的循环引用
    
    def __del__(self):
        print(f"TreeNode {self.value} finalized")

# 构建树
root = TreeNode(1)
child1 = TreeNode(2)
child2 = TreeNode(3)
root.add_child(child1)
root.add_child(child2)

# 删除根节点——但 child1 和 child2 仍然通过 parent 引用 root
del root

# 强制 GC
print(f"Collected: {gc.collect()} objects")  # 应该收集 3 个 TreeNode

# 2. 使用弱引用打破循环
class BetterNode:
    def __init__(self, value):
        self.value = value
        self.parent = None  # 强引用
        self._children = []
    
    def add_child(self, child):
        self._children.append(child)
        child.parent = weakref.ref(self)  # 使用弱引用！不会创建循环

# 这样 parent 对 child 是强引用，child 对 parent 是弱引用
# 删除外部对 parent 的引用后，整个结构可以被引用计数回收

# 3. 调试循环引用问题
import gc

def find_cycles():
    """找出当前所有不可达但有引用的对象"""
    gc.collect()
    for obj in gc.garbage:
        print(f"Uncollectable: {type(obj)}")
        # 进一步分析...

# 4. 监控 GC 统计
import gc
stats = gc.get_stats()
for gen, stat in enumerate(stats):
    print(f"Generation {gen}:")
    print(f"  Collections: {stat['collections']}")
    print(f"  Collected: {stat['collected']}")
    print(f"  Uncollectable: {stat['uncollectable']}")

# 5. 3.13+ 自由线程构建的 GC 注意事项
import sys
if hasattr(sys, '_is_gil_enabled') and not sys._is_gil_enabled():
    print("Free-threaded build: GC uses stop-the-world with per-thread marking")
    print("GC pauses may be more noticeable in highly concurrent workloads")
```

---

## 16.4 Python 中的垃圾收集

**这是 GC 的"调度与调优"层面**——分代收集、触发条件、阈值调优。

**书里（Python 2.5）的描述**：

- 三代模型：年轻代（0 代）、中年代（1 代）、老年代（2 代）
- 默认阈值 (700, 10, 10)
- 新对象放入 0 代，0 代满 700 个对象触发回收；0 代回收 10 次后触发 1 代回收；1 代回收 10 次后触发 2 代回收

**现代（3.14）的精确机制**（依据官方文档）：

**三代模型（截至 3.14.5）**：

```
import gc
print(gc.get_threshold())  # (700, 10, 10) —— 默认值
```

**触发逻辑**：

1. **0 代触发**：自上次 0 代回收以来，分配的对象数减去释放的对象数**超过 threshold0（默认 700）**​ → 触发 0 代回收
2. **1 代触发**：0 代回收次数超过 threshold1（默认 10）次 → 下次 0 代回收时同时回收 1 代
3. **2 代触发**：1 代回收次数超过 threshold2（默认 10）次 → 触发完整回收（0+1+2 代全部扫描）

**3.14 中的重要变化**：

> _"Changed in version 3.14: generation=1 performs an increment of collection. Changed in version 3.14: threshold2 is ignored. Changed in version 3.14.5: threshold2 is restored to match Python 3.13 behavior."_

也就是说：

- **3.14.0 中**：`threshold2` 被忽略，GC 实际上是**两代模型**（0 代和 2 代）
- **3.14.5 中**：恢复到三代模型（threshold2 重新生效）

这反映了 CPython GC 分代模型仍在演化中。**查阅文档时应以你所使用的确切版本为准**。

**自由线程构建（3.13+）的特殊逻辑**：

> _"In the free-threaded build, the increase in process memory usage is also checked before running the collector. If the memory usage has not increased by 10% since the last collection and the net number of object allocations has not exceeded 40 times threshold0, the collection is not run."_

也就是说，在自由线程构建中，GC 触发不仅看分配计数，还看**内存增长**——如果内存没增长 10%，且分配数未超过 40×threshold0，就不运行 GC。这避免了在内存稳定的长期运行服务中不必要的 GC 暂停。

**GC 的 stop-the-world 特性**：

> _"The cyclic GC is stop-the-world: while it runs, the Python thread cannot execute any user code."_

GC 运行时，Python 线程被暂停。这就是为什么 GC 调优对延迟敏感的应用（如 Web 服务器）很重要。

**`gc.collect(generation)` 的精确语义**：

```
import gc
gc.collect(0)  # 只回收 0 代
gc.collect(1)  # 回收 0 代 + 1 代的增量
gc.collect(2)  # 完整回收（0+1+2 代）
gc.collect()   # 等同于 gc.collect(2)
```

**完整回收的副作用**：

> _"The free lists maintained for a number of built-in types are cleared whenever a full collection or collection of the highest generation (2) is run."_

也就是说，**完整 GC 会清空内置类型的空闲列表**——包括元组、列表、字典等的 free list。这可以进一步释放内存，但会增加后续对象分配的开销（需要重新填充 free list）。

**工业调优策略**：

**策略 1：延迟敏感型应用（Web 服务器、实时系统）**

```
import gc

# 大幅调高阈值，减少 GC 频率
gc.set_threshold(50000, 50, 50)

# 或者在安全时间点手动触发
def handle_request(request):
    # 处理请求...
    pass

def between_requests():
    """在请求间隙手动触发 GC"""
    gc.collect()
    # 或者使用增量回收
```

**策略 2：内存敏感型应用（数据处理、长期运行服务）**

```
import gc

# 降低阈值，更积极地回收内存
gc.set_threshold(500, 5, 5)

# 定期监控 GC 统计
import time
def monitor_gc():
    while True:
        stats = gc.get_stats()
        total_uncollectable = sum(s['uncollectable'] for s in stats)
        if total_uncollectable > 0:
            print(f"WARNING: {total_uncollectable} uncollectable objects!")
        time.sleep(60)
```

**策略 3：调试内存泄漏**

```
import gc, objgraph

# 启用 GC 调试
gc.set_debug(gc.DEBUG_LEAK)  # 包含 DEBUG_SAVEALL，垃圾对象保存在 gc.garbage 中

# 强制完整回收
collected = gc.collect()
print(f"Collected {collected} objects")

# 检查不可收集的对象
if gc.garbage:
    print(f"Found {len(gc.garbage)} uncollectable objects")
    for obj in gc.garbage:
        print(f"  {type(obj)}: {obj}")

# 使用 objgraph 找出对象引用链
import objgraph
objgraph.show_backrefs(gc.garbage[:1], filename='backrefs.png')
# 可视化不可达对象的引用链，定位内存泄漏根源
```

**第一性原理**：

- **分代 GC 基于"弱分代假说"**：大多数对象生命周期很短（如函数局部变量、临时列表），少数对象长期存活（如配置、缓存）。因此，频繁扫描年轻代、少扫描老年代，可以最大化 GC 效率。
- **阈值是"经验最优值"而非"理论最优值"**。(700, 10, 10) 是 CPython 团队经过大量基准测试得出的默认值，但对特定应用可能需要调优：
    - 延迟敏感 → 提高阈值（减少 GC 频率）
    - 内存敏感 → 降低阈值（更积极回收）
- **GC 是 stop-the-world 的**——这意味着每次 GC 运行都会导致 Python 线程暂停。在延迟敏感的应用中，需要在"GC 频率"和"单次 GC 停顿时间"之间权衡。

**洞见——为什么有时候 `gc.collect()` 后内存没降下来？**

```
import gc, psutil, os

def rss():
    return psutil.Process(os.getpid()).memory_info().rss / 1024 / 1024

before = rss()

# 创建大量循环引用
cycles = []
for i in range(10000):
    a = []
    b = []
    a.append(b)
    b.append(a)
    cycles.append((a, b))

mid = rss()
print(f"After allocation: {mid:.1f} MB")

# 删除外部引用
del cycles
gc.collect()
after = rss()
print(f"After gc.collect(): {after:.1f} MB")

# 可能 after 仍然接近 mid
# 原因：
# 1. 循环引用被 GC 回收了（对象级别）
# 2. 但 pymalloc 的 arena 不会立即还给 OS（分配器级别）
# 3. 内置类型的 free list 可能仍持有内存
```

**真正释放内存的做法**：

```
import gc, psutil, os

# 1. 完整 GC（包括最高代）
gc.collect(2)  # 这会清空内置类型的 free list

# 2. 如果使用 PyPy 或其他实现，行为可能不同
# 3. 在极端情况下，可以使用 malloc trimming（通过 ctypes 调用）
import ctypes
def trim_memory():
    """尝试将内存归还给 OS（仅限 glibc）"""
    try:
        libc = ctypes.CDLL('libc.so.6')
        libc.malloc_trim(0)
    except:
        pass
```

**工业实际**：

```
import gc
import psutil
import os

# 1. 长期运行服务的 GC 调优
class MemoryAwareService:
    def __init__(self):
        # 根据内存压力动态调整 GC 阈值
        self.base_threshold = (700, 10, 10)
        gc.set_threshold(*self.base_threshold)
    
    def adjust_gc_for_memory_pressure(self):
        """根据当前内存使用情况调整 GC 策略"""
        process = psutil.Process(os.getpid())
        mem_percent = process.memory_percent()
        
        if mem_percent > 80:
            # 内存压力大：更激进的回收
            gc.set_threshold(350, 5, 5)
            gc.collect(2)  # 立即完整回收
        elif mem_percent < 40:
            # 内存充裕：减少 GC 频率
            gc.set_threshold(2000, 20, 20)
        # 否则保持默认
    
    def handle_request(self, request):
        # 处理请求
        self.adjust_gc_for_memory_pressure()
        # ... 业务逻辑 ...

# 2. Web 服务器（如 uWSGI、Gunicorn）的 GC 策略
# 通常在 worker 进程 fork 后：
import gc
gc.set_threshold(10000, 50, 50)  # 减少 GC 频率
# 在每个请求处理后：
# gc.collect()  # 手动触发，避免不可预测的停顿

# 3. 数据处理管道的 GC 策略
def process_large_dataset(data):
    """处理大型数据集时，避免 GC 在关键时刻运行"""
    # 禁用自动 GC
    gc.disable()
    try:
        for batch in data:
            result = transform(batch)
            yield result
            # 在批次间隙手动触发
            if len(batch) % 1000 == 0:
                gc.collect(0)  # 只回收年轻代
    finally:
        gc.enable()
        gc.collect(2)  # 最终完整回收

# 4. 监控 GC 性能
import gc
import time

class GCMonitor:
    def __init__(self):
        self.last_collection_time = time.time()
        self.collection_count = 0
    
    def monitor(self):
        while True:
            stats = gc.get_stats()
            current_time = time.time()
            
            for gen in range(3):
                if stats[gen]['collections'] > self.collection_count:
                    # 检测到了 GC 事件
                    pause_time = current_time - self.last_collection_time
                    print(f"GC pause: {pause_time*1000:.2f} ms (gen {gen})")
                    self.collection_count = stats[gen]['collections']
            
            self.last_collection_time = current_time
            time.sleep(1)

# 5. 3.13+ 自由线程构建的特殊考虑
import sys
if hasattr(sys, '_is_gil_enabled') and not sys._is_gil_enabled():
    print("Free-threaded build considerations:")
    print("- GC uses stop-the-world with per-thread marking")
    print("- Memory growth also triggers GC (10% threshold)")
    print("- Consider larger threshold0 to reduce GC frequency")
    gc.set_threshold(5000, 50, 50)  # 更保守的 GC 策略
```

---

## 第 16 章合体总结：Python 内存管理的全景图

|层次|书里（Python 2.5）|现代（Python 3.14）|
|---|---|---|
|**分配器架构**​|pymalloc 是唯一专用分配器|三层域：RAW/MEM/OBJ；pymalloc 或 mimalloc（3.13+）|
|**小对象边界**​|≤ 512 字节|≤ 512 字节（不变），arena 大小 32 位 256KiB/64 位 1MiB|
|**mimalloc**​|不存在|3.13+ 引入，free-threaded build 中必需且不可禁用|
|**引用计数**​|主要机制|仍然是主要机制，确定性析构|
|**循环 GC**​|经典标记-清除|基于引用计数减法的 trial deletion；3.13+ 两套实现|
|**分代模型**​|三代 (700, 10, 10)|3.14.0 中 threshold2 被忽略（两代），3.14.5 恢复三代|
|**自由线程 GC**​|不存在|3.13+ 使用 stop-the-world + 每线程标记|
|**GC 触发**​|基于分配计数|默认构建基于分配计数；自由线程构建还检查内存增长 10%|
|**PEP 442**​|不存在|3.4+ 引入安全终结，解决 `__del__` 循环引用问题|
|**调试工具**​|基础 gc 模块|新增 tracemalloc、`gc.get_stats()`、`gc.DEBUG_LEAK` 等|

### 三个必须带走的洞见

**洞见一：Python 内存管理是"三层协作"，不是单一机制。**

- **底层**：pymalloc/mimalloc 解决"高效分配/释放小对象"
- **中层**：引用计数解决"绝大多数对象的即时析构"
- **顶层**：分代 GC 解决"引用计数搞不定的循环引用"
- 这三个层次各司其职，缺一不可。理解这一点，你就能理解为什么 Python 既能高效处理海量小对象，又能避免循环引用的内存泄漏。

**洞见二：pymalloc 的"内存粘性"是设计权衡，不是 bug。**

- pymalloc 释放的内存留在 arena 中复用，不会立即还给 OS
- 这是刻意为之——避免频繁的系统调用，提升分配速度
- 代价是 Python 进程的内存占用"只增不减"
- 在工业实践中，需要接受这种行为，或通过 `gc.collect(2)` + `malloc_trim` 主动释放

**洞见三：GC 调优的本质是"在 GC 频率和单次停顿之间权衡"。**

- 阈值过高 → GC 频率低，但单次回收的对象多，停顿时间长
- 阈值过低 → GC 频率高，单次停顿短，但 CPU 开销大
- 延迟敏感应用（Web 服务器）：提高阈值 + 在请求间隙手动 `gc.collect()`
- 内存敏感应用（长期服务）：降低阈值 + 监控 `gc.get_stats()`
- 3.13+ 自由线程构建：GC 还需要考虑"stop-the-world"对多线程的影响

