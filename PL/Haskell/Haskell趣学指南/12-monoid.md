
# Monoid —— 读书笔记

> 💡 在本书的中文目录结构中，Monoid 内容位于"第11章 Functors, Applicative Functors 与 Monoids"的 11.4 节 ；你提到的"第十二章"可能是指这个 Monoid 部分。前几章你爬升了 Functor → Applicative 的抽象阶梯：Functor 解决"在容器内做映射"，Applicative 解决"独立效应的组合"。但有一个更古老、更基础的代数结构一直被隐含使用——**如何把一堆值"合并"成一个值？**​ 答案就是 Monoid。这一章表面在教你 `mempty` 和 `mappend`，实质是在揭示一个数学真理：**任何具备"结合律"和"单位元"的二元操作，都构成了一个可组合、可折叠、可并行的代数结构**。Monoid 不是 Haskell 的语法特性，而是**宇宙间"组合"行为的数学本质**——它是 Foldable 的理论基石，是 Writer Monad 的日志机制，是并行计算的天然抽象。

---

## 本章小节地图

11.4.1 Lists are monoids（列表是幺半群）

11.4.2 Product and Sum（乘积与和）

11.4.3 Any and All（任意与所有）

11.4.4 The Ordering monoid（Ordering 的幺半群实例）

11.4.5 Maybe the monoid（Maybe 的幺半群实例）

11.4.6 Using monoids to fold data structures（用 Monoid 折叠数据结构）

> 📌 注：在《Haskell 趣学指南》中，Monoid 是第11章"Functors, Applicative Functors 与 Monoids"的 11.4 节 。

---

## 🎯 全局脉络：为什么"组合"需要数学公理

命令式思维里，"把一堆东西合并成一个"是随手写的逻辑：`result += x`、`result = result + x`。没人问过——**这种"合并"操作需要满足什么条件，才能保证结果正确、可优化、可并行？**

Monoid 给出了答案：**两个定律**。

### Monoid 的数学定义

一个类型 `a` 是 Monoid，如果它提供 ：

```
class Semigroup a => Monoid a where
  mempty  :: a              -- 单位元
  mappend :: a -> a -> a    -- 结合律二元操作（现代 Haskell 中用 (<>) 表示）
  mconcat :: [a] -> a       -- 默认实现：foldr mappend mempty
```

**必须满足三条定律**​ ：

```
-- 左单位元
mempty `mappend` x = x
-- 右单位元  
x `mappend` mempty = x
-- 结合律
(x `mappend` y) `mappend` z = x `mappend` (y `mappend` z)
```

> 📌 **洞见**：这三条定律看起来平淡无奇，但它们是整个可组合计算宇宙的基石。

**单位元定律的意义**：`mempty` 是"什么都不做"的操作。它让你能安全地初始化累加器、处理空输入、定义默认值。没有单位元，`mconcat []` 就无法定义——你无法把"零个值"合并成"一个值"。

**结合律定律的意义**：**括号怎么加都行**。这意味着：

- **求值顺序无关**——你可以从左到右算，也可以从右到左算，还可以并行算多个片段再合并
    
- **树形组合合法**——`(a <> b) <> (c <> d)` 与 `a <> (b <> (c <> d))` 结果相同
    
- **增量计算可行**——已经合并的结果可以和新值继续合并，不需要重算
    

> 💡 **结合律 = 并行性的通行证**。这是 Monoid 最深刻的含义。在分布式系统和并行计算里，Monoid 结构是"可并行 reduce"的充分必要条件——因为结合律保证了"先局部合并、再全局合并"与"全局顺序合并"结果一致。MapReduce 的 reducer、Spark 的 aggregate、GPU 的 parallel reduce，本质上都在利用 Monoid 的结合律。

**现代 Haskell 的重要变更**：自 `base-4.11.0.0` 起，`Semigroup` 成为 `Monoid` 的父类 ——

```
class Semigroup a => Monoid a where
  mempty :: a
  mappend = (<>)  -- 默认实现来自 Semigroup
```

也就是说，**任何 Monoid 首先必须是 Semigroup**（具备结合律的二元操作），再加上单位元 `mempty` 才算 Monoid 。这一改动让类型类层次更精确——很多类型只有结合律但没有单位元（如非空列表），它们现在是 Semigroup 但不是 Monoid。

---

## 🔍 逐节洞见

### 11.4.1 Lists are monoids —— 原型 Monoid

列表在 `++` 下构成 Monoid ：

```
instance Monoid [a] where
  mempty = []
  mappend = (++)
```

验证定律：

```
ghci> [1,2,3] ++ ([] ++ [4,5])
[1,2,3,4,5]
ghci> ([1,2,3] ++ []) ++ [4,5]
[1,2,3,4,5]
ghci> [] ++ [1,2,3]
[1,2,3]
```

`++` 满足结合律（`(xs ++ ys) ++ zs = xs ++ (ys ++ zs)`），`[]` 是单位元（`[] ++ xs = xs ++ [] = xs`） 。

> 📌 **洞见**：列表的 Monoid 实例是"原型"——其他所有 Monoid 实例都在模仿它。但**不要被 `mappend` 的名字误导**——它不一定要"追加"。`*` 不是追加，是乘法；`&&` 不是追加，是逻辑与。把 `mappend` 理解为"二元组合操作"才是正确的心智模型 。

**`foldl` 与 Monoid 的关系**：第5章学的 `foldl` 在 Monoid 视角下就是：

```
foldl mappend mempty [a, b, c, d] 
  = (((mempty `mappend` a) `mappend` b) `mappend` c) `mappend` d
  = a `mappend` b `mappend` c `mappend` d
```

**`foldl` 就是 Monoid 上的"左折叠"**。这一认识直接通向 11.4.6 节的 `foldMap`。

### 11.4.2 Product and Sum —— 一个数字，多种 Monoid

`Int` 在同一个类型下可以有**多个合法的 Monoid 实例**​ ：

- **加法 Monoid**：`mempty = 0`，`mappend = (+)`
    
- **乘法 Monoid**：`mempty = 1`，`mappend = (*)`
    

但 Haskell 不允许一个类型有多个 Monoid 实例（类型类实例必须唯一）。解决方案是 **`newtype` 包装**​ ：

```
newtype Sum a = Sum { getSum :: a }
newtype Product a = Product { getProduct :: a }

instance Num a => Monoid (Sum a) where
  mempty = Sum 0
  Sum x `mappend` Sum y = Sum (x + y)

instance Num a => Monoid (Product a) where
  mempty = Product 1
  Product x `mappend` Product y = Product (x * y)
```

**使用示例**：

```
ghci> getSum $ mconcat [Sum 1, Sum 2, Sum 3, Sum 4]
10
ghci> getProduct $ mconcat [Product 1, Product 2, Product 3, Product 4]
24
```

> 💡 **洞见**：`newtype` 在这里发挥了关键作用——**它让"同一个底层类型，多种抽象视角"成为可能**。这揭示了 Haskell 类型系统的一大哲学：**类型即语义**。`Int` 本身没有"Monoid 意义"——它只是内存中的整数；但 `Sum Int` 明确表示"我是加法 Monoid 里的 Int"，`Product Int` 明确表示"我是乘法 Monoid 里的 Int"。**类型的名字承载了数学语义**。

> ⚠️ **现代 Haskell 提示**：自 `base-4.8` 起，`Sum` 和 `Product` 在 `Data.Monoid` 中定义，使用 `DerivingVia` 或 `GeneralizedNewtypeDeriving` 可以更简洁地定义 。

### 11.4.3 Any and All —— 布尔值的两种 Monoid

布尔值在 `||` 和 `&&` 下分别构成 Monoid ：

```
newtype Any = Any { getAny :: Bool }
newtype All = All { getAll :: Bool }

instance Monoid Any where
  mempty = Any False
  Any x `mappend` Any y = Any (x || y)  -- "任意为真则为真"

instance Monoid All where
  mempty = All True
  All x `mappend` All y = All (x && y)  -- "全部为真才为真"
```

**使用场景**：

```
ghci> getAny $ mconcat [Any True, Any False, Any True]
True
ghci> getAll $ mconcat [All True, All False, All True]
False
```

> 📌 **洞见**：`Any` 和 `All` 是"存在量化"和"全称量化"的 Monoid 表达。它们与 `foldMap` 配合使用极其强大：

```
-- 检查列表中是否有偶数
hasEven :: [Int] -> Bool
hasEven xs = getAny $ foldMap (Any . even) xs

-- 检查列表中是否全是正数  
allPositive :: [Int] -> Bool
allPositive xs = getAll $ foldMap (All . (>0)) xs
```

这一手直接预示了 11.4.6 节的 `foldMap` 威力——**把"谓词判断"变成 Monoid 组合**。

### 11.4.4 The Ordering monoid —— 字典序的数学本质

`Ordering`（`LT`、`EQ`、`GT`）在"左偏选择"下构成 Monoid ：

```
instance Monoid Ordering where
  mempty = EQ
  LT `mappend` _ = LT
  EQ `mappend` y = y
  GT `mappend` _ = GT
```

**直觉理解**：`mappend` 选择"第一个不是 `EQ` 的结果"。这正是字典序/字符串比较的语义 ：

```
compareStrings :: String -> String -> Ordering
compareStrings [] [] = EQ
compareStrings [] _  = LT
compareStrings _  [] = GT
compareStrings (x:xs) (y:ys) = 
  case compare x y of
    EQ -> compareStrings xs ys  -- 当前字符相等，继续比较下一个
    ne -> ne                     -- 当前字符不等，立即返回
```

`compareStrings "apple" "apply"` 的比较过程，本质就是 `EQ`mappend`EQ`mappend`EQ`mappend`GT`mappend`... = GT`。

> 📌 **洞见**：`Ordering` 的 Monoid 实例揭示了**字典序的数学本质**——它就是 Monoid 组合。当你写 `compare (x:xs) (y:ys) = compare x y`mappend`compare xs ys`，你就在用数学的 Monoid 结构表达字典序。这是"类型即语义"的又一例证。

### 11.4.5 Maybe the monoid —— 两层 Monoid 实例

`Maybe a` 有两种合理的 Monoid 实例 ，取决于 `a` 是不是 Monoid：

**第一种**：`a` 本身是 Monoid 时，`Maybe a` 也是 Monoid

```
instance Monoid a => Monoid (Maybe a) where
  mempty = Nothing
  Nothing `mappend` m = m
  m `mappend` Nothing = m
  Just m1 `mappend` Just m2 = Just (m1 `mappend` m2)
```

语义：`Nothing` 是零（被吸收），`Just` 值内部用 `a` 的 Monoid 操作组合。

**第二种**：用 `First` 或 `Last` newtype 选择"第一个"或"最后一个" `Just`：

```
newtype First a = First { getFirst :: Maybe a }
newtype Last a = Last { getLast :: Maybe a }

instance Monoid (First a) where
  mempty = First Nothing
  First Nothing `mappend` r = r
  l `mappend` _ = l  -- 保留第一个 Just

instance Monoid (Last a) where
  mempty = Last Nothing
  l `mappend` Last Nothing = l
  _ `mappend` r = r  -- 保留最后一个 Just
```

**使用场景**：

```
ghci> getFirst $ mconcat [First Nothing, First (Just 1), First (Just 2)]
Just 1  -- 第一个 Just
ghci> getLast $ mconcat [Last (Just 1), Last (Just 2), Last Nothing]
Just 2  -- 最后一个 Just
```

> 💡 **洞见**：`Maybe` 的 Monoid 实例展示了"**效应组合**"的思想雏形。当你把 `Maybe` 看作"可能失败的计算"时，`mappend` 在说——"两个可能失败的计算组合在一起，只要有一个成功就算成功（对 `First`）"。这正是第13章 `Alternative` / `MonadPlus` 的思想源头 。

### 11.4.6 Using monoids to fold data structures —— 全章高潮

这是 Monoid 最有威力的应用——**`foldMap`**​ ：

```
foldMap :: (Monoid m, Foldable t) => (a -> m) -> t a -> m
foldMap f = foldr (mappend . f) mempty
```

`foldMap` 做三件事 ：

1. **映射**：把容器中的每个 `a` 通过 `f` 转换为 Monoid 值 `m`
    
2. **组合**：用 `mappend` 把所有 `m` 值组合
    
3. **默认**：容器为空时返回 `mempty`
    

**基础示例**：

```
ghci> import Data.Monoid
ghci> getSum $ foldMap Sum [1..10]
55
ghci> getProduct $ foldMap Product [1..10]
3628800
```

**更复杂的示例——提取树的所有元素**：

```
data Tree a = Empty | Node (Tree a) a (Tree a)

instance Foldable Tree where
  foldMap f Empty = mempty
  foldMap f (Node l x r) = foldMap f l `mappend` f x `mappend` foldMap f r

ghci> tree = Node (Node Empty 1 Empty) 5 (Node Empty 3 Empty)
ghci> foldMap Sum tree
Sum {getSum = 9}  -- 1 + 5 + 3
ghci> foldMap (Any . even) tree
Any {getAny = False}  -- 没有偶数
```

> 📌 **洞见**：`foldMap` 是 **Foldable 类型类的概念核心**​ 。任何数据结构，只要你能定义 `foldMap`（或等价的 `foldr`），你就自动获得了：
> 
> - `fold` —— Monoid 值的折叠
>     
> - `toList` —— 转换为列表
>     
> - `null`、`length`、`elem`、`maximum`、`minimum`
>     
> - `sum`、`product`
>     
> - `mapM_`、`traverse_`
>     

**这意味着什么？**​ 一旦你的数据类型是 `Foldable`（通过实现 `foldMap`），**所有列表上的操作都免费获得**​ 。这是 Haskell "抽象重用"的极致体现。

**`Endo`——函数的 Monoid**：

```
newtype Endo a = Endo { appEndo :: a -> a }

instance Monoid (Endo a) where
  mempty = Endo id
  Endo f `mappend` Endo g = Endo (f . g)  -- 函数组合
```

`a -> a` 类型在 `(.)` 和 `id` 下构成 Monoid 。这个实例揭示了**`foldr` 可以用 `foldMap` 表达**​ ：

```
foldr :: (a -> b -> b) -> b -> [a] -> b
foldr f z xs = appEndo (foldMap (Endo . f) xs) z
```

> 💡 **这是本章最震撼的洞见**：**`foldr` 不是列表的专属操作，它是 Monoid 的特例**。`Endo b` 把 `a -> b -> b` 类型的函数转换为 Monoid 值，然后 `foldMap` 做组合，最后 `appEndo` 抽取结果。`foldr` 的"折叠"本质就是"在 `Endo` Monoid 上的 `foldMap`"。

**更进一步**：`foldMap` 是 Foldable 的核心，而 `foldr` 可以用 `foldMap` 定义——这意味着 **Monoid 是 Foldable 的理论基础**​ 。任何可折叠的数据结构，本质上都是在某个 Monoid 上做 `foldMap`。

---

## 🧬 跨章节连接：Monoid 是怎么贯通全书的

```
第5章 高阶函数（foldl/foldr 是列表折叠）
  ↓
第7章 ADT + newtype（Sum/Product/Any/All 都是 newtype）
  ↓
第11章前文 Functor / Applicative（效应组合）
  ↓
第11章 11.4 Monoid（值组合的数学公理）
  │
  ├── 结合律 + 单位元 ──────► 并行计算的理论基础
  │                           MapReduce 的 reducer 就是 Monoid
  │
  ├── foldMap ─────────────► Foldable 类型类（第11章后续/现代 Haskell）
  │                           foldMap 是 Foldable 的核心方法
  │                           ↓
  │                           任何 Foldable 实例自动获得
  │                           sum/product/length/toList/...
  │
  ├── Endo Monoid ────────► foldr 的 Monoid 表达
  │                         (a -> b -> b) 在 Endo b 上是 Monoid
  │                         ↓
  │                         第5章的 foldr 是 Monoid 的特例
  │
  ├── Writer Monad ───────► 第13章 A Few Monads More
  │                         newtype Writer w a = Writer (a, w)
  │                         instance (Monoid w) => Monad (Writer w) where
  │                           return a = Writer (a, mempty)
  │                           (Writer (a, w)) >>= f = 
  │                             let (a', w') = runWriter (f a)
  │                             in Writer (a', w `mappend` w')
  │                         
  │                         日志 w 必须是 Monoid！
  │
  └── Alternative/MonadPlus ──► 第13章 MonadPlus
                              mzero = mempty
                              mplus = mappend
                              Monad 与 Monoid 的深刻联系
```

**一条主线串起来看**：

1. **第5章**：你学了 `foldl` 和 `foldr`——但它们只是针对列表的特定操作，缺乏统一的数学基础
    
2. **第7章**：你学了 `newtype`——它让"同一底层类型，多种抽象视角"成为可能，为 `Sum`/`Product`/`Any`/`All` 奠定基础
    
3. **第11章前文**：Functor 和 Applicative 解决了"效应组合"的问题
    
4. **第11章 11.4（本节）**：Monoid 解决了"值组合"的问题。**`mappend` 是值的组合，`<*>` 是效应的组合**——它们是同一抽象在不同维度的展开
    
5. **第13章**：当你学习 `Writer` Monad 时，会发现 `Writer w a` 的 `w` 必须是 Monoid——**Monoid 是"日志聚合"的数学基础**。当你学习 `MonadPlus` 时，会发现 `mzero` 就是 `mempty`，`mplus` 就是 `mappend`——**Monad 与 Monoid 有着深刻的血缘关系**
    

> 💡 **如果你只记住一件事**：**Monoid 是"组合"的数学本质**。`mappend` 的结合律让组合变得有序无关、可并行、可增量；`mempty` 的单位元让组合有安全的起点和默认值。这一数学结构不是 Haskell 的发明，而是宇宙间"合并多个事物为一个"这一行为的普遍规律。Haskell 只是用类型类把它形式化了。当你理解 Monoid 后，你会发现：
> 
> - **第5章的 `foldl`/`foldr` 是 Monoid 的特例**（`Endo` 实例）
>     
> - **`Foldable` 的理论基础是 Monoid**（`foldMap` 是核心方法）
>     
> - **第13章 `Writer` Monad 的日志要求是 Monoid**（`w`mappend`w'`）
>     
> - **并行计算、MapReduce、GPU reduce 都依赖 Monoid 的结合律**
>     

---

## 🛠 实际工程启示

**1. 用 Monoid 建模"可组合的聚合"**

任何需要"把多个值合并为一个"的场景，都应该考虑 Monoid：

```
-- 配置聚合
data Config = Config { timeout :: Int, retries :: Int, endpoints :: [String] }
  deriving (Semigroup, Monoid) via (Generic)

-- 默认配置 + 用户配置 + 环境变量配置
finalConfig = defConfig <> userConfig <> envConfig
```

**2. 用 `newtype` 为同一类型提供多种 Monoid 视角**

```
-- 数字既可以相加也可以相乘
newtype Sum a = Sum a
newtype Product a = Product a

-- 日志既可以追加也可以取最新
newtype AppendLog = AppendLog [String]
newtype LastLog = LastLog String

-- 验证既可以"全部通过"也可以"任一通过"
newtype AllValid = AllValid Bool
newtype AnyValid = AnyValid Bool
```

**3. 用 `foldMap` 实现遍历即聚合**

```
-- 统计树中偶数的个数
countEvens :: Tree Int -> Int
countEvens = getSum . foldMap (Sum . fromEnum . even)

-- 收集所有叶子节点
collectLeaves :: Tree a -> [a]
collectLeaves = foldMap (:[])

-- 检查是否存在满足条件的元素
exists :: (a -> Bool) -> [a] -> Bool
exists p = getAny . foldMap (Any . p)
```

**4. Monoid 是并行计算的天然抽象**

```
-- 并行 reduce：结合律保证正确性
parallelReduce :: Monoid a => [a] -> a
parallelReduce xs = 
  let halves = splitIntoChunks 1000 xs
      partials = parMap rpar (mconcat) halves  -- 并行计算各部分
  in mconcat partials  -- 合并部分结果
```

**5. `Writer` Monad 的日志必须是 Monoid**

```
-- 日志记录用 Monoid 实现自动聚合
type Logger a = Writer [String] a

logStep :: String -> Logger ()
logStep msg = tell [msg]

computation :: Logger Int
computation = do
  logStep "Starting"
  logStep "Processing"
  result <- pure 42
  logStep "Done"
  pure result
-- 运行后日志自动聚合为 ["Starting", "Processing", "Done"]
```

**6. 用 `Any`/`All` 做谓词组合**

```
-- 检查列表中是否有偶数
hasEven :: [Int] -> Bool
hasEven = getAny . foldMap (Any . even)

-- 检查列表中是否全是正数
allPositive :: [Int] -> Bool
allPositive = getAll . foldMap (All . (> 0))

-- 组合多个谓词
complexCheck :: [Int] -> Bool
complexCheck xs = 
  getAll $ foldMap (\x -> All (x > 0 && even x)) xs
```

**7. `Endo` 用于函数组合优化**

```
-- 多个转换函数的组合
transformations :: [Int -> Int]
transformations = [(+1), (*2), (subtract 3)]

-- 用 Endo 组合
combined :: Int -> Int
combined = appEndo $ foldMap Endo transformations
-- 等价于：(\x -> ((x + 1) * 2) - 3)
```

**8. 现代 Haskell 的 `DerivingVia` 简化 Monoid 实例**

```
{-# LANGUAGE DerivingVia #-}

data Score = Score { points :: Int, bonuses :: Int }
  deriving (Semigroup, Monoid) via (Sum Int)  -- 错误地：这只是演示语法

-- 正确的做法：为具体字段定义 Monoid
data GameState = GameState 
  { score :: Sum Int
  , health :: Product Double
  , achievements :: [String]
  } deriving (Semigroup, Monoid)
```

**9. 避免 Monoid 定律违规**

```
-- ❌ 错误的 Monoid 实例：不满足结合律
instance Monoid Int where
  mempty = 0
  mappend = (-)  -- 减法不满足结合律！

-- ✅ 正确的 Monoid 实例
instance Num a => Monoid (Sum a) where
  mempty = Sum 0
  mappend (Sum x) (Sum y) = Sum (x + y)  -- 加法满足结合律
```

**10. 性能考量：选择合适的 Monoid 容器**

```
-- ❌ 列表 Monoid 的 mappend 是 O(n)
-- 频繁追加大数据时性能差

-- ✅ 使用差异列表 (DList) 或其他高效结构
import qualified Data.DList as D

-- 或对于日志场景，使用 Builder 类型
import Data.ByteString.Builder
```

---

## 一句话收束

> Monoid 不是在教你 `mempty` 和 `mappend` 的语法，而是在揭示**"组合"这一宇宙行为的数学本质**：**结合律让组合变得有序无关、可并行、可增量；单位元让组合有安全的起点和默认值**。三条定律（左单位元、右单位元、结合律）看似平淡，却构成了所有"可折叠、可聚合、可并行"计算的理论基石。`Sum`/`Product`/`Any`/`All`/`Ordering`/`Maybe` 的 Monoid 实例展示了**同一类型可以有多种合法的"组合语义"**，而 `newtype` 让这种"一型多态"在 Haskell 的类型系统中被精确表达。`foldMap` 是本章的高潮——它用 Monoid 统一了所有"遍历并聚合"的操作，成为 `Foldable` 类型类的概念核心；而 `Endo` Monoid 揭示了**第5章的 `foldr` 不过是 `Endo` 上的 `foldMap`**——列表折叠的本质就是 Monoid 组合。更深一层，Monoid 与后续章节血脉相连：`Writer` Monad 的日志聚合要求 `w` 是 Monoid；`MonadPlus` 的 `mzero`/`mplus` 与 `mempty`/`mappend` 同构；并行计算、MapReduce、GPU reduce 都在利用 Monoid 的结合律。从这一章开始，你掌握的不仅是"如何合并值"，而是**"组合"这一行为的数学公理**——这是你理解 Haskell 效应系统（Functor/Applicative/Monad）和并行计算理论的最后一环。当所有抽象归位，你会发现 Haskell 的优雅不在于语法糖，而在于**它用类型系统精确地捕捉了数学结构的本质**——Monoid 就是这一哲学的最美见证。

