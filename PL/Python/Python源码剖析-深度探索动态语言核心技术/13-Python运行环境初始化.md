
## 开篇：第 13 章的主线——"Python 从按下回车到第一行代码执行"之间发生了什么

前几章我们一直在 PVM 内部游走：

- **code object**​ 是编译期产物 [第 7 章]
- **frame**​ 是运行期实例 [第 8 章]
- **函数、类、控制流**​ 都是字节码层面的机制 [第 10-12 章]

但有一个根本问题没回答：**在你敲下 `python main.py` 到 `main.py` 的第一行字节码被执行之间，CPython 做了什么？**​ 第 13 章要揭开的，是 PVM 自身的"bootstrap"——解释器怎么把自己从零搭起来。

**本章与书里的最大时差**：

- **书里（Python 2.5）**：初始化是 `Py_InitializeEx()` 一个大函数，内部依次做线程、内建模块、sys 模块、main 模块等初始化
- **现代（3.14）**：初始化拆为**三段式生命周期**——预初始化（Pre-Initialize）→ 核心初始化（Core Initialize）→ 主初始化（Main Initialize）。`Py_Initialize()` 的语义被明确为：初始化已加载模块表 `sys.modules`，创建基础模块 `builtins`、`__main__`、`sys`，初始化模块搜索路径 `sys.path`
- **GIL 的初始化**：3.7 起 `Py_Initialize()` 内部自动调用 `PyEval_InitThreads()`；3.9 起 `PyEval_InitThreads()` 变成空操作（deprecated），GIL 的创建完全由 `Py_Initialize()` 接管

**本章双轨阅读法**：

- **轨道一（机制，跨版本稳定）**：解释器启动 = 配置解析 → 运行时状态初始化 → 线程/GIL 初始化 → 基础模块创建 → 导入系统就绪 → 主模块激活 → 字节码执行
- **轨道二（关键演化）**：
    - 配置系统从零散的全局变量（`Py_VerboseFlag` 等）演变为 `PyConfig` 结构体（3.8+ PEP 587）
    - GIL 初始化从"用户手动调用 `PyEval_InitThreads()`"变为"Py_Initialize 自动完成"
    - 子解释器隔离度大幅提升（3.12+ PEP 684 每个子解释器可有独立 GIL）
    - `site.py` 以冻结模块（frozen module）形式内建在解释器二进制中

---

## 13.1 线程环境初始化

**书里（Python 2.5）的视角**：

- `Py_InitializeEx()` 中调用 `PyEval_InitThreads()` 创建 GIL 并初始化主线程状态
- 用户嵌入 Python 时必须手动调用 `PyEval_InitThreads()` 来启用多线程支持
- 全局解释器锁（GIL）是保护 Python 内部状态的互斥锁

**现代（3.14）的视角**：

**关键演化**：GIL 的初始化已经"自动化"。

> _"Changed in version 3.7: This function is now called by Py_Initialize(), so you don't have to call it yourself anymore. Changed in version 3.9: The function now does nothing."_

也就是说，在 3.9+ 中：

- 调用 `Py_Initialize()` 时，GIL 会被自动创建
- `PyEval_InitThreads()` 变成了空操作（为了向后兼容保留）
- 用户不再需要（也不应该）手动调用 `PyEval_InitThreads()`

**线程环境的三个核心数据结构**（依据 CPython 官方文档）：

1. **`_PyRuntimeState`**：全局运行时状态，进程级唯一，包含 GIL 状态、内存分配器等
2. **`PyInterpreterState`**：解释器状态，每个（子）解释器一个，包含 `sys.modules`、`sys.path`、`builtins` 等
3. **`PyThreadState`**：线程状态，每个 OS 线程一个，包含当前执行的 frame、异常状态等

**GIL 的本质**（工业视角的精确定义）：

> _"The GIL is a mutex (a lock) that protects access to Python objects, preventing multiple native threads from executing Python bytecodes simultaneously within the same process. It was introduced to simplify CPython's memory management (reference counting) and prevent race conditions on internal state."_

**单线程程序也有 GIL**——这不是 bug，是设计。GIL 的存在不是为了"支持多线程"，而是为了"让引用计数等内存管理机制在单线程下也不用加锁"。多线程只是让 GIL 的竞争变得显性而已。

**第一性原理**：

- **线程环境初始化 = 建立"运行时-解释器-线程"三级状态树**。最顶层是进程唯一的 `_PyRuntimeState`（包含 GIL），中间是 `PyInterpreterState`（每个子解释器独立），最底层是 `PyThreadState`（每个线程独立）。这三层结构决定了 Python 并发能力的边界。
- **GIL 是"进程级资源"**：在 3.12 之前，同一个进程内的所有子解释器共享同一个 GIL；3.12+ 通过 `PyInterpreterConfig_OWN_GIL` 可以让子解释器拥有独立 GIL——这是 PEP 684 多解释器并行的基础。
- **`PyThreadState` 与 OS 线程是"绑定"关系**：通过线程本地存储（TLS）维护当前线程的 `PyThreadState`。这就是为什么在 C++ 中创建的新线程要调用 Python API，必须用 `PyGILState_Ensure()` 把新线程"附着"到 Python 运行时。

**洞见——为什么嵌入 Python 的多线程 C++ 程序容易死锁？**

```
// 错误写法
Py_Initialize();
// 主线程持有 GIL，然后去干别的事，没有释放 GIL
std::thread t([]{
    PyGILState_STATE gstate = PyGILState_Ensure();
    // 死锁！主线程持有 GIL 不放，子线程永远拿不到
    PyRun_SimpleString("print('hello')");
    PyGILState_Release(gstate);
});
t.join();
```

正确做法是主线程在 `Py_Initialize()` 后调用 `PyEval_SaveThread()` 释放 GIL，让子线程能获取：

```
Py_Initialize();
main_thread_state = PyEval_SaveThread();  // 创建 GIL 并立即释放
// 子线程用 PyGILState_Ensure()/Release() 配对获取/释放 GIL
```

**工业实际**：

```
# 1. 验证 GIL 的存在（单线程程序也有）
import threading
print(threading.current_thread())  # <_MainThread(...)>

# 2. 3.12+ 子解释器独立 GIL 的配置
# PyInterpreterConfig config = {
#     .use_main_obmalloc = 0,
#     .gil = PyInterpreterConfig_OWN_GIL,
#     ...
# }
# Py_NewInterpreterFromConfig(&tstate, &config)
# 这样每个子解释器有自己的 GIL，可以真正并行执行 Python 字节码

# 3. 嵌入 Python 的标准初始化序列（现代写法）
# Py_Initialize();                    # 自动创建 GIL
# PyEval_SaveThread();                # 释放 GIL，让其他线程可获取
# ... 主线程做非 Python 工作 ...
# PyEval_RestoreThread(tstate);       # 重新获取 GIL
# Py_Finalize();
```

---

## 13.2 系统 module 初始化

**书里（Python 2.5）**：在 `Py_InitializeEx()` 中依次调用 `_PyBuiltin_Init()` 创建 `builtins` 模块、`_PySys_Init()` 创建 `sys` 模块，然后初始化 `sys.modules` 和 `sys.path`。

**现代（3.14）的核心事实**（依据官方文档）：

> _"This initializes the table of loaded modules (`sys.modules`), and creates the fundamental modules `builtins`, `__main__` and `sys`. It also initializes the module search path (`sys.path`)."_

**三个基础模块的各自职责**：

**1. `builtins` 模块**——内建名称的"全局仓库"

- 包含 `int`、`str`、`list`、`dict`、`open`、`print`、`len` 等所有内建函数和类型
- 通过 `_PyBuiltin_Init()` 创建
- 用户代码能直接使用 `len()` 而不需要 `import builtins`，是因为编译器把内建名称解析为 `LOAD_GLOBAL` 时，会在 `builtins` 命名空间中查找

**2. `sys` 模块**——解释器的"仪表盘"

- 由 `_PySys_Create()` 创建，这是一个**特殊函数**，不走常规导入流程
- 为什么特殊？因为 `sys` 模块承载了太多核心功能（`sys.modules`、`sys.path`、`sys.argv`、标准流等），必须在导入系统本身可用之前就存在
- `sys.modules` 是所有已导入模块的缓存字典
- `sys.path` 是模块搜索路径

**3. `__main__` 模块**——顶层执行环境

- 用户脚本的 `__name__` 就是 `'__main__'`
- 当运行 `python script.py` 时，该脚本作为 `__main__` 模块执行
- `if __name__ == '__main__':` 惯用法正是基于此

**`sys.modules` 的关键语义**：

- 它是一个字典，映射模块名 → 模块对象
- **所有已导入的模块**（包括解释器启动时预加载的）都在这里
- 导入系统首先检查 `sys.modules`：若模块已存在，直接返回缓存；若不存在，才走 `sys.meta_path` 查找器
- 启动后 `sys.modules` 中已有的模块（如 `sys`、`builtins`、`_imp`、`_thread` 等）**不能直接在用户命名空间中访问**——它们只是被加载到了 `sys.modules` 缓存中，没有绑定到任何命名空间

**内建模块的查找优先级**（解释导入机制）：

> _"The standard `sys.meta_path` has three meta path finders, one that knows how to import built-in modules, one that knows how to import frozen modules, and one that knows how to import modules from an import path."_

这意味着：内建模块 > 冻结模块 > 路径模块。所以即使你创建了一个名为 `sys.py` 的文件，导入 `sys` 时仍然会得到内建模块。

**`site.py` 的特殊性**：

- `site.py` 在初始化后期被显式导入（`PyImport_ImportModule("site")`）
- 它以**冻结模块（frozen module）**形式内建在解释器二进制中
- 这就是为什么你修改磁盘上的 `site.py` 往往不生效——解释器优先加载内建版本
- `site.py` 负责添加 `site-packages` 到 `sys.path`、执行 `sitecustomize.py` 和 `usercustomize.py`

**第一性原理**：

- **系统 module 初始化 = 搭建"导入系统自身"的基础设施**。`builtins` 提供内建名称，`sys` 提供运行时信息查询和模块缓存，`__main__` 提供顶层执行命名空间。`sys.path` 决定后续所有 `import` 的搜索范围。
- **`sys.modules` 是导入系统的"缓存层"**：所有导入都先查这里。这也是为什么循环导入有时能工作——模块在完全初始化前就被放入 `sys.modules`，其他模块可以拿到这个"半成品"模块对象。
- **`sys` 模块的特殊创建路径**：它不能用常规导入流程创建，因为常规导入流程本身就需要 `sys` 模块。这是经典的"鸡生蛋"问题，CPython 用 `_PySys_Create()` 这个绕过常规导入的特殊函数来破解。

**洞见——为什么修改 `site-packages/site.py` 不生效？**

```
import site
print(site.__file__)
# 如果输出 'frozen' 或报错，说明加载的是内建冻结版本
# 只有编译 Python 时禁用 --without-frozen-modules 才会用磁盘版本
```

这个设计是为了**启动性能**——冻结模块直接编译进二进制，避免了磁盘 I/O。在工业部署中，这意味着你不能依赖修改 `site.py` 来做全局定制；应该用 `sitecustomize.py` 或 `usercustomize.py`（它们在 `site.py` 执行时被导入）。

**工业实际**：

```
# 1. 观察启动时预加载的模块
import sys
print(sorted(sys.modules.keys()))
# ['__builtin__', '__main__', '_ast', '_codecs', '_sre', '_warnings',
#  '_weakref', 'abc', 'codecs', 'encodings', 'errno', 'genericpath',
#  'os', 'posixpath', 're', 'signal', 'site', 'stat', 'sys', ...]

# 2. 验证内建模块优先级
import sys
print('sys' in sys.builtin_module_names)  # 查看内建模块名列表
# 即使当前目录有 sys.py，import sys 仍然得到内建模块

# 3. 自定义 sitecustomize.py（全局定制）
# 在 site-packages 目录下创建 sitecustomize.py：
#   import sys, os
#   sys.path.insert(0, os.path.expanduser('~/my_modules'))
# 这会在每次 Python 启动时自动执行（除非用 -S 禁用 site）

# 4. 嵌入 Python 时的模块初始化
# Py_Initialize();
# PyRun_SimpleString("import sys");
# 此时 sys.modules 已就绪，builtins/__main__/sys 均已创建
```

**子解释器的模块隔离**（3.12+ 的重要演化）：

```
// 每个子解释器有独立的 builtins/__main__/sys 和 sys.modules/sys.path
PyInterpreterConfig config = {
    .use_main_obmalloc = 0,
    .allow_fork = 0,
    .allow_exec = 0,
    .allow_threads = 1,
    .allow_daemon_threads = 0,
    .check_multi_interp_extensions = 1,
    .gil = PyInterpreterConfig_OWN_GIL,
};
PyThreadState *tstate = NULL;
Py_NewInterpreterFromConfig(&tstate, &config);
```

> _"The new interpreter has separate, independent versions of all imported modules, including the fundamental modules builtins, main and sys. The table of loaded modules (sys.modules) and the module search path (sys.path) are also separate."_

这意味着子解释器之间**模块完全隔离**——一个子解释器导入的模块不会出现在另一个子解释器的 `sys.modules` 中。这是 Python 3.12+ 多解释器并行（PEP 684）的基石。

---

## 13.3 激活 Python 虚拟机

**书里（Python 2.5）**：`Py_InitializeEx()` 完成后，调用 `PyRun_AnyFile()` 或 `PyRun_SimpleFile()` 执行用户脚本，进入字节码执行循环。

**现代（3.14）的启动流程**：

从命令行 `python main.py` 到执行 `main.py` 的第一行代码，完整流程是：

```
C main()
  → Py_BytesMain() / Py_Main()        # 解析命令行参数
    → Py_InitializeFromConfig()        # 核心初始化
      → pyinit_core()                  # 创建解释器状态、线程状态、GIL
        → _PyBuiltin_Init()            # 创建 builtins 模块
        → _PySys_Create()              # 创建 sys 模块
        → _PyImport_Init()             # 初始化导入系统
      → pyinit_main()                  # 主初始化
        → init_interp_main()           # 设置 sys.path、导入 site
    → Py_RunMain()                     # 激活虚拟机，执行主模块
      → pymain_run_python()
        → PyRun_FileExFlags()          # 编译+执行用户代码
          → _PyAST_Compile()           # AST → 字节码
          → PyEval_EvalCode()          # 进入 _PyEval_EvalFrame()
```

**关键函数语义**（依据官方文档）：

- `Py_Main(int argc, wchar_t **argv)`：标准解释器的主程序，封装完整的初始化/终结周期，并处理命令行和环境变量的配置
- `Py_RunMain(void)`：在完全配置好的 CPython 运行时中执行主模块
- `Py_Initialize()`：不设置 `sys.argv`，需要用 Initialization Configuration API 来配置

**命令行接口的执行语义**（依据官方文档）：

- `python script.py`：脚本作为 `__main__` 模块执行
- `python -c "code"`：`sys.argv[0]` 为 `'-c'`，当前目录加入 `sys.path` 开头
- `python -m module`：在 `sys.path` 中搜索模块，作为 `__main__` 模块执行；若是包，则执行 `<pkg>.__main__`
- `python`（无参数）：进入交互模式，不执行任何用户文件

**`__main__` 模块的语义**：

> _"'**main**' is the name of the scope in which top-level code executes. A module's **name** is set equal to '**main**' when read from standard input, a script, or from an interactive prompt."_

这就是为什么 `if __name__ == '__main__':` 成为 Python 的标准惯用法——它让模块既能被导入（此时 `__name__` 是模块名），又能作为脚本直接运行（此时 `__name__` 是 `'__main__'`）。

**PYTHONSTARTUP 与 sitecustomize**：

- `PYTHONSTARTUP` 环境变量指向的脚本**只在交互模式**下执行
- `sitecustomize.py` 在 `site` 模块初始化时被导入，影响每次 Python 调用（除非 `-S` 禁用 site）
- `usercustomize.py` 在 `sitecustomize.py` 之后导入，用于用户级定制

**第一性原理**：

- **激活虚拟机 = 配置解析 → 运行时初始化 → 主模块定位 → 编译 → 字节码执行**。前三章我们讨论的所有机制（frame、code object、控制流、函数、类），都是在 `PyEval_EvalCode()` 这一步才真正开始运转的。
- **`__main__` 是"执行命名空间"而非"模块文件"**：运行 `python my_program.py` 时，脚本作为 `__main__` 模块执行，而不是作为 `my_program` 模块。这解释了为什么同一个文件直接运行和被导入时行为可能不同。
- **配置与执行的解耦**：现代 CPython 通过 `PyConfig` 结构体把"配置"和"执行"分离。`Py_Main()` 约等于 `PyConfig_InitPythonConfig()` + `Py_InitializeFromConfig()` + `Py_RunMain()`。这种设计让嵌入 Python 的应用可以精细控制初始化行为。

**洞见——为什么 Python 启动慢？**

```
python -c "pass"   # 冷启动通常需要 20-40ms
```

这段时间花在哪里？

1. **预初始化**：设置运行时状态、内存分配器（~1ms）
2. **核心初始化**：创建 builtins/sys/**main**、导入系统（~5ms）
3. **主初始化**：设置 sys.path、导入 site、执行 sitecustomize（~10-20ms）
4. **字节码编译与执行**：AST → 字节码 → 执行（~1ms）

**最大的开销是 `site.py` 的执行**——它要扫描 `site-packages` 目录、处理 `.pth` 文件、导入 `sitecustomize.py`。这就是为什么：

- `python -S`（禁用 site）可以显著加快启动
- 工业界的 CLI 工具（如 PyTorch、TensorFlow 的命令行工具）常用 `python -S` 或自定义 bootstrap 来优化启动时间
- Python 3.11+ 对启动流程做了大量优化，冷启动时间相比 3.10 降低约 10-20%

**工业实际**：

```
# 1. 嵌入式 Python 的最小启动序列
# Py_Initialize();
# PyRun_SimpleString("print('Hello from embedded Python')");
# Py_Finalize();

# 2. 使用 Py_Main 获得完整 CPython CLI 行为
# int main(int argc, char **argv) {
#     return Py_Main(argc, argv);  // 等价于直接调用 python 命令
# }

# 3. 优化启动时间的技巧
# python -S script.py              # 禁用 site，跳过 site-packages 扫描
# PYTHONSTARTUP=~/.pythonrc        # 交互模式预加载
# 使用 sitecustomize.py 替代 .pth 文件（后者代码执行已被弃用）

# 4. 子解释器隔离运行（3.12+）
# 每个子解释器独立运行，互不干扰 sys.modules
# 适合插件系统、多租户沙箱等场景

# 5. 验证 __main__ 语义
# script.py:
#   print(__name__)               # 直接运行输出 '__main__'
#   # 被导入时输出 'script'
#   if __name__ == '__main__':
#       print("run as script")
```

**启动流程的可观测性**：

```
# 查看每个模块的初始化信息
python -v script.py 2>&1 | head -50
# 输出每个模块的加载位置和来源

# 查看 sys.path 的初始值
python -c "import sys; print(sys.path)"
# 第一个元素是脚本所在目录或 ''（当前目录）

# 查看预加载的模块
python -c "import sys; print(list(sys.modules.keys()))"
```

---

## 第 13 章合体总结：Python 启动的全景图

|阶段|书里（2.5）|现代（3.14）|核心任务|
|---|---|---|---|
|**预初始化**​|无明确阶段|`Py_PreInitialize()`|设置运行时状态、内存分配器、基础配置|
|**核心初始化**​|`Py_InitializeEx()` 内统一处理|`Py_InitializeFromConfig()` → `pyinit_core()`|创建运行时状态、GIL、builtins/sys/**main**、导入系统|
|**线程环境**​|手动调用 `PyEval_InitThreads()`|`Py_Initialize()` 自动创建 GIL（3.7+）|建立运行时-解释器-线程三级状态|
|**系统模块**​|`_PyBuiltin_Init()` + `_PySys_Init()`|同上，但 sys 通过 `_PySys_Create()` 特殊创建|搭建导入系统基础设施|
|**主初始化**​|无明确阶段|`pyinit_main()` → `init_interp_main()`|设置 sys.path、导入 site|
|**激活虚拟机**​|`PyRun_AnyFile()`|`Py_RunMain()` → `PyRun_FileExFlags()`|编译+执行主模块|

### 三个必须带走的洞见

**洞见一：Python 启动是"三级状态树"的搭建过程。**

`_PyRuntimeState`（进程级）→ `PyInterpreterState`（解释器级）→ `PyThreadState`（线程级）。GIL 属于运行时级，所以在 3.12 之前所有子解释器共享一个 GIL；3.12+ 允许子解释器拥有独立 GIL，这是多解释器真正并行的物理基础。理解这三层结构，你就能理解为什么 Python 的"多线程"受限于 GIL，而"多进程"或"多解释器"可以真正并行。

**洞见二：系统模块的初始化顺序是不可颠倒的。**

`sys` 模块必须先于导入系统本身创建（因为它用特殊函数 `_PySys_Create()` 绕过常规导入）；`builtins` 必须在用户代码执行前就绪（因为编译器解析全局名时要查 `builtins` 命名空间）；`__main__` 必须在执行用户代码前创建（作为顶层命名空间）。这个顺序不是偶然的，而是"鸡生蛋"问题的唯一解。

**洞见三：现代 CPython 把"配置"与"执行"彻底解耦。**

`PyConfig` 结构体（3.8+ PEP 587）让嵌入 Python 的应用可以精细控制初始化行为；`Py_Main()` 等价于 `PyConfig_InitPythonConfig()` + `Py_InitializeFromConfig()` + `Py_RunMain()`。这种解耦让 Python 的启动流程变得可编程、可定制、可优化——这是工业级嵌入（如 Blender、Unreal Engine 的 Python 插件、Spark 的 PySpark）能够精细控制 Python 运行时的基础。

### 立刻可做的实验

1. **观察启动流程的模块加载**：
    
    ```
    python -v -c "pass" 2>&1 | head -30
    # 看到每个模块的加载过程和来源
    ```
    
2. **测量各阶段耗时**：
    
    ```
    import time, sys
    
    # 阶段1：导入 sys 后的时间点
    t0 = time.perf_counter()
    
    # 阶段2：首次导入大模块
    import numpy
    t1 = time.perf_counter()
    
    print(f"numpy import: {t1 - t0:.3f}s")
    # 对比：冷启动 python -c "import numpy" 的总时间
    # 差值主要来自解释器初始化 + site 执行
    ```
    
3. **禁用 site 加速启动**：
    
    ```
    time python script.py          # 正常启动
    time python -S script.py       # 禁用 site，启动更快
    ```
    
4. **验证 **main** 语义**：
    
    ```
    # test_main.py
    print(f"__name__ = {__name__}")
    if __name__ == '__main__':
        print("Running as script")
    
    # python test_main.py
    # 输出：__name__ = __main__
    #       Running as script
    
    # python -c "import test_main"
    # 输出：__name__ = test_main
    ```
    
5. **嵌入 Python 的最小 C 程序**：
    
    ```
    #include <Python.h>
    
    int main(int argc, char **argv) {
        Py_Initialize();  // 自动创建 GIL（3.7+）
        PyRun_SimpleString("print('Hello from embedded Python')");
        Py_Finalize();
        return 0;
    }
    ```
    

### 版本差异速查

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`Py_InitializeEx()` 单一大函数|三段式：预初始化 → 核心初始化 → 主初始化|
|手动调用 `PyEval_InitThreads()` 创建 GIL|`Py_Initialize()` 自动创建（3.7+），`PyEval_InitThreads()` 变空操作（3.9+）|
|全局配置变量（`Py_VerboseFlag` 等）|`PyConfig` 结构体（3.8+ PEP 587）|
|无子解释器独立 GIL|3.12+ PEP 684 允许 `PyInterpreterConfig_OWN_GIL`|
|`PyRun_AnyFile()` 执行脚本|`Py_RunMain()` → `PyRun_FileExFlags()`|
|`site.py` 从磁盘加载|作为冻结模块内建在二进制中|
|`sys` 通过常规导入创建|通过 `_PySys_Create()` 特殊创建|
|无 `PYTHONSTARTUP` 交互模式定制|`PYTHONSTARTUP` + `sitecustomize.py` + `usercustomize.py`|

