
## 开篇：第 4、5 章的共同主线——可变容器的“对偶设计”

第 2、3 章的 `int` / `str` 是**不可变**对象，靠 intern、小整数池、单字符缓冲池、immortal 来优化；第 4、5 章的 `list` / `dict` 是**可变**容器，优化手段完全不同：

- **list**​ = 动态数组（`PyObject*` 连续指针数组），优化靠**过度分配（over-allocation）**把 `append` 做成均摊 O(1)，以及**对象壳 free list**​ 复用。
- **dict**​ = 开放寻址哈希表，优化靠**紧凑表示（索引与条目分离）**、**扰动探测**、**键共享（split table）**，以及**对象壳 + 键对象双层 free list**。

这两章最精彩的第一性原理对偶是：**list 扩容约 1.125 倍，dict 扩容 2 倍**。原因不是随意的——list 扩容只是 `memcpy` 指针（便宜），dict 扩容要 **rehash 全部条目**（昂贵，要重算哈希、重建索引）。**操作成本决定了增长常数**，这是理解容器设计的钥匙。

继续沿用**双轨阅读法**：轨道一吃透 2.5 的最小模型（机制，跨版本稳定）；轨道二用现代 CPython（3.13/3.14/3.15）校验（策略，频繁变化）。

---

## 第 4 章 Python 中的 List 对象

### 4.1 PyListObject 对象

**书里（Python 2.5）**：

```
typedef struct {
    PyObject_VAR_HEAD   // 变长对象头：ob_refcnt, ob_type, ob_size
    PyObject **ob_item; // 指向连续的 PyObject* 指针数组
    Py_ssize_t allocated; // 已分配的槽位容量（capacity）
} PyListObject;
```

- `ob_size`：当前元素个数，即 `len(list)`。
- `allocated`：`ob_item` 数组的容量，`0 <= ob_size <= allocated`。
- **list 存的是对象的引用，不是对象本身**。`ob_item` 里每个元素是一个 8 字节（64 位）的 `PyObject*`，指向真正的对象。所以 list 能存任意类型、随机访问 O(1)，但访问要多一次解引用。

**现代（3.13/3.14）**：

- 默认构建（有 GIL）下，`PyListObject` 的布局与 2.5 **完全一致**，仍是 `ob_item` + `allocated`。64 位平台字段偏移：`ob_refcnt` 0、`ob_type` 8、`ob_size` 16、`ob_item` 24、`allocated` 32（可由 `ctypes` 直接观测）。
- **free-threaded（PEP 703，3.13+）漂移**：`PyObject` 头部被改成“有偏引用计数（biased refcounting）”结构——`ob_tid`（属主线程 id）、`ob_ref_local`（本地计数，非原子）、`ob_ref_shared`（共享计数，原子）、`ob_mutex`（1 字节的 `PyMutex` 每对象锁）。list 的 append / insert / sort 等操作通过 `Py_BEGIN_CRITICAL_SECTION(op->ob_mutex)` 获取该锁来保证线程安全。

**第一性原理**：

- **list = 连续内存 + 指针间接**。用“连续的地址空间”换 O(1) 随机访问（对比链表 O(n)）；用“统一 8 字节指针”换异构存储能力（对比 C 数组必须同类型）。代价是：每次访问多一次访存、且大量小对象时内存膨胀（见 4.4）。
- **`ob_size` vs `allocated` 就是“大小 vs 容量”**，这是所有动态数组（C++ `vector`、Java `ArrayList`）的通用抽象。CPython 的独特之处在 4.2 的增长因子。

**洞见**：list 是“对象的容器”，不是“数据的容器”。`[1,2,3]` 在内存里是 3 个指针（24 字节）+ 3 个独立的 `PyLongObject`（每个 28 字节），总共约 108 字节，而不是 C 语言里 3 个 `int` 的 12 字节。理解了这一点，才明白为什么数值计算要用 `array` / NumPy。

---

### 4.2 PyListObject 对象的创建与维护

**书里（Python 2.5）**：

- `PyList_New(size)`：优先从 `free_list` 取（见 4.3），否则 `PyObject_GC_New`；然后 `ob_item = PyMem_MALLOC(size * sizeof(PyObject*))` 并清零；`allocated = size`。（`size=0` 时 `ob_item = NULL`。）
- 维护核心是 `list_resize(self, newsize)`：
    - 若 `allocated >= newsize` 且 `newsize >= allocated >> 1`（新长度不小于容量的一半），**不重新分配**，直接改 `ob_size`；
    - 否则按增长模式扩容或缩容，然后 `PyMem_Realloc`。
- `append` 走 `app1()` → `list_resize(self, n+1)`；`insert` 走 `ins1()` → `list_resize` 后 `memmove` 搬移元素（O(n)）。

**增长公式的版本漂移（重点）**：

- **2.5 / 老 trunk**：`new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6) + newsize;`
- **现代 CPython 3.12+**：`new_allocated = ((size_t)newsize + (newsize >> 3) + 6) & ~(size_t)3;` 即 `(newsize + newsize//8 + 6)` 再向下对齐到 4 的倍数。
- 两者产生的**容量序列都是 `0, 4, 8, 16, 25, 35, 46, 58, 72, 88, 109, 133, 161, ...`**。增长因子约 **1.125（9/8）**，**不是 2 倍**。
- 实测（`sys.getsizeof`，64 位）：空列表 56 字节（`__sizeof__` 是 40，多出的 16 字节是 GC 头）；`append` 第 1 个变 88（容量 4）、第 5 个变 120（容量 8）、第 9 个变 184（容量 16）、第 17 个变 248（容量 24）……触发扩容的长度是 **1, 5, 9, 17, 25, 33, 41...**。

**第一性原理：为什么 list 用 1.125 倍而不是 2 倍？**

- **均摊 O(1) 只要求增长是“几何级数”（比例 >1 的常数），不要求是 2 倍。**​ 增长因子为 `c` 时，n 次 append 的总复制量是 `n + n/c + n/c² + ... = O(n)`，均摊 O(1)。
- 2 倍（如 C++ `vector` 常见）复制次数最少，但**内存浪费最多可达 50%**；1.125 倍内存浪费仅约 11%，但 realloc 次数更多。CPython 选择 1.125 是**在内存效率和复制成本之间偏向内存**（服务端程序常驻大列表，内存比偶尔的 realloc 更贵）。
- 还有**滞回（hysteresis）缩容**：只有当新长度 < 容量的一半时才真正缩小，避免在阈值附近反复扩缩。

**工业实际**：

- **预分配远快于循环 append**：已知最终大小时，用 `[None] * n` 再按索引填，或列表推导式。n=100,000 时预分配约 0.4ms，`append` 循环约 8ms，**快 20 倍**。3.14 还引入了切片特化（GH-132626）进一步优化切片操作。
- **别用 list 做队列头操作**：`insert(0, x)` / `pop(0)` 是 O(n)（要 `memmove` 全部元素），用 `collections.deque`（双端 O(1)）。`append` / `pop()`（尾部）是 O(1)。
- **free-threaded 下的原子性**：`lst.append(x)`、`lst.pop()`（尾部）、`lst[i] = x`、`lst.clear()` 是**原子的**（对象锁保护）；`lst.sort()` **不是**原子的（排序期间其他线程看列表像空）；`lst.insert(idx, x)`、`lst.pop(idx)` 会搬移，其他无锁读可能看到中间态。而 `sum(lst)`、`zip()`、`reduce()`、`list(iterable)` 等 C 层迭代**不是原子的**（每次 `__next__` 独立加锁，可能读到并发修改的中间快照）。要求一致视图就先 `tuple(lst)` 或 `lst.copy()`（拷贝在锁内原子完成），或自行加锁。
- **C 扩展（3.13+）**：`PyList_GetItem` 返回借用引用，在 free-threaded 下可能失效，应改用 **`PyList_GetItemRef`**（返回强引用）。

---

### 4.3 PyListObject 对象缓冲池

**书里（2.5）**：

```
#ifndef PyList_MAXFREELIST
#define PyList_MAXFREELIST 80
#endif
static PyListObject *free_list[PyList_MAXFREELIST];
static int numfree = 0;
```

`list_dealloc` 先 `Py_XDECREF` 每个元素、`PyMem_FREE(ob_item)`，然后把 `PyListObject` **对象壳**放进 `free_list`（若 `numfree < 80`）；`PyList_New` 优先 `free_list[--numfree]` 复用。

**现代（3.13/3.14）**：

- 常量仍是 **80**，但移到了 `Include/internal/pycore_freelist.h` 的 `struct _Py_list_state`（或 `_Py_list_freelist`）中，且可通过 `WITH_FREELISTS` 编译开关关闭（关闭时定义为 0，free list 完全不存在）：
    
    ```
    struct _Py_list_state {
        PyListObject *free_list[PyList_MAXFREELIST]; // 80
        int numfree;
    };
    ```
    
- **关键**：缓冲池只缓存 `PyListObject` 这个**固定大小的对象壳**，**不缓存 `ob_item` 指针数组**（`ob_item` 在 dealloc 时被 `PyMem_Free`，new 时再 `PyMem_Calloc`）。
- 解释器结束/GC 时通过 `_PyList_ClearFreeList` / `PyList_Fini` 释放缓存对象。打开 `SHOW_ALLOC_COUNT` 宏可以打印 `count_alloc` 与 `count_reuse` 的复用率。

**第一性原理**：

- **可变容器的“壳”（固定大小结构体）适合 free list，“负载”（变长数据区）不适合**——负载大小不确定、缓存命中率低、占用内存大。这和 int 的 free list、str 的缓冲池同理：只缓存高频创建/销毁的**固定形态**部分。
- free list 是**单线程时代**的优化。free-threaded 下 free list 的 push/pop 本身需要同步（CPython 用 `_PyFreeListState` 管理，并在 GC/终结时清理）；若追求简单可关闭 `WITH_FREELISTS`。

---

### 4.4 Hack PyListObject

**书里**：改 `listobject.c`，在 `list_resize` 加日志观察容量变化，或打开 `SHOW_ALLOC_COUNT` 看缓冲池命中。

**现代零改动实验（推荐先做）**：

1. **观测过度分配与扩容跳跃**：
    
    ```
    import sys
    lst = []
    prev = 0
    for i in range(60):
        if sys.getsizeof(lst) != prev:
            print(len(lst), sys.getsizeof(lst),
                  "capacity:", (sys.getsizeof(lst) - 56) // 8)
        prev = sys.getsizeof(lst)
        lst.append(i)
    ```
    
      
    会看到长度 1→88（容量 4）、5→120（容量 8）、9→184（容量 16）、17→248（容量 24）、25→312（容量 32）……亲手验证 **1.125 倍增长**，而不是 2 倍。
2. **用 `ctypes` 直接读 `ob_size` 与 `allocated`**（偏移 16 与 32）：
    
    ```
    import ctypes
    l = [1, 2, 3]
    print("len:", ctypes.c_ssize_t.from_address(id(l) + 16).value)       # 3
    print("allocated:", ctypes.c_ssize_t.from_address(id(l) + 32).value) # 4（或更大）
    ```
    
      
    这把第 1 章学到的“对象头偏移”真正用起来。
3. **预分配 vs append 基准**：
    
    ```
    import time
    n = 100_000
    t = time.perf_counter(); a = [None] * n; t1 = time.perf_counter() - t
    t = time.perf_counter()
    b = []
    for i in range(n): b.append(i)
    t2 = time.perf_counter() - t
    print(t1, t2)   # 预分配明显更快
    ```
    

**改源码实验（现代文件是 `Objects/listobject.c`）**：

- 在 `list_resize` 里打印 `printf("resize: allocated=%zd -> new_allocated=%zd\n", self->allocated, new_allocated)`，重编译后 `append` 观察。
- 在 `PyList_New` / `list_dealloc` 里统计 free list 命中率（启用 `SHOW_ALLOC_COUNT`）。

**工业实际（承 4.2）**：

- **同构数值用 `array` / NumPy**：list 存 1M 个 int 要 8MB 指针 + 1M×28B 的 int 对象 ≈ 36MB；`array('i')` 只要 4MB。这是从“list 存指针不存对象”直接推出的最大优化。
- **free-threaded 迁移**：并发 `append` 到共享 list 是安全的（有锁，不会破坏内部数组），但读-改-写复合逻辑（先 `if x in lst` 再 `lst.append`）不是原子，需要自己用 `threading.Lock`；纯消费者-生产者场景用 `queue.Queue` 更省心。
- 3.13+ 写 C 扩展：`PyList_GetItem` → **`PyList_GetItemRef`**；`PyList_SET_ITEM` 在 free-threaded 下无内部同步，仅用于填充“其他线程还看不到的新列表”，共享列表要用 `PyList_SetItem`（走对象锁）。

---

## 第 5 章 Python 中的 Dict 对象

### 5.1 散列表概述

**冲突解决的两种范式**：

- **链地址法（separate chaining）**：每个槽挂链表，冲突元素串起来。
- **开放寻址（open addressing）**：**CPython 从始至终都用开放寻址**——所有条目直接存在哈希表数组里，冲突了就按探测序列找下一个空槽。

**CPython 的扰动探测（perturbation probing）**（不是简单线性探测！）：

```
#define PERTURB_SHIFT 5
size_t perturb = hash;               // 初始扰动 = 完整哈希
size_t i = hash & mask;              // 起始槽位，mask = 表大小-1（表大小是 2 的幂）
for (;;) {
    // 检查槽位 i（通过 dk_indices[i] 拿到条目索引）
    perturb >>= PERTURB_SHIFT;       // 每探测一次，扰动右移 5 位
    i = (i * 5 + perturb + 1) & mask; // 下一个槽位
}
```

- `i*5 + 1` 在表大小为 2ⁿ 时能生成**完整排列**（因为 gcd(5, 2ⁿ)=1），保证能探测到所有槽位。
- `perturb` 用哈希的**高比特位**参与，打破“只有低位起作用”导致的聚集（clustering）；随着 `perturb` 右移为 0，探测退化为 `5*i+1`，仍是完整排列。
- 探测序列示例（表大小 32、起始 3）：3 → 11 → 19 → 29 → 5 → 6 → 16 → 31 → 28 → 13 → 2 ...

**删除用墓碑（tombstone）**：

- 索引槽标记为 `DKIX_DUMMY (-2)`，条目 `me_key` 置 NULL；**绝不能标为 `DKIX_EMPTY (-1)`**，否则会截断探测链，导致后面插入的元素找不到。
- 只有下一次扩容（`dictresize`）才清理所有 dummy，回收空间。

**装载因子与扩容**：

- 最大装载因子是 **2/3**（`USABLE_FRACTION`）。当已用条目达到容量的 2/3 就扩容。
- `GROWTH_RATE(d) = ma_used * 3`。因为 2/3 满载时 `used*3 == 2*size`，所以**新容量 = ≥ `ma_used*3` 的最小 2 的幂 = 原容量翻倍**（无删除时）：“dicts double in size when growing without deletions”。
- 初始表大小 `PyDict_MINSIZE = 8`（空字典就有 8 个槽位，`dk_usable` 初始为 5，因为 `USABLE_FRACTION(8)=5`）。

**第一性原理（对偶洞见）**：

- **dict 用 2 倍扩容，list 用 1.125 倍**——因为 dict 扩容要 **rehash 所有条目**（重算哈希 + 重建索引，O(n) 且常数大），必须一次给足空间把 rehash 次数压到 O(log n)；list 扩容只是 `memcpy` 指针数组（便宜），所以可以用更省内存的 1.125 倍。**操作成本决定增长常数**，这是本章最漂亮的第一性原理。
- 装载因子 2/3 是**冲突概率与内存浪费的平衡点**：太大则冲突剧增、探测变长；太小则内存浪费、缓存不友好。

---

### 5.2 PyDictObject

**书里（2.5）：单体稀疏表（combined table 雏形）**：

```
typedef struct {
    Py_ssize_t me_hash;   // 缓存的哈希值
    PyObject *me_key;
    PyObject *me_value;
} PyDictEntry;             // 24 字节（64 位）

struct _dictobject {
    PyObject_HEAD
    Py_ssize_t ma_fill;   // 活跃 + 非活跃（dummy）条目总数
    Py_ssize_t ma_used;   // 活跃条目数，即 len(dict)
    Py_ssize_t ma_mask;   // 表大小 - 1
    PyDictEntry *ma_table;// 条目数组
    PyDictEntry *(*ma_lookup)(PyDictObject *, PyObject *, long); // 查找函数
    PyDictEntry ma_smalltable[PyDict_MINSIZE]; // 内嵌的 8 槽小表
} PyDictObject;
```

2.5 是**一张稀疏大数组**：即使空槽也占 24 字节的 `PyDictEntry`，浪费严重。

**现代（3.6+ 紧凑字典，PEP 468 / Raymond Hettinger 提案，bpo-27350 实现）**：彻底重写为**“索引表 + 条目表”分离**。

`PyDictObject`（`Include/cpython/dictobject.h`）退化成一个小头部：

```
typedef struct {
    PyObject_HEAD
    Py_ssize_t ma_used;            // 元素个数，len(dict)
    uint64_t _ma_watcher_tag;      // 原 ma_version_tag，3.12+ 用于 watchers（见下）
    PyDictKeysObject *ma_keys;     // 指向真正的哈希表（键对象）
    PyDictValues *ma_values;       // split table 的值数组；combined table 时为 NULL
} PyDictObject;
```

真正的哈希表是 `PyDictKeysObject`（`Include/internal/pycore_dict.h`）：

```
struct _dictkeysobject {
    Py_ssize_t dk_refcnt;             // 引用计数（共享键时 >1）
    uint8_t dk_log2_size;             // 表大小 = 2^dk_log2_size（必须是 2 的幂）
    uint8_t dk_log2_index_bytes;      // 每个索引占几字节（log2）
    uint8_t dk_kind;                  // DICT_KEYS_GENERAL / UNICODE / SPLIT
    uint32_t dk_version;              // 键版本（共享键/特化失效用）
    Py_ssize_t dk_usable;             // 可用条目数（装载上限 = 2/3 容量）
    Py_ssize_t dk_nentries;           // 已用条目数
    // 后面紧跟：dk_indices[]（哈希表本体），再后面是 dk_entries[]（密集条目数组）
};
```

- **`dk_indices[]`**：开放寻址的哈希表本体，存的是 `dk_entries` 的下标，或哨兵 `DKIX_EMPTY(-1)` / `DKIX_DUMMY(-2)` / `DKIX_ERROR(-3)`。**索引宽度随表大小自适应**：槽数 ≤128 用 `int8_t`，≤32768 用 `int16_t`，≤2³¹ 用 `int32_t`，更大用 `int64_t`。
- **`dk_entries[]`**：**密集数组**，按插入顺序追加（这就是 3.7+ 插入有序的实现基础）。条目有两种：
    - `PyDictKeyEntry`（GENERAL 表）：`{ Py_hash_t me_hash; PyObject *me_key; PyObject *me_value; }` = 24 字节，自带哈希。
    - `PyDictUnicodeEntry`（UNICODE / SPLIT 表）：`{ PyObject *me_key; PyObject *me_value; }` = **16 字节，不存哈希**——因为 Unicode 对象自己缓存了 hash（`PyASCIIObject.hash`）。
- **combined vs split（PEP 412 键共享）**：
    - `ma_values == NULL` → **结合表（combined）**：键值都在 `dk_entries` 里，`dk_refcnt == 1`。普通 `{'a':1}`、`dict()` 建的字典都是 combined。
    - `ma_values != NULL` → **分离表（split）**：`ma_keys` 里只有键，**被同类所有实例共享**（`dk_refcnt > 1`），值存在 `ma_values` 数组；**只允许字符串（unicode）键**。用于同类实例的 `__dict__`——1000 个 `Student` 实例，键名 `"name"`/`"grade"` 只在共享的 `PyDictKeysObject` 里存一次，每个实例只存自己的值数组。这是面向对象程序省 10%–20% 内存的关键。

**洞见：紧凑字典的本质是“把稀疏性从昂贵的条目移到廉价的索引”**。

2.5 的稀疏大数组里，一个空槽占 24 字节；新实现里空槽只占 1–8 字节的索引。同时拿到三个好处：**省内存 20%–25%**、**保持插入顺序**（entries 追加）、**迭代更快**（只遍历密集 entries，不需要跳过空槽）。

**现代漂移：`ma_version_tag` → `_ma_watcher_tag`**

- Python 3.6 引入 `ma_version_tag`（PEP 509），字典每次修改递增，供 Cython 等扩展做快速全局查找。
- **PEP 699**（3.14 落实）宣布 `ma_version_tag` 不再对外保证，成为解释器内部字段；3.11 起内部优化已改用 **dict watchers**​ 做特化失效，C 扩展不应再依赖它（会发编译警告，并将在两个版本后移除）。
- 3.12+ 该字段更名为 `_ma_watcher_tag`，位布局：bits 0-7 是 **dict watchers 位图**（哪些观察器在监听），bits 8-11 是监听变更计数器（Tier 2 优化用），bits 12-31 未用，bits 32-63 在 free-threaded 下是每线程引用的唯一 id。也就是说，**旧“版本号”语义被回收给观测器使用**。

---

### 5.3 PyDictObject 的创建和维护

**创建**：`PyDict_New()` → `new_dict(interp, Py_EMPTY_KEYS, NULL, 0, 0)`。`Py_EMPTY_KEYS` 是一个静态的、大小为 8 的空键对象（`dk_log2_size = 3`），所以**新字典初始就有 8 个槽位，`dk_usable = 5`**。（`PyDict_New` 优先从 5.4 的 free list 取对象壳。）

**插入 / 查找**：

- `PyDict_SetItem(mp, key, val)` → 计算/复用哈希 → `_Py_dict_lookup` 探测 → 若 combined 表，写入 `dk_entries[dk_nentries]` 并在 `dk_indices` 记录下标；若 split 表，写入 `ma_values`。
- 查找入口 `_Py_dict_lookup` 按 `dk_kind` 分派到 `unicodekeys_lookup_unicode`（纯字符串键，快）、`unicodekeys_lookup_generic`、`dictkeys_generic_lookup`。
- **当 `ma_used` 达到 `dk_usable`（2/3 容量）触发扩容**：`insertion_resize` → `dictresize(interp, mp, calculate_log2_keysize(GROWTH_RATE(mp)), unicode)`。

**扩容（`dictresize`）细节**：

1. `GROWTH_RATE = ma_used * 3`；`calculate_log2_keysize` 找最小的 `log2_size` 使 `2^log2_size >= ma_used * 3`（即翻倍到下一个 2 的幂）。
2. 分配新的 `PyDictKeysObject`（新的 `dk_indices` + `dk_entries`）。
3. 搬移旧条目：若旧表没有删除过（无 dummy），`memcpy` 整块拷 `dk_entries`；否则遍历并跳过 `me_value == NULL` 的 dummy 条目。
4. 用 `build_indices_generic` / `build_indices_unicode` **重建 `dk_indices`**（所有条目重新探测位置）。
5. **分离表（split）在扩容时会被转换成结合表（combined）**——所以实例字典扩容（属性数超当前共享键容量）会丢失键共享（3.11+ 有改善，见下）。
6. 更新 `dk_usable` 与 `dk_nentries`。

**删除**：`PyDict_DelItem` 找到条目后，combined 表把索引槽设为 `DKIX_DUMMY`、`me_key = NULL`；split 表设为 Pending。删除**不立即缩容**，也不恢复 `dk_usable`（dummy 仍占探测链），只有下次 `dictresize` 才清理。`popitem()` 用 `me_hash` 字段做搜索指纹，并且总是把字典转成 combined 表。

**PEP 412 键共享的脆弱性与 3.11+ 改进（工业重点）**：

- **旧行为（3.3–3.10）**：任何一个实例**删除属性**，或**在已有多个实例后新增属性导致共享键数组扩容**，整个类的键共享就被破坏，所有实例（含之后新建的）退化成 combined 表——实例 `__dict__` 从约 104 字节膨胀到约 232 字节。
- **3.11+ 改进**：共享字典的值被**内联（inline）到实例**（托管字典 `Py_TPFLAGS_MANAGED_DICT`），只要类的唯一属性数 **< 30**，**删除属性或以不同顺序插入键都不再破坏键共享**；只有在第二个实例创建后、新增属性导致共享键数组需扩容时才退化。
- 实践：尽量在 `__init__` 里设完所有属性（或在第二个实例创建前设好），就能稳定享受键共享；用 `__slots__` 则完全去掉 `__dict__`，更省。

**free-threaded 维护**：dict 有每对象锁。`PyDict_SetItem` / `PyDict_DelItem` / `d[k]` 读取，在键是 `str/int/float/bool/bytes` 时是**原子的**；但 `PyDict_Next` 迭代**不上锁**，必须用 `Py_BEGIN_CRITICAL_SECTION` 包裹。借用引用 `PyDict_GetItem` 在并发下失效，要改用 **`PyDict_GetItemRef`**（返回强引用）。

---

### 5.4 PyDictObject 对象缓冲池

**书里（2.5）**：

```
#define PyDict_MAXFREELIST 80
static PyDictObject *free_list[PyDict_MAXFREELIST];
static int numfree = 0;
```

`dict_dealloc` 先 decref 所有键值，然后若 `numfree < 80` 把 `mp` 放进 `free_list`；`PyDict_New` 优先复用。**注意**：2.5 的 `ma_smalltable[8]` 内嵌在结构体里，所以小字典随对象壳一起被缓存，不额外分配 `ma_table`。

**现代（3.13/3.14，`pycore_dict_state.h`）**：依然是 **80**，但有**两个 free list**：

```
#define PyDict_MAXFREELIST 80
#define DICT_MAX_WATCHERS 8
struct _Py_dict_state {
    uint64_t global_version;
    uint32_t next_keys_version;
    PyDictObject *free_list[80];          // 缓存 dict 对象壳
    int numfree;
    PyDictKeysObject *keys_free_list[80]; // 缓存键对象（哈希表）
    int keys_numfree;
    PyDict_WatchCallback watchers[DICT_MAX_WATCHERS];
};
```

- `dict_dealloc`：解引用键值 → `free_values` → `dictkeys_decref` → 若 `numfree < 80` 把 `mp` 放入 `free_list`。
- **`PyDictKeysObject` 也有 free list（`keys_free_list`）**，但**仅当哈希表的 `dk_log2_size == 3`（表大小为 8 槽）且是 unicode 键表**时才缓存。这正好覆盖“空字典 / 小字典 / 关键字参数字典”的键对象。大表或非 unicode 键表不缓存，避免浪费内存、保证命中率。
- `PyDict_New` 时 `new_dict` 弹 `free_list` 复用对象壳；`new_keys_object` 若 `keys_numfree` 有货且大小为 8 就复用 `PyDictKeysObject`。
- 同样可用 `WITH_FREELISTS=0` 编译关闭（此时两个 MAXFREELIST 为 0）。

**第一性原理**：与 list 缓冲池同理，但 dict **多缓存了一层 `PyDictKeysObject`**。因为**小字典（8 槽）的键对象创建极其频繁**（函数调用传 `**kwargs`、小配置字典等），且其大小固定（8 个索引 + 条目区），非常适合缓存。条件限制（必须 8 槽且 unicode 键）体现了**“只缓存固定形态、高频使用的对象”**的取舍——否则缓存大表或杂键表会浪费内存且命中率低。

**工业实际**：`DICT_MAX_WATCHERS = 8` 意味着最多注册 8 个字典观察器（CPython 内部用掉一些给 PEP 659 特化解释器，剩余可供调试器 / profiler / APM 用）。

---

### 5.5 Hack PyDictObject

**书里**：改 `dictobject.c`，观察 `ma_table` 的散列分布、验证探测序列。

**现代零改动实验（推荐）**：

1. **验证插入有序（3.7+ 语言规范）**：
    
    ```
    d = {}
    d['z'] = 1; d['a'] = 2; d['m'] = 3
    print(list(d.keys()))   # ['z', 'a', 'm']
    ```
    
      
    迭代顺序 = `dk_entries` 的插入顺序（**不是哈希序**），直接验证“紧凑字典 entries 按插入追加”。
2. **观察扩容触发点（初始 8 槽，可用 5）**：
    
    ```
    import sys
    d = {}
    for i in range(12):
        d[f"key{i}"] = i
        print(len(d), sys.getsizeof(d))
    ```
    
      
    会看到 `sys.getsizeof(d)` 在插入**第 6 个键**时跳跃（8 槽表 `dk_usable=5`，第 6 个触发扩容到 16 槽），然后在第 11 个键（16 槽 `USABLE_FRACTION(16)=10`）再次跳跃。（`sys.getsizeof` 对 dict 返回 `PyDictObject + PyDictKeysObject`（含 indices 和 entries）的总大小，所以能看到增长。）
3. **检测 split table（键共享，PEP 412）**：
    
    ```
    import sys
    class A: pass
    a = A(); a.x = 1; a.y = 2
    b = A(); b.x = 3; b.y = 4
    print(sys.getsizeof(vars(a)))          # 分离表/共享键，较小（约 104~112）
    print(sys.getsizeof(dict(vars(a))))    # 强制转 combined 字典，较大（约 232~240）
    ```
    
      
    在 3.10 及以前，`del a.x` 会破坏共享，之后新建的 `c = A()` 也变大；在 **3.11+**​ 删除属性不再破坏共享。也可用 `ctypes` 读 `ma_values`（偏移 40）：非 NULL 即 split table。
4. **键类型污染实验（性能陷阱）**：对一个纯字符串键字典，一旦插入或查找**一个非字符串键**（哪怕是失败的 `d[1]` 触发 `KeyError`），内部的 `lookdict_unicode` 特化会**永久**降级为通用 `lookdict`，后续所有字符串键查找慢约 **30%**，且**不可逆**。所以：不要在以字符串为主的 dict 里混入非字符串键（包括用 `1 in d` 做试探性检查）。

**改源码实验（`Objects/dictobject.c`）**：

- 在 `lookdict` / `_Py_dict_lookup` 的探测循环里 `printf("probe i=%zu perturb=%zx\n", i, perturb)`，亲眼看到扰动探测序列。
- 在 `dictresize` 里打印 `ma_used * 3` 与 `calculate_log2_keysize` 的返回值，验证“翻倍”。
- 在 `PyDict_New` / `dict_dealloc` 里打印 `numfree` / `keys_numfree`，观察小字典键对象的复用。

**工业实际（dict 相关）**：

- **键要同质**：全用字符串键（享受 unicode 特化：条目不带 `me_hash` 字段省 8 字节/条目，比较直接用缓存哈希）。混入非字符串键永久降级 30%。
- **插入有序**：3.7+ 可依赖此做配置 / JSON 序列化保序（`json.dump` 保持插入序）。
- **省内存**：大量同类小对象用 **`__slots__`**​ 去掉 `__dict__`（省 50%–70% 内存）；或至少保证在 init__ 里一次性设完所有属性，以稳定享受 PEP 412 的键共享优化，避免后续动态增删属性导致共享键退化成 combined table，从而让每个实例都背负完整的 232 字节字典开销。**

### 5.5 Hack PyDictObject（续：工业实际与总结）

**工业实际（接上，并补充 free-threaded 与 C 扩展要点）**：

- **`PyDict_GetItem` vs `PyDict_GetItemRef`（3.13+ free-threaded）**：旧 API 返回借用引用，在并发修改下可能悬空；新 API 返回强引用（ borrowed → owned），用完后需 `Py_DECREF`。写 C 扩展支持 free-threaded 时必须迁移。
- **字典推导式优于循环赋值**：`{k: v for k, v in iterable}` 在字节码层面是 `BUILD_MAP` + `MAP_ADD`，比 `d = {}; for k, v in iterable: d[k] = v` 少一次 `STORE_SUBSCR` 的属性查找，且能更好地触发内部快速路径。
- **`json.loads` 与有序字典**：由于 3.7+ dict 保序，`json.load` 默认就能保持 JSON 对象的键顺序（虽然 JSON 标准不保证顺序，但 Python 实现会保留插入序）。如果需要 `OrderedDict` 的特定语义（相等性比较考虑顺序），才用 `object_pairs_hook=OrderedDict`。
- **监控与调试**：在 free-threaded 构建下，如果多个线程频繁修改同一个大字典，探测链的锁竞争会成为瓶颈。此时应考虑**分片字典**（按 key hash 分到多个小 dict）或改用 `queue.Queue` 做生产者-消费者隔离。

---

## 第 4、5 章合体总结：可变容器的第一性原理

### 主线：可变性的代价与优化对偶

|维度|list（第 4 章）|dict（第 5 章）|
|---|---|---|
|**核心结构**​|连续 `PyObject*` 数组|开放寻址哈希表（索引 + 条目分离）|
|**增长因子**​|**1.125 倍**（省内存，复制便宜）|**2 倍**（省 rehash，复制昂贵）|
|**缓冲池**​|对象壳 free list（80）|对象壳 free list（80）+ 键对象 free list（80，仅 8 槽 unicode）|
|**线程安全（free-threaded）**​|对象锁保护 append/pop/setitem|对象锁保护单个键值操作，迭代无锁|
|**关键优化**​|过度分配、均摊 O(1)|紧凑表示、键共享（split table）、扰动探测|

### 三个必须带走的洞见

**洞见一：增长因子不是玄学，是操作成本的显式表达。**

list 的 1.125 和 dict 的 2，背后是 `memcpy` 指针 vs `rehash` 全部条目的成本差异。当你自己设计缓存或池化策略时，问自己：**“扩容一次要搬多少数据？”**​ 答案直接决定你的增长常数。

**洞见二：CPython 的优化史，就是一部“把稀疏性从昂贵结构搬到廉价结构”的历史。**

- 2.5 的 dict：稀疏大数组（24 字节/空槽）。
- 3.6+ 的 dict：廉价索引（1–8 字节/槽） + 密集条目。
- 字符串的 PEP 393：按内容选宽度，避免固定 4 字节的浪费。
- 整数 immortal：把计数从“每次写”变成“永不写”。  
    **本质都是同一招：让不变量（空槽、小整数、ASCII 字符串）尽可能少占空间，把变化的部分集中管理。**

**洞见三：共享的前提是“不变集”。**

- 小整数池：值域不变 → 可共享。
- intern 字符串：标识符集不变 → 可共享。
- dict 的 split table：类的属性名集合不变 → 键可共享。  
    **一旦不变集被打破（删除属性、混入非字符串键），共享就退化。**​ 这是所有缓存/共享机制的共同命运——**不是 bug，是设计上的优雅降级**。

### 立刻可做的实验（按性价比排序）

1. **list 扩容观测**：运行 4.4 节的 `sys.getsizeof` 循环，亲手画出容量跳跃点（1, 5, 9, 17...），验证 1.125 倍。
2. **dict 扩容观测**：运行 5.5 节的 `sys.getsizeof` 循环，观察第 6 个键（8 槽满）和第 11 个键（16 槽满）时的跳跃。
3. **split table 验证**：用 `sys.getsizeof(vars(a))` vs `sys.getsizeof(dict(vars(a)))` 感受共享键的省内存效果。
4. **键污染实验**：对一个纯字符串字典，执行 `1 in d` 触发 `KeyError`，然后用 `timeit` 测后续字符串查找，对比污染前后的性能差异（约 30% 退化）。
5. **ctypes 读对象头**：用 4.4 节的代码读 `ob_size` 和 `allocated`，把抽象的“过度分配”变成可见的内存布局。

### 版本差异速查表（贴书边）

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`intobject.c` / `PyIntObject`|`longobject.c` / `PyLongObject`|
|`stringobject.c` / `PyStringObject`|`unicodeobject.c` / `PyUnicodeObject`|
|`listobject.c`：`free_list[80]`|`listobject.c` + `pycore_freelist.h`：`_Py_list_state.free_list[80]`|
|`dictobject.c`：单体稀疏表 `ma_table`|`dictobject.c` + `pycore_dict.h`：紧凑表 `ma_keys->dk_indices` + `dk_entries`|
|`ma_version_tag`|3.12+ 变为 `_ma_watcher_tag`（内部 watchers 用）|
|无 free-threaded|3.13+ 有偏引用计数、每对象锁、`*Ref` API|

