
## 开篇：第 8、9 章的共同主线——"静态代码"如何变成"动态执行"

第 7 章我们得到了 `PyCodeObject`——编译期的产物，包含了字节码和所有静态信息。但 code object 自己**不会动**：它没有变量值、没有求值栈、不知道 `globals` 是谁。第 8、9 章要解决的问题就是——**code object 如何在"执行环境"中被驱动起来**。

这两章的第一性原理可以凝练成一句话：**code object 是"死"的蓝图，`PyFrameObject` 是"活"的实例；字节码是 PVM 的机器语言，表达式求值就是 PVM 对栈的操作**。

延续双轨阅读法：

- **轨道一（机制，跨版本稳定）**：code object + frame + 栈式求值 + LEGB 名字查找。这条主线从 Python 1.x 到现在 3.14 都没变。
- **轨道二（策略，频繁演化）**：frame 的内部布局（3.11 起公开 `PyFrameObject` 结构体成员被移除，改用 `_PyInterpreterFrame` + 稳定 API）、求值循环的分派方式（3.11 自适应解释器、3.13 实验性 JIT、3.14 tail-call 解释器）、`f_locals` 语义（3.13 PEP 667 变成写入穿透代理）。

---

## 第 8 章 Python 虚拟机框架

### 8.1 Python 虚拟机中的执行环境

**书里（Python 2.5）的执行环境 = `PyFrameObject`**：

```
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;     // 调用者的帧，形成帧链表
    PyCodeObject *f_code;      // 对应的 PyCodeObject
    PyObject *f_builtins;      // 内建名字空间（字典）
    PyObject *f_globals;       // 全局名字空间（字典）
    PyObject *f_locals;        // 局部名字空间（字典）
    PyObject **f_valuestack;   // 求值栈栈底
    PyObject **f_stacktop;     // 求值栈栈顶
    // ... f_lasti, f_lineno, f_blockstack, f_localsplus 等
    PyObject *f_localsplus[1]; // 变长区域：局部变量+cell+free+栈空间
} PyFrameObject;
```

书里的核心洞察：**PVM 执行的不是 `PyCodeObject`，而是 `PyFrameObject`**。code object 提供"程序"（字节码 + 静态数据），frame 提供"运行时的全部上下文"（名字空间 + 栈 + 指令指针）。每次函数调用都创建新帧，通过 `f_back` 串成帧链表，模拟 x86 的调用栈。

**现代（3.14）的执行环境演化**：

- **3.11 起 `PyFrameObject` 的公开结构体成员被整体移除**，成为不透明结构体（opaque struct），只能通过稳定 API 访问：`PyFrame_GetCode()`、`PyFrame_GetGlobals()`、`PyFrame_GetBuiltins()`、`PyFrame_GetLocals()`、`PyFrame_GetBack()`。
- 内部实现改为 `_PyInterpreterFrame`（在 `Include/internal/pycore_frame.h`），大多数帧**不再单独堆分配**，而是在每线程的栈上连续分配（参见 `_PyThreadState_PushFrame`），极大提升了局部性和性能。只有生成器/协程的帧嵌入在 `PyGenObject` 里。
- 当 `sys._getframe()` 或回溯被触发时，才会**按需**创建 Python 可见的 `PyFrameObject` 包装（堆分配），并拷贝 `_PyInterpreterFrame` 的状态。这是一种"lazy materialization"优化。
- 帧的内存布局（现代 `_PyInterpreterFrame`）是 **Specials → Locals → Stack**：specials 区固定大小（含 globals/builtins/code/instruction pointer 等），locals 紧随其后，stack 在最后。这种布局让解释器只需维护 frame pointer 和 stack pointer 两个寄存器。

**第一性原理**：

- **执行环境 = 静态程序 + 动态状态**。code object 是静态的（字节码、常量、名字表），frame 是动态的（变量值、栈、指令指针）。两者的分离使得同一个函数可以被递归调用多次（多个 frame 共享一个 code object），也使得协程/生成器可以"暂停-恢复"（frame 不销毁，保留在堆上）。
- **为什么帧要形成链表？**​ 因为 Python 没有 x86 那样的硬件栈指针，`f_back` 就是软件版的 `%rbp`——函数返回时靠它找到调用者。这也意味着 Python 的回溯（traceback）可以轻松遍历整个调用链。
- **为什么 3.11 要把帧分配在线程栈上？**​ 因为绝大多数帧生命周期短暂（函数调用开始到结束），线程栈分配/释放几乎是零成本（只移动 `rsp`），而堆分配要走过内存分配器。只有需要"逃逸"的帧（生成器、被 `sys._getframe()` 捕获的）才上升到堆——典型的"快速路径优化"。

**洞见**：理解 frame 是现代 Python 性能优化的关键。3.11 的性能飞跃（相比 3.10 快约 25%），帧分配从堆改到线程栈贡献了相当一部分。当你在生产环境看到"栈溢出"或"递归深度超限"时，本质上就是帧链表达到了 C 栈或 Python 递归限制的天花板。

---

### 8.2 名字、作用域和名字空间

**书里（Python 2.5）的视角**：

- 三个名字空间：`f_locals`（局部）、`f_globals`（全局）、`f_builtins`（内建），都是 `PyDictObject`。
- 名字查找遵循 **LEGB 规则**：Local → Enclosing → Global → Built-ins。
- 编译期通过符号表分析确定每个名字属于哪种作用域，决定它进入 `co_varnames`（最快，数组下标访问）、`co_cellvars`/`co_freevars`（闭包）、`co_names`（最慢，字典查找）。

**现代（3.14）的视角（主线不变，细节强化）**：

- LEGB 规则依然是 Python 语义的核心，但底层实现更精细：
    - **Local**：对函数而言，`f_locals` 不是真正的字典，而是 `f_localsplus` 数组的视图。变量访问通过 `LOAD_FAST` 等指令直接下标访问数组，**不碰字典**。
    - **Enclosing（闭包）**：通过 cell 对象实现。外层函数的局部变量如果被内层函数引用，就会被"升级"为 cell（在堆上分配，允许多个帧共享）。
    - **Global**：`f_globals` 是模块的 `__dict__`，真实字典。
    - **Builtins**：`f_builtins` 指向 `builtins` 模块的 `__dict__`，真实字典。
- **3.13 PEP 667 的重要变化**：`frame.f_locals` 不再返回真实的底层映射，而是返回一个 **write-through proxy**（`PyFrameLocalsProxy_Type`）。对代理的修改会**实时反映**到帧的实际局部变量中，反之亦然。这修复了长期存在的"修改 `f_locals` 不影响实际局部变量"的怪异行为。

**第一性原理**：

- **作用域是编译期的概念，名字空间是运行期的容器**。编译期决定"这个名字去哪个名字空间找"（静态），运行期才真正去容器里取值（动态）。这就是为什么 Python 是"静态作用域、动态查找"的语言。
- **LEGB 不是四个平等的字典查找**——Local 是数组下标 O(1)，Enclosing 是通过 cell 指针间接 O(1)，Global/Builtins 才是真正的字典 O(1) 平均但常数更大。所以"局部变量访问远快于全局变量"的本质是**数组 vs 哈希表**的差异。
- **闭包的成本**：每次外层函数被调用，都要在堆上为被引用的局部变量创建 cell 对象。这就是为什么在热循环里频繁创建闭包会有开销。

**工业实际**：

```
import dis

def outer():
    x = 10
    def inner():
        return x  # x 被闭包捕获，x 变成 cell
    return inner

co_outer = outer.__code__
print(co_outer.co_cellvars)   # ('x',) —— x 被"升级"为 cell
print(co_outer.co_consts)     # 包含 inner 的 code object

dis.dis(outer.__code__.co_consts[-1])  # 反汇编 inner
# 会看到 LOAD_DEREF 而非 LOAD_FAST —— 通过 cell 间接访问
```

**性能法则**：

- 热循环里把全局变量绑定到局部：`cos = math.cos` 然后 `cos(x)`，可以把 `LOAD_GLOBAL` + `LOAD_ATTR` 两次字典查找变成一次 `LOAD_FAST` 数组访问，性能提升 2-5 倍。
- 尽量避免在热路径上创建闭包；如果必须，考虑用 `__slots__` 类或 `functools.partial` 代替。

---

### 8.3 Python 虚拟机的运行框架

**书里（Python 2.5）的主循环**：

```
PyObject *PyEval_EvalFrameEx(PyFrameObject *f, int throwflag) {
    // 1. 初始化：从 f->f_code 取出 co_code
    // 2. 三个指针遍历字节码：
    //    first_instr —— 永远指向字节码序列开头
    //    next_instr —— 下一条待执行指令
    //    f->f_lasti —— 上一条已执行指令的偏移
    // 3. 主循环：
    while (1) {
        // 取指令 → 解码 opcode + oparg → switch(opcode) 分派
        // 每条指令操作 f->f_valuestack
    }
}
```

**现代（3.14）的主循环（依据 `InternalDocs/interpreter.md`）**：

```
// 入口：PyEval_EvalCode() 构造帧 → _PyEval_EvalFrame()
// 默认实现：_PyEval_EvalFrameDefault()

_Py_CODEUNIT *first_instr = code->co_code_adaptive;
_Py_CODEUNIT *next_instr = first_instr;

while (1) {
    _Py_CODEUNIT word = *next_instr++;
    unsigned char opcode = _Py_OPCODE(word);
    unsigned int oparg = _Py_OPARG(word);
    switch (opcode) {
        // ... 每条 opcode 一个 case
    }
}
```

关键演化：

- **字节码是 16 位 code unit**：每条指令 2 字节，opcode 在前、oparg 在后（与机器字节序无关）。
- **指令集由 DSL 生成**：`Python/bytecodes.c` 用专门 DSL 定义所有 opcode，`ceval.c` 的主循环 `switch` 由这个 DSL 自动生成（不再是手写的大 `switch`）。
- **PEP 523 可插拔**：默认 `_PyEval_EvalFrame` 调用 `_PyEval_EvalFrameDefault`，但可以通过 `interp->eval_frame` 替换为自定义帧执行器（这是 PyTorch 的 TorchDynamo、PyPy 的 C-API 兼容层等的基石）。
- **3.11+ 自适应解释器**：主循环在执行中会改写字节码为专用化版本（如 `LOAD_ATTR` → `LOAD_ATTR_SLOT`）。
- **3.14 tail-call 解释器**：每个 opcode handler 以尾调用下一个 handler 的方式编写，消除中央分派的循环开销，性能再提升 3-5%。

**求值栈操作示例**（表达式 `x = 2 + 3 * 4`）：

```
LOAD_CONST 2    | []          → [2]
LOAD_CONST 3    | [2]         → [2, 3]
LOAD_CONST 4    | [2, 3]      → [2, 3, 4]
BINARY_MULTIPLY | [2, 3, 4]   → [2, 12]   （弹出 3,4，压入 12）
BINARY_ADD      | [2, 12]     → [14]      （弹出 2,12，压入 14）
STORE_NAME x    | [14]        → []        （弹出 14，存入名字空间）
```

**第一性原理**：

- **PVM 是栈式虚拟机**：指令不直接操作寄存器，而是从求值栈取操作数、把结果压回栈。这与 x86 等寄存器式机器相反。栈式的优势是指令短小（2 字节）、编译器简单；劣势是栈操作有额外开销。
- **主循环 = 取指 → 解码 → 分派**。这个三元组是所有虚拟机（JVM、BEAM、PVM）的共同骨架。Python 的性能演化史，本质上是"如何让这三步更快"的历史：3.6 的 wordcode（定长指令让解码 O(1)）、3.11 的自适应解释器（让分派针对类型特化）、3.13 JIT（让热点代码跳过解释器直接执行机器码）、3.14 tail-call（让分派变成直接跳转）。

**工业实际**：

- **理解主循环是性能调优的基石**：任何减少栈操作、减少字典查找、增加类型稳定性的改写，都能在 PVM 层面获得加速。
- **`sys.setprofile` / `sys.settrace` 的代价**：这两个钩子会让 PVM 在每条指令边界做回调，性能暴跌 100 倍。生产环境绝对禁用，只在调试时用。
- **PEP 523 框架**：如果你想实现自己的帧执行器（比如做字节码级 APM、DSL 加速），可以通过 `PyThreadState_GetEvalFrameDefault()` / `PyThreadState_SetEvalFrameDefault()` 替换 `eval_frame` 指针。

---

### 8.4 Python 运行时环境初探

**书里（Python 2.5）的启动流程**：

1. `Py_Initialize()`：创建 `PyInterpreterState`（解释器状态）和初始 `PyThreadState`（线程状态）。
2. 初始化内建模块（`builtins`）、`sys` 模块。
3. 初始化 `builtins` 名字空间（把 `print`、`len`、`dict` 等塞进去）。
4. 创建 `__main__` 模块，执行用户的 `__main__` 字节码。

**现代（3.14）的运行时状态层次**：

```
PyRuntimeState           # 进程级（真正全局）
└── PyInterpreterState   # 解释器级（每个子解释器独立）
    ├── builtins 模块
    ├── sys 模块
    ├── import 系统
    ├── 内存分配器状态
    └── PyThreadState    # 线程级（每个 OS 线程一个）
        ├── 当前帧（frame chain）
        ├── 异常状态（exc_type/value/traceback）
        ├── 递归深度
        └── eval_frame 指针（PEP 523）
```

**关键现代演化**：

- **多解释器隔离（PEP 554，3.12+）**：每个 `PyInterpreterState` 完全隔离（独立的 GIL、独立的 `builtins`、独立的模块表）。这是 Python 实现真正并行（绕过 GIL）的方向之一。
- **每个线程的帧栈**：现代实现里，帧在线程栈上连续分配（`_PyThreadState_PushFrame`），避免了每帧的堆分配。
- **3.13+ 的"免 GIL"构建（PEP 703，实验性）**：通过 `./configure --disable-gil` 编译的 Python，每个 `PyInterpreterState` 有自己的 GIL，多线程可以真正并行。这是 Python 并发模型的重大演化。

**第一性原理**：

- **运行时环境的层次性反映了"作用域层次"**：进程 → 解释器 → 线程 → 帧 → 代码块。每一层都有自己的状态和生命周期。理解这个层次，才能理解为什么 `global` 声明影响的是模块级名字空间、为什么子解释器之间不共享 `sys.modules`、为什么多线程在 CPython 默认构建下受 GIL 约束。
- **`builtins` 名字空间的特殊性**：它是所有模块的"最后一道查找"。每个帧的 `f_builtins` 通常都指向同一个 `builtins` 模块的 `__dict__`（除非用 `exec` 自定义）。这就是为什么你在任何地方都能直接调用 `print`、`len`——它们在内建名字空间里。

**工业实际**：

- **Web 服务多租户隔离**：可以用子解释器（PEP 554）实现真正的隔离，每个租户一个解释器，避免 `sys.modules` 污染。mod_wsgi、uWSGI 等 WSGI 服务器早就用了类似技术。
- **`site.py` 与 `PYTHONSTARTUP`**：Python 启动时 `site.py` 会把 `site-packages` 加到 `sys.path`，`PYTHONSTARTUP` 指定的脚本会在交互模式初始化时执行。理解这个机制才能排查"为什么我的模块导入不了"。
- **嵌入式 Python**：当你把 Python 嵌入到 C/C++ 应用时，`Py_Initialize()` / `Py_Finalize()` 的配对调用、线程状态的创建与切换（`PyGILState_Ensure` / `PyGILState_Release`）是必须掌握的。

---

## 第 9 章 Python 虚拟机中的一般表达式

### 9.1 简单内建对象的创建

**书里（Python 2.5）的简单对象创建**：

- 整数：`LOAD_CONST` 从 `co_consts` 取常量（小整数可能走 `PyInt_FromLong` 的小整数池）。
- 字符串字面量：同样是 `LOAD_CONST`，从 `co_consts` 取（intern 机制可能复用）。
- 编译期常量折叠：`2 + 3` 在编译期被折叠为 `5`，字节码里根本看不到加法。
- `None` / `True` / `False`：特殊的单例对象，`LOAD_CONST` 取到的是同一个对象。

**现代（3.14）的简单对象创建**：

- **常量折叠依然在编译期完成**：`dis` 模块可以验证——`x = 1 + 2` 编译出的字节码是 `LOAD_CONST 3`，而不是 `LOAD_CONST 1; LOAD_CONST 2; BINARY_ADD`。
- **字面量的 interning**：
    - 字符串：标识符（变量名、属性名）一定被 intern；普通字符串字面量在编译期去重（相同字面量共享 `PyUnicodeObject`）。
    - 整数：Python 3 的 `int` 是不限定位数的 `PyLongObject`，小整数池机制依然存在（默认 -5 到 256），但这个范围可以通过 `Py_LONG_CACHE` 编译选项调整。
    - `None` / `True` / `False`：单例，`LOAD_CONST` 取到的永远是同一个对象（身份相等 `is`）。
- **`LOAD_CONST` 的底层**：从 `co_consts` 元组里取下标为 oparg 的元素压栈。因为 `co_consts` 是元组（数组），所以是 O(1) 的数组访问。

**第一性原理**：

- **"简单对象"的创建成本趋近于零**：因为编译期已经完成了对象创建和常量折叠，运行期只是 `LOAD_CONST` 一次数组访问。这就是为什么 `x = 1000000` 和 `x = 1` 在字节码层面完全一样快——对象在编译期就存在于 `co_consts` 里了。
- **常量折叠是第一层优化**：编译器在 AST → 字节码阶段就计算所有编译期可确定的表达式。`1+2+3+4` 变成 `LOAD_CONST 10`，而不是三次加法。
- **interning 是空间换时间**：相同的字面量共享一个对象，既省内存，又让 `is` 比较和字典键查找更快（身份比较比等值比较快）。

**工业实际**：

```
import dis

# 常量折叠验证
dis.dis(compile("x = 1 + 2 * 3", "<string>", "exec"))
# 输出：LOAD_CONST 7 —— 1+2 * 3 已被折叠为 7

# 字符串 interning 验证
a = "hello"
b = "hello"
print(a is b)  # True（编译期 interning）

# 但运行时拼接的字符串不一定 intern
c = "hel" + "lo"
print(a is c)  # 通常 True（常量折叠 + interning）
d = "hel"
e = d + "lo"
print(a is e)  # False（运行时创建新对象）
```

---

### 9.2 复杂内建对象的创建

**书里（Python 2.5）的复杂对象创建**：

- **list 字面量**：`BUILD_LIST` 指令，从栈上弹出 n 个元素构建 list。
- **tuple 字面量**：`BUILD_TUPLE`，类似 `BUILD_LIST` 但不可变。
- **dict 字面量**：`BUILD_MAP` + `STORE_MAP`（2.5 时代），现代改成 `BUILD_MAP` + 键值对在栈上成对出现。
- **set 字面量**：`BUILD_SET`。
- **属性访问**：`LOAD_ATTR` / `STORE_ATTR`，涉及哈希表查找。
- **下标访问**：`BINARY_SUBSCR` / `STORE_SUBSCR`。

**现代（3.14）的复杂对象创建**：

- **列表/元组/字典/集合字面量**：对应 `BUILD_LIST` / `BUILD_TUPLE` / `BUILD_MAP` / `BUILD_SET`。这些指令从栈上收集元素，调用对应的 C API 构造对象。
- **字典字面量的现代形态**：`{a: b, c: d}` 编译为：
    
    ```
    LOAD_NAME a
    LOAD_NAME b
    LOAD_NAME c
    LOAD_NAME d
    BUILD_MAP 2     # 从栈上取 4 个元素（2 对键值），构建字典
    ```
    
- **关键演化：自适应专用化（3.11+）**：
    - `LOAD_ATTR` 在观察到属性访问稳定时，会被改写为 `LOAD_ATTR_SLOT`（slot 属性）、`LOAD_ATTR_MODULE`（模块属性）、`LOAD_ATTR_INSTANCE_VALUE`（实例 `__dict__` 属性）等专用化版本。
    - `BINARY_OP` 在观察到操作数类型稳定时，会被改写为 `BINARY_OP_ADD_INT`、`BINARY_OP_ADD_FLOAT` 等，绕过通用的类型分派。
    - 专用化过程：冷代码（泛型 opcode）→ 预热（执行 8 次后变 adaptive）→ 专用化（稳定后变 specialized）。遇到新类型时**反专用化（de-specialize）**回退到泛型。
- **3.13 实验性 JIT**：当 Tier 1 专用化字节码成为热点，会被翻译成 Tier 2 的 uops（微指令），再 JIT 编译为机器码（copy-and-patch 技术）。

**第一性原理**：

- **复杂对象创建的本质是"栈上元素收集 + C 构造函数调用"**。`BUILD_*` 系列指令把栈上的元素打包成对象。
- **字面量 vs 运行时构造**：`[1,2,3]` 是 `BUILD_LIST 3`，栈上已有 1、2、3；`list(range(3))` 是 `CALL` 调用 `list` 构造函数。前者更快，因为省去了函数调用开销。
- **自适应专用化是"类型稳定性"的胜利**：Python 是动态类型语言，但实践中大多数变量的类型是稳定的（一个循环里的 `i` 始终是 int）。PVM 通过观察这种稳定性，把"每次都要查类型、分派操作"变成"第一次观测后直接走快路径"。这是 Python 3.11 性能飞跃的核心。

**工业实际**：

```
import dis

def f(obj):
    return obj.value

dis.dis(f, adaptive=True, show_caches=True)
# 第一次：LOAD_ATTR（泛型）
f(SomeClass(value=42))
f(SomeClass(value=42))
dis.dis(f, adaptive=True, show_caches=True)
# 第二次：LOAD_ATTR_INSTANCE_VALUE（专用化）
```

**性能法则**：

- 热循环里用字面量而非构造函数：`[0]*100` 比 `for i in range(100): lst.append(0)` 快得多。
- 保持类型稳定：在循环里混用 `int` 和 `float` 会导致 `BINARY_OP_ADD_INT` 反复 de-specialize，性能损失约 15%。
- **`__slots__` 的隐藏收益**：有 `__slots__` 的类，其属性访问走 `LOAD_ATTR_SLOT` 专用化路径（通过固定偏移访问），比 `LOAD_ATTR_INSTANCE_VALUE`（字典查找）更快。

---

### 9.3 其他一般表达式

**书里（Python 2.5）的其他表达式**：

- **二元运算**：`BINARY_ADD` / `BINARY_SUBTRACT` / `BINARY_MULTIPLY` 等，从栈弹两个操作数，调用 `PyNumber_Add` 等，结果压栈。
- **一元运算**：`UNARY_NEGATIVE` / `UNARY_NOT`。
- **比较运算**：`COMPARE_OP`，参数表示比较类型（<、<=、==、!=、>、>=、in、not in、is、is not）。
- **布尔运算**：`POP_JUMP_IF_FALSE` / `POP_JUMP_IF_TRUE`，短路求值通过条件跳转实现。
- **条件表达式**：`x if cond else y` 编译为 `cond` 求值 + `POP_JUMP_IF_FALSE` + 两条分支。
- **lambda**：`MAKE_FUNCTION`，从 `co_consts` 取出 code object 创建函数对象。

**现代（3.14）的演化**：

- **统一算术指令**：3.12 起，所有二元运算合并为 `BINARY_OP`，参数表示具体操作（5 代表乘法等）。这使得自适应专用化框架更统一。
- **比较运算**：`COMPARE_OP` 依然保留，但参数编码方式改变。
- **布尔短路**：`POP_JUMP_IF_FALSE` / `POP_JUMP_IF_TRUE` 仍是核心指令。
- **lambda / 函数定义**：`MAKE_FUNCTION`，但 3.11+ 增加了多种 flag 支持 `posonlyargs`、`kwdefaults`、`annotations` 等新特性。
- **3.11+ 的 `RESUME` 指令**：每个函数帧的开头都有 `RESUME 0`，用于生成器/协程的恢复。

**第一性原理**：

- **表达式是"栈机的舞蹈"**：每个表达式对应一串栈操作指令。复杂表达式的求值过程就是栈的动态变化过程。理解这一点，就能预测任何 Python 表达式的性能特征。
- **短路求值是"条件跳转"**：`a and b` 不是函数调用，而是 `LOAD a; POP_JUMP_IF_FALSE; LOAD b` 的指令序列。这就是为什么 `a and b` 中如果 `a` 为假，`b` 根本不会被求值（也不会有副作用）。
- **统一 `BINARY_OP` 的智慧**：3.12 把几十个二元运算 opcode 合并为一个，参数区分操作类型。这让自适应专用化框架只需处理一个 opcode，专用化版本通过参数区分（`BINARY_OP_ADD_INT`、`BINARY_OP_ADD_FLOAT` 等）。这是"以抽象换优化空间"的典型设计。

**工业实际**：

```
import dis

# 短路求值验证
dis.dis(compile("a and b", "<string>", "exec"))
# LOAD_NAME a
# POP_JUMP_IF_FALSE 2   # 若 a 为假，跳转到结尾
# LOAD_NAME b

# 条件表达式
dis.dis(compile("x if cond else y", "<string>", "exec"))
# LOAD_NAME cond
# POP_JUMP_IF_FALSE L
# LOAD_NAME x
# JUMP_FORWARD M
# L: LOAD_NAME y
# M: ...

# 统一 BINARY_OP（3.12+）
def add(a, b): return a + b
dis.dis(add)
# LOAD_FAST 0
# LOAD_FAST 1
# BINARY_OP 0 (+)    # 参数 0 表示加法
# RETURN_VALUE
```

**洞见——表达式性能的三个层级**：

1. **编译期可确定**​ → 常量折叠，运行期零成本（`1+2 * 3` → `7`）
2. **类型稳定**​ → 自适应专用化，绕过类型分派（`BINARY_OP_ADD_INT`）
3. **类型多态**​ → 泛型 opcode，完整类型分派 + 字典查找（最慢）

写出高性能 Python 代码的本质，就是**尽可能让表达式落到第 1 或第 2 层级**。

---

## 第 8、9 章合体总结：PVM 的执行模型

|维度|书里（2.5）|现代（3.14）|
|---|---|---|
|**执行环境**​|`PyFrameObject`（堆分配，含 f_back/f_code/f_builtins/f_globals/f_locals）|`_PyInterpreterFrame`（线程栈分配）+ 按需创建 `PyFrameObject`（不透明结构体）|
|**主循环**​|手写大 `switch`|DSL 生成 + 自适应解释器 + tail-call 分派|
|**字节码**​|不定长指令|定长 2 字节 wordcode（opcode + oparg）|
|**名字查找**​|LEGB + 字典查找|LEGB + 数组下标（局部）/ cell（闭包）/ 字典（全局、内建）|
|**f_locals 语义**​|直接返回局部变量字典|PEP 667 写入穿透代理（3.13+）|
|**表达式求值**​|栈式操作 + 泛型 opcode|栈式操作 + 自适应专用化 + 实验性 JIT|
|**运行时层次**​|解释器状态 → 线程状态 → 帧|运行时 → 解释器（多解释器隔离）→ 线程 → 帧|

### 三个必须带走的洞见

**洞见一：frame 是"代码"与"执行"的婚姻。**

code object 提供"程序"（静态），frame 提供"上下文"（动态）。两者的结合才产生执行。这就是为什么同一个函数可以被递归调用、为什么生成器可以暂停-恢复、为什么闭包可以捕获外层变量——所有这些"魔法"的本质，都是 frame 在堆上存活并持有状态。

**洞见二：LEGB 不是四个平等的字典，而是四个不同成本的查找路径。**

Local（数组下标 O(1)）< Enclosing（cell 指针间接 O(1)）< Global/Builtins（哈希表 O(1) 平均但常数大）。写出高性能 Python 的第一步，就是把热路径上的 Global/Builtins 查找降级为 Local 查找（`cos = math.cos` 技巧的底层原理）。

**洞见三：自适应专用化是"类型稳定性"在工程上的胜利。**

Python 是动态类型语言，但实践中类型往往是稳定的。PVM 通过观察这种稳定性，把"每次都做类型分派"变成"第一次观测后走专用化快路径"。3.11 的 25% 性能提升、3.12 的统一 opcode、3.13 的实验性 JIT，都是这条主线的延伸。**理解自适应专用化，你就拿到了现代 Python 性能调优的钥匙**。

### 立刻可做的实验

1. **观察帧链**：
    
    ```
    import sys
    def f():
        frame = sys._getframe()
        while frame:
            print(frame.f_code.co_name, frame.f_locals)
            frame = frame.f_back
    def g(): f()
    g()
    ```
    
2. **验证常量折叠**：
    
    ```
    import dis
    dis.dis(compile("x = 1 + 2 * 3 - 4 / 2", "<string>", "exec"))
    # 只有一个 LOAD_CONST，值是 5.0
    ```
    
3. **观察自适应专用化**：
    
    ```
    import dis
    def add(a, b): return a + b
    for _ in range(10):
        add(1, 2)
    dis.dis(add, adaptive=True, show_caches=True)
    # BINARY_OP 变成 BINARY_OP_ADD_INT
    ```
    
4. **LEGB 实战**：
    
    ```
    import dis
    glob = "global"
    def outer():
        enc = "enclosing"
        def inner():
            local = "local"
            return local, enc, glob, len  # Local, Enclosing, Global, Builtin
        return inner()
    dis.dis(outer.__code__.co_consts[-1])  # 反汇编 inner
    # 会看到 LOAD_FAST (local), LOAD_DEREF (enc), LOAD_GLOBAL (glob, len)
    ```
    

### 版本差异速查

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`PyFrameObject` 公开结构体成员|3.11+ 成为不透明结构体，用 `PyFrame_GetCode/Globals/Builtins/Locals` 访问|
|帧堆分配|3.11+ 大多数帧在线程栈上连续分配（`_PyInterpreterFrame`）|
|不定长字节码|3.6+ 定长 2 字节 wordcode|
|`BINARY_ADD` 等具体操作码|3.12+ 统一为 `BINARY_OP`（参数区分操作）|
|泛型 opcode 直接执行|3.11+ 自适应专用化（`BINARY_OP_ADD_INT` 等）|
|`f_locals` 返回真实字典|3.13+ PEP 667 写入穿透代理|
|单层解释器|3.12+ PEP 554 多解释器隔离|
|GIL 全局唯一|3.13+ PEP 703 可禁用 GIL（实验性）|
|纯解释执行|3.13+ 实验性 JIT（Tier 2 uops → 机器码）|

