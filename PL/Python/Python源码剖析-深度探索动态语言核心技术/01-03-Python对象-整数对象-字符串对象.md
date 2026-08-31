
# 开篇：这三章的共同主线（先给你一个心智模型）

陈儒这本书剖析的是 **Python 2.5**。所以第 1–3 章里你会看到 `PyIntObject`、`PyStringObject`、`ob_sval`、`characters[256]` 这些在 Python 3 里已经改名或消失的东西。但请先记住一句总纲，它能救你免于被版本差异淹没：

> **这三章讲的是同一件事的三个实例：不可变性（immutability）→ 可共享性（shareability）→ 缓存/池化/intern/immortal。**

- 第 1 章建立"身份 / 类型 / 值"三元模型，并给出共享的技术前提——`PyObject` 统一头 + 引用计数；
- 第 2 章的整数是**不可变**的，所以可以预建小整数池、可以共享同一个对象；
- 第 3 章的字符串也是**不可变**的，所以可以 intern、可以做单字符缓冲池。

而现代 CPython 把这根链条又往前推了一步：**既然不可变，那就连引用计数一起省掉**——这就是 PEP 683 的 immortal objects。整条演进脉络是：`不可变 → 可共享 → 可缓存 → 可免同步`。抓住这条线，你就不是在读三章孤立的源码，而是在看同一个第一性原理被反复应用到极限。

继续沿用第 0 章的**双轨阅读法**：轨道一吃透 2.5 的最小模型（机制，跨版本稳定，投入 80% 精力）；轨道二用现代英文资料校验（策略，频繁变化，投入 20% 精力跟踪）。

---

# 第 1 章 Python 对象初探

## 1.1 Python 内的对象

### 书里在讲什么

开篇立论：**一切皆对象**。对象 = 数据 + 行为。CPython 里所有对象共享同一段最小头部：

```
typedef struct _object {
    Py_ssize_t  ob_refcnt;   // 引用计数
    PyTypeObject *ob_type;   // 类型指针
} PyObject;

#define PyObject_HEAD        PyObject ob_base;
#define PyObject_VAR_HEAD    PyVarObject ob_base;   // 多一个 ob_size
```

定长对象（int/float）嵌入 `PyObject_HEAD`；变长对象（str/list/tuple）嵌入 `PyObject_VAR_HEAD`，多出来的 `ob_size` 表示元素个数。

### 第一性原理：这是"运行时类型信息"的最小完备集

**原理 A：对象头 = 让任意 `PyObject*` 变成"自描述数据"的最小代价。**

只需要两个字段，运行时就能回答关于任意对象的两个根本问题：**谁还在用它**（`ob_refcnt`）和**它能干什么**（`ob_type`）。这是整个动态语言运行时最关键的架构权衡——**用 16 字节（64 位平台）的空间 + 一层指针间接，换取"无需静态类型信息也能安全操作任意值"的能力**。

> 洞见：CPython 里你几乎看不到 C++ 式的虚函数表。它选择把"类型信息"显式地做成**数据**（一个可被读写、可被替换的 `ob_type` 指针），而不是**编译期固定的代码布局**。这个选择带来两个后果：一是动态性（运行时可以改 `__class__`、可以 monkey patch），二是每次操作都要多一次访存。CPython 二十多年的优化史，本质上就是"如何减少这次访存的代价"——PEP 659 的 specialization 就是最典型的答案。

**原理 B：`id()` 就是地址——这是 CPython 的语言级泄漏。**

官方数据模型文档明确写着：对象的 identity 一经创建就不再改变，`is` 比较身份，`id()` 返回身份；而 CPython 实现细节里 `id(x)` 就是对象的内存地址。这意味着 Python 里的"身份"概念不是抽象的，它就是**指针相等**。理解这点，你才能理解为什么 `a is b` 有时为 `True` 有时为 `False`——它问的从来不是"值相等吗"，而是"是同一个指针吗"。

### 现代漂移（必须知道的补丁）

- **`Py_TYPE()` 在 3.11 起变成了 inline static 函数**，参数类型不再是 `const PyObject*`。这条变化看起来琐碎，实则是 CPython 为后续优化（比如 3.12 起的 immortal objects 位打包）清理出来的空间。
- **3.10 起新增 `Py_Is()` / `Py_IsNone()` / `Py_IsTrue()` / `Py_IsFalse()`**。这是把"身份比较"从宏提升为一等 API 的信号。
- **对象头在 64 位平台上做了位打包**：`ob_refcnt` 与若干 flag（immortality、溢出检测）挤在同一个 64 位整数里。也就是说，你在 2.5 里学到的"头就是 refcnt + type 两个字段"，在现代已经不再字面成立。

### 工业实际

理解 `PyObject` 头是**所有 Python 内存问题的起点**。`True` 在 C 里只需 1 字节，在 CPython 里占 28 字节；`0` 占 24 字节，`2**30` 占 32 字节。这不是"Python 浪费"，而是**统一对象模型的必然对价**。任何 Python 内存优化的第一步，都是先接受这个 24–28 字节的地板价，然后去优化_对象的数量_，而不是优化单个对象的大小。

---

## 1.2 类型对象

### 书里在讲什么

类型本身也是对象，其 C 结构是巨大的 `PyTypeObject`——一张**行为表（slot table）**。完整结构见官方文档，核心分几组：

|分组|关键 slot|作用|
|---|---|---|
|身份与尺寸|`tp_name`, `tp_basicsize`, `tp_itemsize`|类型名、实例大小、变长元素大小|
|协议套件|`tp_as_number`, `tp_as_sequence`, `tp_as_mapping`, `tp_as_buffer`|数字/序列/映射/缓冲四套协议|
|核心操作|`tp_new`, `tp_init`, `tp_dealloc`, `tp_free`, `tp_alloc`, `tp_hash`, `tp_call`, `tp_str`, `tp_repr`|构造、析构、哈希、调用|
|属性访问|`tp_getattro`, `tp_setattro`, `tp_descr_get`, `tp_descr_set`, `tp_dict`, `tp_dictoffset`|描述符协议|
|继承与方法解析|`tp_base`, `tp_bases`, `tp_mro`|单继承基类、基类元组、C3 线性化 MRO|
|GC 与遍历|`tp_traverse`, `tp_clear`, `tp_is_gc`|循环垃圾回收|
|缓存|`tp_version_tag`|方法缓存版本号，`PyType_Modified()` 时失效|

还有 `PyType_Ready()`——把一张"半成品"类型表补齐（继承 slot、算 MRO、初始化 `tp_dict`）的初始化函数。

### 第一性原理：类型 = 一张可被继承、可被修改的行为查找表

**原理 C：CPython 的"类"，在 C 层面就是一张函数指针表 + 一个字典。**

`tp_as_number` 里挂着 `nb_add`，于是 `a + b` 就变成"查 `a` 的类型 → 取 `nb_add` → 调用"。**行为的动态分派，被降级为一次结构体字段访问 + 一次函数指针调用。**

> 洞见（我自己的提法）：**`PyTypeObject` 是 C 语言对"面向对象"的一次手写实现，而且它是"表驱动"而非"继承驱动"的。**
> 
> C++ 的虚函数表是编译器生成、布局固定、不可运行时修改的；`PyTypeObject` 是**运行时可读写的普通内存**。这直接解释了 Python 与 C++ 的能力分野：C++ 的静态性换来零成本抽象，Python 的可变性换来 monkey patch、动态 `__class__`、运行时生成类。两者不是优劣，是**同一组需求在"编译期/运行期"两个方向上的不同投射**。

**原理 D：`PyType_Ready()` 的存在说明——类型定义是"声明式"的，类型的可用性是"计算出来"的。**

你写下 `tp_as_number = &long_as_number` 只是一份声明；MRO 要算、继承的 slot 要填、`tp_dict` 要建——这些都在 `PyType_Ready()` 里完成。这是典型的**"声明 + 求值"两阶段设计**，在你写 C 扩展时踩过的绝大多数坑，都源于在 `PyType_Ready()` 之前就去用这个类型。

### 现代漂移

- **静态类型 vs 堆类型（static types vs heap types）是现代必须分清的二分**：
    - **静态类型**：C 代码里写死的内建类型（`PyLong_Type`、`PyUnicode_Type`），静态分配，**通常被标记为 immortal**。
    - **堆类型**：运行时动态创建的类（`class Foo`），由 `PyHeapTypeObject` 承载。
    - 关键区别：静态类型的实例不算作对类型的引用，堆类型的实例**算**引用。这意味着堆类型存在"实例还活着 → 类型不能死"的循环，GC 必须介入。
- **方法缓存从"全局表"改成了"每类型缓存"**（现代主线，3.16 已完成这一改造）。这是个漂亮的演进：全局表需要全局 version tag 失效，每类型缓存则把失效范围缩小到单个类型及其子类型。`tp_version_tag` 的语义也随之细化——`MAX_VERSIONS_PER_CLASS` 限制了版本号的分配，自定义 MRO 会直接禁用缓存。

### 工业实际

写 C 扩展时，最实用的心智模型是：**`tp_new` 负责"造出对象"，`tp_init` 负责"填内容"，`tp_alloc` 负责"要内存"，`tp_dealloc` 负责"拆干净"`。四者职责混淆是 C 扩展内存泄漏的头号来源。

另外，`tp_flags` 与 `Py_TPFLAGS_*` 是你与解释器之间的"能力协商"通道——声明了 `Py_TPFLAGS_HAVE_GC` 就必须实现 `tp_traverse`，否则 GC 会漏扫或崩溃。

---

## 1.3 Python 对象的多态性

### 书里在讲什么

CPython 没有 C++ 的虚函数，多态靠的是：**所有对象都能转成 `PyObject*`，运行时通过 `ob_type` 找到对应行为**。同一个 `PyObject_Print(op)`，传入 int 和传入 str 会走到不同的 `tp_print`/`tp_repr`。

### 第一性原理：多态 = 查表，且这张表在运行时可读可改

**原理 E：动态语言的多态，本质是"把虚函数表从代码段搬到数据段"。**

C++ 的 vtable 在 `.rodata`，编译期定死；CPython 的 `ob_type` 指针在堆对象里，运行期可改。这个搬迁换来的是**反射、动态类型检查、运行时补丁**，付出的是**每次分派一次额外访存 + 无法内联**。

> 洞见：**现代 CPython 正在把"运行时多态"悄悄换回"编译期多态"——只不过编译发生在运行时。**
> 
> PEP 659 的 specialization 做的事情是：解释器观察到 `a + b` 里的 `a`、`b` 一直是 int，就把通用的 `BINARY_OP` 指令**就地替换**成 `BINARY_OP_ADD_INT`，后者直接内联整数加法，跳过查表。一旦类型变了，再退化（deopt）回通用指令。
> 
> 这就是我在第 0 章说的"闭环反馈"在多态层面的具体形态：**用"运行时的类型统计"把动态分派特化为静态分派**。第 1.3 节学到的"多态靠查表"，在现代已经被这条路径大幅优化掉了——但**语义模型没变**，变的是实现。这正是"机制稳定、策略多变"的完美例证。

### 工业实际

`isinstance()` 比 `type(x) == T` 慢还是快？现代 CPython 对 `Py_IS_TYPE()`（3.9 新增）这类"精确类型检查"有快速路径，而 `isinstance` 要遍历 MRO。在热路径上：

- **用 `type(x) is T` 做精确匹配**（对应 C 层 `Py_IS_TYPE`）；
- **用 `isinstance` 做继承匹配**，接受 MRO 遍历成本；
- **别用 `==` 比较类型**——它会走 `type.__eq__`，是三者里最慢的。

---

## 1.4 引用计数

### 书里在讲什么

`Py_INCREF(op)` / `Py_DECREF(op)` 一对宏；`ob_refcnt` 归零时调用类型对象的 `tp_dealloc` 析构。官方文档对 CPython 内存管理的表述是：**引用计数为主 +（可选的）循环垃圾延迟检测**——注意"可选"二字：分代 GC 只负责引用计数解决不了的**循环引用**。

### 第一性原理：引用计数 = 把"生命周期"化归为"局部可判定的计数问题"

**原理 F：引用计数的真正价值不是"自动回收"，而是"确定性析构"。**

追踪式 GC（标记-清除）只能保证"最终会回收"，回收时机不可预测；引用计数保证"最后一个引用消失的那一刻就析构"。这让 RAII 风格成为可能在 Python 里成立——`with` 语句、`close()`、文件句柄的及时释放，根基都在这里。

代价有三，这本书里讲了前两个，第三个是近五年才成为焦点：

1. **循环引用无解**​ → 必须外挂分代 GC；
2. **每次引用传递都要写内存**​ → 空间与时间开销；
3. **写放大与并发冲突**​ → 这是 free-threading 时代的核心难题。

> 洞见：**immortal objects（PEP 683）是"引用计数"这个概念的一次漂亮的自反。**
> 
> 它问的问题是：**如果有些对象注定永远活着，为什么还要给它记数？**
> 
> 答案：把 `ob_refcnt` 设成一个哨兵值（`_Py_IMMORTAL_REFCNT`），让 `Py_INCREF()` / `Py_DECREF()` / `Py_SET_REFCNT()` 对这些对象直接变成 no-op。
> 
> PEP 683 列举的动机正是原理 F 的代价清单：
> 
> - **缓存行失效**：每次改 `None` 的 refcnt，都是一次写、一次缓存行失效、一次跨核传播；
> - **数据竞争**：多核下两个线程同时 incref 同一个 `None`，会互相把对方的缓存刷掉——**即使有 GIL 也一样**（只是影响小些）；
> - **多解释器隔离**：若做 per-interpreter GIL，两个解释器连 `None` 都不能共享，除非 `None` 真正不可变。
> 
> 本质上：**immortality 把"引用计数"从"必须维护的不变量"降级为"可以豁免的优化"**。这是原理 F 的直接推论——既然这些对象生命周期是"永远"，那"局部计数"这件事对它们就是多余的。

### 现代漂移（本章变化最剧烈的一节）

**① immortal objects（3.12 起，PEP 683）**

- `None`、`True`、`False`、小整数、interned 字符串、静态 `PyTypeObject`、从 `.pyc` 加载的 code object 都被标记为 immortal。
- `Py_INCREF()` / `Py_DECREF()` 开头多了一句 `if (_Py_IsImmortal(op)) return;`。
- **3.14 新增 `sys._is_immortal(obj)`**​ 可以直接观测。在 3.16 上你会看到 `sys.getrefcount(1)` 返回一个天文数字（3221225472）——**这正说明它已经不再被计数**。
- **3.16 把静态 immortal 化推到了极致**：构建期就生成 **1030 个整数单例（范围 [-5, 1024]）、256 个 Unicode 单例（U+0000–U+00FF）、256 个 bytes 单例（b'\x00'–b'\xff'）、约 865 个静态 Unicode 字符串**。

**② biased reference counting（3.13+ free-threaded，PEP 703）**

对象头被彻底改写成：

```
struct _object {
    uintptr_t ob_tid;        // 拥有者线程 ID
    uint16_t  _padding;
    struct {
        uint32_t   local;    // 线程本地计数（非原子，快路径）
        Py_ssize_t shared;   // 共享计数（原子，慢路径）
    } ob_ref;
    PyTypeObject *ob_type;
};
```

- **拥有者线程**走 `local++`，非原子、无内存屏障，成本与 GIL 时代几乎相同；
- **其他线程**走 `shared`，用原子操作（约 3 倍慢）；
- `shared` 计数左移 1–2 位，低位用作状态标志：`0b00` default / `0b01` weakrefs / `0b10` queued / `0b11` merged，构成单向状态机；
- 真实负载中 **85–95% 的 refcount 操作命中快路径**。

这叫"biased（有偏）"：**赌大多数引用操作发生在拥有者线程上**。这是典型的"优化常见路径、兜住罕见路径"的工程哲学。

**③ deferred reference counting + stack references（3.16 正在开发）**

更进一步：连 incref/decref 都不调用。`LOAD_CONST` 不再 `Py_INCREF(value)`，而是用 `PyStackRef_FromPyObjectBorrow()` 打一个 tag 标记成"借用"，`POP_TOP` 用 `PyStackRef_XCLOSE()`——**因为当初就没 incref，所以现在也不必 decref**。

**④ 对象锁与 mimalloc**​ —— free-threaded 构建下每个对象头带一个 1 字节的 `_PyMutex`（无竞争时一次 CAS，竞争时退化为 futex）；内存分配器也从 pymalloc 换成 **mimalloc**（在 free-threaded 构建中是强制且不可关闭的）。

### 工业实际：一个教科书级的真实收益

**Gunicorn pre-fork 的 copy-on-write 问题。**​ 这是个能把原理讲透的例子：

- pre-fork 模型下，子进程通过 copy-on-write 与父进程共享内存页；
- 每次 `Py_INCREF(None)` 都是**一次写**——它会把 `None` 所在的整页变成子进程私有；
- `None` 几乎出现在每个函数里（默认返回值、`if x is None` 检查），所以**每个 worker 在启动后立刻就把包含这些单例的页全弄脏了**；
- 结果：本可共享的几百 KB 单例数据，每个 worker 各持一份私有副本。

**immortal objects 让这些页永远不被写脏**——3.12 之后 pre-fork 部署的内存占用实实在在下降了。这是"教科书级"的：**一个语言运行时的内部优化，直接变成了生产环境的内存账单优化。**

---

## 1.5 Python 对象的分类

### 书里在讲什么

书里按用途把对象分成几大类：**Fundamental（类型对象）、Numeric（数值）、Sequence（序列）、Mapping（映射）、Internal（虚拟机内部对象：code / frame / function / module 等）**。分类的意义是提供一个阅读地图——后面章节基本按这个分类逐个剖析。

### 第一性原理：分类维度不止一个，每个维度对应一类优化

**原理 G：书里的分类是"语义分类"，但对理解实现更有用的是三个"实现分类"：**

|维度|取值|为什么重要|
|---|---|---|
|**定长 vs 变长**​|`PyObject_HEAD` vs `PyObject_VAR_HEAD`|决定能否用 `PyObject_NewVar`；变长对象（str/list/tuple）的 `ob_size` 是长度|
|**静态类型 vs 堆类型**​|C 里写死 vs 运行时创建|决定是否 immortal、是否被 GC 追踪、实例是否构成对类型的引用|
|**是否 GC 追踪**​|`PyObject_GC_New` vs `PyObject_New`|GC 追踪的对象头部前面多一个 `PyGC_Head`|

> 洞见：**书里的"语义分类"与这里的"实现分类"正交。**​ 比如 `int` 语义上是 Numeric，`PyLong_Type` 却是静态类型、非 GC 追踪（不可变、不可能产生循环引用）；而 `list` 语义上是 Sequence，却是堆分配、GC 追踪。**判断一个对象走不走 GC，看的不是它是什么，而是它"能不能引用别的对象"**——不可变对象天然不可能形成环，所以 `int`/`str`/`tuple`（可含可变元素，所以 tuple 其实是被追踪的，此处需留意）的 GC 策略各不相同。这条判据比任何分类表都好记。

### 现代漂移：内存分配视角的分类

现代 CPython 的对象分配是**分层**的：

```
对象请求 → 类型专属 free list（int/tuple/list/frame...）
        → pymalloc：arena(256KiB/1MiB) → pool(4KiB) → block(8,16,24...512 字节)
        → malloc（>512 字节直接回落）
        → OS（mmap / VirtualAlloc）
```

- **arena**：32 位平台 256 KiB，64 位平台 1 MiB；
- **pool**​ 4 KiB，每个 pool 只服务一种 size class；
- **block**​ 按 8 字节对齐分档，最高 512 字节；
- **free list**​ 是"类型专属、绕过 pymalloc 的最快路径"——tuple（按尺寸分组）、空 list、frame 都有。

**只有 arena 能真正还给操作系统。**​ 这就解释了那个著名的运维困惑：**Python 进程 RSS 为什么"降不下来"**——对象释放只是把 block 还给 pool 的空闲链表，arena 里只要还有一个 pool 在用，整个 arena 就不能 `munmap`。

### 工业实际

- **内存 profiling 的正确姿势**：`sys.getsizeof()` 只告诉你对象本身，不含它引用的对象，也不含分配器粒度带来的取整。要看真实占用用 `tracemalloc`，要看总量用 `psutil` 的 RSS，两者差异正是 pymalloc 的arena 碎片。
- **调试内存破坏**：`PYTHONMALLOC=debug` 会给每个 block 加 canary（`deadbeef`）与记账头；`PYTHONMALLOC=malloc` 完全绕过 pymalloc，让 ASan / Valgrind 能看到每一次分配。写 C 扩展遇到段错误时，这两个环境变量是最快的第一步。

---

# 第 2 章 Python 中的整数对象

> **版本警告（最重要的一条）**：`PyIntObject` 在 Python 3 里**已经不存在了**。PEP 237 统一了 int 与 long，Python 3.0 起只有一种整数类型 `int`，它在 C 层就是原来的 `PyLongObject`；`PyInt_*` API 全部转发到 `PyLong_*`，`intobject.h` 在 3.1 被彻底删除。**读这两章时，请把书里的 `PyIntObject` 在脑中全部替换成 `PyLongObject`。**

---

## 2.1 初识 PyIntObject 对象

### 书里在讲什么

Python 2.5 的 `PyIntObject` 极简：

```
typedef struct {
    PyObject_HEAD
    long ob_ival;      // 一个 C long
} PyIntObject;
```

一个 C `long` 装值，溢出就升级成 `PyLongObject`。`PyInt_Type` 是它的类型对象。

### 现代：`PyLongObject` 完全是另一种东西

Python 3 的 `int` 是**任意精度大整数**，结构是"符号-数值"表示：

> `|value| = Σ ob_digit[i] × 2^(SHIFT×i)`，符号编码在 `ob_size` 的正负里。

- **digit 位宽**：3.0 全是 15 位；3.1–3.10 在 64 位构建上是 30 位、32 位是 15 位；**3.11 起默认全是 30 位**。
- **3.12+ 引入"紧凑表示"**：小整数的值直接打包进 `long_value.lv_tag` 这个位域里（高位存 digit 个数，低 3 位存符号与标志），不再需要额外的 digit 数组。
- **乘法**：单 digit 走朴素算法，多 digit 走 **Karatsuba**（O(n^1.585) 而非 O(n²)）。

### 第一性原理：为什么是 30 位而不是 32 位？

**原理 H：digit 位宽的选择 = "进位溢出"与"乘法中间结果溢出"之间的博弈。**

这是个非常漂亮的第一性原理问题。为什么不直接用一个 32 位无符号整数当 digit？

- 如果 digit 是 32 位，两个 digit 相乘的**中间结果需要 64 位**——在某些平台上拿不到高效的 64 位乘法，或者需要额外处理溢出。
- 留出 2 位余量（30 位存值），则**任意两个 digit 相乘仍落在 32 位内**，进位也可以安全地累加而不溢出。
- 现代 x86-64 有原生 64 位乘法，所以 3.11 起统一到 30 位（其实仍有余量空间，因为可以用 64 位做中间结果）。

> 洞见：**这个"浪费 2 位"的设计，是"用空间换算法简单性"的经典案例。**​ 它让整个大整数加法/乘法的实现里，不需要任何一处溢出检查——**正确性由位宽的算术性质静态保证，而不是由运行时的分支判断保证。**​ 这类"让不变量在类型/位宽层面成立，从而消灭运行时检查"的技巧，在高性能 C 代码里遍地都是，值得你专门去体会。

### 工业实际

- **`sys.getsizeof()` 的阶梯**：`0` → 24 字节，`1` / `2**30-1` → 28 字节，`2**30` → 32 字节，`2**60` → 36 字节。公式大致是 `28 + 4 × ceil(bit_length/30)`。
- **30 位是性能分水岭**：幅值 ≤30 位的操作（绝大多数现实代码）走单 digit 快路径，直接一次 CPU 运算，跳过所有循环。**这意味着 `int` 在"人类常用范围"内其实很快——慢的是大数运算。**
- **真要算大数，用 gmpy2**：密码学、大数分解这类场景，`gmpy2` 基于 GMP，乘法能快一个数量级。
- **别为了"避开大数"把运算拆小**：Python 层的循环远比 C 层的 `longobject.c` 慢。**算术就自然写，交给解释器。**

---

## 2.2 PyIntObject 对象的创建和维护

### 书里在讲什么

三级缓存体系（这是本书最经典的段落之一）：

1. **小整数池**：`NSMALLNEGINTS=5`、`NSMALLPOSINTS=257`，`static PyLongObject *small_ints[262]`，预建 **-5 到 256**​ 的所有整数。落在这个区间的整数直接返回池中对象。
2. **通用整数对象池（free list）**：Python 2.5 用 `PyIntBlock` 链表（`block_list`）+ `free_list` 管理已被释放的 int 对象，复用时不走 malloc。
3. **真正的 malloc**：池都空了才向系统申请。

于是 `a = 256; b = 256; a is b` → `True`；而 `a = 257; b = 257; a is b` → `False`。

### 现代：这套体系经历了三次重要改造

**① 小整数池：从"每解释器"回到"全局静态"**

这段历史很有教育意义，值得完整记住：

|时间|变更|动机|
|---|---|---|
|3.9（bpo-38858）|小整数**改为每个子解释器一份**，且**不再支持**在构建时通过 `NSMALLPOSINTS` 宏覆盖，必须改 `pycore_pystate.h`|为 per-interpreter GIL 铺路——每个解释器要能拥有自己的单例|
|3.11（bpo-45691）|Mark Shannon 把它**改回 `_PyRuntime` 里的静态数组**​|修复 use-after-free：解释器销毁顺序会让 small_ints 悬空|

> 洞见：**这是一次"为了并发做隔离 → 隔离引入了生命周期 bug → 退回共享并改用 immortality 解决并发"的完整往返。**
> 
> 最终的答案是：不靠"每解释器一份拷贝"来避免竞争，而是靠"对象不可变 + 免计数"来让共享变安全。**当隔离的成本（生命周期复杂度）高于共享的成本（同步开销）时，回头用"不可变性"重构掉这个权衡——这是 CPython 近五年反复出现的解法模式。**​ 我在第 1.4 节说的"不可变 → 可共享 → 可免同步"，在这里得到了完整的实证。

**② 整数 free list：3.14 才正式引入**

有意思的是，Python 3 早期一度**没有**整数的 free list，2.5 时代的 `PyIntBlock` 机制消失了。直到 **gh-126868（3.14）**​ 才重新为"紧凑整数对象"加回 free list：

```
// _PyLong_FromMedium：先试着从 free list 弹，没有再 malloc
PyLongObject *v = (PyLongObject *)_Py_FREELIST_POP(PyLongObject, ints);
if (v == NULL) {
    v = PyObject_Malloc(sizeof(PyLongObject));
    ...
}
```

`long_dealloc` 里对应地把紧凑、非小整数的 int 推回 free list：

```
if (_PyLong_IsCompact((PyLongObject *)self)) {
    if (compact_int_is_small(self)) { _Py_SetImmortal(self); return; }
    if (PyLong_CheckExact(self)) { _Py_FREELIST_FREE(ints, self, PyObject_Free); return; }
}
```

有个很直观的实验佐证[citation：64]：循环 10 万次 `print(i+1)` 会分配约 10 万个 int 对象（都是 print 造成的），而只做 `a = i + 1` 则只分配 905 个——**加法结果几乎全部命中 free list 或小整数池，真正的 malloc 极少**。

**③ 小整数 immortal 化与扩容**

- 小整数在 free-threaded 构建下被标记为 immortal；
- **正在开发的 3.16 把静态整数单例从 262 个扩到 1030 个（范围 [-5, 1024]）**。

### 第一性原理：三级缓存 = "局部性原理"在分配器上的直接应用

**原理 I：池化的全部依据是三条经验规律——时间局部性、值分布的长尾、以及"不可变对象可以被任何人共享"。**

- **时间局部性**：刚释放的 int 马上又会被分配 → free list；
- **值分布长尾**：0/1/-1/2 这些值占绝对多数 → 小整数池；
- **不可变**：共享同一个 5 是安全的，因为没人能把它改成 6 → 池化才成立。

> 洞见：**"不可变"是池化的许可证。**​ 你永远不会看到 `list` 有"小列表池"——因为共享一个可变对象会立刻导致别名 bug。而 `tuple` 有 free list（按尺寸分组），恰恰因为它是不可变的（尽管元素可变，但容器拓扑不变）。
> 
> 这条判据极其好用：**问"这个类型能不能池化"，等价于问"它可不可变"。**

### 工业实际

- **`a is b` 绝不能用于值比较。**​ 小整数池是 CPython 实现细节，官方文档对 `a = 1; b = 1` 的原话是"may or may not refer to the same object... **should not be relied upon**"。判空请用 `is None`，判值请用 `==`。
- **大数场景注意内存**：`10**1000` 约 3322 bits，占 ~460 字节。用 `bit_length()` 提前估算。
- **密码学用 `pow(a, b, mod)` 三参数形式**——模幂在内部保持中间结果小，比 `a**b % mod` 快几个数量级。

---

## 2.3 Hack PyIntObject

### 书里在讲什么

书里让你动手改 `intobject.c`，观察小整数池的行为、对象池的复用、引用计数的变化——核心目的是**把 2.2 节的静态结论变成你亲眼看到的事实**。

### 现代可做的等价实验（从易到难）

**① 纯 Python 层，零改动——先看现象**

```
import sys

# 小整数池边界
print(sys.getsizeof(0), sys.getsizeof(1), sys.getsizeof(2**30), sys.getsizeof(2**60))
# 24 28 32 36

# immortality（3.14+）
print(sys._is_immortal(1))      # True  —— 小整数已 immortal
print(sys._is_immortal(1000))   # False（3.14）；在 3.16 上是 True（[-5,1024] 都是静态单例）
print(sys.getrefcount(1))       # 天文数字 3221225472 —— 它已不再被计数

# 身份
a = 256; b = 256; print(a is b)   # True
a = 257; b = 257; print(a is b)   # False（交互式/同一 code object 内可能不同，注意编译期常量折叠）
```

**② 用 `ctypes` 直接读对象头**——把第 1 章学到的 16 字节头变成可见的数据

```
import ctypes
x = 42
refcnt = ctypes.c_ssize_t.from_address(id(x)).value
ob_type = ctypes.c_void_p.from_address(id(x) + 8).value
print(refcnt, hex(ob_type), hex(id(int)))   # ob_type 应等于 id(int)
```

**③ 改源码加探针**（现代版，改对文件）

- 文件是 `Objects/longobject.c`，不是 `intobject.c`；
- 在 `_PyLong_FromMedium` 里 `printf` 一次 `free list POP hit` / `malloc`，重跑上面那个 10 万次循环的实验，你会亲眼看到命中率；
- 在 `long_dealloc` 里打印，观察 free list 的 push/pop 配对。

**④ 加 opcode / 改语义**——现代要改 `Python/bytecodes.c` 再 `make regen-cases`（见第 0 章的"改源头不改生成物"纪律）。

### 工业实际

企业里真实有价值的"hack"通常不是改整数实现，而是**用这些知识做性能诊断**：

- 看到进程内存异常 → 怀疑是不是创建了海量 int（比如用 dict 存了大量计数，而非用 array / numpy）；
- **用 `array('i')` 或 NumPy 存同构数值**，能把 28 字节/元素的 Python int 压到 4–8 字节/元素——**这是 Python 数值密集型代码最大的一次数量级优化，而且它直接来自你在本章学到的对象头知识。**

---

# 第 3 章 Python 中的字符串对象

> **版本警告**：`PyStringObject` 在 Python 3 里分裂成了两个类型——**`str`（`PyUnicodeObject`，Unicode 文本）和 `bytes`（`PyBytesObject`，字节序列）**。Python 2 的 `str` 语义更接近 Python 3 的 `bytes`。**读这一章时，请把 `PyStringObject` 主要映射到 `PyUnicodeObject`（文件 `Objects/unicodeobject.c`）。**

---

## 3.1 PyStringObject 与 PyString_Type

### 书里在讲什么

Python 2.5 的 `PyStringObject` 是个变长对象：

```
typedef struct {
    PyObject_VAR_HEAD
    long ob_shash;      // 缓存的 hash 值，-1 表示未计算
    int  ob_sstate;     // intern 状态
    char ob_sval[1];    // 变长字符数组，以 '\0' 结尾
} PyStringObject;
```

三个字段各有深意：`ob_shash` 是 hash 缓存（字符串不可变，所以 hash 只需算一次）；`ob_sstate` 是 intern 状态机；`ob_sval` 是柔性数组的经典 C 技巧。

### 现代：`PyUnicodeObject` 是 PEP 393 的三态结构

Python 3.3 的 PEP 393（Flexible String Representation）彻底重写了字符串表示。核心是**一个三层结构体 + 一个位域状态**：

```
typedef struct {
    PyObject_HEAD
    Py_ssize_t length;
    Py_hash_t  hash;
    struct {
        unsigned int interned : 2;   // SSTATE_NOT_INTERNED / MORTAL / IMMORTAL / IMMORTAL_STATIC
        unsigned int kind     : 2;   // 1=UCS1, 2=UCS2, 4=UCS4
        unsigned int compact  : 1;   // 字符数据是否紧跟结构体
        unsigned int ascii    : 1;
        unsigned int ready    : 1;
    } state;
    wchar_t *wstr;
} PyASCIIObject;

typedef struct { PyASCIIObject _base; Py_ssize_t utf8_length; char *utf8; ... } PyCompactUnicodeObject;
typedef struct { PyCompactUnicodeObject _base; union { void *any; Py_UCS1 *latin1; Py_UCS2 *ucs2; Py_UCS4 *ucs4; } data; } PyUnicodeObject;
```

**三种形态**：

|形态|条件|布局|
|---|---|---|
|**compact ASCII**​ (`PyASCIIObject`)|全部字符 < U+0080|单次分配，字符数据紧跟结构体；**UTF-8 与字符数据共享同一块内存**（ASCII 天然是合法 UTF-8）|
|**compact non-ASCII**​ (`PyCompactUnicodeObject`)|有字符 ≥ U+0080|单次分配，字符数据紧跟结构体；UTF-8 单独缓存（惰性分配）|
|**legacy**​ (`PyUnicodeObject`)|通过旧 API `PyUnicode_FromStringAndSize(NULL, n)` 创建|结构体与数据分离，可变大小直到 `PyUnicode_READY()` 调用|

**三种字符宽度（kind）**：

|kind|类型|范围|字节/字符|
|---|---|---|---|
|`PyUnicode_1BYTE_KIND`|`Py_UCS1`|U+0000–U+00FF（Latin-1）|1|
|`PyUnicode_2BYTE_KIND`|`Py_UCS2`|U+0000–U+FFFF（BMP）|2|
|`PyUnicode_4BYTE_KIND`|`Py_UCS4`|U+0000–U+10FFFF|4|

CPython 在创建时自动选最紧凑的表示。

### 第一性原理：PEP 393 解决的是"空间 vs 随机访问"的两难

**原理 J：字符串表示的根本矛盾是——UTF-8 省空间但 O(n) 索引；定宽编码 O(1) 索引但浪费空间。PEP 393 的答案是"按内容自适应选宽度"。**

这是个纯粹的工程权衡，而且解法极其优雅：

- 用 UTF-8 存储 → 省内存，但 `s[i]` 要 O(n) 扫描（因为字符变长）；
- 用 UTF-32/UCS-4 定宽 → `s[i]` 是 O(1)，但一个纯 ASCII 字符串要浪费 4 倍内存；
- **PEP 393：看内容里最大的码点是多少，选最小够用的宽度。**​ 于是纯 ASCII 字符串 1 字节/字符（和 UTF-8 一样省），同时 `s[i]` 仍是 O(1)。

> 洞见：**PEP 393 是"自适应表示"这一设计范式的典范，而且它和 PEP 659 的 specialization 是同一个思路的两个实例。**
> 
> 两者的骨架完全同构：**先观察输入的特征（字符串的最大码点 / 操作数的实际类型），再选一个专门化的表示，并保证退化路径存在。**
> 
> 更有意思的是 ASCII 的额外优化：**纯 ASCII 字符串连 UTF-8 缓存都省了——因为 ASCII 本身就是合法的 UTF-8，`utf8` 指针直接指向字符数据。**​ 这不是"缓存"，是"同一份数据的两种解释"。**能用"重新解释"代替"复制"时，永远不要复制。**​ 这条原则同样适用于你写零拷贝的网络/序列化代码。

### 工业实际

- **内存**：一个 1000 字符的纯 ASCII 字符串在现代 CPython 里约 1049 字节；如果里面混入一个 emoji，kind 会跳到 UCS4，变成约 4049 字节。所以——**"字符串里混入一个 emoji，内存可能翻 4 倍"**。日志系统、文本处理流水线里这是真实会踩到的坑。
- **C 扩展互操作**：`PyUnicode_AsUTF8AndSize()` 会缓存 UTF-8 表示，反复调用不重复转换。而旧的 `PyUnicode_AsUnicode()`（返回 `wchar_t*`）已被标记为低效、应避免。
- **`sys.getsizeof(s)` 不含 intern 表与 UTF-8 缓存**，只反映结构本身。

---

## 3.2 创建 PyStringObject 对象

### 书里在讲什么

`PyString_FromString` / `PyString_FromStringAndSize` 是入口。几个关键优化：

- **空字符串复用**：全局 `nullstring` 单例；
- **单字符复用**：走字符缓冲池（见 3.4）；
- **intern**：某些路径会 intern（见 3.3）。

### 现代：一次分配完成结构 + 数据

创建的核心在 `PyUnicode_New(size, maxchar)`——**根据 `maxchar` 决定 kind，然后一次性分配结构体 + 字符数据**：

```
obj = (PyObject *) PyObject_MALLOC(struct_size + (size + 1) * char_size);
```

末尾那个 `+1` 是为了补 `'\0'`，让字符串能直接当 C 字符串用。

### 第一性原理

**原理 K：`compact` 标志的本质是"把两次分配合并成一次"。**

好处不是省了一个指针，而是**缓存局部性**——读字符串时结构体头和数据在同一个/相邻缓存行，少一次 cache miss。这正是文档里说的"减少内存碎片、改善缓存局部性"。

> 洞见：**"compact vs legacy" 这个二分，是 CPython 里反复出现的"新旧 API 共存"主题的又一例。**
> 
> PEP 393 明说：之所以还要保留 legacy 形态，是因为旧的 C API（允许先建空壳再填数据）不能一夜之间删掉。**于是新表示不能强制，只能"新 API 给你最好的，旧 API 给你兼容的"。**
> 
> 这个模式你在 `Python/bytecodes.c`（DSL 生成）、`Grammar/python.gram`（PEG）、`longintrepr.h`（3.0 起变私有 ABI）里都会再遇到。**CPython 的现代化，几乎总是通过"引入更好的平行路径 + 把旧路径降级为兼容层"来完成的，而不是原地重写。**

### 工业实际

写 C 扩展时的对应纪律：**用 `PyUnicode_New` + 填充（一次分配、compact），而不是 `PyUnicode_FromStringAndSize(NULL, n)` 再填（legacy、两次分配）。**​ 后者在现代 CPython 里是明确的次优路径。

---

## 3.3 字符串对象的 intern 机制

### 书里在讲什么

intern 就是**字符串池化**：内容相同的字符串在内存里只存一份。书里讲 `PyString_InternInPlace()`、全局 interned 字典、`ob_sstate` 三态（`SSTATE_NOT_INTERNED` / `SSTATE_INTERNED_MORTAL` / `SSTATE_INTERNED_IMMORTAL`）。

### 现代：机制基本没变，但边界和工具都更清晰了

**① 什么时候自动 intern？**

- **长得像标识符的字符串**（匹配 `[a-zA-Z0-9_]*`）会被自动 intern。所以 `a = "hello"; b = "hello"; a is b` → `True`，而 `"hello world!"`（含空格、不像标识符）不会被自动 intern。
- **字典的 key 也会被 intern**（模块、类、实例的 `__dict__` 用的就是 interned key）。

**② 强制 intern：`sys.intern()`**

官方文档给的理由非常具体，值得逐字记住：

> "Interning strings is useful to gain a little performance on dictionary lookup – if the keys in a dictionary are interned, and the lookup key is interned, the key comparisons (after hashing) can be done by a **pointer compare instead of a string compare**."

即：**intern 之后，字典查找在 hash 命中之后，可以用一次指针比较代替字符串内容比较。**​ 这是把 O(len) 的比较降成 O(1)。

**③ 关键的警告（书里没强调，现代文档明确写了）**

> "**Interned strings are not immortal**; you must keep a reference to the return value of `intern()` around to benefit from it."

这是一个非常容易踩的坑：intern 只是"放进表里"，不会阻止它被回收。你必须自己持有引用。

**④ 新的观测工具**

- **3.13 新增 `sys._is_interned(s)`**——终于可以从 Python 层直接观测 intern 状态了。
- **free-threaded 构建下，interned 字符串被标记为 immortal**——因为 intern 表是全局共享的，多线程并发访问时免计数能消除热点竞争。

### 第一性原理

**原理 L：intern 是"用一次哈希查表，换取后续所有等值比较变成指针比较"。**

它的适用条件极其明确，我把它总结成一条可以照着判断的规则：

> **当同一批字符串值会被反复用作字典 key 或反复做等值比较，且这批值的集合是封闭、有限、高重复时，intern 才划算。**

- 划算：编译器/解释器的标识符表、JSON 解析的字段名、NLP 的词表、日志的 tag 字段、ORM 的列名。
- 不划算：用户生成的一次性字符串、UUID、带时间戳的消息——**intern 表只增不减，会把它们永久钉在内存里，变成内存泄漏。**

> 洞见：**intern 表是一个"永久增长的全局哈希表的特例"，它是一个不受 GC 管理的强引用容器。**​ 这解释了官方文档为什么要警告"interned strings are not immortal"——它说的是"表本身不持有强引用"，但反过来，**只要你持有引用，那个字符串就永远活在表里**。
> 
> 换句话说：**`sys.intern()` 是一次"用全局表的确定性增长，换局部比较的确定性加速"的交易。**​ 用错了（对无限集合 intern），就是把性能优化写成内存泄漏。**绝大多数"优化"都是这样的交易——先问清楚你付出的是什么。**

### 工业实际

- **NLP / 编译器 / 解析器**：对词表、token、AST 节点名做 intern，是标准操作。有报告称对 100 万个重复字符串可省约 50% 内存，`is` 比 `==` 快约 10 倍（该数据来自二手来源，建议自己用业务数据复测）。
- **JSON / 日志解析**：字段名高度重复，intern 收益明显；字段值通常不重复，**不要 intern**。
- **反模式**：`sys.intern(request_id)` / `sys.intern(timestamp_str)` → 内存泄漏。

---

## 3.4 字符缓冲池

### 书里在讲什么

Python 2.5 有一个 `characters[UCHAR_MAX + 1]` 数组——**缓存 256 个单字符字符串**。创建长度为 1 的字符串时直接复用池中对象，不分配内存。

### 现代：这个池不但还在，而且被 immortal 化了

这条 2.5 时代的优化，在现代 CPython 里以一种更彻底的形式活着：

> 3.16 在构建期就创建 **256 个 Unicode 单例（U+0000–U+00FF）**、**256 个 bytes 单例（`b'\x00'`–`b'\xff'`）**，全部标记为 **immortal**。

也就是说，字符缓冲池从"运行时懒初始化的缓存数组"升级成了"**构建期生成的静态 immortal 单例**"。

### 第一性原理

**原理 M：这是"值域封闭 + 不可变"带来的终极优化——把对象从"运行时分配"变成"编译期常量"。**

判断能否这么做的两个条件：

1. **值域封闭且极小**：单字符只有 256 种可能（Latin-1 范围内），可以穷举；
2. **不可变**：没人能改掉 `'\x41'` 的内容，所以共享安全。

满足这两条，就可以**在构建 CPython 时就把这 256 个对象造好**，放进静态数据段，标记为 immortal，之后运行时永远不需要为它们分配内存或维护引用计数。

> 洞见：**2.5 的字符缓冲池 → 3.16 的静态 immortal 单例，这条演进线展示了"缓存"的终局形态。**
> 
> 缓存的三个阶段，正好对应本书第 2、3 章与现代 CPython 的差异：
> 
> 1. **运行时池化**（2.5 的 `characters[]`、小整数池）——省分配，但仍占堆、仍需计数；
> 2. **免计数**（PEP 683 immortal，3.12）——省 incref/decref，消除缓存行失效与并发竞争；
> 3. **构建期固化**（3.16 静态单例）——连"初始化"都省了，对象直接躺在二进制的静态段里。
> 
> **每一步都在把"运行时的工作"往"构建期"搬。**​ 这就是"零成本抽象"的实操定义：**能在编译期做完的事，绝不留到运行时。**​ 这条原则对你自己的代码同样成立——查表代替计算、常量折叠、预分配池，本质上都是同一件事。

---

## 3.5 PyStringObject 效率相关问题

### 书里在讲什么

核心问题：**字符串拼接**。因为字符串不可变，`s = s + t` 每次都要新建对象并复制，n 次拼接是 **O(n²)**。

### 现代：结论完全成立，而且还有几条书里没讲的

**① `+` vs `join()`——结论不变**

在循环里累加字符串用 `+` 是 O(n²)：每次拼接要复制已有的全部内容。用 `''.join(parts)` 是 O(n)——**一次性算出总长度、一次分配、逐个拷入**。

**② 现代新增的注意点：`kind` 的选择会影响内存**

拼接、切片、格式化都可能改变 kind。比如把一个纯 ASCII 字符串和一个含中文的字符串拼起来，结果的 kind 会升到 UCS2，内存翻倍。**在内存敏感的场景（大文本处理、日志缓冲），留意 kind 的"污染"。**

**③ f-string vs `%` vs `.format()` vs `join`**

现代 CPython 里 f-string 通常最快（编译期就能确定结构，3.11+ 还有专门的 `FORMAT_SIMPLE` / `BUILD_STRING` 优化）。3.14 还引入了 **t-string（`t"..."`）**​ 用于需要"延迟/安全插值"的场景（如 SQL 模板）。

**④ hash 缓存**

`ob_shash`（现代是 `PyASCIIObject.hash`）缓存字符串的 hash。字符串不可变，所以 hash 只需算一次。这直接解释了**为什么字符串能当字典 key 且很快**——也解释了为什么你自定义的可变对象不该有 `__hash__`。

### 第一性原理

**原理 N：不可变性的性能账单是双向的——它既带来"可以缓存、可以池化、可以 intern"的红利，也带来"每次修改都要复制"的成本。**

这是整章（乃至整本书）最重要的一条平衡观：

|不可变带来的收益|不可变带来的成本|
|---|---|
|hash 可缓存|拼接 O(n²)|
|可 intern / 池化|修改要全量复制|
|可免引用计数（immortal）|大量中间对象产生 GC 压力|
|线程安全（无锁读）|—|

> 洞见：**"不可变"从来不是免费的，它只是把成本从"并发正确性"转移到了"内存与复制"。**​ CPython 对字符串做的所有优化（intern、字符池、kind 自适应、immortal），本质上都是**在偿还这笔账单的利息**。
> 
> 而你在业务代码里的选择（`+` 还是 `join`、要不要 intern、用 list 累积还是 str 累积），就是**在决定这笔利息由谁来付**。理解了这一点，你就不是在背"用 join 更快"这种口诀，而是在做有依据的权衡。

### 工业实际

- **日志/大文本构建**：一律 `''.join(parts)` 或 `io.StringIO`。
- **大量小字符串**：考虑 intern（若值域封闭）。
- **超大文本**：考虑用 `array`/`bytearray`/NumPy，或流式处理（generator + `writelines`），避免一次性 materialize。
- **内存 profiling**：`sys.getsizeof()` 看单个字符串，`tracemalloc` 看整体分配热点。

---

## 3.6 Hack PyStringObject

### 书里在讲什么

动手验证 intern 行为、观察 `ob_sstate` 变化、打印字符缓冲池的复用情况。

### 现代可做的等价实验

**① 零改动观测（推荐先做这个）**

```
import sys

# intern 的自动触发条件：像不像标识符
a = "hello";      b = "hello";      print(a is b)   # True  —— 像标识符，自动 intern
c = "hello world!"; d = "hello world!"; print(c is d) # False —— 有空格，不自动 intern

# 强制 intern 并观测（3.13+）
e = sys.intern("hello world!"); f = sys.intern("hello world!")
print(e is f, sys._is_interned(e))    # True True

# 观察 kind 对内存的影响
print(sys.getsizeof("hello" * 100))        # 纯 ASCII
print(sys.getsizeof("你好" * 100))          # UCS2，注意对比
print(sys.getsizeof("😊"))    # UCS4

# 拼接的 O(n²) 实证
import time
def bench(n):
    t = time.perf_counter(); s = ""
    for i in range(n): s += "x"
    plus = time.perf_counter() - t
    t = time.perf_counter(); "".join("x" for _ in range(n))
    return plus, time.perf_counter() - t
print(bench(200000))
```

**② 加探针改源码**

- 文件是 `Objects/unicodeobject.c`；
- 在 `PyUnicode_InternInPlace` / `intern_common` 里打印 key，观察哪些字符串被 intern 了；
- 在 `PyUnicode_New` 里打印 `maxchar` 与选中的 `kind`，观察 kind 的选择过程；
- 在 `unicode_dealloc` 里观察 free list（`unicode_writers` 等）。

**③ 用 C 扩展或 ctypes 读 `state` 位域**——把 `interned` / `kind` / `compact` / `ascii` 这四个 bit 直接打出来看。

### 工业实际

这个 hack 的真正价值不在字符串本身，而在**建立"不可变对象的共享行为可被观测"的直觉**。有了这个直觉，你以后调试任何"对象身份"相关的问题（`is` 意外为 True/False、内存异常、意外的别名共享）都会快得多。

---

# 三章合起来：一条主线、三个洞见

## 主线：不可变性 → 可共享性 → 可缓存/池化 → 可免同步

```
不可变 (immutable)
   ↓ 没人能改，所以共享安全
可共享 (shareable)
   ↓ 共享就不用重复创建
可缓存/池化 (intern / small int cache / char pool / free list)
   ↓ 共享的对象被反复读写 → 引用计数成为热点
可免同步 (immortal → 不计数；biased refcount → 本地计数；deferred → 干脆不计数)
```

第 1 章给你**身份与计数的模型**，第 2、3 章给你**两个不可变类型的优化实例**，而现代 CPython 把这根链条推到了"免计数"的终点。**这三章不是三块知识，是同一条逻辑链的三次展开。**

## 洞见一：`PyObject` 头不是"实现细节"，它是整个动态性的物理载体

16 字节里装的是"谁在用它"和"它能干什么"两个问题的答案。CPython 二十多年的优化史，就是一部"如何减少解这两个答案的代价"的历史——从 specialization（缓存类型判断结果）到 immortal（取消计数）到 biased refcount（消除跨线程同步），**全部围绕这两个字段展开**。

## 洞见二：CPython 的现代化，从来不是"重写"，而是"引入更好的平行路径 + 把旧路径降级为兼容层"

PEP 393 保留了 legacy 字符串；`longintrepr.h` 从公开变成私有 ABI 而不是删掉；小整数池从"每解释器"退回到全局静态，但用 immortality 而非隔离来解决竞争。**认识到这个模式，你就能预测 CPython 未来会怎么变**——永远是新路径先并行、旧路径后降级，激进的原地重写在这个社区里几乎不会发生。

## 洞见三：所有"优化"都是交易，先问清楚你付出的是什么

- 池化付出**内存与复杂度**，换来**分配速度**；
- `+` 付出 **O(n²) 复制**，换来**代码直观**；
- `sys.intern()` 付出**全局表永久增长**，换来**比较变指针相等**；
- free-threading 付出**单线程 5–15% 回退**，换来**多核并行**；
- immortal 付出**"这些对象永不回收"**，换来**免计数与缓存行安宁**。

**能清楚说出代价的优化，才是工程决策；说不出代价的，只是信仰。**

---

# 立刻可做的实验（按性价比排序）

1. **跑一遍 3.1 节的 `sys._is_immortal` / `sys.getrefcount(1)`**，亲眼看到 `3221225472` 这个"不再被计数"的数字。
2. **用 `ctypes` 读出 `42` 的 `ob_refcnt` 和 `ob_type`**，验证 `ob_type == id(int)`。
3. **跑 3.6 节那个 `"hello"` vs `"hello world!"` 的 intern 对比**，再用 `sys._is_interned()` 确认。
4. **跑 `+` vs `join` 的 20 万次基准**，亲手感受 O(n²)。
5. **打开 `Objects/longobject.c` 的 `_PyLong_FromMedium` 与 `long_dealloc`**，读 free list 的 push/pop 这两段。

---

### 版本差异速查表（贴在书边上）

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`Objects/intobject.c` / `PyIntObject`|`Objects/longobject.c` / `PyLongObject`（PEP 237，`intobject.h` 3.1 删除）|
|`Objects/stringobject.c` / `PyStringObject`|`Objects/unicodeobject.c`（`str`）/ `Objects/bytesobject.c`（`bytes`）|
|`ob_shash` / `ob_sstate`|`PyASCIIObject.hash` / `state.interned`（2 bit）|
|`characters[256]` 字符缓冲池|3.16 起为 256 个构建期静态 immortal Unicode 单例|
|`PyIntBlock` + `free_list`|3.14 起为紧凑 int 的 `_Py_FREELIST_POP(ints)`|
|单一 `ob_refcnt`|immortal（3.12）/ biased refcount（3.13 free-threaded）/ deferred + stack ref（3.16 开发中）|

