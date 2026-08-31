
## 开篇：第 7 章的主线——源码如何变成"可执行的代码块"

前 6 章我们都在看**对象**（int/str/list/dict），从第 7 章开始进入**执行模型**：Python 源码不是直接被解释的，而是先被编译成一个**与源码等价、但更接近机器**的中间表示——`PyCodeObject`，再被 Python 虚拟机（PVM）一条条执行。`PyCodeObject` 是"编译结果"，`.pyc` 是它在磁盘上的持久化形式，`co_code` 里的字节码是 PVM 真正消费的指令流。

本书基于 Python 2.5，那时的 `PyCodeObject` 字段、`.pyc` 头部格式（只有 magic + 时间戳）、字节码指令集（不定长、无专用化）都和今天有显著差异。**本章的双轨阅读尤其重要**——书里讲的是"机制"（编译→code object→pyc→字节码这条主线跨版本稳定），但"策略"（字段布局、pyc 失效模式、指令编码、自适应专用化）在 3.6→3.11→3.12→3.14 间发生了多次演化。

最关键的三个现代漂移：

- **`PyCodeObject` 字段大幅重组**：3.11 加入 `co_qualname`、`co_exceptiontable`；3.12 起 `PyCode_New` 被重命名为 `PyUnstable_Code_New`（明确标注为不稳定 API）；字节码从独立的 `co_code` bytes 对象变成内嵌的可变长 `co_code_adaptive` 数组（为专用化腾空间）。
- **pyc 头部从"magic + 时间戳"扩展到 16 字节，支持 PEP 552 三种失效模式**（时间戳 / 校验哈希 / 不校验哈希）。
- **字节码从不定长指令变成定长 2 字节 wordcode（3.6+）**，3.11 起引入**自适应专用化**（adaptive bytecode），3.12 把 `BINARY_ADD` 等合并为统一的 `BINARY_OP`，3.13 加入实验性 JIT，3.14 加入 tail-call 解释器。

---

## 7.1 Python 程序的执行过程

**书里（Python 2.5）的执行流水线**：

1. 解析器（`Parser`）把源码变成**抽象语法树（AST）**。
2. 编译器（`Compiler`）把 AST 编译成 `PyCodeObject`（字节码 + 元数据）。
3. 运行时把 `PyCodeObject` 封装进 `PyFrameObject`，交给 PVM 的 `PyEval_EvalFrame` 主循环执行。
4. 若 `co_code` 里有嵌套的代码对象（函数定义、类定义、lambda、推导式），在执行到对应字节码时再递归创建帧。

**现代（3.14）的流水线（主线不变，细节演化）**：

- **AST → 符号表 → 字节码**：CPython 先建 AST，再做符号表分析（`_PySymtable_Build`）解析作用域（local/global/free/cell），最后遍历 AST 生成字节码。符号表阶段决定了变量落在 `co_varnames`（最快）、`co_cellvars`、`co_freevars` 还是 `co_names`（最慢）。
- **"code object 不等于可执行的函数"**：官方明确定义——_"Each one represents a chunk of executable code that hasn't yet been bound into a function"_。同一个 `PyCodeObject` 可以被多个 `PyFunctionObject` 共享（闭包就是这么实现的：不同 `func_closure` + 同一个 code object）。
- **PVM 主循环**：`_PyEval_EvalFrameDefault` 不断从 `co_code` 取指令、解码、执行。3.11+ 这个主循环变成了"自适应解释器"——指令在执行过程中会被改写成专用化版本（如 `LOAD_ATTR` → `LOAD_ATTR_SLOT`）。3.14 进一步改成 tail-call 解释器（每个 opcode handler 直接尾调用下一个），消除中央分派开销，pyperformance 上提升 3-5%。

**第一性原理**：

- **编译与执行分离**是 Python 性能演化的根基。正因为源码先变成 code object，才有了 pyc 缓存、专用化、JIT 的可能性——如果每次都从头解释 AST，这些优化都不存在。
- **code object 是不可变的（3.11+ 部分可变）**：大多数字段只读，使得它可以安全地在多线程间共享、缓存到磁盘。例外是 `co_code_adaptive`（为专用化可变）和 `_co_monitoring`（运行时监控信息）。
- **frame 是 code object 的一次"实例化"**：code object 是"类"，frame 是"对象"。同一个函数被递归调用 5 次，就有 5 个 frame 共享 1 个 code object。理解了这点，才明白为什么函数的本地变量不冲突——它们存在 frame 里，不在 code object 里。

---

## 7.2 Python 编译器的编译结果——PyCodeObject 对象

**书里（Python 2.5）的 PyCodeObject 核心字段**：

```
typedef struct {
    PyObject_HEAD
    int co_argcount;        // 位置参数个数
    int co_nlocals;         // 局部变量个数
    int co_stacksize;       // 求值栈最大深度
    int co_flags;           // 标志位（CO_GENERATOR 等）
    PyObject *co_code;      // 字节码字符串
    PyObject *co_consts;    // 常量元组
    PyObject *co_names;     // 名字元组（全局/属性）
    PyObject *co_varnames;  // 局部变量名元组
    PyObject *co_freevars;  // 自由变量名元组
    PyObject *co_cellvars;  // cell 变量名元组
    PyObject *co_filename;  // 源文件名
    PyObject *co_name;      // 代码块名（函数名/类名/'<module>'）
    int co_firstlineno;     // 首行号
    PyObject *co_lnotab;    // 行号表（字节码偏移 → 源码行号）
} PyCodeObject;
```

**现代（3.12+，依据 `Include/cpython/code.h`）的 PyCodeObject**：

```
struct PyCodeObject {
    PyObject_HEAD
    // ---- 执行引擎最热的字段放最前面（cache line 友好）----
    PyObject *co_consts;     // 常量元组
    PyObject *co_names;      // 名字元组
    PyObject *co_exceptiontable; // 异常处理表
    int co_flags;
    int co_argcount;
    int co_posonlyargcount;  // 仅位置参数（3.8+）
    int co_kwonlyargcount;   // 仅关键字参数
    int co_stacksize;
    int co_firstlineno;
    int co_nlocalsplus;      // 局部+cell+free 变量总数
    int co_framesize;        // 帧大小（字长）
    int co_nlocals;
    int co_ncellvars;
    int co_nfreevars;
    uint32_t co_version;     // 版本号
    PyObject *co_localsplusnames;
    PyObject *co_localspluskinds;
    PyObject *co_filename;
    PyObject *co_name;
    PyObject *co_qualname;   // 限定名（3.11+）
    PyObject *co_linetable;  // 行号表（3.10+ 紧凑格式，取代 lnotab）
    PyObject *co_exceptiontable; // 异常处理表（3.11+）
    // 末尾：变长字节码数组
    char co_code_adaptive[1]; // 自适应字节码（3.11+）
};
```

**关键演化**：

- **`co_lnotab` → `co_linetable`**：3.10 起因 PEP 626（精确行号/列号）引入新紧凑格式，支持字节码偏移映射到精确的 (start_line, start_col, end_line, end_col)。
- **`co_qualname`**：3.11 加入，存储带嵌套信息的限定名（如 `OuterClass.inner_method`），对递归嵌套的类和函数调试至关重要。
- **`co_exceptiontable`**：3.11 加入，用二进制格式编码异常处理器范围（try/except/finally 的字节码区间），取代了旧的线性扫描。
- **`co_code` → `co_code_adaptive`**：3.11 前 `co_code` 是一个独立的 bytes 对象；3.11 起字节码内嵌到 code object 尾部（flexible array member），允许解释器在运行时**原地改写**指令做专用化。
- **`PyCode_New` 标记不稳定**：3.12 起重命名为 `PyUnstable_Code_New`，官方明确警示——_"Since the definition of the bytecode changes often, calling PyUnstable_Code_New() directly can bind you to a precise Python version"_。这意味着任何依赖构造 code object 的 C 扩展，都必须随 Python 小版本重新编译。

**第一性原理**：

- **code object 是"静态信息"的容器**：它包含了一切执行前就能确定的东西——字节码、常量池、变量名表、栈大小、标志位。**所有动态的东西（变量值、求值栈、异常状态）都在 frame 里**。这种静态/动态分离，是 code object 能被缓存、共享、甚至跨线程安全访问的根本原因。
- **变量名表的"四元组"决定了变量访问速度**：
    - `co_varnames` → `LOAD_FAST`：数组下标访问，最快
    - `co_cellvars` / `co_freevars` → `LOAD_DEREF`：通过 cell 对象间接访问（闭包）
    - `co_names` → `LOAD_GLOBAL` / `LOAD_ATTR`：哈希表查找，最慢
    - 这就是为什么**局部变量访问比全局变量快**——它本质上是数组下标 vs 字典查找的差异。
- **常量池去重**：编译期会对相同字面量去重（两个 `"hello"` 可能共享同一个常量池条目），嵌套的函数/类/推导式的 code object 也作为常量出现在父 code object 的 `co_consts` 里——这构成了 code object 的树状嵌套结构。

**洞见——code object 树的工业意义**：

一个 `.py` 文件编译后产生的顶层 code object，其 `co_consts` 里嵌套着所有函数、类、lambda、推导式的 code object。所以 `marshal.load(pyc文件)` 得到的不仅是一个 code object，而是整棵代码对象树。这是 Python 静态分析工具（如 `ast` + `dis` 组合）能遍历整个模块的基础。

**Python 层观察**（现代）：

```
import dis
def demo(a, b=10, *args, key=None, **kwargs):
    x = a + b
    return x

co = demo.__code__
print(co.co_argcount)        # 2（a 和 b）
print(co.co_kwonlyargcount)  # 1（key）
print(co.co_names)           # ('a', 'b', 'x') —— 实际是 co_varnames
print(co.co_consts)          # (10, None) —— 默认值和 None
print(co.co_stacksize)       # 求值栈最大深度
dis.dis(demo)
```

---

## 7.3 PyCodeObject 对象与 pyc 文件的生成

**书里（Python 2.5）的 pyc 生成**：

- 导入模块时，若 `sys.dont_write_bytecode` 为 False 且源文件有写入权限，CPython 在 `__pycache__`（PEP 3147 之前是直接同目录）写 `.pyc`。
- 旧的 pyc 头部只有 8 字节：`magic(4) + 时间戳(4)`，后面紧跟 `marshal` 序列化的 code object。
- 失效判断：比较源文件的 mtime 与 pyc 头部存的时间戳。

**现代（3.14）的 pyc 生成（PEP 552 + PEP 3147）**：

- 文件路径：`__pycache__/<module>.cpython-<version>.pyc`（如 `demo.cpython-314.pyc`）。
- **16 字节头部**：
    
    ```
    [0..4)   magic number（4 字节，标识 Python 版本+字节码格式）
    [4..8)   bit field（4 字节，PEP 552 失效模式）
    [8..16)  校验字段（8 字节）
    ```
    
- **bit field 的最低位决定失效模式**：
    - `0`：传统时间戳模式。后 8 字节 = (源文件 mtime: u32, 源文件 size: u32)。
    - `1`：哈希模式。bit 1 是 `check_source` 标志。后 8 字节 = SipHash-1-3(源文件内容) 的 64 位哈希值。
- **三种失效模式（PycInvalidationMode）**：
    - `TIMESTAMP`：比较 mtime + size
    - `CHECKED_HASH`：每次导入重新哈希源文件并比对
    - `UNCHECKED_HASH`：信任 pyc，不校验（由外部系统如 Linux 包管理器保证一致性）
- **magic number 几乎每个版本都变**：Python 3.10=3439, 3.11=3495, 3.12=3531, 3.13=3571, 3.14=3627（即 `2b0e0d0a`）。字节码指令集变了，magic 就必须变——这也是为什么 `.pyc` 不能跨版本使用。
- **尾部 0D 0A 的妙处**：magic number 的后两字节固定为 `\r\n`（0D 0A）。如果 pyc 文件被当作文本传输导致换行符被破坏，magic 校验会失败，Python 会丢弃缓存重新编译——这避免了加载损坏的字节码。

**生成 pyc 的现代 API**：

```
import py_compile
from py_compile import PycInvalidationMode

# 时间戳模式（默认）
py_compile.compile('demo.py')

# 哈希模式（可复现构建）
py_compile.compile('demo.py',
                   invalidation_mode=PycInvalidationMode.CHECKED_HASH)
```

`compileall` 也支持 `--invalidation-mode` 命令行选项。

**第一性原理**：

- **pyc 是 code object 的"记忆"**：code object 是编译的产物，pyc 让这个产物跨进程、跨时间存活。没有 pyc，每次导入都要重走"解析→AST→符号表→字节码"全流程。
- **失效模式的演化反映了"构建确定性"的需求**：传统的 mtime 模式依赖文件系统时间戳，在可复现构建（reproducible builds）场景下不够——同样的源码，在不同机器、不同时刻编译出的 pyc 应该字节级相同。PEP 552 的哈希模式正是为此而生，配合 `SOURCE_DATE_EPOCH` 环境变量可实现完全确定性的构建。
- **marshal 格式的不稳定性是刻意设计**：官方明确说 marshal 格式"may change between Python versions"，且反序列化不可信数据是**不安全**的（`marshal.load` 从不信任的输入可能崩溃或被利用）。这印证了一个工程哲学——**内部缓存格式可以随时演进，因为它只对当前 Python 版本负责；跨版本/跨信任边界的序列化请用 pickle**。

**工业实际**：

- **部署时预编译**：Docker 镜像构建阶段跑 `python -m compileall`，避免运行时重复编译。配合 `UNCHECKED_HASH` 模式，生产环境导入零哈希开销。
- **可复现构建**：Linux 发行版打包 Python 包时用 `CHECKED_HASH` 或 `UNCHECKED_HASH`，确保构建确定性（Debian/Fedora 的实践）。
- **只读文件系统**：设置 `PYTHONDONTWRITEBYTECODE=1` 或 `-B` 参数禁止写 pyc，适用于只读挂载的场景。
- **跨版本兼容**：永远不要把 A 版本 Python 生成的 pyc 喂给 B 版本——magic number 不匹配会被直接拒绝。

---

## 7.4 Python 的字节码

**书里（Python 2.5）的字节码**：

- **不定长指令**：每条指令 1-3 字节，操作码（opcode）后可能跟 2 字节参数。
- 栈式虚拟机：`LOAD_FAST` / `LOAD_CONST` 压栈，`BINARY_ADD` 等弹出操作数、压回结果。
- 指令集包含 `BINARY_ADD`、`BINARY_MULTIPLY`、`BINARY_SUBTRACT` 等具体操作码。

**现代（3.14）的字节码**：

**1. 定长 2 字节 wordcode（3.6+）**：

- 每条指令恰好 2 字节：1 字节 opcode + 1 字节参数。
- 参数不够用时，用 `EXTENDED_ARG` 前缀指令拼装更大参数（每多一个 `EXTENDED_ARG` 提供 8 位）。
- 好处：指令定位 O(1)，跳转目标计算简单，PVM 主循环更紧凑。

**2. 自适应专用化（3.11+）**：

- 字节码在执行过程中会被**原地改写**为专用化版本。例如 `LOAD_ATTR` 第一次执行时发现 attribute 在类型上稳定，会被改写为 `LOAD_ATTR_SLOT` 或 `LOAD_ATTR_MODULE` 等快速路径。
- `dis` 模块可以通过 `adaptive=True` 显示专用化后的字节码，通过 `show_caches=True` 显示内联缓存条目。
- 每个需要缓存的指令后面跟着若干 `CACHE` 指令（占位 2 字节），作为专用化的存储空间。

**3. 统一算术指令（3.12+）**：

- `BINARY_ADD` / `BINARY_MULTIPLY` / `COMPARE_OP` 等被合并为统一的 `BINARY_OP`，参数表示具体操作（如 `+` 对应 5，`*` 对应 5 的不同子码）。
- 简化指令集，让专用化框架更统一。

**4. 字节码示例（3.12+）**：

```
def square(n):
    return n * n

dis.dis(square)
# 输出：
# 2 RESUME 0
# 3 LOAD_FAST 0 (n)
#   LOAD_FAST 0 (n)
#   BINARY_OP 5 (*)
#   RETURN_VALUE
```

- `RESUME` 是 3.11+ 新增的帧恢复指令（处理生成器/协程恢复）。
- `BINARY_OP 5` 的 `5` 代表乘法操作。
- `LOAD_FAST 0` 表示取 `co_varnames[0]`，即局部变量 `n`。

**5. 字节码与源码位置的精确映射（PEP 626）**：

- 3.10+ 的 `co_linetable` 不仅记录行号，还记录列号。
- 3.11+ 的精确错误提示（如 `SyntaxError` 下划线标注具体表达式）正是依赖这个精确映射。
- `dis` 的 `show_positions=True`（3.14+）可以显示每条指令对应的源码行列。

**第一性原理**：

- **栈式虚拟机的本质是"隐式数据总线"**：指令不直接指定操作数地址，而是从栈上取。这让指令短小（2 字节）、编译器简单，代价是栈操作有开销。这正是 Python 与寄存器式虚拟机（如 Lua 5.x）的核心差异。
- **自适应专用化 = "观察驱动的运行时编译"**：PVM 在第一次执行某条指令时观察操作数类型，若发现热点模式就把指令改写成专用版本。这本质上是 JIT 的轻量级形式——3.11 的自适应解释器是 3.13 实验性 JIT 的基础。
- **wordcode 定长化是一次"以空间换时间"**：定长指令让 PVM 主循环的分派更高效，虽然某些单字节参数用不满 2 字节略有浪费，但换来整体执行速度提升。

**工业实际**：

- **性能调优**：用 `dis.dis` 对比不同写法的字节码长度，是判断性能差异的最直接方式。例如列表推导式 vs for 循环，`dis` 会显示推导式少了 `LOAD_ATTR` + `CALL` 的开销。
- **调试黑魔法**：`code.co_code` 可以直接修改（3.11+ 通过 `co_code_adaptive`），但这属于极度危险的 hack，仅供研究。
- **3.14 新特性**：`python -m dis -S` 显示专用化字节码，`-P` 显示源码位置信息，帮助深入理解 PVM 行为。

---

## 7.5 解析 pyc 文件

**书里的方法**：用 `marshal.load(fp)` 直接反序列化（那时头部只有 8 字节）。

**现代（3.14）的解析步骤**：

```
import marshal
import importlib.util

def parse_pyc(path):
    with open(path, 'rb') as fp:
        magic = fp.read(4)          # magic number
        flags = fp.read(4)          # bit field
        # 后 8 字节的语义取决于 flags
        if int.from_bytes(flags, 'little') & 1:
            # 哈希模式
            hash_value = fp.read(8) # SipHash-1-3
        else:
            # 时间戳模式
            mtime = fp.read(4)      # 源文件 mtime
            size = fp.read(4)       # 源文件 size
        # 剩下的全是 marshal 序列化的 code object
        code = marshal.load(fp)
    return code

# 使用
code = parse_pyc('__pycache__/demo.cpython-314.pyc')
print(code.co_name, code.co_firstlineno)
print(code.co_consts, code.co_names)
```

**关键细节**：

- **必须跳过 16 字节头部**：3.7+ 的 pyc 头部固定 16 字节（PEP 552），不再是 2.5 时代的 8 字节。
- **marshal 是版本绑定的**：`marshal.version` 当前是 5（3.14）。用 A 版本的 Python 去 `marshal.load` B 版本生成的 pyc，行为是未定义的（可能崩溃）。
- **`allow_code=True`**：3.13+ 起 `marshal.load` 需要显式传 `allow_code=True` 才能反序列化 code object，这是安全加固（防止不可信数据触发代码对象加载）。
- **遍历嵌套 code object 树**：

```
import types

def walk_code(co, indent=0):
    print('  ' * indent + f"{co.co_name} (line {co.co_firstlineno})")
    for c in co.co_consts:
        if isinstance(c, types.CodeType):
            walk_code(c, indent + 1)

walk_code(code)
```

**洞见——pyc 解析的工业价值**：

- **静态分析工具的基础**：IDE 的跳转定义、lint 工具、覆盖率工具，本质上都在解析 code object 树。
- **反编译的入口**：uncompyle6、decompyle3 等反编译工具，第一步就是 `marshal.load(pyc)` 得到 code object，然后根据字节码指令集反向构造 AST。
- **热重载（hot reload）机制**：开发服务器（如 Django dev server、Flask debug）检测源文件变化时，重新编译并替换 code object，实现无需重启的代码更新。

**安全警示**：

> ⚠️ **绝不**对不可信来源的 pyc 文件执行 `marshal.load`。官方文档明确："The marshal module is not intended to be secure against erroneous or maliciously constructed data"。加载恶意构造的 pyc 可能导致解释器崩溃甚至任意代码执行。

**3.14 的 pyc 实操**：

```
import py_compile, marshal, importlib.util

# 1. 编译生成 pyc
py_compile.compile('demo.py')

# 2. 查看当前 Python 的 magic number
print(importlib.util.MAGIC_NUMBER)  # b'+\x0e\r\n' for 3.14

# 3. 手工解析
with open('__pycache__/demo.cpython-314.pyc', 'rb') as fp:
    header = fp.read(16)
    magic = header[:4]
    flags = int.from_bytes(header[4:8], 'little')
    code = marshal.load(fp, allow_code=True)

print(f"Magic: {magic.hex()}")
print(f"Flags: {flags:#x}")
print(f"Code object name: {code.co_name}")
print(f"Constants: {code.co_consts}")
```

---

## 第 7 章合体总结：编译结果的三层结构

|层级|书里（2.5）|现代（3.14）|核心职责|
|---|---|---|---|
|**源码 → 中间表示**​|PyCodeObject + co_lnotab|PyCodeObject + co_linetable + co_exceptiontable|静态信息容器|
|**磁盘持久化**​|8 字节头（magic + mtime）|16 字节头（magic + flags + 校验）|pyc 缓存|
|**PVM 指令流**​|不定长字节码|定长 2 字节 wordcode + 自适应专用化|可执行指令|

### 三个必须带走的洞见

**洞见一：code object 是"编译期"与"运行期"的分水岭。**

编译期决定一切静态可知的东西（常量、变量名、栈大小、字节码），运行期只负责动态执行。这种分离使得：

- code object 可以被缓存到磁盘（pyc）
- code object 可以被多线程安全共享
- code object 可以作为一等公民传递（函数是一等公民的根本原因）

**洞见二：pyc 的演化史是"构建确定性"的演进史。**

从 2.5 的 `magic + mtime`，到 3.7 的 PEP 552 三种失效模式，本质是 Python 社区对"可复现构建"需求的响应。当你做 Docker 镜像优化、CI/CD 流水线、Linux 发行版打包时，这个演化直接决定了你的 pyc 策略。

**洞见三：字节码的演化方向是"专用化 + 定长化"。**

3.6 的 wordcode（定长 2 字节）→ 3.11 的自适应专用化 → 3.12 的统一 `BINARY_OP` → 3.13 的实验性 JIT → 3.14 的 tail-call 解释器。每一步都在减少 PVM 主循环的开销，让"解释执行"越来越接近"编译执行"。**理解字节码的现代形态，是理解 Python 性能优化的钥匙**——你写的每一行代码，最终都会变成这些指令的组合，`dis.dis` 是你的显微镜。

### 立刻可做的实验

1. **解析 pyc 三层结构**：
    
    ```
    import marshal, importlib.util
    with open('__pycache__/your_module.cpython-314.pyc', 'rb') as f:
        print("Magic:", f.read(4).hex())
        print("Flags:", int.from_bytes(f.read(4), 'little'))
        print("Validation:", f.read(8).hex())
        code = marshal.load(f, allow_code=True)
    print(code.co_name, code.co_consts, code.co_names)
    ```
    
2. **观察自适应专用化**：
    
    ```
    import dis
    def f(obj):
        return obj.attr
    dis.dis(f, adaptive=True, show_caches=True)
    # 第一次执行前是 LOAD_ATTR，执行后变成 LOAD_ATTR_SLOT 等专用化版本
    ```
    
3. **对比不同写法的字节码**：
    
    ```
    dis.dis(lambda: [x*x for x in range(10)])
    dis.dis(lambda: [x*x for x in range(10)])
    # 对比列表推导式 vs for 循环的字节码差异
    ```
    
4. **递归遍历 code object 树**：
    
    ```
    import types
    def walk(co, depth=0):
        print("  "*depth + co.co_name)
        for c in co.co_consts:
            if isinstance(c, types.CodeType):
                walk(c, depth+1)
    walk(your_module_code_object)
    ```
    

### 版本差异速查

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`co_lnotab`（行号表）|`co_linetable`（3.10+ 紧凑格式，PEP 626）|
|无 `co_qualname`|`co_qualname`（3.11+）|
|无 `co_exceptiontable`|`co_exceptiontable`（3.11+）|
|`co_code` 是 bytes 对象|`co_code_adaptive` 内嵌可变长数组（3.11+）|
|`PyCode_New`|`PyUnstable_Code_New`（3.12+，不稳定 API）|
|pyc 头部 8 字节（magic + mtime）|pyc 头部 16 字节（magic + flags + 校验，PEP 552）|
|时间戳失效|三种模式：TIMESTAMP / CHECKED_HASH / UNCHECKED_HASH|
|不定长字节码指令|定长 2 字节 wordcode（3.6+）|
|`BINARY_ADD` 等具体操作码|统一 `BINARY_OP`（3.12+）|
|无专用化|自适应专用化（3.11+），实验性 JIT（3.13+）|

---

**下一章预告**：第 8 章进入**"Python 虚拟机"**——`PyFrameObject` 的结构、`PyEval_EvalFrame` 主循环、名字空间（local/global/builtin）的查找链、以及函数调用的完整流程。读完第 7 章你知道了"code object 是什么"，第 8 章会告诉你"code object 如何被执行"。重点留意现代 CPython 的 **Tier 1 自适应解释器**和 **Tier 2 uop 微指令**——这是 3.11+ 性能飞跃的核心架构。

需要我按同样的方式继续整理第 8 章吗？