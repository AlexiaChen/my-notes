
# 第11章 Applicative 函子（Applicative Functors）——读书笔记

> 💡 前10章你构建了一个完整的能力栈：ADT 建模领域、纯函数表达逻辑、`foldl` 驱动状态机、`IO` 包裹效应边界。但有一个问题始终没被正面解决——**当函数有多个参数，且每个参数都"困在"某个效应容器里（Maybe、IO、[]）时，你怎么调用它？**​ `fmap` 只能把一个参数送进容器，第二个参数就无能为力了。第11章给出的答案 `Applicative`，本质是**"把函数应用本身，提升为在效应容器里进行"**。这一章是 Functor（第7章）与 Monad（第12章）之间的关键桥梁——它揭示了一个深刻真相：**不是所有"多步骤计算"都需要 Monad 那么强的"步骤间依赖"，很多场景下各步骤是独立的，Applicative 恰好捕捉了这种"独立效应的组合"**。

---

## 本章小节地图

11.1 函子再现（Functors Redux）—— 作为函子的 I/O、作为函子的函数

11.2 函子定律（Functor Laws）—— 为什么定律是契约而非装饰

11.3 使用 Applicative 函子（Applicative Functors）—— `pure` 与 `<*>`、Maybe 实例、Applicative 风格、列表实例、IO 实例、函数作为 Applicative、ZipList、Applicative 定律

11.4 Applicative 的实用函数（Useful Functions for Applicatives）—— `liftA2`、`sequenceA`、`*` 系列

> 📌 以上小节名依据《Haskell 趣学指南》第11章目录 。

---

## 🎯 全局脉络：为什么 Functor 不够，为什么 Monad 太强

要理解 Applicative 为什么存在，先看 Functor 的能力边界。

**Functor 的本质**：`fmap` 把一个普通函数"提升"为在容器中工作的函数 ：

```
fmap :: (a -> b) -> f a -> f b
```

`fmap` 的威力在于——**它让你在"不关心容器内部结构"的前提下，对容器内的值做变换**。`fmap (+1) (Just 3)` 得到 `Just 4`，`fmap (++"!") getLine` 得到一个"读取一行并加感叹号"的 I/O 动作。

但 `fmap` 有个**致命限制**：

```
-- 我想把 (+) 应用于两个 Maybe 值
-- (+)) :: Num a => a -> a -> a

fmap (+) (Just 3) :: Num a => Maybe (a -> a)  
-- 得到一个"装在 Maybe 里的函数"，但第二个参数还是 Maybe a
-- fmap (+) (Just 3) (Just 5)  -- ❌ 类型错误！fmap 只接受一个参数
```

`fmap` 只能"吃掉"第一个参数，剩下的函数还困在 `Maybe` 里，而第二个参数也在 `Maybe` 里——**两个都在容器里，但 `fmap` 无法让它们"相遇"**。

这就是 Applicative 要解决的问题。

**Applicative 的本质**：提供两个操作 ：

```
class Functor f => Applicative f where
  pure  :: a -> f a                    -- 把纯值装进"无效应"的默认容器
  (<*>) :: f (a -> b) -> f a -> f b    -- 在容器里应用函数
```

`<*>` 的类型是 `f (a -> b) -> f a -> f b`——它接受一个"装在容器里的函数"和一个"装在容器里的值"，返回"装在容器里的结果"。**这正是 `fmap` 做不到的事情**。

> 📌 **洞见**：`<*>` 与 `$` 的关系，正如 `fmap` 与函数应用的关系——`<*>` 就是"**在效应上下文里的函数应用**"。`f $ x` 把 `f` 应用于 `x`；`f <*> x` 把"装在效应里的 `f`"应用于"装在效应里的 `x`"。

**为什么需要 `pure`？**​ 因为 `<*>` 要求函数已经在容器里，但很多时候你有的只是普通函数。比如 `(+)` 不是 `Maybe (a -> a -> a)`，你需要 `pure (+)` 把它装进去：

```
pure (+) :: Num a => Maybe (a -> a -> a)
pure (+) <*> Just 3 <*> Just 5 :: Num a => Maybe a
```

**Functor 与 Applicative 的包含关系**：

```
Functor  ⊂  Applicative  ⊂  Monad
```

- 每个 Applicative 都是 Functor（`pure f <*> x = fmap f x` 是 Applicative 定律之一）
    
- 每个 Monad 都是 Applicative（`pure = return`，`mf <*> mx = mf >>= \f -> mx >>= \x -> return (f x)`）
    
- **但 Applicative 比 Monad 弱**——它无法表达"后一步计算依赖前一步结果"
    

> 💡 **为什么"弱"反而是优势？**​ 因为**更弱的抽象 = 更强的可组合性、更多的优化空间、更清晰的语义**。Applicative 的计算结构是**静态可知的**——你可以预先知道"要组合哪些效应"，而 Monad 的 `>>=` 允许"根据上一步的结果动态决定下一步"，这导致 Monad 计算结构是动态的、不可静态分析的。因此：
> 
> - **当各步骤效应独立时**——用 Applicative（解析器组合、表单验证、并发请求）
>     
> - **当后一步依赖前一步时**——才用 Monad（状态传递、I/O 依赖、错误处理链）
>     

这是本章最深刻的洞见：**Applicative 不是"弱化版的 Monad"，而是"独立效应组合"的标准抽象**。很多真实场景的效应是独立的，用 Applicative 恰好捕捉了这种独立性，而 Monad 反而会"过度表达"。

---

## 🔍 逐节洞见

### 11.1 函子再现 —— I/O 和函数也是 Functor

**I/O 是 Functor**：

```
fmap (++"!") getLine :: IO String
```

`getLine :: IO String`，`fmap (++"!")` 把它变成 `IO (String -> String)`——等等，不对，是 `IO String` 里的内容被 `(++"!")` 变换。实质上：**I/O 动作"读取一行"被提升为"读取一行并加感叹号"**。这是第8章 I/O 思想的延续——`IO` 动作是值，可以被 `fmap` 变换。

**函数是 Functor**（最反直觉的一节）：

```
instance Functor ((->) r) where
  fmap = (.)
```

`(->) r` 是一个 Functor——意思是"接受 `r` 作为参数的函数"可以被 `fmap`。`fmap f g = f . g`——**函数组合就是函数在 Functor 意义上的 `fmap`**。

> 📌 **洞见**：当 `f` 是 `(->) r` 时，`f a` 就是 `r -> a`。所以 `fmap :: (a -> b) -> (r -> a) -> (r -> b)`——这正是函数组合的类型！这意味着**"函子"这个概念远比"容器"更广**：它不仅涵盖 `Maybe`、`[]`、`IO` 这些"装了值的容器"，也涵盖"(->) r 这种"等待一个输入的函数"。

> 💡 **更深一层**：把函数看作 Functor，是"读者语境"（Reader Context）思想的萌芽。一个函数 `r -> a` 可以理解为"在环境 `r` 下产生 `a`"。`fmap` 在这个语境下就是"在相同的环境中，对产出值做变换"——这正是第13章 `Reader` Monad 的核心思想。函数作为 Functor 的这一节，看似抽象游戏，实则为 `Reader` Monad 埋下了伏笔。

### 11.2 函子定律 —— 为什么定律是契约而非装饰

Functor 必须满足两条定律 ：

```
fmap id = id                           -- 定律1：映射 id 不改变函子
fmap (f . g) = fmap f . fmap g        -- 定律2：映射函数组合 = 组合映射
```

**为什么这两条定律至关重要？**

**定律1（Identity）**：`fmap id fa` 必须返回与 `fa` 完全相同的函子。这意味着 `fmap` 不能偷偷"做点额外的事"——它只能"在容器内做变换"，不能改变容器的"效应结构"。对于 `Maybe`，这意味着 `fmap id (Just x) = Just x`、`fmap id Nothing = Nothing`——不能把 `Nothing` 变成 `Just undefined`。

**定律2（Composition）**：`fmap (f . g)` 必须等于 `fmap f . fmap g`。这意味着**两次 `fmap` 可以合并为一次 `fmap`**，且结果相同。这条定律让编译器可以做"融合优化"（fusion optimization）——`fmap f . fmap g` 可以被优化为 `fmap (f . g)`，避免中间数据结构的分配。GHC 的 `map` fusion 优化就是基于这条定律。

> ⚠️ **违反定律的后果**：如果你定义了一个 Functor 实例但违反了定律，依赖 `fmap` 的抽象（如 `traverse`、`foldMap`）会产生诡异行为，编译器也无法做融合优化。**定律是类型类契约的一部分**——编译器不检查，但整个生态系统的代码都依赖于此。

**洞见**：Haskell 的类型类之所以强大，不是因为编译器强制了定律，而是因为**整个社区都自觉遵守定律**。这是一种"社会契约"——你写一个 `Functor` 实例，就等于承诺遵守这两条定律。这种"基于约定的正确性"是 Haskell 工程文化的核心。

### 11.3 使用 Applicative 函子 —— 全章核心

**`Maybe` 的 Applicative 实例**：

```
instance Applicative Maybe where
  pure = Just
  (Just f) <*> (Just x) = Just (f x)
  _ <*> _ = Nothing
```

`pure = Just`——把纯值装入 `Maybe` 的"默认无失败"形态。

`<*>`——如果两个都是 `Just`，就应用函数；任何一个为 `Nothing`，结果就是 `Nothing`。

**Applicative 风格的威力**：

```
-- 不用 Applicative：嵌套模式匹配
addMaybes :: Maybe Int -> Maybe Int -> Maybe Int
addMaybes (Just x) (Just y) = Just (x + y)
addMaybes _ _ = Nothing

-- 用 Applicative：一行搞定
addMaybes :: Maybe Int -> Maybe Int -> Maybe Int
addMaybes mx my = (+) <$> mx <*> my
```

`(<$>)` 是 `fmap` 的中缀形式，`f <$> x <*> y <*> z` 就是 Applicative 风格——**它看起来和普通函数应用 `f x y z` 几乎一样，只是 sprinkled with `<$>` 和 `<*>`**。

> 📌 **洞见**：Applicative 风格的美在于——**它让"在效应容器里调用多参数函数"变得和"普通函数调用"一样自然**。你不需要写嵌套的 `case`、`do` 块或 `>>=`，只需用 `<$>` 和 `<*>` 把普通函数"提升"进容器。这是"**让非常规计算看起来像常规计算**"的极致——Haskell 的哲学"**让抽象透明化**"在此达到高峰。

**列表的 Applicative 实例**：

```
instance Applicative [] where
  pure x = [x]
  fs <*> xs = [f x | f <- fs, x <- xs]
```

`pure x = [x]`——单元素列表是"无效应"的默认形态。

`<*>`——**笛卡尔积式的应用**：左边的每个函数应用于右边的每个值。

```
ghci> [(+3), (*2)] <*> [1, 2]
[4, 5, 2, 4]  -- (+3) 1, (+3) 2, (*2) 1, (*2) 2
```

> 💡 **列表作为 Applicative 的语义是"非确定性计算"**：`[1,2,3]` 可以被理解为"一个计算，其结果可能是 1、2 或 3"。`fs <*> xs` 就是"把第一个非确定性计算（产生一组函数）应用于第二个非确定性计算（产生一组值），得到所有可能的组合"。

这种"全组合"语义在处理排列组合时极为强大：

```
ghci> (,) <$> [1,2] <*> ['a','b']
[(1,'a'),(1,'b'),(2,'a'),(2,'b')]
```

一行代码生成所有可能的配对——这是列表推导式 `[ (x,y) | x <- [1,2], y <- ['a','b'] ]` 的 Applicative 等价物。

**`ZipList`：另一种列表 Applicative**：

列表不能有两个 Applicative 实例（类型类实例必须唯一），Haskell 用 `newtype` 包装器解决：

```
newtype ZipList a = ZipList { getZipList :: [a] }

instance Applicative ZipList where
  pure x = ZipList (repeat x)
  ZipList fs <*> ZipList xs = ZipList (zipWith ($) fs xs)
```

`ZipList` 的 `<*>` 是**按位置配对**而非全组合：

```
ghci> getZipList $ (,) <$> ZipList [1,2,3] <*> ZipList ['a','b','c']
[(1,'a'),(2,'b'),(3,'c')]
```

> 📌 **洞见**：`ZipList` 展示了 Applicative 的"效应"本质——同一种数据结构（`[]`），可以有两种完全不同的"效应语义"：
> 
> - 作为非确定性计算（笛卡尔积）
>     
> - 作为"同步流"（按位置配对）
>     
> 
> **不同的 `pure` 和 `<*>` 实现，赋予同一种数据结构不同的"效应语义"**。这是 Applicative 抽象威力的极致——它把"效应"本身参数化了。这一思想直接通往第12章 Monad：不同的 Monad 实例（`Maybe`、`[]`、`IO`、`State`、`Reader`）就是不同的"效应语义"。

**I/O 的 Applicative 实例**：

```
instance Applicative IO where
  pure = return
  mf <*> mx = do
    f <- mf
    x <- mx
    return (f x)
```

`pure = return`——把纯值包装为"不做任何 I/O 的动作"。

`<*>`——**顺序执行两个 I/O 动作，把第一个的结果（函数）应用于第二个的结果（值）**。

```
-- 读取两行并拼接
readTwoLines :: IO String
readTwoLines = (++) <$> getLine <*> getLine

-- 读取 N 行并拼接
getLines :: Int -> IO String
getLines 0 = pure []
getLines n = (++) <$> getLine <*> getLines (n - 1)
```

> 💡 **洞见**：`(++) <$> getLine <*> getLine` 是 `do` 表示法的 Applicative 等价物：

```
readTwoLines = do
  a <- getLine
  b <- getLine
  return (a ++ b)
```

但请注意——**这里的 `a` 和 `b` 是独立的**（`getLine` 不依赖前一个 `getLine` 的结果）。这正是 Applicative 的甜区：**各步骤效应独立时，Applicative 风格比 `do` 块更简洁、更清晰地表达了"独立性"**。

**函数作为 Applicative**：

```
instance Applicative ((->) r) where
  pure x = \_ -> x
  f <*> g = \x -> f x (g x)
```

`pure x = \_ -> x`——常函数，忽略环境，永远返回 `x`。

`<*>`——`f <*> g` 在环境 `x` 下，先计算 `f x`（得到函数），再计算 `g x`（得到参数），应用之。

这意味着 `(+) <$> (+10) <*> (*5)` 是一个函数，当它被应用于 `n` 时，相当于 `(n+10) + (n*5)`。**这是"Reader Monad"的 Applicative 版本**——把"环境"当作共享的只读状态，多个依赖于同一环境的函数可以组合。

> 📌 **洞见**：函数作为 Applicative 揭示了"**Applicative 组合就是依赖注入**"。`f <$> g1 <*> g2 <*> g3` 可以理解为"把 `g1`、`g2`、`g3` 这三个从环境取值的函数，组合成一个更大的从环境取值的函数"。这是第13章 `Reader` Monad 和现代 Haskell 依赖注入模式的基础。

**Applicative 定律**：

Applicative 必须满足四条定律 ：

```
pure id <*> v = v                          -- 恒等定律
pure f <*> pure x = pure (f x)             -- 同态定律
u <*> pure y = pure ($ y) <*> u            -- 交换定律
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)  -- 组合定律
```

**这四条定律的直觉意义**：

- **恒等定律**：`pure id` 在 `<*>` 下是单位元——正如 `id` 在 `$` 下是单位元
    
- **同态定律**：`pure` 保持函数应用——在效应里应用纯函数，等于先在纯世界里应用再 `pure` 结果
    
- **交换定律**：效应函数应用于纯值，与纯值先被 `($ y)` 包装再应用，结果相同——**求值顺序不重要**
    
- **组合定律**：`<*>` 是结合的——`pure (.)` 让函数组合在效应里依然成立
    

> ⚠️ **定律的重要性**：这四条定律保证了 Applicative 风格的代码可以**自由重组而不改变语义**。你可以把 `f <$> a <*> b <*> c` 重排为 `f <$> c <*> a <*> b`（如果 `a`、`b`、`c` 效应独立），编译器也可以做各种优化。**定律是"效应组合"代数的公理**——正如群论的公理保证了群运算的性质。

### 11.4 Applicative 的实用函数 —— 抽象的第 N 层

**`liftA2`**：

```
liftA2 :: Applicative f => (a -> b -> c) -> f a -> f b -> f c
liftA2 f a b = f <$> a <*> b
```

`liftA2` 把一个二进制函数"提升"为在两个 Applicative 上工作的函数。它清晰地展示了 Applicative 相对于 Functor 的强大：**Functor 只能提升一元函数，Applicative 能提升任意元函数**。

**`sequenceA`**：

```
sequenceA :: Applicative f => [f a] -> f [a]
sequenceA [] = pure []
sequenceA (x:xs) = (:) <$> x <*> sequenceA xs
```

`sequenceA` 把"一列效应"转换为"效应里的一列"——**把效应"外翻"**。对于 `Maybe`：`sequenceA [Just 1, Just 2, Just 3] = Just [1,2,3]`；如果任何一个是 `Nothing`，结果是 `Nothing`。对于 `IO`：`sequenceA [getLine, getLine]` 是"读取两行并返回字符串列表"的 I/O 动作。

> 📌 **洞见**：`sequenceA` 是"效应翻转"的标准操作——`[f a]` 变成 `f [a]`。这个操作在 Traversable 类型类（第11章后续/现代 Haskell）中是核心。`traverse` 就是 `sequenceA . fmap`——**"映射一个产生效应的函数，然后翻转效应"**。这是 Haskell 处理"效应性映射"的标准手法。

**`*` 系列运算符**：

```
(*>) :: Applicative f => f a -> f b -> f b
(<*) :: Applicative f => f a -> f b -> f a
```

`(*>)` 执行两个动作，丢弃第一个的结果；`(<*)` 执行两个动作，丢弃第二个的结果。在解析器和 I/O 中极为常用：

```
-- 读取一行，丢弃，再读取一行，返回
getSecondLine :: IO String
getSecondLine = getLine *> getLine

-- 解析器：匹配 "let"，然后解析表达式，返回表达式
parseLet :: Parser Expr
parseLet = string "let" *> parseExpr
```

---

## 🧬 跨章节连接：Applicative 是怎么"长"在 Functor 与 Monad 之间的

```
第7章 Functor
  │  fmap :: (a -> b) -> f a -> f b
  │  只能提升一元函数
  ↓
第11章（本节） Applicative
  │  pure :: a -> f a
  │  (<*>) :: f (a -> b) -> f a -> f b
  │  能提升任意元函数，组合独立效应
  │
  ├── 函子再现 ─────────► 第8章 IO 是 Functor
  │                     I/O 动作可以被 fmap 变换
  │
  ├── Maybe 实例 ──────► 第10章 RPN 计算器
  │                     用 pure/Just 和 <*> 组合多参数函数
  │                     替代嵌套 case 匹配
  │
  ├── 列表实例 ────────► 第1章 列表推导式
  │                     [f x y | x <- xs, y <- ys] 
  │                     = f <$> xs <*> ys
  │                     笛卡尔积语义
  │
  ├── ZipList 实例 ────► 矢量/数组编程
  │                     按位置配对（类似 NumPy 的广播）
  │
  ├── 函数作为 Applicative ──► 第13章 Reader Monad
  │                           (->) r 的 Applicative 是 Reader 的基础
  │
  └── Applicative 定律 ──► 第12章 Monad
                        Monad 的 return = pure
                        Monad 的 ap = <*>
                        定律保证 Monad ⊃ Applicative
```

**一条主线串起来看**：

1. **第7章**：你学了 `Functor`——`fmap` 提升一元函数。但这不够：多参数函数怎么办？
    
2. **第11章（本节）**：Applicative 用 `pure` 和 `<*>` 填补了这个空白。**`pure f <*> x <*> y <*> z` 让"在效应里调用多参数函数"变得和自然函数调用一样**。`<$>` 是 `fmap` 的中缀别名，`<*>` 是"在容器里应用函数"——两者组合，Applicative 风格诞生。
    
3. **第12章**：Monad 进一步泛化——`>>=` 允许"后一步计算依赖前一步结果"。但 Monad 比 Applicative 强，**只在真正需要依赖时才用 Monad**。
    
4. **第13章**：具体 Monad 实例——`Maybe`、`[]`、`IO`、`State`、`Reader`、`Writer`。其中 `Reader` 的 Applicative 实例就是第11章学的"函数作为 Applicative"；`ZipList` 思想是 `Applicative` 在矢量计算中的应用。
    

> 💡 **如果你只记住一件事**：**Applicative 捕捉的是"独立效应的组合"**。`f <$> a <*> b <*> c` 表达的是——"用效应 `a`、`b`、`c` 分别产生值，然后用普通函数 `f` 把它们组合起来"。这里 `a`、`b`、`c` 的效应是**独立的**——`b` 不依赖 `a` 的结果，`c` 不依赖 `a` 或 `b` 的结果。这种"独立性"是 Applicative 的本质特征，也是它比 Monad 更适合某些场景的原因：解析器组合、表单验证、并发请求——所有这些场景中，各子计算互不依赖，Applicative 是完美的抽象。

---

## 🛠 实际工程启示

**1. 优先用 Applicative 风格，而非手动嵌套 case/do**

```
-- ❌ 反模式：嵌套模式匹配
combine :: Maybe Int -> Maybe Int -> Maybe Int
combine (Just x) (Just y) = Just (x + y)
combine _ _ = Nothing

-- ✅ Applicative 风格
combine :: Maybe Int -> Maybe Int -> Maybe Int
combine x y = (+) <$> x <*> y

-- 三元函数也一样优雅
combine3 :: Maybe Int -> Maybe Int -> Maybe Int -> Maybe Int
combine3 x y z = (+) <$> x <*> y <*> z
```

**2. 用 `pure` 注入纯值**

```
-- 当函数的一个参数是纯值时
addThree :: Maybe Int -> Int -> Maybe Int
addThree mx n = (+n) <$> mx

-- 等价于
addThree mx n = pure (+) <*> mx <*> pure n
```

**3. 列表的 Applicative 用于组合生成**

```
-- 生成所有坐标对
coordinates :: [(Int, Int)]
coordinates = (,) <$> [1..3] <*> ['a'..'c']
-- [(1,'a'),(1,'b'),(1,'c'),(2,'a'),...]

-- 等价于列表推导式，但更函数式
coordinates = [ (x,y) | x <- [1..3], y <- ['a'..'c'] ]
```

**4. 用 `ZipList` 做矢量计算**

```
import Control.Applicative (ZipList(..))

-- 矢量加法
vectorAdd :: [Int] -> [Int] -> [Int]
vectorAdd xs ys = getZipList $ (+) <$> ZipList xs <*> ZipList ys

-- 等价于 zipWith (+)，但 Applicative 风格更统一
```

**5. I/O 中的 Applicative 风格**

```
-- 读取多个输入并组合
getUserInput :: IO (String, Int)
getUserInput = (,) <$> getLine <*> (read <$> getLine)

-- 注意：第二个 getLine 不依赖第一个的结果
-- 如果是依赖的，才需要用 Monad 的 >>=
```

**6. `sequenceA` 翻转效应**

```
-- 验证多个条件，全部通过才返回结果
validateForm :: Form -> Maybe ValidatedForm
validateForm form = 
  ValidatedForm <$> checkName (name form) 
                <*> checkAge (age form) 
                <*> checkEmail (email form)

-- 如果任何 checkX 返回 Nothing，整体就是 Nothing
-- 这就是 "fail-fast" 验证模式
```

**7. 解析器组合子（工业级应用）**

Parsec 等解析库的核心是 Applicative：

```
-- 解析 "(123, hello)"
parsePair :: Parser (Int, String)
parsePair = char '(' *> ((,) <$> number <*> (char ',' *> word)) <* char ')'

-- *> 和 <* 用于"匹配但丢弃"
-- <$> 和 <*> 用于"组合结果"
```

**8. Applicative 与 Monad 的抉择**

```
-- 用 Applicative：各步骤效应独立
combineIndependent :: Maybe Int -> Maybe Int -> Maybe Int
combineIndependent x y = (+) <$> x <*> y

-- 用 Monad：后一步依赖前一步
dependent :: Maybe Int -> Maybe Int
dependent mx = mx >>= \x -> 
  if x > 0 then Just (x * 2) else Nothing
```

**经验法则**：**能用 Applicative 就不要用 Monad**。Applicative 的"静态结构可知"特性让编译器能做更多优化，也让代码意图更清晰（"这些效应是独立的"）。

**9. 定律是契约**

写 Applicative 实例时，心里要有四条定律。违反定律的实例会导致：

- `pure f <*> pure x` 不等于 `pure (f x)` —— `sequenceA` 行为诡异
    
- 组合定律不成立 —— 无法自由重组计算
    
- 恒等定律不成立 —— `f <$> x` 不等于 `pure f <*> x`
    

**10. `liftA2` 的威力**

```
-- 不需要为二元函数专门写 Applicative 风格
liftA2 (+) (Just 3) (Just 5)  -- Just 8

-- 等价于 (+) <$> Just 3 <*> Just 5
-- 但 liftA2 更明确：把二元函数提升为 Applicative 组合
```

---

## 一句话收束

> 第11章不是在教你 `pure` 和 `<*>` 的语法，而是在揭示 Haskell 抽象体系的关键一层：**Applicative 捕捉的是"独立效应的组合"**。`fmap` 只能提升一元函数，无法处理"多参数函数 + 多效应参数"的场景；Applicative 用 `pure`（注入纯值）和 `<*>`（在效应里应用函数）填补了这个空白，使得 `f <$> a <*> b <*> c` 这种 Applicative 风格，**让"在效应容器里调用多参数函数"变得和自然函数调用一样自然**。`Maybe` 用它处理"可能失败"的组合，`[]` 用它表达非确定性计算，`ZipList` 用它做矢量配对，`IO` 用它组合独立 I/O 动作，`(->) r` 用它实现依赖注入——**同一种抽象，统一了所有"独立效应组合"的场景**。而四条 Applicative 定律（恒等、同态、交换、组合）保证了这种组合的代数性质，使得代码可以**自由重组而不改变语义**。Applicative 位于 Functor 与 Monad 之间：`pure = return`、`<*> = ap`——每个 Monad 都是 Applicative，但 Applicative 比 Monad 弱，恰好捕捉了"效应独立"这一常见模式。当你在第12章学习 Monad 时，请记得——**Monad 的 `>>=` 允许"后一步依赖前一步"，但很多真实场景并不需要这种依赖**（解析、验证、并发请求），这时 Applicative 才是恰到好处的抽象。从这一章开始，你掌握了 Haskell 效应系统的"中间层"——它让你能精确表达"哪些效应是独立的"，这是写出清晰、可优化、可组合代码的关键。

