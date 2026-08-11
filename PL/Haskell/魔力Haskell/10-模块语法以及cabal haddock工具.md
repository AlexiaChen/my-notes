
# 第十章 模块语法以及 cabal、Haddock 工具 —— 读书笔记

> 💡 如果说前九章一直在回答"Haskell 语言本身怎么表达计算"，那么第十章就是在回答一个工程化的转折问题：**当你的 `data`、`type class`、函数越来越多，怎么把它们组织成可复用、可发布、可文档化的单元？**​ 这一章看似是"工具章节"，实则是 Haskell 工程哲学的集中体现——**模块是抽象数据类型的封装边界，cabal 是包的构建与分发规范，Haddock 让类型签名成为 API 文档的单一事实源**。三块内容共同回答了一个问题：如何让"类型驱动的正确性"从单个文件延伸到整个生态系统。

韩冬在这一章安排了三个小节：**10.1 模块语法**、**10.2 使用 cabal**（含 10.2.1 使用 cabal 安装依赖、10.2.2 项目的 cabal 配置）、**10.3 Haddock**。表面是"语法 + 工具"的组合，实则在揭示 Haskell 工程化的三条主线：**命名空间控制、包管理、文档即类型**。

---

## 🎯 全局脉络：为什么是"模块 + cabal + Haddock"？

```
第九章: type / newtype / 惰性求值（语言层面的零成本抽象）
   ↓
第十章: 模块 + cabal + Haddock  ← 工程层面的封装与分发  【你在这里】
   ↓
第二部分: 函子 / 应用函子 / 单子 / 解析器  ← 用工程化思维构建抽象
```

**这一章是整本书的"工程化转折点"**。前九章你学的所有语言特性——`data` 创造的代数数据类型、`class`/`instance` 定义的类型类、高阶函数表达的递归模式——它们如果只能写在一个 `.hs` 文件里，价值就极其有限。**模块让这些抽象能跨文件复用，cabal 让它们能跨项目分发，Haddock 让它们的接口能被人和工具理解**。

Haskell 报告里有一句话点明了模块的本质：

> _"A module defines a collection of values, datatypes, type synonyms, classes, etc., in an environment created by a set of imports... It exports some of these resources, making them available to other modules."_

**模块不只是"代码分割"，它是 Haskell 里构建抽象数据类型（ADT）的唯一途径**。这正是第十章最值得深挖的底层动机。

---

## 10.1 模块语法：命名空间控制与抽象数据类型

### 模块的基本结构

一个完整的 Haskell 程序由多个模块组成，**约定其中一个必须叫 `Main`，且必须导出 `main` 函数，类型为 `IO a`**​ ：

```
module Main (main) where
import A
import B
main = A.f >> B.f
```

模块声明的完整形态：

```
module Xxx.Yyy.Zzz
  ( -- 导出列表：指定哪些绑定/类型/类型类对外可见
    binding,
    module OtherModule,  -- 重导出
    DataType(Constructor1, Constructor2...),  -- 导出类型及指定构造子
    DataType(),  -- 仅导出类型，不导出构造子
    DataType(..),  -- 导出类型及所有构造子
    ClassDef(classMethod1, classMethod2...),
    ...
  ) where

-- 导入声明
import Aaa
import Aaa.Bbb
import Aaa.Bbb.Ccc (Type, value...)
import qualified Aaa.Bbb.Ccc.Ddd as D
import qualified Aaa.Bbb.Ccc.Eee as E hiding (Type, value)
import OtherModule

-- 正文
...
```

### 🔑 洞见一：模块是 Haskell 构建抽象数据类型的唯一途径

这是 Haskell 教程里被反复强调的核心观点：

> _"Aside from controlling namespaces, modules provide the only way to build abstract data types (ADTs) in Haskell."_

为什么这么说？回想第二章——你用 `data` 定义了一个类型，比如二叉搜索树：

```
data BST a = Empty | Node (BST a) a (BST a)
```

如果只是 `module BST where ...`，**所有构造子 `Empty` 和 `Node` 都会被导出**。这意味着用户可以绕过你的 `insert`、`delete` 函数，直接构造一个违反 BST 不变量的树：

```
-- 用户能干这种事：
badTree = Node Empty 5 (Node Empty 3 Empty)  -- 3 在 5 的右子树里！违反 BST 不变量
```

**解决方案是精确控制导出列表**：

```
module BST (BST, empty, insert, delete, member) where
-- 注意：BST 后面没有 (..)，构造子被隐藏
```

这样用户只能通过你提供的 `empty`、`insert`、`delete` 来操纵 `BST`——**不变量被编译器强制保护**​ 。

> 💡 **这是"make illegal states unrepresentable"工程化的关键手段**。类型系统在类型层面排除非法状态，模块系统在模块边界层面排除非法构造。两者配合，让"破坏不变量"在编译期就不可能。

### 🔑 洞见二：导出列表的精确控制是一门艺术

Haskell 报告明确说明，导出列表有几种精细写法 ：

|导出形式|含义|
|---|---|
|`DataType`|仅导出类型，不导出任何构造子|
|`DataType(..)`|导出类型及所有构造子|
|`DataType(C1, C2)`|导出类型及指定的构造子|
|`ClassDef(m1, m2)`|导出类型类及指定的方法|
|`module OtherModule`|重导出整个模块的所有绑定|

**选择导出什么，是 API 设计的艺术**：

- **隐藏构造子**​ → 强制用户通过你的智能构造函数（smart constructor）创建值，确保不变量
    
- **导出部分构造子**​ → 允许用户模式匹配，但不允许随意构造
    
- **重导出**​ → 设计"门面模块"（facade），让用户从一个入口导入所有功能
    

### 🔑 洞见三：qualified 导入是命名空间冲突的解药

Haskell 的模块命名空间是平的（flat），这意味着不同模块里可能有同名函数。Haskell 的解决方案是 `qualified` 导入：

```
import qualified Data.Set as Set
import qualified Data.Map as Map

-- 使用时必须加前缀
let s = Set.fromList [1,2,3]
let m = Map.fromList [("a", 1)]
```

图灵社区的摘录里，韩冬明确建议：

> _"除一些常用的模块外，推荐使用 qualified 关键字或选择性导入这两种方式来避免命名空间冲突"_

**为什么这个建议重要？**​ 因为 Haskell 生态里大量模块导出了同名函数——`Data.List`、`Prelude`、`Data.Set` 都有 `map`、`filter`、`null`。如果不加 `qualified` 或选择性导入，代码会迅速陷入命名冲突的泥潭。

### 🔑 洞见四：模块名是层级结构，但语言层面是平的

Haskell 允许模块名用点分隔（`Data.Bool`、`Control.Monad.ST`），形成层级：

> _"Module names can be thought of as being arranged in a hierarchy... This is purely a convention, however, and not part of the language definition"_

这意味着 `Data.Set` 和 `Data.Set.Extra` 在语言层面是两个完全独立的名字，点只是社区约定。**这套约定让 Haskell 生态的包组织极度规整**：`Data.*` 是数据结构，`Control.*` 是控制抽象，`System.*` 是系统操作，`Network.*` 是网络相关。

### 🔑 洞见五：模块可以互相递归

Haskell 报告特别指出：

> _"Because they are allowed to be mutually recursive, modules allow a program to be partitioned freely without regard to dependencies."_

**模块可以互相引用、形成递归依赖**——这是 Haskell 声明"顶层作用域是互相递归"的延伸（呼应第一章的绑定声明）。这意味着你不必像某些语言那样操心"先定义哪个模块"的问题，编译器会处理好。

### 实际应用

```
-- 一个设计良好的模块导出列表
module Bank.Account
  ( -- * 类型
    Account,
    AccountId,
    Balance,
    
    -- * 智能构造函数
    createAccount,
    
    -- * 操作
    deposit,
    withdraw,
    transfer,
    
    -- * 查询
    balance,
    accountId,
  ) where

-- 注意：Account 的构造子被隐藏
-- 用户必须通过 createAccount 创建账户
-- 确保余额永远不会为负（不变量保护）
```

---

## 10.2 使用 cabal：Haskell 的构建与包管理系统

### cabal 的三重身份

Haskell 官网明确说明：

> _"The term cabal can refer to either: cabal-the-spec (.cabal files), cabal-the-library (code that understands .cabal files), or cabal-the-tool (the cabal-install package which provides the cabal executable)"_

**cabal 既是规范（.cabal 文件格式），也是库（理解 .cabal 的库），也是工具（cabal 命令行）**。它身兼**包管理工具**和**自动化编译工具**双重功能 。

### 10.2.1 使用 cabal 安装依赖

韩冬在书中介绍了 cabal 的依赖管理：

```
cabal update                    # 第一次使用时更新包索引
cabal install containers -j --global   # 安装到全局
ghc-pkg                         # 管理全局空间下安装的函数库
```

**🔑 洞见六：全局 vs 沙盒——依赖隔离的工程纪律**

韩冬在书中给出了明确的实践建议：

> _"一般建议只向全局空间安装一些非常常用的函数库。对于项目相关的函数库依赖，统统使用沙盒 sandbox 来管理"_

这是 Haskell 工程化的关键纪律。原因：

- **全局空间**是 GHC 的"系统级"包仓库，混合多个项目的依赖会导致"依赖地狱"
    
- **沙盒（sandbox）**为每个项目创建独立的包环境，避免版本冲突
    

```
mkdir test && cd test
cabal sandbox init     # 为项目初始化沙盒
cabal init             # 交互式创建 .cabal 文件
```

> 💡 在现代 Haskell 工具链中，sandbox 已被 `cabal.project` + `v2-build` 系统取代（new-build 系统），但其核心思想不变——**每个项目应该有隔离的依赖环境**。

### 10.2.2 项目的 cabal 配置

`cabal init` 会生成一个 `.cabal` 文件，它是项目的"说明书"：

```
name:                bst
version:             0.1.0.0
synopsis:            A module for efficient BSTs
license-file:        LICENSE
author:              Haskell Curry
maintainer:          hcurry@haskell.org
category:            Data
build-type:           Simple
cabal-version:       >=1.10

library
  exposed-modules:     BST
  build-depends:       base >=4.7 && <5, 
                       containers >=0.5 && <0.6
  default-language:    Haskell2010
```

**🔑 洞见七：.cabal 文件是"类型安全"从代码延伸到工程的体现**

注意 `build-depends` 里的版本约束：`base >=4.7 && <5`。这是一种**区间约束**——它声明"我的代码需要 base 4.7 以上、5 以下"。

这种版本约束的背后是 Haskell 社区的 **PVP（Package Versioning Policy）**​ 规范：

- **主版本号**变更 → 可能破坏 API
    
- **次版本号**变更 → 向后兼容的 API 添加
    
- **修订号**变更 → 向后兼容的 bug 修复
    

**PVP 让类型安全从"函数签名"延伸到"包版本"**——如果你只依赖 `^>=1.2`（即 `>=1.2 && <1.3`），Haskell 工具链能保证 API 兼容。

### 常用 cabal 命令

```
cabal run        # 编译并运行项目
cabal build      # 编译整个项目
cabal clean      # 清理编译结果
cabal sdist      # 生成可上传发布的代码压缩包
cabal upload     # 上传到 Hackage
```

> 📌 这些命令构成了 Haskell 项目的完整生命周期：开发（`cabal run`）、构建（`cabal build`）、打包（`cabal sdist`）、发布（`cabal upload`）。

### 🔑 洞见八：cabal 让"类型驱动的正确性"跨越包边界

cabal 不仅仅是构建工具，它是 Haskell "类型驱动开发"理念的延伸：

- **代码层面**：类型签名保证函数正确
    
- **模块层面**：导出列表保证抽象不被破坏
    
- **包层面**：版本约束保证 API 兼容性
    

三层叠加，Haskell 生态的依赖管理比其他语言严格得多——这也是为什么 Haskell 项目在生产环境中极少出现"依赖地狱"。

---

## 10.3 Haddock：让类型签名成为文档的单一事实源

### Haddock 的定位

Haskell 官网对 Haddock 的定义：

> _"This is Haddock, a tool for automatically generating documentation from annotated Haskell source code."_

**Haddock 是从"带注解的 Haskell 源码"自动生成文档的工具**。它的设计目标有三条 ：

1. **文档靠近接口**：文档写在源码旁边，避免文档与代码脱节
    
2. **文档易读易写**：注解语法轻量，不喧宾夺主
    
3. **超链接导航**：生成的文档完全可点击跳转——类型名链接到定义，标识符自动识别
    

### Haddock 注释语法

Haskell 官方教程给出了清晰的示例 ：

```
-- | Insert an element into a BST
insert :: Ord a =>
          a       -- ^ Value to be inserted
       -> Tree a  -- ^ Tree to insert into
       -> Tree a
insert x Empty = Node Empty x Empty
insert x (Node l y r)
  | x < y     = Node l y (insert x r)
  | otherwise = Node l x r
```

**🔑 洞见九：Haddock 注释的两种形式**

|注释形式|作用对象|
|---|---|
|`-- \\|`|注释紧随其后的定义（函数/类型/类）|
|`-- ^`|注释紧随其前的标识符（通常是函数参数）|
|`-- \*`|章节标题（`*` 越多层级越深）|

### 🔑 洞见十：Haddock 与模块导出列表的深度集成

这是 Haddock 最精妙的设计——**文档的结构由模块的导出列表决定**：

> _"All of this happens within the module's export list. The export list defines the module's API... Everything in the document that Haddock generates is listed according to the ordering of module's export list"_

也就是说，**你写导出列表的顺序，就是 Haddock 生成文档的顺序**。更巧妙的是，你可以在导出列表里穿插章节标题：

```
module Network.Wreq
  ( -- * HTTP verbs
    -- ** GET
    get, getWith,
    -- ** POST
    post, postWith,
    
    -- * Configuration
    Options, defaults,
    
    -- ** Authentication
    Auth, basicAuth, oauth1Auth,
  ) where
```

Haddock 会生成带目录的层级文档——`*` 是一级标题，`**` 是二级标题 。

> 💡 **这意味着：模块导出列表 = API 契约 = 文档结构**。三者是同一个东西！这是 Haskell "单一事实源"哲学的极致体现——你不需要在代码之外再维护一份 API 文档，导出列表本身就是文档大纲。

### 🔑 洞见十一：Haddock 理解 Haskell 的模块系统

Haddock 的一个独门绝技：

> _"Haddock understands Haskell's module system, so you can structure your code however you like without worrying that internal structure will be exposed in the generated documentation. For example, it is common to implement a library in several modules, but define the external API by having a single module which re-exports parts of these implementation modules."_

**这意味着你可以用多个内部模块实现库，用一个门面模块重导出公共 API，Haddock 能正确处理这种结构**——内部模块的 Haddock 注释会自动"传播"到门面模块的文档里 。

这种模式在 Haskell 生态里极其常见：

```
-- Internal/Impl.hs（内部实现）
module Internal.Impl (...) where
-- | 智能构造函数
createFoo :: Int -> Foo
...

-- Foo.hs（门面模块）
module Foo (module Internal.Impl) where
import Internal.Impl
```

用户在 Hackage 上看到的文档，就是从 `Foo` 模块导出的干净 API，但 `createFoo` 的 Haddock 注释来自 `Internal.Impl` 里的定义——**Haddock 自动完成了文档的传播**。

### 🔑 洞见十二：Haddock 是 Haskell 生态的"通用语言"

Hackage（Haskell 的中央包仓库）**只用 Haddock 生成 HTML 文档**​ 。这意味着：

- 每个上传到 Hackage 的包，都会自动获得 Haddock 文档
    
- 这些文档互相超链接——点击一个类型名，跳转到它的定义
    
- Hoogle（Haskell 的函数搜索引擎）直接索引 Haddock 文档
    

**Haddock 让整个 Haskell 生态的文档形成了一个互联的网络**。你在使用任何第三方库时，都能获得与标准库一致的文档体验——这极大降低了学习和使用成本。

### 实际应用

```
-- 一个 Haddock 友好的模块示例
module Data.Queue
  ( -- * 类型
    Queue,
    
    -- * 构造
    empty,
    singleton,
    fromList,
    
    -- * 操作
    enqueue,
    dequeue,
    peek,
    
    -- * 查询
    null,
    size,
  ) where

-- | 一个 FIFO 队列
data Queue a = Queue [a] [a]

-- | 创建一个空队列
empty :: Queue a
empty = Queue [] []

-- | 入队：O(1) 时间复杂度
enqueue :: a -> Queue a -> Queue a
enqueue x (Queue front back) = Queue front (x:back)

-- | 出队：摊还 O(1) 时间复杂度
dequeue :: Queue a -> Maybe (a, Queue a)
dequeue (Queue [] []) = Nothing
dequeue (Queue (x:xs) back) = Just (x, Queue xs back)
dequeue (Queue [] back) = 
  case reverse back of
    (x:xs) -> Just (x, Queue xs [])
    []     -> Nothing  -- 不会发生，但类型安全
```

运行 `cabal haddock` 后，会生成带目录、带类型签名、带超链接的 HTML 文档——与你在 Hackage 上看到的任何标准库文档完全一致。

---

## 🧭 本章在知识体系中的位置

```
前九章: 语言特性（data/type class/高阶函数/惰性求值）
   ↓
第十章: 模块 + cabal + Haddock  ← 工程化封装  【你在这里】
   ↓
第二部分: 函子/应用函子/单子/解析器  ← 用工程化思维构建抽象
```

**本章给你三件行李**：

1. **模块是抽象数据类型的封装边界**——通过导出列表精确控制 API，隐藏构造子以保护不变量
    
2. **cabal 是包的生命周期管理工具**——构建、依赖隔离、版本约束、发布
    
3. **Haddock 让类型签名成为文档的单一事实源**——导出列表即 API 契约即文档结构
    

---

## 📝 本章核心洞见总结

> **洞见一：模块是 Haskell 构建抽象数据类型的唯一途径**。
> 
> 通过隐藏构造子，模块强制用户只能通过智能构造函数创建值——这让"不变量"在编译期就被保护。这是"make illegal states unrepresentable"从类型层面延伸到模块边界的关键手段。

> **洞见二：导出列表是 API 设计的艺术**。
> 
> `DataType` vs `DataType(..)` vs `DataType(C1, C2)`——每种写法都传达不同的设计意图。精确的导入控制（`qualified`、选择性导入、`hiding`）是大型 Haskell 项目的生存必需品。

> **洞见三：模块名层级是约定而非语言特性**。
> 
> `Data.Set` 和 `Data.Set.Extra` 在语言层面完全独立，点分隔只是社区约定。这套约定让 Haskell 生态的包组织极度规整。

> **洞见四：cabal 是"类型安全"从代码延伸到工程的载体**。
> 
> PVP 版本约束让依赖兼容性在类型系统之外再添一道保险。`base >=4.7 && <5` 这样的约束，本质是"API 兼容性"的类型签名。

> **洞见五：Haddock 的文档结构由导出列表决定**。
> 
> 你写导出列表的顺序，就是文档的顺序；在导出列表里穿插 `-- *` 标题，就是定义文档的章节结构。**导出列表 = API 契约 = 文档大纲**——三者是同一个事实源的三种视图。

> **洞见六：Haddock 理解 Haskell 的模块系统**。
> 
> 内部模块的实现细节 + 门面模块的重导出 = 干净的公共 API 文档。Haddock 自动把内部模块的文档注释"传播"到门面模块——这让"实现分散、接口统一"的库设计模式成为可能。

> **洞见七：模块 + cabal + Haddock 三位一体**。
> 
> 模块解决"命名空间与抽象边界"，cabal 解决"包的生命周期"，Haddock 解决"文档与接口的同步"。三者共同构成了 Haskell 工程化的基石——让类型驱动的正确性从单个文件扩展到整个生态系统。

---

## 🛠️ 实践建议

1. **永远写显式导出列表**：
    
    ```
    module MyLib (importantFunc, ImportantType) where
    -- 而不是 module MyLib where
    ```
    
    显式导出列表不仅控制 API，还能启用某些编译优化 。
    
2. **默认用 qualified 导入**：
    
    ```
    import qualified Data.Map as Map
    import qualified Data.Text as T
    ```
    
    只有 `Prelude`、项目内部模块才用普通导入。
    
3. **隐藏构造子以保护不变量**：
    
    ```
    module Bank (Account, createAccount, deposit, withdraw) where
    data Account = Account { balance :: Int }
    
    createAccount :: Int -> Account
    createAccount initial
      | initial < 0 = error "余额不能为负"
      | otherwise = Account initial
    ```
    
4. **为项目使用依赖隔离**：
    
    ```
    # 现代方式：使用 cabal.project + v2-build
    cabal build --project-file=cabal.project
    ```
    
5. **Haddock 注释是每个导出项的标配**：
    
    ```
    -- | 简短描述函数的意图
    -- 详细说明复杂行为、边界条件、不变量
    -- 使用示例：
    --
    -- > take 5 (repeat 1)
    -- [1,1,1,1,1]
    myFunc :: Int -> [Int]
    myFunc n = ...
    ```
    
    用 `>` 开头的注释会被 Haddock 当作代码示例执行测试（doctest）。
    
6. **在 .cabal 文件里精确声明版本约束**：
    
    ```
    build-depends: 
      base >=4.7 && <5,
      containers >=0.5 && <0.7,
      text ^>=1.2.3
    ```
    
    `^>=` 表示 `>=1.2.3 && <1.3`（PVP 兼容）。
    

---

## 🔮 承上启下

第二部分"重要的类型和类型类"即将开启——第十一章的**函子（Functor）**、第十三章的**应用函子（Applicative）**、第十六章的**单子（Monad）**——这些高阶抽象若没有本章的模块系统，就无法被组织成 `Data.Functor`、`Control.Applicative`、`Control.Monad` 这样的标准库模块；若没有 cabal，就无法作为包分发到 Hackage；若没有 Haddock，就无法生成我们在 Hackage 上看到的精美文档。

更深远的连接：

- **第十一章 Functor**：`fmap :: Functor f => (a -> b) -> f a -> f b` 是定义在 `Data.Functor` 模块里的类型类。你能在自己的项目里 `import Data.Functor`，正是模块系统的功劳
    
- **第十二章 Lens**：镜头库（lens）是 Hackage 上最复杂的包之一，它用数百个模块组织代码，通过精确的导出列表隐藏实现细节——这是第十章"模块封装"理念的极致展示
    
- **第十三章 Applicative**：`optparse-applicative`（第十五章）是一个独立的 cabal 包，你通过 `cabal install optparse-applicative` 安装它，然后在代码里 `import Options.Applicative`——这就是"模块 + cabal + Haddock"三位一体的完整闭环
    

**第十章是你从"Haskell 程序员"迈向"Haskell 工程师"的关键一跃**。吃透模块的导出控制、cabal 的包管理、Haddock 的文档生成，第二部分的 Functor/Applicative/Monad 就不只是"语言抽象"——它们是"可复用、可发布、可文档化的工程抽象"。你将在 Hackage 上看到标准库的文档，理解它们是如何被模块组织、被 cabal 构建、被 Haddock 文档化的——这种"全栈理解"是掌握 Haskell 生态的关键。

> 💡 学习建议：读完本章后，立刻动手创建一个自己的 cabal 项目：
> 
> ```
> mkdir my-first-lib && cd my-first-lib
> cabal init  # 选择 library 模式
> # 编辑 .cabal 文件，写几个带 Haddock 注释的函数
> cabal haddock --open  # 生成并在浏览器打开文档
> ```
> 
> 亲眼看见自己的类型签名变成 Hackage 风格的文档，是理解本章精髓的最佳方式。

