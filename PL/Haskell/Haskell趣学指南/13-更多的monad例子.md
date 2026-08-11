
# 第13章 再来看看更多 Monad（For a Few Monads More）——读书笔记

> 💡 前几章你已经建立了 Monad 的核心认知：Monad 类型类用 `return` 和 `>>=` 捕捉"带效应的计算组合"。但 Monad 不是一个具体的东西，而是一类**计算模式的抽象**——不同的"效应"对应不同的 Monad 实例。本章就是带你**参观效应动物园**：Writer（日志）、Reader（只读环境）、State（可变状态）、Error（错误处理）。每一个具体 Monad 都在解决一个真实的工程问题，而它们共享同一个 `>>=` 组合子——这正是 Monad 抽象威力的终极证明：**用同一套组合语法，统一表达日志、环境、状态、失败等截然不同的效应**。

> 📌 章节说明：按《Haskell 趣学指南》中文目录，本章为"第十三章 再来看看更多 Monad"；对应的英文原书是 Chapter 14 "For a Few Monads More"，而英文 Chapter 13 "A Fistful of Monads" 在中文版中对应第12章"Monad 类型类"等内容。以下按你指定的"第十三章 更多的 monad 的例子"展开。

---

## 本章小节地图

§13.1 你所不知道的 Writer Monad（13.1.1 Monoids 的好处 · 13.1.2 The Writer type · 13.1.3 Using do notation with Writer · 13.1.4 Adding logging to programs · 13.1.5 Inefficient list construction · 13.1.6 Difference lists · 13.1.7 Comparing Performance）

§13.2 Reader Monad

§13.3 State Monad（13.3.1 Stack and Stones · 13.3.2 The State Monad · 13.3.3 随机性与 state monad）

§13.4 Error Monad

§13.5 一些实用的 Monadic functions（13.5.1 liftM · 13.5.2 The join function · 13.5.3 filterM · 13.5.4 foldM · 13.5.5 Making a safe RPN calculator · 13.5.6 Composing monadic functions）

§13.6 定义自己的 Monad

---

## 🎯 全局脉络：为什么需要"更多 Monad"

第10章"函数式地解决问题"交付了 Haskell 工程的标准配方：**纯核心 + I/O 外壳**。但纯核心里经常出现一些"准效应"——你想要日志、想要配置访问、想要状态传递、想要优雅的错误处理。如果每次都手写这些模式的样板代码，会淹没业务逻辑。

**Monad 的本质价值**：把"效应"本身变成**可组合的一等公民**。一旦某种效应有了 Monad 实例，你就可以用 `do` 表示法写出"看起来像纯代码、实际在跑效应"的程序。

本章的四个核心 Monad，对应四类最常见的效应需求：

|Monad|效应语义|典型场景|
|---|---|---|
|**Writer**​|附加日志值（累加）|调试日志、审计跟踪|
|**Reader**​|只读环境（共享）|配置访问、依赖注入|
|**State**​|可变状态（传递）|解析器、随机数、模拟|
|**Error / Maybe**​|可能失败（短路）|错误处理、验证链|

> 📌 **洞见**：这四个 Monad 不是"四个独立的东西"，而是**同一个 `>>=` 组合子在四种不同效应语义下的实例化**。它们的 `return` 和 `>>=` 实现各不相同，但都满足 Monad 定律——这意味着你可以用**完全相同的 `do` 语法**来组合它们。这就是抽象的力量：**语法不变，语义可插拔**。

更深一层——本章的 Monad 与前几章的知识血脉相连：

```
第11章 Monoid（值的组合）
  ↓
第11章 Applicative（独立效应的组合）
  ↓
第12章 Monad（依赖效应的组合）
  ↓
第13章（本节）具体 Monad 实例
  │
  ├── Writer ──────► 依赖 Monoid（w 必须是 Monoid）
  ├── Reader ──────► 对应第11章"函数作为 Applicative"
  ├── State ───────► 对应第10章 RPN 计算器的"栈状态机"
  └── Error ──────► 对应第7章 Maybe 的失败处理
```

**本章最深刻的洞见**：你以为在学习"四个新东西"，其实你在学习"一个抽象（Monad）的四种实例化"。理解了这一点，你就真正跨过了 Monad 学习的门槛——**Monad 不再是一个神秘的概念，而是"效应组合"的代数结构，Writer/Reader/State/Error 只是这个结构的具体例子**。

---

## 🔍 逐节洞见

### §13.1 Writer Monad —— 日志即效应，Monoid 是引擎

**动机**：假设你有一个函数 `isBigGang :: Int -> Bool`，你想让它"除了返回 Bool，还附加一个日志字符串解释它在做什么"。朴素的做法是改成 `Int -> (Bool, String)`——但这样你就得手动在每个调用点拼接日志，代码迅速膨胀。

**Writer 类型**：

```
newtype Writer w a = Writer { runWriter :: (a, w) }
```

`Writer w a` 是一个"装有值 `a` 和日志 `w`"的计算。

**Monad 实例**：

```
instance (Monoid w) => Monad (Writer w) where
  return x = Writer (x, mempty)
  (Writer (x, v)) >>= f = 
    let (Writer (y, v')) = f x 
    in Writer (y, v `mappend` v')
```

> 💡 **注意 `instance (Monoid w) =>` 这个约束**——Writer 的日志聚合**要求 `w` 是 Monoid**。这正是上一章 Monoid 的直系应用：`mempty` 是空日志，`mappend` 是日志拼接。没有 Monoid，Writer 就无法工作——这是"Monoid 是组合的数学本质"这一论点的活生生证据。

**`>>=` 的语义**：`(Writer (x, v)) >>= f` 取出值 `x` 和已有日志 `v`，把 `x` 传给 `f` 得到新的 Writer `(y, v')`，然后把两段日志 `v`mappend`v'` 拼接起来。日志在 `>>=` 过程中**自动累积**——你完全不需要手动管理。

**用 `do` 表示法写带日志的计算**：

```
logNumber :: Int -> Writer [String] Int
logNumber x = writer (x, ["Got number: " ++ show x])

multWithLog :: Writer [String] Int
multWithLog = do
  a <- logNumber 3
  b <- logNumber 5
  tell ["Multiplying " ++ show a ++ " and " ++ show b]
  return (a * b)

-- 运行结果：
-- ghci> runWriter multWithLog
-- (15, ["Got number: 3", "Got number: 5", "Multiplying 3 and 5"])
```

`tell` 函数是 Writer 的专用操作，等价于 `tell w = writer ((), w)`——只写日志不产生值。

> 📌 **洞见**：Writer 的优雅之处在于——**业务逻辑 `a <- logNumber 3; b <- logNumber 5; return (a*b)` 看起来和纯代码一模一样**，但日志在背后自动累积。这就是 Monad 的核心价值：**让你专注于"值"，让"效应"在类型系统里自动流动**。

**§13.1.5-13.1.7 性能陷阱与 Difference Lists**：

Writer 用列表 `[String]` 做日志时，`mappend` 是 `++`——**左关联拼接是 O(n²)**​ 的。Writer 的 `>>=` 是左关联的：

```
(((mempty `mappend` w1) `mappend` w2) `mappend` w3)
-- 每次 mappend 都要遍历左日志
```

解决方案是 **Difference List（DList）**——一种 O(1) 拼接的列表表示：

```
newtype DList a = DList { runDList :: [a] -> [a] }

instance Monoid (DList a) where
  mempty = DList id
  DList f `mappend` DList g = DList (f . g)  -- O(1) 组合
```

DList 的本质是**函数组合**——`f . g` 是 O(1) 的，只有最终 `runDList dl []` 一次性生成列表时才付出 O(n) 代价。这是上一章 `Endo` Monoid 思想的实战应用。

> ⚠️ **工程提醒**：用 `Writer [String]` 处理大量日志时，务必切换到 `Writer (DList String)` 或用 `Data.Text` 等高效结构，否则性能会雪崩。这是 Haskell 工程中最经典的"隐性 O(n²)"陷阱之一。

**§13.1.3 的现代变化**：mtl 库从 1.x 升级到 2.x 后，`Writer` 从数据构造器变成了函数 `writer`。旧版 `Writer (x, ["log"])` 要改为新版 `writer (x, ["log"])`。这是 LYAH 教材的一个过时点——**遇到这种"教材代码编译不过"的情况，查 mtl 的当前 API 而非盲从教材**。

---

### §13.2 Reader Monad —— 只读环境的依赖注入

**动机**：很多计算需要访问一个"全局环境"——配置、数据库连接、用户信息。每次都把环境作为参数传递，代码会很笨重。

**Reader 类型**：

```
newtype Reader e a = Reader { runReader :: e -> a }
```

`Reader e a` 是一个"在环境 `e` 下产生 `a`"的计算。

**Monad 实例**：

```
instance Monad (Reader e) where
  return a = Reader $ \_ -> a
  (Reader g) >>= f = Reader $ \e ->
    let (Reader h) = f (g e)
    in h e
```

**`>>=` 的语义**：`g e` 在环境 `e` 下得到值 `a`，把 `a` 传给 `f` 得到新的 Reader `h`，在**同一个环境 `e`**​ 下运行 `h`。环境在 `>>=` 过程中**不变地传递**——这正是"只读"的含义。

**与第11章"函数作为 Applicative"的关系**：

回想第11章学的——函数 `(->) r` 是 Functor、Applicative、Monad。Reader e a 本质上就是 `e -> a` 的 newtype 包装！**Reader Monad 就是"函数作为 Monad"的特例**。这意味着：

```
-- 这两个类型是等价的
Reader e a  ≅  e -> a
```

**使用场景**：

```
type Config = Map String String

getConfig :: String -> Reader Config String
getConfig key = Reader $ \cfg -> 
  fromMaybe "default" (Map.lookup key cfg)

app :: Reader Config String
app = do
  host <- getConfig "host"
  port <- getConfig "port"
  return $ host ++ ":" ++ port

-- 运行
-- ghci> runReader app configMap
```

> 📌 **洞见**：Reader Monad 是 Haskell 版的"依赖注入"。与 OOP 框架（Spring、Guice）不同，Haskell 不需要任何框架——**Reader Monad 用纯粹的函数组合就实现了依赖注入**。环境 `e` 在 `>>=` 链中自动传递，业务逻辑完全不感知"配置从哪里来"。这是"让非法状态无法被表示"理念的延伸——你根本无法写出"忘记传递配置"的代码，因为类型系统强制了。

---

### §13.3 State Monad —— 状态传递的终极化

**动机**：第10章 RPN 计算器里，你用手写的"栈"作为状态，用 `foldl` 驱动状态机。但手写状态传递容易出错，且难以组合。State Monad 把这种模式**抽象成类型类实例**。

**Stack and Stones（13.3.1）**：作者用走钢索的比喻——一个人左右两边各有数字栈，每次可以选择从左边或右边拿数做操作。这就是一个典型的"状态机"问题。

**State 类型**：

```
newtype State s a = State { runState :: s -> (a, s) }
```

`State s a` 是一个"接受状态 `s`、产生值和点滴新状态 `s`"的计算。

**Monad 实例**：

```
instance Monad (State s) where
  return x = State $ \s -> (x, s)
  (State h) >>= f = State $ \s ->
    let (a, newState) = h s
        (State g) = f a
    in g newState
```

**`>>=` 的语义**：

```
(State op1) >>= f = State $ \firstState ->
  let (a, secondState) = op1 firstState
      (State op2) = f a
      (b, thirdState) = op2 secondState
  in (b, thirdState)
```

`op1` 在 `firstState` 下运行，产生 `a` 和 `secondState`；`f a` 产生 `op2`；`op2` 在 `secondState` 下运行，产生最终结果 `b` 和 `thirdState`。**状态在 `>>=` 链中自动线程化（threaded）**——这正是 State Monad 的精髓。

**基本操作**：

```
get :: State s s
get = State $ \s -> (s, s)        -- 读取当前状态

put :: s -> State s ()
put s = State $ \_ -> ((), s)     -- 设置新状态

modify :: (s -> s) -> State s ()
modify f = do
  s <- get
  put (f s)                       -- 读取、变换、写回
```

**走钢索问题**：

```
type Stack = ([Int], [Int])  -- 左栈和右栈

pushLeft :: Int -> State Stack ()
pushLeft x = modify $ \(left, right) -> (x:left, right)

popLeft :: State Stack (Maybe Int)
popLeft = do
  (left, right) <- get
  case left of
    [] -> return Nothing
    (x:xs) -> do
      put (xs, right)
      return (Just x)

-- 钢索行走逻辑
walk :: State Stack Bool
walk = do
  leftTop <- popLeft
  rightTop <- popRight
  case (leftTop, rightTop) of
    (Just l, Just r) -> do
      -- 执行平衡操作...
      walk  -- 递归继续
    _ -> return False  -- 失败
```

> 💡 **洞见**：State Monad 把第10章 RPN 计算器里"手写的栈状态机"**提升为类型类级别的抽象**。你不再需要手动传递状态——`do` 块里的每一行都在"当前的 State 语境下"运行，`get`/`put`/`modify` 让你自然地读写状态。这是"效应组合"思想的最高成就：**状态传递的样板代码完全消失，业务逻辑脱颖而出**。

**§13.3.3 随机性与 State Monad**：

第9章你学过 `random :: (Random a, RandomGen g) => g -> (a, g)`——它显式传递生成器。这正是 State Monad 的完美场景：

```
import System.Random
import Control.Monad.State

randomSt :: (RandomGen g, Random a) => State g a
randomSt = State random

threeCoins :: State StdGen (Bool, Bool, Bool)
threeCoins = do
  a <- randomSt
  b <- randomSt
  c <- randomSt
  return (a, b, c)

-- 运行
-- ghci> runState threeCoins (mkStdGen 42)
-- ((True, False, True), <new generator>)
```

> 📌 **洞见**：第9章"显式传递随机数生成器"的样板代码，在 State Monad 下消失了。`randomSt` 把 `random` 函数包装成 State 动作，`threeCoins` 看起来就像一串普通的 `do` 语句——但背后生成器状态在自动线程化。这是 State Monad 最经典的应用案例。

---

### §13.4 Error Monad —— 错误处理的 Monad 化

**动机**：第7章你学过用 `Maybe` 建模"可能失败"。但当失败时，你往往想知道**为什么失败**——`Nothing` 携带不了错误信息。`Either String a` 可以携带错误信息，而 Error Monad 让 `Either` 成为 Monad。

**Error Monad 的本质**：`Either e` 是 Monad 实例，`Left e` 表示失败（携带错误），`Right a` 表示成功。

```
instance Monad (Either e) where
  return = Right
  Right x >>= f = f x
  Left e >>= _ = Left e  -- 失败短路
```

**关键语义**：`>>=` 在遇到 `Left e` 时**立即短路**——后续计算不再执行，错误直接向上传播。这正是"错误处理链"的理想语义。

**使用场景**：

```
validateAge :: Int -> Either String Int
validateAge age
  | age < 0 = Left "Age cannot be negative"
  | age > 150 = Left "Age seems unrealistic"
  | otherwise = Right age

validateName :: String -> Either String String
validateName name
  | null name = Left "Name cannot be empty"
  | length name > 50 = Left "Name too long"
  | otherwise = Right name

createUser :: String -> Int -> Either String User
createUser name age = do
  validName <- validateName name
  validAge <- validateAge age
  return $ User validName validAge
```

> 💡 **洞见**：Error Monad 让"验证链"变得优雅——`do` 块里任何一步失败，整个计算立即返回 `Left` 错误。这比嵌套的 `case` 匹配清爽得多。这是第10章"用 Maybe 建模失败"思想的自然延伸——**`Either` 是携带错误信息的 `Maybe`**。

---

### §13.5 实用的 Monadic 函数 —— Monad 生态的工具箱

这一节巡览 `Control.Monad` 里的通用组合子——它们都是针对任意 Monad 的，因此对 Writer/Reader/State/Error 都适用。

**§13.5.1 liftM**：

```
liftM :: Monad m => (a -> b) -> m a -> m b
liftM f ma = ma >>= \a -> return (f a)
```

`liftM` 就是 `fmap`！在 Monad 语境下，`fmap = liftM`。这再次验证了 Functor ⊂ Applicative ⊂ Monad 的层次关系。

**§13.5.2 join**：

```
join :: Monad m => m (m a) -> m a
join mma = mma >>= id
```

`join` 把"两层 Monad"压扁为一层。它的存在揭示了 Monad 的另一个定义方式——**Monad = Functor + pure + join**（而非 Functor + pure + `>>=`）。这两种定义在数学上等价。

**§13.5.3 filterM**：

```
filterM :: Monad m => (a -> m Bool) -> [a] -> m [a]
```

`filterM` 是"效应化的 filter"——判定函数返回 `m Bool` 而非 `Bool`。在 IO 中，`filterM` 可以做"基于文件系统检查的过滤"：

```
import System.Directory

-- 过滤出存在的文件
existingFiles :: [FilePath] -> IO [FilePath]
existingFiles paths = filterM doesFileExist paths
```

**§13.5.4 foldM**：

```
foldM :: Monad m => (a -> b -> m a) -> a -> [b] -> m a
```

`foldM` 是"效应化的 foldl"——累加函数返回 `m a` 而非 `a`。这是第10章"用 foldl 驱动状态机"思想的 Monad 化版本：

```
-- 效应化的 RPN 计算器
solveRPN :: String -> Maybe Double
solveRPN expr = do
  result <- foldM step [] (words expr)
  case result of
    [x] -> Just x
    _ -> Nothing
  where step (x:y:ys) "+" = Just ((x + y):ys)
        step (x:y:ys) "*" = Just ((x * y):ys)
        step stack num = Just (read num : stack)
        step _ _ = Nothing
```

> 💡 **这比第10章的手写 RPN 更好**：`foldM` 让"栈状态机"跑在 `Maybe` Monad 里，失败时自动短路；`step` 函数返回 `Maybe`，错误传播完全自动化。

**§13.5.5 安全的 RPN 计算器**：

作者用 Monad 重新实现第10章的 RPN 计算器——用 `Maybe` Monad 处理错误，代码比原来的 `head . foldl` 版本**更安全、更清晰**。

**§13.5.6 组合 Monadic 函数**：

```
(<=<) :: Monad m => (b -> m c) -> (a -> m b) -> (a -> m c)
f <=< g = \x -> g x >>= f
```

`(<=<)` 是 Monad 世界的函数组合——它对应纯函数世界的 `(.)`。这是第5章函数组合的 Monad 版本：

```
-- 纯函数组合
(f . g) x = f (g x)

-- Monadic 函数组合
(f <=< g) x = g x >>= f
```

> 📌 **洞见**：`(<=<)` 与 `(.)` 的同构关系，揭示了 Monad 与纯函数的深刻对应——**Monad 不过是在"效应维度"上推广了函数组合**。这一认识直接通往第12章 Monad 定律的数学本质。

---

### §13.6 定义自己的 Monad

章节末尾，作者引导你为自定义类型编写 Monad 实例——把本章学的所有知识融会贯通。

**关键提醒**：要成为 Monad，必须满足三条 Monad 定律：

```
-- 左单位元
return a >>= f  ≡  f a

-- 右单位元
m >>= return  ≡  m

-- 结合律
(m >>= f) >>= g  ≡  m >>= (\x -> f x >>= g)
```

> ⚠️ **定律是契约**：编译器不检查 Monad 定律，但整个 Monad 生态（包括 `do` 表示法、`liftM`、`foldM` 等）都依赖于此。违反定律的 Monad 实例会导致诡异行为。

---

## 🧬 跨章节连接：具体 Monad 是怎么"长"在抽象体系上的

```
第7章 ADT + 类型类
  ↓
第11章前文 Functor / Applicative
  ↓
第11章 11.4 Monoid（值的组合）
  ↓
第12章 Monad（依赖效应的组合）
  ↓
第13章（本节）具体 Monad 实例
  │
  ├── Writer w a ─────────► 依赖 Monoid（w 必须是 Monoid）
  │                         ★ 上一章 Monoid 的直系应用
  │                         ★ 日志聚合 = mappend
  │                         ↓
  │                         工程应用：调试日志、审计跟踪
  │
  ├── Reader e a ─────────► 对应第11章"函数作为 Applicative/Monad"
  │                         e -> a 的 newtype 包装
  │                         ★ 依赖注入的函数式实现
  │                         ↓
  │                         工程应用：配置管理、环境访问
  │
  ├── State s a ─────────► 对应第10章 RPN 计算器的"栈状态机"
  │                         s -> (a, s) 的 newtype 包装
  │                         ★ 状态传递的 Monad 化
  │                         ↓
  │                         工程应用：解析器、模拟、随机数
  │
  ├── Either e a ────────► 对应第7章 Maybe 的错误携带版
  │                         ★ 错误处理的 Monad 化
  │                         ↓
  │                         工程应用：验证链、错误处理
  │
  └── 实用函数 (liftM, join, filterM, foldM, <=<) ──► 任意 Monad 通用
                                                    ★ 第5章高阶函数的 Monad 版
                                                    ★ 组合子的抽象复用
```

**一条主线串起来看**：

1. **第7章**：你学了 ADT 和类型类——为自定义类型写实例
    
2. **第10章**：你手写"栈状态机"解决 RPN 问题——用 `foldl` 驱动状态
    
3. **第11章 Monoid**：你学了"组合的数学本质"——`mappend` 和 `mempty`
    
4. **第11章 Applicative**：你学了"独立效应的组合"
    
5. **第12章 Monad**：你学了 `>>=` 和 `return`——"依赖效应的组合"
    
6. **第13章（本节）**：你把 Monad 抽象**实例化**为具体效应：
    
    - **Writer**​ 用 Monoid 聚合日志
        
    - **Reader**​ 是函数作为 Monad 的 newtype 包装
        
    - **State**​ 把第10章手写的"栈状态机"提升为类型类抽象
        
    - **Error (Either)**​ 把第7章的 Maybe 失败处理升级为携带错误
        
    
7. **实用函数**：`liftM`、`join`、`filterM`、`foldM`、`<=<`——它们都是**针对任意 Monad 的通用组合子**，体现了"抽象级别的重用"
    

> 💡 **如果你只记住一件事**：**本章的四个 Monad 不是四个独立概念，而是 Monad 抽象在四种效应语义下的实例化**。它们共享同一个 `>>=` 组合子，区别在于 `return` 和 `>>=` 的具体实现：
> 
> - **Writer**​ 的 `>>=` 在组合时 **mappend 日志**（依赖 Monoid）
>     
> - **Reader**​ 的 `>>=` 在组合时 **保持环境不变**
>     
> - **State**​ 的 `>>=` 在组合时 **线程化状态**
>     
> - **Either**​ 的 `>>=` 在组合时 **遇错短路**
>     
> 
> 理解这一点，你就真正掌握了 Monad 的精髓——**Monad 不是"容器"，而是"计算的组合代数"**。不同的效应只是这个代数结构的不同"模型"（model），正如群论中的"整数加法群"和"矩阵乘法群"都是群的具体实例。

---

## 🛠 实际工程启示

**1. 用 Writer 做调试日志，但小心性能**

```
-- ❌ 反模式：用 [String] 做大量日志
heavyComputation :: Writer [String] Result
heavyComputation = do
  -- ... 产生大量日志 ...
  -- O(n²) 拼接！

-- ✅ 推荐：用 DList 或 Text
import Data.DList (DList)
import qualified Data.DList as D

heavyComputation :: Writer (DList String) Result
heavyComputation = do
  tell (D.singleton "Step 1")
  tell (D.singleton "Step 2")
  -- O(1) 拼接
```

**2. 用 Reader 做配置管理**

```
data AppConfig = AppConfig
  { dbConnection :: Connection
  , apiKey :: String
  , debugMode :: Bool
  }

type App a = Reader AppConfig a

getDB :: App Connection
getDB = Reader dbConnection

runApp :: App a -> AppConfig -> IO a
runApp app cfg = return $ runReader app cfg
```

**3. 用 State 写解析器和状态机**

```
-- 解析器是 State Monad 的经典应用
type Parser = State String

char :: Parser Char
char = do
  s <- get
  case s of
    [] -> fail "Unexpected end of input"
    (c:cs) -> do
      put cs
      return c
```

**4. 用 Either 做错误处理链**

```
-- 业务验证
validateInput :: Input -> Either ValidationError ProcessedInput
validateInput input = do
  v1 <- validateField1 (field1 input)
  v2 <- validateField2 (field2 input)
  v3 <- validateField3 (field3 input)
  return $ ProcessedInput v1 v2 v3
  -- 任何一步失败，整个计算返回 Left
```

**5. 用 foldM 写效应化的遍历**

```
-- 数据库事务：逐步执行，任何一步失败则回滚
processTransactions :: [Transaction] -> StateT DBState IO (Either Error ())
processTransactions txns = do
  result <- foldM step (Right ()) txns
  return result
  where step (Left err) _ = return (Left err)
        step (Right ()) txn = do
          res <- liftIO $ executeTxn txn
          case res of
            Left err -> return (Left err)
            Right _ -> return (Right ())
```

**6. 用 (<=<) 组合 Monadic 函数**

```
-- 纯函数组合
h = f . g . k

-- Monadic 函数组合
h' = f <=< g <=< k

-- 管道式数据处理
processUser :: UserId -> App (Either Error UserProfile)
processUser = fetchUser <=< validateId <=< parseId
```

**7. 现代 mtl 的变化要点**

```
-- 旧版 LYAH 代码
Writer (x, ["log"])

-- 新版 mtl 2.x 必须改为
writer (x, ["log"])

-- 或者更惯用的 tell
do
  tell ["log"]
  return x
```

**8. Monad 定律是契约**

写 Monad 实例时，心里要有三条定律。违反定律的实例会导致：

- `return a >>= f` 不等于 `f a` —— `do` 表示法行为异常
    
- `m >>= return` 不等于 `m` —— 计算被意外修改
    
- 结合律不成立 —— `foldM` 等组合子行为诡异
    

**9. Monad 栈的组合（进阶）**

真实应用往往需要多种效应叠加——用 `MTL` 风格的类型类约束：

```
type App a = ReaderT Config (StateT AppState (ExceptT AppError IO)) a

-- 这个类型表达了：
-- - 读配置（Reader）
-- - 可变状态（State）
-- - 错误处理（Except）
-- - I/O 能力（IO）
-- 所有效应通过 Monad Transformer 栈组合
```

这是工业级 Haskell 应用的标配架构。

**10. Writer 的日志是 Monoid 的活广告**

```
-- 任何 Monoid 都可以作为 Writer 的日志类型
type LogWriter a = Writer (Sum Int) a     -- 计数日志
type TimeWriter a = Writer (Sum Double) a -- 耗时日志
type ErrWriter a = Writer (Any) a         -- 错误标志日志

-- 不同的日志类型，相同的 Monad 抽象
```

---

## 一句话收束

> 第13章不是在教你"四个新 Monad"，而是在揭示 Monad 抽象的实例化力量：**Writer、Reader、State、Either 不是四个独立概念，而是同一个 `>>=` 组合子在四种效应语义下的具体模型**——Writer 用 Monoid 聚合日志（直接依赖上一章的 Monoid 理论）、Reader 是"函数作为 Monad"的 newtype 包装（呼应第11章 Applicative）、State 把第10章 RPN 计算器手写的"栈状态机"提升为类型类抽象、Either 把第7章 Maybe 的失败处理升级为携带错误。它们的 `return` 和 `>>=` 实现各异，但都遵循 Monad 定律——这意味着你可以用**完全相同的 `do` 语法**组合它们，实现"语法不变、语义可插拔"。而 `liftM`、`join`、`filterM`、`foldM`、`<=<` 这些通用组合子，则是针对**任意 Monad**​ 的抽象工具——它们把第5章高阶函数的思想提升到了效应维度。从这一章开始，你真正掌握了 Haskell 效应系统的全貌：**Functor 解决"在容器内做映射"、Applicative 解决"独立效应的组合"、Monad 解决"依赖效应的组合"、而具体 Monad 实例则是这些抽象在真实工程场景中的 instantiation**。当你把这些 Monad 与第10章"纯核心 + I/O 外壳"的工程配方结合时，你就拥有了构建工业级 Haskell 应用的完整工具箱——配置用 Reader、状态用 State、日志用 Writer、错误用 Except、边界用 IO，所有效应通过 Monad Transformer 栈优雅组合。Monad 不再是神秘的概念，而是**"效应组合"的代数结构**——本章的四个实例，正是这个代数结构最经典的模型。

