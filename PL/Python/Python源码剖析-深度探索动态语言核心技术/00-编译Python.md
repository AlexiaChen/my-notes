
# 第 0 章 读书笔记：Python 源码剖析——编译 Python

## 开篇：先给这一章定个位

陈儒这本书 2008 年出版，剖析的是 **Python 2.5**​ 的 CPython 源码——作者在后记里明确说过，2006 年底 Python 2.5 发布后，他把全部剖析内容又跟 2.5 对照了一遍，所以"你手上这本书跟最新的 Python 实现是一致的"。这句话在 2008 年成立，在 2026 年已经是一句需要加注释的话了。

我建议你采用一套**双轨阅读法**，这是本笔记贯穿全书的读法：

- **轨道一（骨架）**：把 Python 2.5 当作"最小可理解模型"。2.5 的运行时更简单——int 和 long 还是两个类型、str 和 unicode 还是两个类型、解释器是单一 Tier、没有 specialization、没有 JIT、GIL 铁板一块。正因为简单，它的**骨架**反而比现代 CPython 更清晰。
- **轨道二（校验）**：每读完一节，用现代 CPython（3.13/3.14）检验"这个结论还成立吗"。英文资料在这里是刚需，因为 CPython 的演进几乎全部以 PEP 和 devguide 为准。

**第一性原理（本章总纲）**：读源码书的目标不是"记住实现"，而是建立一个**可被证伪**的心智模型，然后用"编译 → 修改 → 观测"的闭环去验证它。第 0 章交付给你的不是知识，是一台**实验装置**。

---

## 0.1 Python 总体架构

### 这一节在讲什么

书里给出的是那张经典三层图（数据流 + 两条支撑总线）：

- **左侧：Module**——Python 自带的标准库 + 用户自定义模块（可消费的输入）
- **中间：核心解释器 Interpreter**——Scanner（词法）→ Parser（语法）→ Compiler（生成字节码）→ Code Evaluator（执行字节码）
- **右侧：运行时环境 Runtime**——对象/类型系统（Object/Type structures）、内存分配器（Memory Allocator）、Python 当前状态（Current State）

执行路径是：源码 → 词法/语法分析 → 字节码 → 虚拟机逐条执行；执行过程中不断与对象系统、内存分配器双向交互。

### 第一性原理：把这个架构拆到不能再拆

**原理 A：任何运行时 = 状态 + 状态转移规则。**

CPython 的三块正好各司其职：Object/Type + Memory 是"状态"，Interpreter 是"转移规则"，Modules/Code 是"被转移规则消费的输入"。这个三元分解适用于你以后要读的 **任何**​ 语言运行时（JVM、V8、Ruby/YARV、Lua）。

**原理 B：动态语言的全部代价，集中在"执行前不知道操作数类型"这一件事上。**

CPython 的解法是 `PyObject`——一个所有对象共享的最小通用头（引用计数 + 类型指针）。有了它，"不知道类型"也能安全地做状态转移：先查类型指针，再分派到对应行为。你在后面每一章看到的复杂度，几乎都是这条原理的推论或补丁。

**原理 C：字节码不是性能手段，而是"抽象层的可序列化边界"。**

它把"编译"和"执行"解耦，才让 pyc 缓存成为可能。更重要的是——**正因为指令流是一个可以被读写的中间表示，后来的运行时自我优化才有了操作的抓手**。这是理解现代 CPython 的钥匙（见下）。

### 洞见：书里是开环流水线，现代 CPython 是闭环反馈系统

这是本章最需要升级的认知。现代执行模型已经是分层自适应的：

```
Tokenizer → PEG Parser → AST → Compiler(CFG) → bytecode
   → Tier 1 解释器（执行中观察类型、就地替换指令 = quickening/specialization）
   → Tier 2（uop trace 优化）
   → Tier 3（copy-and-patch JIT 生成机器码）
```

关键在于：**执行不再只读字节码，它会写回字节码**。PEP 659 的 specialization 就是在 `co_quickened`（一份可变的指令副本，因为 `co_code` 本身是 bytes、不可变）上把通用指令替换成特化指令。

所以请把书里那张静态架构图，在脑中升级成一条**带反馈回路的闭环**：编译一次、执行中持续再编译自己。这个转变是 CPython 近五年（Faster CPython 项目，2021 年由微软赞助 Guido 团队发起，目标 4 年 5 倍提速）最本质的变化。

### 工业实际

- **免费的午餐**：3.11 因 PEP 659 specialization + 零开销 frame，pyperformance 平均提速约 25%；3.12/3.13 继续小幅递进。你升级解释器版本就自动拿到，不需要改一行业务代码——这是过去十年 ROI 最高的 Python 性能优化。
- **PEP 659 的粒度选择是工程智慧的典范**：它把优化粒度压到**单条指令**，于是去优化（deopt）不可能发生在区域中间，实现极其简单；而且它**刻意不做 JIT**，为的是在 iOS 这类因 codesign 禁止动态生成代码页的平台上也能用。

---

## 0.2 Python 源代码的组织

### 这一节在讲什么（书中目录 vs 现代目录）

书里 Python-2.5 的目录是：`Demo / Doc / Grammar / Include / Lib / Mac / Misc / Modules / Objects / Parser / PC / PCbuild / PCbuild8 / Python / RISCOS / Tools`。

现代 CPython（3.14）的官方 devguide 给了一份高度稳定、与之一脉相承的划分：

|目录|职责|现代变化|
|---|---|---|
|`Grammar/`|语法定义|`python.gram`(PEG) + `Tokens`，**语法的唯一真源**​|
|`Parser/`|词法/语法分析|3.9 起是 PEG parser（`parser.c` 为生成物 + `peg_api.c` + `tokenizer/` + `Python.asdl`）|
|`Python/`|**核心运行时**​|`ceval.c`(Tier1 解释器)、`compile.c`、`symtable.c`、`bytecodes.c`(字节码 DSL)、`gc.c`、`import.c`、`jit.c`|
|`Objects/`|所有内建类型|`longobject.c`=int、`unicodeobject.c`=str、`listobject.c`、`dictobject.c`、`codeobject.c`、`frameobject.c`、`genobject.c`|
|`Modules/`|C 实现的 stdlib|`_json.c`、`_struct.c`、`mathmodule.c`…|
|`Include/`|C API 头文件|已分裂为 `cpython/`（公开）与 `internal/`（`pycore_*.h`，私有）|
|`Lib/`|纯 Python stdlib（含 `test/`）|—|
|`Programs/`|可执行程序入口（含 `python.c` 的 main）|—|
|`PC/` `PCbuild/`|Windows 特有代码 / MSBuild 工程|现代为 `.vcxproj`|
|`Tools/`|维护工具|新增 `peg_generator/`、`build/`、`c-analyzer/` 等|

### 第一性原理：目录结构就是架构决策的化石

**原理 D：目录划分严格对应"抽象层级"和"变化频率"。**

`Objects/` 是数据模型（变化慢）、`Python/` 是执行引擎（变化快）、`Modules/` 是可插拔的 I/O 与算法、`Include/` 是契约（最稳定）、`PC/PCbuild` 是平台适配层（隔离变化）。**读懂目录结构，等于读懂了一半架构决策。**

**原理 E：`Include` 是"契约层"，这是 CPython 可扩展性的架构级来源。**

它定义了 C 扩展与解释器之间的 ABI 边界。现代把它显式拆成 `Include/cpython/`（你可以依赖的公开 API）和 `Include/internal/`（`pycore_*`，明天就可能变）——这是一次非常重要的架构收紧：**明确区分"承诺"与"实现细节"**。

### 必须知道的两次"地图漂移"（书 → 现代）

**漂移一：类型文件改名。**​ 这是最容易浪费你两小时的地方。

- `int` 在 `Objects/longobject.c`（不是 `intobject.c`）——因为 3.x 把 int/long 统一了
- `str` 在 `Objects/unicodeobject.c`（不是 `stringobject.c`）——因为 3.x 把 str/unicode 统一了
- 书里对应的 Python 2.5 源码确实是 `Objects/intobject.c` 与 `Include/stringobject.h`（`PyStringObject` 带 `ob_shash`、`ob_sstate`）
- 其他例外：`sys` 在 `Python/sysmodule.c`，`marshal` 在 `Python/marshal.c`，`winreg` 在 `PC/winreg.c`

**漂移二：Parser 从 pgen/LL(1) 换成了 PEG。**

- 2.5 时代是 LL(1) + pgen（书里说它"与 YACC 非常类似"，是能自动生成词法/语法分析器的工具）
- 3.9 起换成 PEG（**PEP 617**），`Grammar/python.gram` 成为语法唯一真源，旧 LL(1) 在 3.10 被彻底删除
- 意义：PEG 天然无二义性、有序选择、支持左递归（生成器的 packrat memoization + cut 优化），这才让 3.10 的 `match` 语句这类新语法能干净地加入

### 原理延伸：现代 CPython 大量使用"DSL → 生成 C 代码"

`Grammar/python.gram` 生成 `Parser/parser.c`；`Python/bytecodes.c`（一份用 `inst/op/macro/family` 宏写的 DSL）生成 `Python/generated_cases.c.h`。

**所以读现代 CPython 的第一条纪律：你看到的很多 `.c` 文件是产物，不是源头。改它们会在下次 regen 时被覆盖。**​ 这条纪律在 0.5 节会变成具体操作。

---

## 0.3 Windows 环境下编译 Python

### 这一节在讲什么

Python 2.5 同时提供 VS2003（`PCbuild`）和 VS2005（`PCbuild8`）两套工程文件。书里的步骤是：打开 `pcbuild.sln` → 把 Startup Project 从默认的 `_bsddb` 改成 `Python` → 在配置对话框里取消无关子工程、**只保留 `pythoncore` 和 `python`**​ → 此时直接编译仍会失败，必须先单独编译 `make_buildinfo` 和 `make_versioninfo` 生成缺失的文件，再全量编译。

### 第一性原理：这节的真正价值不在 VS2003 的操作细节

**原理 F：构建系统的复杂度 ≈ 平台差异 × 可选依赖的组合爆炸。**

Windows 上 VC 版本、SDK 版本、zlib/openssl/sqlite 等第三方库缺失，是全部摩擦的来源。书里"裁剪子工程、只留最小核心"的做法，本质是**降低问题维度的工程直觉：先构造一个最小可验证系统，再逐步把变量加回来**。

这个思路比 VS2003 的点击顺序有价值一万倍，而且永不过时——它同样适用于今天的 Windows 构建、乃至任何复杂系统的调试。

### 现代 Windows 构建（3.14）

- **要求**：Visual Studio 2017 或更高（3.11 起要求 C11 编译器，Windows 上即 VS2017+；现在主流用 VS2022）
- **流程**：`git clone` → `cd PCbuild` → `build.bat`（现代 build.bat 会自动下载 nuget 依赖到 `externals/`，含 openssl/libffi/zlib/sqlite 等；旧教程里需要先手工跑 `get_externals.bat`）
- **常用参数**：`-c Debug|Release`、`-p x64|win32|ARM64`、`-t Build|Rebuild`；ARM64 自 3.11 起原生支持
- **产物**：在 ``PCbuild\amd64\`（``python.exe`/`python_d.exe`）；也可以在 VS 里打开`pcbuild.sln` 继续开发
- **坑**：安装时勾选 "Python development" workload + "Python native development tools"；注意 Windows SDK 与 platform toolset 的重定向；离线环境要预先准备 `externals`

### 工业实际

Windows 上自编译最常见的真实动机，按价值排序：

1. **调试 C 扩展崩溃**——官方 release 没有断言和完整符号，必须自建 debug 构建（`python_d.exe` + `python3XX_d.dll`）配合 VS 调试器定位段错误、引用计数泄漏。
2. **打包定制解释器**——给 Windows 服务/客户端嵌入、静态链接自家 C 扩展、或在 air-gapped 环境产出与官方 ABI 一致的 `python3xx.dll`。
3. **打安全补丁 / 移植到特定平台**。

**实践建议：两个二进制并存。**​ 用官方 release（本身就是 PGO+LTO 构建）跑性能和生产，用自编译 debug 构建做正确性与调试，绝不混用。

---

## 0.4 Unix/Linux 环境下编译 Python

### 这一节在讲什么

2.5 时代就是经典三步：`./configure` → `make` → `make install`。

### 现代构建（devguide 官方流程）

```
git clone <你的 fork>; git remote add upstream https://github.com/python/cpython
./configure --with-pydebug
make -s -j$(nproc)
./python          # 就地运行，通常根本不需要 install
```

devguide 特别强调：**"你应该永远在 pydebug 构建下开发，唯一的例外是做性能测量"**。

### 关键 flag 全景（这是现代比 2.5 丰富得多的地方）

|Flag|作用|代价|
|---|---|---|
|`--with-pydebug`|断言、引用计数追踪、额外类型检查|慢，但可观测|
|`--enable-optimizations`|PGO（+ 可选 LTO）|构建极慢，只用于性能|
|`--with-lto`|链接期优化|构建慢|
|`--disable-gil`|**free-threaded 构建**（3.13+，PEP 703），产物 `python3.13t`|ABI 不兼容，单线程有回退|
|`--enable-experimental-jit`|3.13+ copy-and-patch JIT（`=interpreter` 可只启用 uop 解释器便于调试）|实验性|
|`--with-tail-call-interp`|3.14+ 新解释器|需 Clang 19+、x86-64/AArch64，官方强烈建议配 PGO|
|`--prefix=$HOME/...` + `make altinstall`|与系统 Python 并存|**绝不覆盖系统 python3**​|
|out-of-tree (`mkdir build && cd build && ../configure`)|多配置并存|—|

**make 生态**（现代远比 2.5 丰富）：

- `./python` 就地运行；`make test` / `./python -m test -j8 -uall`；`make buildbottest`
- `make regen-all`：重新生成所有生成文件（parser、字节码表、opcode ids、AST…）
- `make regen-pegen` / `make regen-cases` / `make clinic`（Argument Clinic 胶水代码）
- `make pythoninfo`：提交 bug 时的诊断 dump
- `make clean` / `make distclean`

### 第一性原理

**原理 G：构建配置 = 你在选择"要观察哪个宇宙"。**

同一份源码树：`--with-pydebug` 给你可观测性、丢掉速度；`--enable-optimizations` 给你速度、丢掉可观测性；`--disable-gil` 给你并行、丢掉部分单线程性能；`--enable-experimental-jit` 给你一条尚未成熟的执行路径。**理解每个 flag 背后的 trade-off，比记住 flag 本身重要。**

### 工业实际：三个"前沿特性"的现实版本

**① free-threading（去 GIL）**

- 3.13 实验性（需 `--disable-gil`，产物 `python3.13t`）。当时 specialization 在 free-threaded 下被禁用，单线程回退 20–40%。
- **3.14 转正为官方支持**，specialization 重新启用，单线程回退降到 **5–10%**。
- 致命约束：ABI 不兼容，所有 C 扩展必须重建并用 `Py_mod_gil` 槽位声明支持；**未声明的扩展在被 import 时会自动把 GIL 重新打开**——这是个优雅降级，但它会悄悄抵消你正在测的并行收益。
- 建议：先在非关键路径试点，用 **pyperformance + 你自己的业务 benchmark**​ 双重验证。别信网上"快 10 倍"的单一 benchmark。

**② JIT（copy-and-patch，PEP 744）**

- 3.13 起实验性，默认关闭，官方二进制不含（需 `--enable-experimental-jit` 自编译）。
- **核心开发者 Ken Jin 2025 年公开表示："JIT 常常比解释器还慢"**——用 Clang 20 构建时，解释器往往胜过 JIT。目前 pyperformance 上只是个位数百分比。
- 这是**极好的第一性原理案例**：copy-and-patch 在 CPython 构建时就用 LLVM 把每个 uop 编译成机器码"模板(stencil)"，运行时只做"复制 + 填洞"。LWN 报道里的原话是"分配内存的时间几乎是 JIT 生成代码时间的两倍"——快到离谱，但代价是放弃了跨区域的深度优化（内联、寄存器分配、跨函数分析）。
- **选它不是因为它最快，而是因为它可被一个小团队维护。**​ 理解了这个 trade-off，你就能预测 Python 性能演进的形状：渐进、务实、工程驱动，而不是激进重写。

**③ 尾调用解释器（3.14）**

- 传统 CPython 用一个巨大的 C `switch/case` 分派字节码；新实现改为**在实现各 opcode 的小 C 函数之间做尾调用**。
- 官方数据：pyperformance 几何平均 **3–5%**（部分字节码密集型负载报告更高）；仅支持 Clang 19+ 的 x86-64/AArch64；**必须配合 PGO**，这是官方唯一验证过的配置；opt-in，且不改变任何可见行为。
- 注意别混淆：这不是 Python 层面的尾调用优化（TCO），CPython 至今不支持 TCO。

---

## 0.5 修改 Python 源代码

### 这一节在讲什么

书里把"带领读者一起动手对 Python 虚拟机进行改造"作为卖点——后面每一章末尾的 `Hack PyXXXObject` 就是这个精神的落点。0.5 讲的是：改完之后怎么重新编译，以及从哪儿开始改。

### 第一性原理：修改是唯一能证伪"我以为我懂了"的手段

**原理 H：阅读会给你"一致性错觉"。**

源码读起来很顺，不代表你能预测它改了会怎样。只有 **提出假设 → 改一行 → 重新编译 → 观测差异**​ 这个闭环，才是真正的学习。这是第 0 章对整本书方法论最大的贡献。

**原理 I（我自己的提法）：把解释器当成"实验装置"，而不是"文本"。**

最优的改动不是加功能，而是**加探针**——在关键路径（对象分配、字节码分派、GC）上加计数器或日志，把不可见的运行时行为变成可观测数据。这比"hack 出一个新语法"更能建立正确的心智模型。

### 现代必须知道的操作纪律

1. **改源头，不改生成物**：`Parser/parser.c` 由 `python.gram` 经 `Tools/peg_generator` 生成；`Python/generated_cases.c.h` 由 `Python/bytecodes.c` 生成。直接改会在 regen 时丢失。
2. **正确的 regen 路径**：改 `Grammar/python.gram` → `make regen-pegen`；改 `Python/bytecodes.c` → `make regen-cases`；不确定就 `make regen-all`。
3. **加 opcode 的完整链路**：`bytecodes.c` 里用 `inst/op/macro/family` 这套 DSL 定义（还要考虑 specialization family），再到 Tier 2 的 uop 表示，再到 JIT stencil 生成。
4. **加新 C 文件**：改 `Makefile.pre.in` 与 Windows 的 `PCbuild/*.vcxproj`。
5. **参数解析用 Argument Clinic**（`Tools/clinic`）→ `make clinic`。
6. **测试**：`./python -m test test_xxx`；提交前跑 `Tools/patchcheck` 与 pre-commit。

### 三个最值得做的起步实验（按性价比排序）

1. **用 `dis` 亲眼看见 specialization**：写个 `def f(a, b): return a + b`，调用几次后执行 `dis.dis(f, adaptive=True, show_caches=True)`。会看到 `BINARY_OP` 变成 `BINARY_OP_ADD_INT`。warmup 阈值在 3.11 起默认是 8 次执行。**这是"看见"0.1 那条闭环反馈的最短路径。**
2. **在 `Objects/longobject.c` 的分配路径加 `printf`**：观察小整数池（`-5..256`）的命中与 free list 的复用。
3. **改 `Python/bytecodes.c` 里一个 `inst` 的语义并 regen**：亲手体会"单一真源 → 生成"的范式。

### 工业实际

企业里"修改 CPython"的真实形态通常**不是**改核心，而是：写 C 扩展 / 用 Cython / 用 PyO3(Rust)。改核心的高价值场景只有两类：**把性能 patch 提回上游**（远比自己维护 fork 便宜），或**为特定硬件/平台做移植**。

**成本提醒**：fork CPython = 长期承担安全补丁与版本升级债务。devguide 存在的意义就是让你把 patch 提上游。

**一个更实用的中间路线**：完全不改 CPython，用 `sys.settrace` / `sys.monitoring`（3.12+）/ audit hooks 做观测与治理。在 APM、沙箱、供应链安全这些场景里，这比改解释器实用得多。

---

## 0.6 通往 Python 之路

书里给的是学习路径建议（先懂架构 → 逐个对象 → 虚拟机 → 高级话题）。我把它映射成**现代版路线图**，把书的目录直接对接到 3.14 的现实：

|书的章节|现代对照|需要打的补丁|
|---|---|---|
|第 1–5 章 内建对象|`longobject.c` / `unicodeobject.c` / `listobject.c` / `dictobject.c`|int/str 已统一；dict 保序（3.7+ 语言规范保证）；3.12+ 引入 immortal objects，部分常量的生命周期管理变了|
|第 7 章 Code 对象与 pyc|`codeobject.c`、`marshal`、`bytecodes.c`、`dis`|用 `dis(adaptive=True)` 才能看见特化后的指令|
|第 8–12 章 虚拟机|`ceval.c` = Tier 1|必须额外读 specialization（`specialize.c` + `bytecodes.c`）；3.11 起 frame 布局改为堆上的 chunked 数组，Python→Python 调用不再消耗 C 栈|
|第 13–16 章 高级话题|`pylifecycle.c`、`import.c`、`obmalloc.c`、`gc.c`|GIL → 可选（3.13/3.14）；GC 在 3.14 起改为**增量式**，大堆的最大暂停时间大幅缩短|

**书之外必须补的第五阶段**：PEP 阅读（**659**​ specialization、**617**​ PEG parser、**703**​ free-threading、**684**​ 每解释器 GIL、**554**​ 多解释器、**734**​ 3.14 多解释器标准库、`concurrent.interpreters`）、Faster CPython 的 ideas 仓库、pyperformance 基准、devguide。

### 英文资料清单（专治"书已过时"）

- **官方**：`devguide.python.org`（Setup and building / Directory structure）；`docs.python.org/3/using/configure.html`（构建要求与全部 flag）；What's New 3.9/3.11/3.13/3.14
- **PEP**（一手权威）：PEP 659、PEP 617、PEP 703、PEP 744
- **书/课程**：Real Python《CPython Internals》及其 _Your Guide to the CPython Source Code_；Anthony Shaw 的 CPython Internals
- **深度报道**：LWN 的 copy-and-patch JIT 分析；faster-cpython ideas 仓库；PyCon 上 Mark Shannon / Brandt Bucher / Ken Jin 关于 specialization 与 JIT 的演讲

### 第一性原理（学习法）

**原理 J：先建立"最小可运行心智模型"，再用现实版本打补丁，而不是反过来。**

2.5 的简单性在这个方法论下反而是**资产**。先吃透 2.5 的骨架，再叠加"3.x 改了什么"，比一上来啃 3.14 的三层执行模型要省力得多。

**原理 K：区分"机制"与"策略"。**

- **机制（跨版本稳定，值得投入 80% 精力）**：对象头、引用计数、栈式 VM、code object / pyc、符号表与作用域。
- **策略（频繁变化，只花 20% 精力跟踪）**：GIL、specialization、JIT、GC 算法、解析器实现。

把精力押在机制上，你就能准确说出"书里哪些内容永远有效、哪 20% 已经过期"——这才是读一本老书的正确姿势。

---

## 0.7 一些注意事项（现代升级版清单）

书里的提醒我就不复述了，直接给一份**2026 年版避坑清单**：

1. **版本漂移**：int→`longobject.c`、str→`unicodeobject.c`、LL(1)→PEG、单 Tier→三 Tier、GIL→可选、GC→增量式、有了 JIT。每一章读完先问一句："这个结论在 3.14 还成立吗？"
2. **绝不覆盖系统 Python**：用 `make altinstall`，或干脆就地 `./python` 不安装。
3. **`--enable-shared` 的幽灵陷阱**：就地运行工作副本时别开它，否则可能意外链接到旧的系统 libpython——你会调试一个不存在的解释器。
4. **pydebug 构建 ≠ 性能构建**：devguide 明确"唯一不做 pydebug 的场合是测性能"。测性能用官方 release 或 `--enable-optimizations`。
5. **区分生成物与源头**：别改 `generated_cases.c.h` / `parser.c`，改 `.gram` 与 `bytecodes.c` 后 regen。
6. **缺依赖是静默的**：编译末尾会列出 missing modules，用 `python -c "import ssl, sqlite3"` 之类做冒烟测试。
7. **磁盘与时间**：完整构建 + 测试要数十分钟到数小时，PGO 更久。务必 `make -j$(nproc)`。
8. **前沿特性别当生产力**：JIT 常常更慢；free-threading 单线程仍有 5–10% 开销且 ABI 不兼容；尾调用解释器要 Clang 19+ 且依赖 PGO。**先量后上。**
9. **git 规范**：双 remote（origin=fork, upstream=cpython），Windows 开 `autocrlf`，装 pre-commit。
10. **读源码的纪律**：一次只追**一条纵向切片**（比如"一次函数调用到底发生了什么"），横向铺开必迷路。

---

## 本章小结：三个必须带走的洞见

**① CPython 已从"静态流水线"进化为"运行时自适应闭环"。**

书里第 0 章那张架构图是开环的。现代 CPython 会在执行中重写自己的指令流（quickening），并在热路径继续下沉到 uop 与机器码（Tier 2 / JIT）。读完这本书，你要在脑中把这张图升级成带反馈回路的版本。

**② 这门学科的底层统一原理是：用"观察 + 猜测 + 快速退回来"对抗动态性带来的不确定性。**

PEP 659 是这个原理最干净的表达——把优化粒度压到单条指令，从而让去优化变得平凡（不可能发生在区域中间）；并且刻意不生成机器码，换取在 iOS 等禁止 JIT 的平台也可用。copy-and-patch JIT 是同一原理的下一步，但选了"可维护性优先"而非"峰值性能优先"。**理解了这一个 trade-off，你就能预测 Python 性能演进的形状。**

**③ 本章真正交付的能力不是"会编译 Python"，而是"拥有了一台可证伪的实验装置"。**

从这章开始，你对 Python 的每一个断言，都应该能用 `dis`、pydebug 构建、或一次源码改动来验证。**这是"读源码"与"读博客"的分界线。**

---

### 立刻可做的三件事

1. 按 devguide 跑通 `./configure --with-pydebug && make -j$(nproc) && ./python`（或 Windows 的 `PCbuild\build.bat -p x64`）。
2. 用 `dis.dis(f, adaptive=True)` 亲眼看见 `BINARY_OP` → `BINARY_OP_ADD_INT` 的变形。
3. 打开 `Objects/longobject.c` 和 `Objects/dictobject.c`，各读 200 行——这是全书最友好的起点。

