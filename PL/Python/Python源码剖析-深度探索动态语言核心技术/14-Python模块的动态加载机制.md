
## 开篇：第 14 章的主线——"import" 不是语句，而是一套可扩展的查找-加载协议

前几章我们回答了：

- **函数**​ 是 code object + 上下文的封装 [第 11 章]
- **类**​ 是 type 的实例，属性访问走描述器协议 [第 12 章]
- **PVM 启动**​ 时搭建了 `builtins` / `sys` / `__main__` 三大基础模块 [第 13 章]

但还有一个根本问题没回答：**当你写下 `import requests` 时，CPython 是怎么把这个字符串变成一个模块对象，并绑定到当前名字空间的？**​ 第 14 章要揭开的，是 Python "动态加载" 的完整链路。

**本章与书里的最大时差**：

- **书里（Python 2.5）**：import 机制是 C 层的 `PyImport_ImportModuleEx()` 硬逻辑 + `sys.path` 遍历 + 文件后缀匹配，扩展性有限
- **现代（3.14）**：import 机制在 Python 3.3 时被完全重写为 `importlib` 包——查找器（Finder）与加载器（Loader）彻底分离，3.4 引入 `ModuleSpec`（PEP 451）作为导入的"规格载体"，3.12 彻底移除旧的 `find_module()` 接口

**本章双轨阅读法**：

- **轨道一（机制，跨版本稳定）**：import = 缓存查询 → 规格查找（Finder）→ 模块加载（Loader）→ 名字空间绑定
- **轨道二（关键演化）**：
    - Finder/Loader 分离，导入系统完全基于 `importlib`（3.3+）
    - `ModuleSpec` 取代直接返回 Loader（3.4+ PEP 451）
    - 隐式命名空间包，无需 `__init__.py`（3.3+ PEP 420）
    - `sys.meta_path` 三类默认查找器：内建模块、冻结模块、路径查找器
    - `sys.path_hooks`：zipimport + FileFinder
    - 3.12 移除 `find_module()`，强制使用 `find_spec()`
    - 3.15 计划引入惰性导入（PEP 810）

---

## 14.1 import 前奏曲

**书里（Python 2.5）的视角**：

`import` 语句在编译期生成 `IMPORT_NAME` 字节码，运行期由 `PyEval_EvalCodeEx()` 中的 `import_name()` 调用 `PyImport_ImportModuleEx()` 完成。整个流程是 C 层的硬逻辑，用户无法轻易介入。

**现代（3.14）的视角**：

`import` 语句在编译期生成 `IMPORT_NAME(name, fromlist, level)` 字节码：

```
def f():
    import name
    return name
```

反编译：

```
LOAD_CONST 1 (0)        # level = 0 (absolute import)
LOAD_CONST 0 (None)     # fromlist = None
IMPORT_NAME 0 (name)    # 执行导入
STORE_FAST 0 (name)     # 绑定到名字空间
```

**`IMPORT_NAME` 字节码的执行路径**（依据 CPython 开发者文档）：

1. 从当前 frame 的 builtins 中获取 `__import__` 函数
2. 调用 `__import__(name, frame_globals, frame_locals, fromlist, level)`
3. 如果 builtins 中的 `__import__` 就是内建的 `__import__`，走快速路径：直接调用 `PyImport_ImportModuleLevelObject()`
4. `PyImport_ImportModuleLevelObject()` 调用 `importlib._bootstrap._find_and_load()`

**关键架构决策**（3.3+ 的核心演化）：

> _"The importlib package was added to Python 3.1. In Python 3.3, the import machinery was rewritten based on the importlib package."_

也就是说，今天你调用的 `import`，底层的查找-加载逻辑完全是用 Python 写的（位于 `Lib/importlib/_bootstrap.py` 和 `_bootstrap_external.py`），这两个模块以**冻结模块（frozen module）**形式内嵌在 CPython 二进制中。

**`sys.meta_path` 的三类默认查找器**：

```
import sys
print(sys.meta_path)
# [<class '_frozen_importlib.BuiltinImporter'>,
#  <class '_frozen_importlib.FrozenImporter'>,
#  <class '_frozen_importlib_external.PathFinder'>]
```

- **BuiltinImporter**：处理内建模块（如 `sys`、`gc`）
- **FrozenImporter**：处理冻结模块（如 `importlib._bootstrap` 自身）
- **PathFinder**：处理文件系统上的模块（通过 `sys.path`）

**第一性原理**：

- **import 的本质是"字符串 → 模块对象"的映射**。这个映射不是硬编码的，而是通过"查找器链"动态完成的。每个查找器都尝试把模块名解析为一个 `ModuleSpec`，第一个成功的查找器胜出。
- **"前奏曲"的真正含义**：在真正查找模块之前，CPython 要做三件事：① 检查 `sys.modules` 缓存；② 确定是绝对导入还是相对导入（由 `level` 参数决定）；③ 准备 `fromlist` 处理（如 `from a.b import c` 中的 `c`）。这些都是 import 语句执行前的"序曲"。
- **import 机制本身是用 Python 实现的**——这是一个优雅的自举：CPython 用 C 写了最小化的导入引导，然后用 Python 实现了完整的导入系统，最后把这个 Python 实现冻结进二进制。这使得导入系统高度可扩展（用户可以轻松编写自定义查找器）。

**洞见——为什么 `from __future__ import annotations` 要放在文件最顶部？**

因为 `IMPORT_NAME` 字节码在编译期就确定了导入行为，且 `from __future__ import X` 是一种特殊的编译期指令，必须在任何其他代码（包括其他 import）之前执行。如果放在文件中间，编译器会报错——这体现了"前奏曲"的概念：某些导入是"元导入"，必须在所有其他导入之前完成。

**工业实际**：

```
import dis, sys

# 1. 观察 IMPORT_NAME 字节码
def demo():
    import os
    from sys import path
    import json as j

dis.dis(demo)
# 会看到三个 IMPORT_NAME 指令，分别对应三种导入形式

# 2. 查看默认 meta_path 查找器
print(sys.meta_path)
# 三类查找器：内建、冻结、路径

# 3. 快速路径验证
# 当 builtins.__import__ 未被覆盖时，IMPORT_NAME 直接调用
# PyImport_ImportModuleLevelObject()，跳过 __import__ 的函数调用开销
import builtins
print(builtins.__import__ is __import__)  # True（默认情况）
```

---

## 14.2 Python 中 import 机制的黑盒探测

**书里（Python 2.5）**：通过实验观察 `import` 的行为——当模块不存在时抛 `ImportError`，当模块已导入时直接返回缓存等。

**现代（3.14）的黑盒探测**：

我们可以用现代工具深入观测 import 的内部过程：

**1. 观测 sys.modules 缓存**：

```
import sys

# 导入前
print('os' in sys.modules)  # False

import os

# 导入后
print('os' in sys.modules)  # True
print(sys.modules['os'] is os)  # True——同一个对象
```

**2. 观测 sys.path 的影响**：

```
import sys
print(sys.path)
# 第一个元素是脚本所在目录或 ''（当前目录）
# 然后是 PYTHONPATH 环境变量指定的路径
# 然后是安装依赖的默认路径
# site.py 会在启动时添加 site-packages
```

**3. 观测 ModuleSpec**：

```
import importlib.util

spec = importlib.util.find_spec('json')
print(spec.name)                        # 'json'
print(spec.origin)                      # '/usr/lib/python3.12/json/__init__.py'
print(spec.loader)                      # <SourceFileLoader ...>
print(spec.submodule_search_locations)  # ['/usr/lib/python3.12/json']
```

**4. 观测命名空间包（PEP 420）**：

```
# 假设有两个目录：
# dir1/company/project_a/module_a.py
# dir2/company/project_b/module_b.py
# 都没有 __init__.py

import sys
sys.path += ['dir1', 'dir2']

import company.project_a.module_a
import company.project_b.module_b

print(company.__path__)
# _NamespacePath(['dir1/company', 'dir2/company'])
# company 是一个跨越两个目录的命名空间包
```

**5. 观测 import 钩子的执行顺序**：

```
import sys

class LoggingFinder:
    def __init__(self, name):
        self.name = name
    def find_spec(self, fullname, path, target=None):
        print(f"[Finder {self.name}] Looking for {fullname}")
        return None  # 不处理，让其他查找器继续

sys.meta_path.insert(0, LoggingFinder("first"))
sys.meta_path.append(LoggingFinder("last"))

import os
# 输出：
# [Finder first] Looking for os
# （BuiltinImporter 处理 os，返回 spec）
```

**第一性原理**：

- **黑盒探测的核心是"观测三个东西"**：`sys.modules`（缓存）、`sys.meta_path`（查找器链）、`sys.path`（路径列表）。理解了这三者，import 的所有行为都可以预测。
- **`ModuleSpec` 是导入过程的"护照"**：它携带了模块的名称、加载器、来源路径、是否为包等信息。查找器的工作就是生产这个"护照"，加载器的工作是凭"护照"放行模块进入运行时。
- **命名空间包打破了"包必须有 `__init__.py`"的旧假设**：在 3.3+ 中，如果一个目录没有 `__init__.py` 但包含模块或子包，Python 会将其视为命名空间包，可以跨多个 `sys.path` 目录聚合。

**洞见——为什么 `sys.modules` 的缓存机制既是性能优化也是潜在陷阱？**

```
import sys
import json

# 手动修改缓存
sys.modules['json'] = type(sys)('fake_json')

import json
print(json)  # <module 'fake_json'>——已经不是原来的 json 了！

# 修复
del sys.modules['json']
import json
print(json)  # 恢复正常
```

`sys.modules` 是 import 系统的"短路"——只要模块名在缓存中，直接返回缓存对象，不再走查找-加载流程。这既是性能优化的关键（避免重复导入），也是危险操作的入口（恶意修改缓存可以替换模块）。**工业实践中，不应该随意修改 `sys.modules`**，除非你清楚后果（如实现热重载）。

**工业实际**：

```
# 1. 探测模块的完整导入信息
import importlib.util

def probe_module(name):
    spec = importlib.util.find_spec(name)
    if spec is None:
        print(f"{name}: not found")
        return
    print(f"Module: {spec.name}")
    print(f"Origin: {spec.origin}")
    print(f"Loader: {spec.loader}")
    print(f"Is package: {spec.submodule_search_locations is not None}")
    if spec.submodule_search_locations:
        print(f"Submodule paths: {spec.submodule_search_locations}")

probe_module('os')
probe_module('json')
probe_module('sys')

# 2. 探测 import 的实际查找路径
import sys
print("sys.path entries:")
for i, p in enumerate(sys.path):
    print(f"  [{i}] {p!r}")

# 3. 观测 sys.path_hooks 如何影响路径查找
print(sys.path_hooks)
# [<class 'zipimport.zipimporter'>,
#  <function _path_hooks_... at 0x...>]
# 每个 sys.path 项都会被这些 hook 尝试，成功则创建路径查找器
```

---

## 14.3 import 机制的实现

**这是本章的技术核心**——完整展开现代 CPython 的 import 实现。

**书里（Python 2.5）的实现**：

`PyImport_ImportModuleEx()` 在 C 层直接遍历 `sys.path`，根据文件后缀（`.py`、`.pyc`、`.so`）匹配加载器，调用对应的加载函数。

**现代（3.14）的实现**（依据 CPython 开发者文档）：

```
__import__(name, globals, locals, fromlist, level)
    ↓
PyImport_ImportModuleLevelObject()  [Python/import.c]
    ↓
importlib._bootstrap._find_and_load(name, import_func)  [冻结的 Python 代码]
    ↓
    ├─ 检查 sys.modules 缓存
    ├─ _find_spec(name, path)  # path 是父包的 __path__ 或 None
    │     ↓
    │   遍历 sys.meta_path 中的每个 finder
    │   调用 finder.find_spec(name, path, target)
    │   第一个返回非 None 的 spec 胜出
    │
    ├─ module = module_from_spec(spec)  # 创建模块对象
    ├─ sys.modules[name] = module       # 缓存到 sys.modules（在 exec_module 之前！）
    └─ spec.loader.exec_module(module)  # 执行模块代码
```

**关键步骤详解**：

**1. sys.modules 缓存检查**：

```
if name in sys.modules:
    return sys.modules[name]
```

这是 import 系统的"短路"——已导入的模块直接返回缓存。

**2. 规格查找（find_spec）**：

```
def _find_spec(name, path, target=None):
    for finder in sys.meta_path:
        spec = finder.find_spec(name, path, target)
        if spec is not None:
            return spec
    raise ModuleNotFoundError(f"No module named {name!r}")
```

对于 `foo.bar.baz` 这样的带点模块名，导入系统会**逐级导入**：

- 首先导入 `foo`：`find_spec("foo", None)`
- 然后导入 `foo.bar`：`find_spec("foo.bar", foo.__path__)`
- 最后导入 `foo.bar.baz`：`find_spec("foo.bar.baz", foo.bar.__path__)`

**3. PathFinder 的查找逻辑**：

PathFinder 是 `sys.meta_path` 中的最后一个查找器，它处理文件系统上的模块：

```
class PathFinder:
    @classmethod
    def find_spec(cls, fullname, path, target=None):
        if path is None:
            path = sys.path
        for entry in path:
            finder = cls._get_finder(entry)  # 使用 sys.path_importer_cache
            if finder is not None:
                spec = finder.find_spec(fullname)
                if spec is not None:
                    return spec
        return None
```

对于每个 `sys.path` 项，PathFinder 调用 `sys.path_hooks` 中的钩子尝试创建路径查找器：

```
# 默认 sys.path_hooks
sys.path_hooks = [
    zipimport.zipimporter,           # 处理 .zip 文件
    importlib._bootstrap_external.FileFinder  # 处理目录
]
```

**4. FileFinder 的查找逻辑**：

FileFinder 针对一个具体目录，根据文件后缀匹配加载器：

```
# 默认的文件后缀 → 加载器映射
# .py  → SourceFileLoader
# .pyc → SourcelessFileLoader  
# .so  → ExtensionFileLoader
```

**5. 模块加载（Loader）**：

现代加载器实现两个方法：

- `create_module(spec)`：创建模块对象（通常返回 None 使用默认创建逻辑）
- `exec_module(module)`：执行模块代码，初始化模块命名空间

**6. ModuleSpec 的结构**（PEP 451）：

```
class ModuleSpec:
    name                            # 模块的完全限定名
    loader                          # 使用的加载器
    origin                          # 模块来源（如文件路径）
    submodule_search_locations     # 包的 __path__（如果不是包则为 None）
    loader_state                    # 加载器专用状态
    cached                          # .pyc 文件路径
    parent                          # 父包名
    has_location                    # origin 是否指向可加载位置
```

**第一性原理**：

- **import 实现 = "Finder 生产 Spec + Loader 消费 Spec"**。Finder 和 Loader 的职责彻底分离：Finder 只负责"找到模块并描述它"（生产 ModuleSpec），Loader 只负责"创建并执行模块"（消费 ModuleSpec）。这种分离让导入系统高度可扩展。
- **`sys.modules` 缓存在 `exec_module` 之前**——这是一个关键设计：模块在代码执行前就被放入缓存。这解决了循环导入问题——当模块 A 导入模块 B，而 B 又导入 A 时，B 拿到的是 A 的不完整对象（已创建但未执行完），但至少不会无限递归。
- **`find_spec()` 取代了 `find_module()`**：3.4 引入 ModuleSpec 后，`find_spec()` 成为标准接口，3.12 彻底移除了 `find_module()`。这是因为 ModuleSpec 携带的信息远比单纯的 Loader 丰富。

**洞见——为什么 import 系统是 Python 灵活性的基石？**

因为 Finder 和 Loader 都可以被用户自定义：

- 想要从数据库中导入模块？写一个自定义 Finder。
- 想要导入加密的 .py 文件？写一个自定义 Loader。
- 想要实现插件系统？利用 `sys.meta_path` 插入自定义 Finder。

这种"一切皆可挂钩"的设计，让 Python 的 import 系统成为了一个通用的"资源加载框架"，不仅限于 .py 文件。

**工业实际**：

```
# 1. 自定义 Finder 实现"从字符串导入"
import sys
import importlib.abc
import importlib.util

class StringFinder(importlib.abc.MetaPathFinder):
    def __init__(self):
        self.modules = {}
    
    def register(self, name, source_code):
        self.modules[name] = source_code
    
    def find_spec(self, fullname, path, target=None):
        if fullname in self.modules:
            # 创建内存中的模块 spec
            loader = StringLoader(fullname, self.modules[fullname])
            return importlib.util.spec_from_loader(fullname, loader)
        return None

class StringLoader(importlib.abc.Loader):
    def __init__(self, name, source_code):
        self.name = name
        self.source_code = source_code
    
    def create_module(self, spec):
        return None  # 使用默认模块创建
    
    def exec_module(self, module):
        exec(self.source_code, module.__dict__)

# 使用
finder = StringFinder()
finder.register('my_dynamic_module', '''
def hello():
    return "Hello from dynamic module!"
''')
sys.meta_path.insert(0, finder)

import my_dynamic_module
print(my_dynamic_module.hello())  # Hello from dynamic module!

# 2. 自定义路径钩子处理特殊路径格式
import sys
import importlib.abc
import importlib.machinery

class DatabaseImporter:
    """从数据库加载模块的路径钩子"""
    def __init__(self, path_entry):
        self.path_entry = path_entry
        # 验证 path_entry 是否是数据库连接字符串
        if not path_entry.startswith('db://'):
            raise ImportError(f"Not a database path: {path_entry}")
        # 连接数据库...
    
    def find_spec(self, fullname, target=None):
        # 从数据库查询模块
        # 返回 ModuleSpec 或 None
        pass

def database_path_hook(path):
    return DatabaseImporter(path)

sys.path_hooks.append(database_path_hook)
sys.path.append('db://localhost/mymodules')
# 现在可以从数据库导入模块了
```

---

## 14.4 Python 中的 import 操作

**书里（Python 2.5）**：`import X`、`from X import Y`、`import X as Z` 三种形式的语义和字节码差异。

**现代（3.14）的 import 操作语义**：

**1. `import X` 的完整语义**：

```
import X
# 等价于：
X = __import__('X')
```

- 如果 X 是带点名称（如 `foo.bar`），则导入整个链，但只绑定最顶层名字：

```
import foo.bar.baz
# 等价于：
tmp = __import__('foo.bar.baz', fromlist=['baz'])  # 实际绑定 foo
foo = tmp
# 访问 foo.bar.baz 时，每层都已经导入完成
```

**2. `from X import Y` 的完整语义**：

```
from X import Y
# 等价于：
_X = __import__('X', fromlist=['Y'])
Y = getattr(_X, 'Y')  # 如果 Y 是子模块，则触发子模块导入
```

关键点：`fromlist` 非空时，`__import__` 会确保 `fromlist` 中的每个名字都可用。如果 Y 是 X 的子模块，`__import__` 会自动导入子模块。

**3. `import X as Z` 的语义**：

```
import X as Z
# 等价于：
Z = __import__('X')
```

**4. 相对导入（PEP 328，3.0+）**：

```
from . import sibling
from .subpackage import module
from ..parent import uncle
```

- 单个点 `.` 表示当前包
- 两个点 `..` 表示父包
- 三个点 `...` 表示祖父包
- 相对导入只能在包内使用（即 `__package__` 不为 None）

**5. 动态导入的现代 API**：

```
import importlib

# 等价于 import json
json = importlib.import_module('json')

# 等价于 from os import path
path = importlib.import_module('os.path')

# 相对导入
module = importlib.import_module('.sibling', package='mypackage')
```

**6. 从文件路径加载（工业动态加载标准做法）**：

```
import importlib.util
import sys

def import_from_file(module_name, file_path):
    spec = importlib.util.spec_from_file_location(module_name, file_path)
    if spec is None:
        raise ImportError(f"Cannot load module from {file_path}")
    module = importlib.util.module_from_spec(spec)
    sys.modules[module_name] = module  # 重要：先注册到 sys.modules
    spec.loader.exec_module(module)
    return module

# 使用
my_module = import_from_file('my_plugin', '/path/to/plugin.py')
```

**⚠️ 过时 API 警告**：

- `SourceFileLoader.load_module()` 在 3.6 弃用，3.12 移除
- 必须使用 `module_from_spec()` + `exec_module()` 的现代模式

**第一性原理**：

- **所有 import 操作最终都归结为 `__import__()` 函数调用**。语法糖的背后，是统一的 `__import__(name, globals, locals, fromlist, level)` 接口。
- **`from X import Y` 的 `Y` 可能是属性也可能是子模块**：如果 Y 是 X 的子模块（即 X 包的 `__path__` 下存在 Y.py 或 Y/ 目录），`__import__` 会自动导入子模块并绑定到 X.Y。
- **相对导入依赖于 `__package__` 属性**：这个属性在包内执行时自动设置，标识当前包名。这就是为什么相对导入不能在脚本（非包内）中使用。

**洞见——为什么 `from module import *` 是不良实践？**

```
# bad_module.py
def foo(): pass
def bar(): pass
_private = "secret"

# 在另一个文件中
from bad_module import *
# 导入了 foo、bar、_private——包括"私有"成员
```

`from X import *` 的行为取决于 X 的 `__all__` 属性：

- 如果定义了 `__all__`，只导入列表中的名字
- 如果没定义 `__all__`，导入所有不以 `_` 开头的名字

这种不确定性使得 `from X import *` 在生产代码中应该避免使用——它会污染当前名字空间，且行为难以预测。现代 Python 代码应该用显式导入。

**工业实际**：

```
# 1. 插件系统的标准实现
import importlib
import pkgutil
import sys

def load_plugins(package_name):
    """动态加载包中的所有插件"""
    package = importlib.import_module(package_name)
    plugins = []
    for _, name, _ in pkgutil.iter_modules(package.__path__):
        full_name = f"{package_name}.{name}"
        module = importlib.import_module(full_name)
        if hasattr(module, 'Plugin'):
            plugins.append(module.Plugin())
    return plugins

# Django 的 INSTALLED_APPS 就是这样工作的
# 每个 app 名通过 importlib.import_module() 动态导入

# 2. 热重载的实现
import importlib
import sys

def reload_module(module_name):
    """重新加载模块（更新代码后）"""
    if module_name in sys.modules:
        module = sys.modules[module_name]
        return importlib.reload(module)
    else:
        return importlib.import_module(module_name)

# 3. 条件导入（处理可选依赖）
try:
    import ujson as json  # 优先使用快速 JSON
except ImportError:
    import json  # 回退到标准库

# 4. 延迟导入（优化启动时间）
def process_data(data):
    # 只在需要时导入重型库
    import pandas as pd
    df = pd.DataFrame(data)
    return df.describe()
```

---

## 14.5 与 module 有关的名字空间问题

**这是 import 机制中最微妙的部分**——模块对象与名字空间的关系。

**书里（Python 2.5）**：模块对象有一个 `__dict__` 属性作为全局名字空间，import 语句将模块对象绑定到当前 frame 的 `f_globals`。

**现代（3.14）的名字空间机制**：

**1. 模块对象本身就是名字空间**：

```
import os
print(type(os))                    # <class 'module'>
print(type(os.__dict__))           # <class 'dict'>
print(os.path is os.__dict__['path'])  # True
```

模块的属性访问就是对其 `__dict__` 的字典查找。

**2. 模块对象的属性与 `__spec__` 的关系**：

依据官方文档，`module.__spec__` 是模块的 `ModuleSpec` 对象，它与模块的直接属性存在对应关系：

- `module.__spec__.origin == module.__file__`（通常）
- `module.__spec__.submodule_search_locations == module.__path__`（对包而言）
- `module.__spec__.name == module.__name__`
- `module.__spec__.loader == module.__loader__`

> ⚠️ **关键警告**：_“while the values are usually equivalent, they can differ since there is no synchronization between the two objects”_——`__spec__` 和模块直接属性之间**没有同步机制**。如果运行时修改 `module.__file__`，`module.__spec__.origin` 不会自动更新，反之亦然。

**3. import 语句的名字绑定**：

```
import os
# 等价于：
os = __import__('os')
# os 这个名字被绑定到当前 frame 的 f_globals 中
# 实际上，'os' 被添加到当前模块的 __dict__ 中
```

**4. `from X import Y` 的名字绑定**：

```
from os import path
# 等价于：
_path = __import__('os', fromlist=['path']).__dict__['path']
# 更准确：path 被绑定为 os.path 的引用
path = getattr(__import__('os', fromlist=['path']), 'path')
# 'path' 被添加到当前模块的 __dict__ 中
```

**5. 循环导入问题**：

```
# module_a.py
import module_b
def func_a():
    return module_b.func_b()

# module_b.py
import module_a
def func_b():
    return module_a.func_a()
```

当导入 module_a 时：

1. 创建 module_a 对象，注册到 `sys.modules['module_a']`
2. 执行 module_a 代码，遇到 `import module_b`
3. 创建 module_b 对象，注册到 `sys.modules['module_b']`
4. 执行 module_b 代码，遇到 `import module_a`
5. 由于 `module_a` 已在 `sys.modules` 中，直接返回**未完成初始化的 module_a 对象**
6. module_b 继续执行，func_b 定义完成
7. 回到 module_a 继续执行

**问题在于**：如果 module_b 在 `import module_a` 之后**立即**调用 `module_a.func_a()`，而 func_a 尚未定义，就会抛出 `AttributeError`。

**解决方案**：

```
# 方案1：延迟导入（在函数内部导入）
# module_b.py
def func_b():
    import module_a  # 在函数调用时导入，此时 module_a 已完成初始化
    return module_a.func_a()

# 方案2：重组代码结构，避免循环依赖
# 方案3：将共享代码提取到第三个模块
```

**6. `__init__.py` 的作用与演化**：

- **传统包（regular package）**：包含 `__init__.py` 的目录
- **命名空间包（namespace package，PEP 420）**：不包含 `__init__.py`，但可以跨多个目录聚合

```
# 传统包结构
mypackage/
    __init__.py          # 包初始化代码
    module_a.py
    module_b.py

# 命名空间包结构（无 __init__.py）
mypackage/
    module_a.py
    module_b.py
# 可以分布在多个 sys.path 目录中
```

**7. 模块级名字的可见性**：

```
# module.py
x = 1
def foo():
    print(x)  # 访问模块级名字 x
foo()  # 输出 1

# 在另一个模块中
import module
module.foo()  # 输出 1
print(module.x)  # 1
module.x = 2
print(module.x)  # 2
print(module.__dict__['x'])  # 2
```

**第一性原理**：

- **模块对象 = 名字空间的物理载体**。模块的 `__dict__` 就是模块的全局名字空间。当你在模块中定义一个函数、类或变量，它就被添加到 `__dict__` 中。
- **import 语句的本质是"名字绑定"**：`import X` 把 X 模块对象绑定到当前名字空间的一个名字；`from X import Y` 把 X 模块中的 Y 属性绑定到当前名字空间的一个名字。
- **循环导入之所以能"部分工作"，是因为 `sys.modules` 缓存在 `exec_module` 之前**——模块对象已创建但未执行完时就被其他模块获取，所以可以访问已定义的属性，但不能访问尚未定义的属性。

**洞见——为什么修改 `module.__file__` 不会反映在 `module.__spec__.origin` 上？**

```
import json
json.__file__ = "/fake/path/json.py"
print(json.__file__)              # /fake/path/json.py
print(json.__spec__.origin)       # /usr/lib/python3.12/json/__init__.py（不变）
```

官方文档明确指出：_“there is no synchronization between the two objects”_。`module.__file__` 和 `module.__spec__.origin` 是两个独立的属性，修改一个不会影响另一个。这意味着：

- 依赖 `module.__file__` 的代码（如某些序列化库）和依赖 `module.__spec__.origin` 的代码（如 importlib 内部）可能看到不一致的值
- 这就是为什么不应该随意修改模块的这些属性——它们虽然通常相等，但本质上是两个独立的存储

**工业实际**：

```
# 1. 观测模块的名字空间
import os
print(type(os.__dict__))   # <class 'dict'>
print('path' in os.__dict__)  # True
print(os.path is os.__dict__['path'])  # True

# 2. 循环导入的安全处理
# 方案A：延迟导入
# module_a.py
def get_b():
    from module_b import func_b  # 延迟导入
    return func_b()

# 方案B：使用 importlib 动态导入
# module_a.py
import importlib
def get_b():
    module_b = importlib.import_module('module_b')
    return module_b.func_b()

# 3. 运行时修改模块名字空间
import math
math.PI = 3.14  # 不推荐！但技术上可行
print(math.PI)  # 3.14

# 正确的方式：使用模块级常量或配置
# config.py
PI = 3.141592653589793

# 4. 命名空间包的工业应用
# 插件系统：不同厂商的插件可以贡献到同一个命名空间
# myapp/plugins/auth.py  （来自 package-a）
# myapp/plugins/db.py    （来自 package-b）
# 两者都没有 __init__.py，但都属于 myapp.plugins 命名空间

import myapp.plugins.auth
import myapp.plugins.db
print(myapp.plugins.__path__)
# 包含两个路径：package-a/myapp/plugins 和 package-b/myapp/plugins

# 5. 模块的 __spec__ 属性详解
import json
print(json.__spec__.name)                       # 'json'
print(json.__spec__.origin)                     # 文件路径
print(json.__spec__.loader)                     # SourceFileLoader
print(json.__spec__.submodule_search_locations) # 包的 __path__
```

---

## 第 14 章合体总结：import 机制的完整图景

|阶段|书里（Python 2.5）|现代（Python 3.14）|
|---|---|---|
|**导入入口**​|`PyImport_ImportModuleEx()` C 函数|`IMPORT_NAME` 字节码 → `__import__` → `PyImport_ImportModuleLevelObject` → `importlib._bootstrap._find_and_load`|
|**查找机制**​|C 层硬逻辑遍历 `sys.path`|`sys.meta_path` 查找器链，每个 finder 的 `find_spec()` 返回 `ModuleSpec`|
|**加载机制**​|直接调用加载函数|Loader 的 `create_module()` + `exec_module()`|
|**模块规格**​|无（直接返回 Loader）|`ModuleSpec` 对象（PEP 451，3.4+）|
|**包的概念**​|必须有 `__init__.py`|支持命名空间包（无 `__init__.py`，PEP 420）|
|**路径处理**​|`sys.path` 直接遍历|`sys.path_hooks` + `sys.path_importer_cache`|
|**默认查找器**​|内建 + 文件系统|`BuiltinImporter` + `FrozenImporter` + `PathFinder`|
|**find_module**​|使用 `find_module()`|3.12 移除，强制使用 `find_spec()`|
|**动态导入 API**​|`imp` 模块|`importlib.import_module()` / `importlib.util.spec_from_file_location()`|

### 三个必须带走的洞见

**洞见一：import 不是语句，而是一套"Finder 生产 Spec + Loader 消费 Spec"的可扩展协议。**

- Finder 和 Loader 的职责彻底分离，使得导入系统高度可扩展
- `ModuleSpec` 是导入过程的"护照"，携带模块的所有元数据
- `sys.meta_path` 和 `sys.path_hooks` 提供了两层扩展点

**洞见二：`sys.modules` 缓存在 `exec_module` 之前设置——这是循环导入能"部分工作"的物理基础。**

- 模块对象在代码执行前就被注册到 `sys.modules`
- 这解决了无限递归问题，但也意味着不完整的模块对象可能被其他模块获取
- 这就是为什么循环导入中，只能访问已定义的属性，不能访问尚未定义的属性

**洞见三：模块对象本身就是名字空间，`__dict__` 是模块全局变量的物理存储。**

- `import X` 是把模块对象绑定到当前名字空间
- `from X import Y` 是把 X 的 Y 属性绑定到当前名字空间
- `module.__spec__` 和模块直接属性（如 `__file__`）之间没有同步机制，修改一个不会影响另一个

### 立刻可做的实验

1. **观测完整导入流程**：
    
    ```
    import sys, importlib.util
    
    # 1. 检查 sys.modules 缓存
    print('os' in sys.modules)  # False
    import os
    print('os' in sys.modules)  # True
    
    # 2. 查看 ModuleSpec
    spec = importlib.util.find_spec('json')
    print(f"Name: {spec.name}")
    print(f"Origin: {spec.origin}")
    print(f"Loader: {spec.loader}")
    
    # 3. 查看默认查找器
    print(sys.meta_path)
    ```
    
2. **自定义 Finder 实现内存模块**：
    
    ```
    import sys
    import importlib.abc
    import importlib.util
    
    class MemoryFinder(importlib.abc.MetaPathFinder):
        def __init__(self):
            self.code_store = {}
        def register(self, name, code):
            self.code_store[name] = code
        def find_spec(self, fullname, path, target=None):
            if fullname in self.code_store:
                loader = MemoryLoader(fullname, self.code_store[fullname])
                return importlib.util.spec_from_loader(fullname, loader)
            return None
    
    class MemoryLoader(importlib.abc.Loader):
        def __init__(self, name, code):
            self.name = name
            self.code = code
        def create_module(self, spec):
            return None
        def exec_module(self, module):
            exec(self.code, module.__dict__)
    
    finder = MemoryFinder()
    finder.register('greet', 'def hello():\n    return "Hi!"')
    sys.meta_path.insert(0, finder)
    
    import greet
    print(greet.hello())  # Hi!
    ```
    
3. **观测命名空间包**：
    
    ```
    # 创建两个目录：pkg1/ns/pkg/mod1.py 和 pkg2/ns/pkg/mod2.py
    # 都没有 __init__.py
    import sys
    sys.path += ['pkg1', 'pkg2']
    import ns.pkg.mod1
    import ns.pkg.mod2
    print(ns.pkg.__path__)  # 包含两个路径
    ```
    
4. **循环导入的实验**：
    
    ```
    # module_a.py: import module_b; print(module_b.attr)
    # module_b.py: import module_a; attr = 42
    # 导入 module_a 会触发：
    # 1. 创建 module_a 对象 → 注册到 sys.modules
    # 2. 执行 module_a: import module_b
    # 3. 创建 module_b 对象 → 注册到 sys.modules
    # 4. 执行 module_b: import module_a（从缓存获取不完整的 module_a）
    # 5. module_b 定义 attr = 42
    # 6. 回到 module_a: print(module_b.attr) → 42
    ```
    

### 版本差异速查

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`PyImport_ImportModuleEx()` C 函数直接处理|`IMPORT_NAME` 字节码 → `importlib._bootstrap._find_and_load`|
|查找和加载耦合在一起|Finder 和 Loader 彻底分离|
|无 ModuleSpec|PEP 451 引入 `ModuleSpec`（3.4+）|
|包必须有 `__init__.py`|支持命名空间包（PEP 420，3.3+）|
|`find_module()` 接口|3.12 移除，使用 `find_spec()`|
|`imp` 模块|`importlib` 包（3.1+ 引入，3.3+ 成为核心）|
|`SourceFileLoader.load_module()`|3.6 弃用，3.12 移除，使用 `exec_module()`|
|单一 `sys.path` 遍历|`sys.meta_path` + `sys.path_hooks` + `sys.path_importer_cache`|
|无惰性导入|PEP 810 计划在 3.15 引入惰性导入|

