
# 第十九章 State 单子 —— 读书笔记

> 💡 如果说第十八章的 Reader 单子解决了"只读环境的隐式传递"，那么第十九章就是把 Monad 用在了一个更普遍的场景——**"可读写状态的隐式传递"**。表面看是 `State s a` 类型和 `get`/`put`/`modify` 操作，实则在揭示一个深层真相：**任何需要"隐式状态传递"的计算——随机数生成、计数器、解释器、计算器、游戏状态——都可以被建模为 `s -> (a, s)` 这个函数，而 State 单子让我们可以用 `do` 语法糖优雅地组合这类计算**。这一章让你亲眼看见：Monad 不只是 IO 和列表和 Reader——它是"计算上下文"的统一抽象，而"可读写状态"只是其中最常见的一种。

韩冬在这一章安排了三个小节：**19.1 什么是 State 单子**、**19.2 随机数**、**19.3 简易计算器**。表面是从"State 单子的定义"到"随机数生成的应用"再到"简易计算器的应用"，实则在铺设一条主线：**State 单子 = `(->) s` 的升级版——它不仅"读取"状态，还"写入"状态；`>>=` 自动把"旧状态"传给第一步计算，拿到"新状态"再传给下一步计算，状态就这样从左到右流过整个 `do` 块**。

---

## 🎯 全局脉络：为什么 State 单子是 Monad 家族的关键成员？

```
第十六章: Monad（>>= 让后续依赖前文）
   ↓
第十七章: 列表单子（非确定性）  ← Monad 实例 #1
第十八章: Reader 单子（只读环境）  ← Monad 实例 #2
   ↓
第十九章: State 单子（可读写状态）  ← 你在这里  ← Monad 实例 #3
   ↓
第二十章: IO 单子的工程实践
第二十四章: Monad Transformers（StateT 组合）
```

**这一章是 Monad 抽象"第三次震撼亮相"**。前几章你看到：

- 列表 `[]` 是 Monad——建模"非确定性"
    
- `(->) r` 是 Monad——建模"只读环境"
    

本章要揭示的是：**`s -> (a, s)` 也是 Monad——建模"可读写状态"**。

Haskell 官方文档给出了 State 单子的核心定义 ：

```
newtype State s a = State { runState :: s -> (a, s) }
```

**🔑 洞见一：State s a 是一个"状态转换函数"**

`State s a` 可以理解为一个函数：

- **输入**：当前状态 `s`
    
- **输出**：`(a, s)`——计算结果 `a` 和**更新后的状态**​ `s`
    

与 Reader 的对比：

|维度|Reader r a|State s a|
|---|---|---|
|底层类型|`r -> a`|`s -> (a, s)`|
|读取状态|✅ 能|✅ 能（`get`）|
|写入状态|❌ 不能|✅ 能（`put`）|
|典型用途|配置、环境|计数器、随机数、解释器|

Monday Morning Haskell 给出了精辟总结 ：

> _"Just like the Reader has a single type we read from, the State has a single type we can both read from and write to."_

---

## 19.1 什么是 State 单子

### 类型定义与运行函数

Haskell 官方文档 ：

```
newtype State s a = State { runState :: s -> (a, s) }

-- 运行 State 计算，返回 (计算结果, 最终状态)
runState :: State s a -> s -> (a, s)

-- 只取计算结果，丢弃最终状态
evalState :: State s a -> s -> a
evalState act = fst . runState act

-- 只取最终状态，丢弃计算结果
execState :: State s a -> s -> s
execState act = snd . runState act
```

### 🔑 洞见二：State 单子的三种"视角"

`State s a` 可以从三个角度理解：

1. **函数视角**：`s -> (a, s)`——一个状态转换函数
    
2. **计算视角**：一个"在状态 `s` 上下文中产出 `a`"的计算
    
3. **Monad 视角**：`State s` 是一个 Monad 实例，可以用 `do` 语法糖组合
    

### 核心操作：get、put、modify

Haskell 官方文档 ：

```
-- 获取当前状态
get :: MonadState s m => m s
get = State $ \s -> (s, s)  -- 返回状态，状态不变

-- 设置新状态（替换旧状态）
put :: MonadState s m => s -> m ()
put s = State $ \_ -> ((), s)  -- 返回 ()，状态变为 s

-- 用函数修改状态
modify :: MonadState s m => (s -> s) -> m ()
modify f = State $ \s -> ((), f s)  -- 返回 ()，状态变为 f s

-- 获取状态的投影
gets :: MonadState s m => (s -> a) -> m a
gets f = State $ \s -> (f s, s)  -- 返回 f s，状态不变
```

### 🔑 洞见三：get 和 put 的对称性

芝加哥大学讲义给出了精炼的定义 ：

```
get :: State s s
get = State $ \s -> (s, s)      -- 状态作为结果返回，状态本身不变

put :: s -> State s ()
put t = State $ \s -> ((), t)   -- 结果抛弃，状态被替换为 t

modify :: (s -> s) -> State s ()
modify f = State $ \s -> ((), f s)  -- 结果抛弃，状态变为 f s
```

**直观理解**：

- `get`：看看当前状态是什么（状态不变）
    
- `put`：把状态改成某个值（旧状态被丢弃）
    
- `modify`：基于旧状态计算新状态（如 `modify (+1)` 让状态加 1）
    

Haskell wiki 给出了经典示例 ：

```
runState (return 'X') 1            -- ('X', 1)   return 不改变状态
runState get 1                     -- (1, 1)     get 返回状态，状态不变
runState (put 5) 1                 -- ((), 5)    put 设置状态为 5
runState (do { put 5; return 'X' }) 1  -- ('X', 5)  先 put 再 return
```

### State 单子的 Monad 实例

芝加哥大学讲义揭示了 `>>=` 的本质 ：

```
instance Monad (State s) where
  m >>= f = State $ \s ->
    let (a, t) = runState m s      -- 第一步：在状态 s 下运行 m，得到 (a, t)
        (b, u) = runState (f a) t  -- 第二步：在状态 t 下运行 f a，得到 (b, u)
    in (b, u)                      -- 最终结果是 (b, u)
```

**🔑 洞见四：`>>=` 让状态"从左到右"流过计算**

这是 State 单子最关键的机制：

1. **第一步计算**​ `m` 在初始状态 `s` 下运行，产生结果 `a` 和新状态 `t`
    
2. **第二步计算**​ `f a` 在新状态 `t` 下运行（**不是**​ `s`！），产生结果 `b` 和最终状态 `u`
    
3. **状态 `t` 自动传递给第二步**——你不需要手动传递
    

芝加哥大学讲义明确指出 ：

> _"As with (), the state will flow through the expression in left-to-right order."_

**对比 Reader 单子**：

```
-- Reader 的 >>=：环境 r 被共享
f >>= k = \r -> k (f r) r

-- State 的 >>=：状态 s 被传递和更新
m >>= f = \s -> let (a, t) = m s in f a t
```

Reader 中环境 `r` 是不变的（只读）；State 中状态 `s` 每一步都可能变化（可读写）。

### 🔑 洞见五：`state` 函数——从状态转换函数构造 State

Haskell 官方文档和随机数教程中频繁使用 `state` 函数 ：

```
state :: MonadState s m => (s -> (a, s)) -> m a
state f = State f
```

**它的作用**：把一个"状态转换函数" `s -> (a, s)` 包装成 `State s a`。

这在随机数生成中极有用（见 19.2 节）：

```
-- randomR :: (a, a) -> g -> (a, g)  -- 纯函数式随机数生成
-- 用 state 把它提升为 State 单子
rollDie :: State StdGen Int
rollDie = state $ randomR (1, 6)
```

### 实际应用

```
import Control.Monad.State

-- 一个计数器：每次调用加 1，返回旧值
tick :: State Int Int
tick = do
  n <- get        -- 读取当前状态
  put (n + 1)     -- 状态加 1
  return n        -- 返回旧值

-- 运行
runState tick 0   -- (0, 1)  -- 返回 0，状态变为 1
runState (tick >> tick) 0   -- (1, 2)  -- 返回 1，状态变为 2
evalState (replicateM 5 tick) 0  -- [0,1,2,3,4]  -- 5 次 tick 的结果
execState (replicateM 5 tick) 0  -- 5  -- 最终状态

-- 用 modify 简化
increment :: State Int ()
increment = modify (+1)

-- 用 gets 提取状态的投影
isEven :: State Int Bool
isEven = gets even
```

---

## 19.2 随机数：State 单子隐藏生成器状态

### 问题背景

图灵社区文章指出 ：

> _"按照一定的算法对这个种子计算，得出一个随机数和一个新的种子，下次取的时候会基于上次得出的种子，重复此过程"_

**纯函数式随机数的本质**：

```
getRandom :: seed -> (a, seed)
```

- 输入一个种子，输出一个随机数和一个**新种子**
    
- 下次调用必须用新种子——否则会得到相同的"随机"数
    
- 这是伪随机数生成器（PRNG）的标准模式
    

**痛点**：在多次随机数调用中，你必须手动传递生成器状态 ：

```
-- 手动传递生成器（繁琐且易错）
rollTwo :: StdGen -> ((Int, Int), StdGen)
rollTwo g0 = 
  let (d1, g1) = randomR (1, 6) g0
      (d2, g2) = randomR (1, 6) g1
  in ((d1, d2), g2)
```

### 🔑 洞见六：State 单子自动隐藏生成器传递

Haskell 随机数教程给出了优雅的解决方案 ：

```
import Control.Monad.State
import System.Random

type RandState = State StdGen

-- 用 state 函数把 randomR 提升为 State 单子
roll :: Int -> RandState Int
roll n = state $ uniformR (1, n)

-- 掷骰子 1 到 6
rollDie :: RandState Int
rollDie = roll 6

-- 掷两个骰子
rollDice :: RandState (Int, Int)
rollDice = do
  d1 <- rollDie
  d2 <- rollDie
  return (d1, d2)

-- 掷 k 面骰 n 次
kSidedRolls :: Int -> Int -> StdGen -> ([Int], StdGen)
kSidedRolls k n g = 
  let kSided = state $ uniformR (1, k)
  in runState (replicateM n kSided) g
```

**🔑 洞见七：do 块中的生成器自动传递**

当你写：

```
rollDice = do
  d1 <- rollDie    -- 这里用到了生成器 g0，产生 d1 和 g1
  d2 <- rollDie    -- 这里自动用 g1，产生 d2 和 g2
  return (d1, d2)
```

**`>>=` 的魔法**：`d1 <- rollDie` 脱糖为 `rollDie >>= \d1 -> ...`。`>>=` 在状态 `g0` 下运行 `rollDie`，得到 `(d1, g1)`；然后把 `g1` 传给后续计算 `\d1 -> ...`，其中再运行 `rollDie`，得到 `(d2, g2)`。

**你完全不需要手动写 `g1`、`g2`——State 单子的 `>>=` 自动完成了生成器状态的传递**。

### 🔑 洞见八：随机数是"确定性"的

教程明确指出 ：

> _"Unless the value of the seed to mkStdGen is changed, we will get the same result every time... This is to be expected since rollFourAndSix is a pure function"_

```
runState fourSidedDie' (mkStdGen 134)
-- (3, StdGen {unStdGen = SMGen 17500645228062596230 16743392734764332447})
-- 每次用相同的种子 134，结果都是 3

runState fourSidedDie' (mkStdGen 999)
-- 不同的种子，不同的结果
```

**这是纯函数式编程的核心特性**：没有"真正的"随机性，只有"伪随机性"——相同的种子产生相同的序列。但这正是我们想要的：**可复现的随机性**——便于调试、测试、回放。

### 🔑 洞见九：replicateM 与随机数序列

教程展示了 `replicateM` 的威力 ：

```
-- 掷 12 面骰 10 次
kSidedRolls 12 10 (mkStdGen 42)
-- ([...10 个随机数...], 最终生成器状态)
```

**`replicateM n action` 在 State 单子中做了什么**：

1. 第一次运行 `action`，得到结果 `r1` 和新状态 `g1`
    
2. 第二次在 `g1` 上运行 `action`，得到 `r2` 和 `g2`
    
3. ...重复 n 次
    
4. 返回 `[r1, r2, ..., rn]` 和最终状态 `gn`
    

**这正是非确定性计算中"序列生成"的标准模式**——State 单子让生成器状态的传递完全自动化。

### 完整示例

```
import Control.Monad.State
import System.Random

type RandState a = State StdGen a

-- 掷 k 面骰
roll :: Int -> RandState Int
roll k = state $ uniformR (1, k)

-- 模拟掷 n 次 k 面骰
rollMany :: Int -> Int -> RandState [Int]
rollMany k n = replicateM n (roll k)

-- 模拟掷两个六面骰的和
rollDiceSum :: RandState Int
rollDiceSum = do
  d1 <- roll 6
  d2 <- roll 6
  return (d1 + d2)

-- 直到掷出和为 8 或更大
untilAtLeastEight :: RandState [(Int, Int)]
untilAtLeastEight = go []
  where
    go acc = do
      d1 <- roll 6
      d2 <- roll 6
      let pair = (d1, d2)
          newAcc = pair : acc
      if d1 + d2 >= 8
        then return (reverse newAcc)
        else go newAcc

main :: IO ()
main = do
  let gen0 = mkStdGen 42
  
  -- 掷 10 次 20 面骰
  let (rolls, gen1) = runState (rollMany 20 10) gen0
  putStrLn $ "Rolls: " ++ show rolls
  
  -- 掷两个六面骰的和
  let (sum, gen2) = runState rollDiceSum gen1
  putStrLn $ "Dice sum: " ++ show sum
  
  -- 直到和为 8 或更大
  let (pairs, gen3) = runState untilAtLeastEight gen2
  putStrLn $ "Pairs until sum >= 8: " ++ show pairs
  
  -- 用 evalState 只取结果
  let finalSum = evalState rollDiceSum gen3
  putStrLn $ "Final dice sum: " ++ show finalSum
```

### 🔑 洞见十：State 单子与 IO 的随机数对比

|维度|`System.Random` 的 IO|`State StdGen`|
|---|---|---|
|随机性来源|系统熵（真随机）|种子（伪随机）|
|可复现性|❌ 不可复现|✅ 完全可复现|
|适用场景|生产环境、密码学|测试、模拟、游戏|
|组合性|IO 单子|State 单子（可与 ReaderT/WriterT 组合）|

**实践建议**：

- 需要"真随机"且不在乎可复现性 → 用 `IO` + `randomIO`
    
- 需要"可控的伪随机"（测试、回放、模拟）→ 用 `State StdGen`
    

---

## 19.3 简易计算器：State 单子建模"全局变量"

### 问题设定

图灵社区文章给出了一个独特的计算器设计 ：

```
-- 对"全局变量"做加减乘除
(~+) :: Double -> State Double (Double -> Double)
(~+) x = State $ \s -> ((+x), s + x)

(~-) :: Double -> State Double (Double -> Double)
(~-) x = State $ \s -> (((-) x), s - x)

(~*) :: Double -> State Double (Double -> Double)
(~*) x = State $ \s -> ((*x), s * x)

(~/) :: Double -> State Double (Double -> Double)
(~/) x = State $ \s -> ((/x), s / x)

-- 重复某个计算
(~~) :: (Double -> Double) -> State Double (Double -> Double)
(~~) f = State $ \s -> (f, f s)

-- 组合运算
op :: State Double (Double -> Double)
op = do
  (~+) 10
  (~*) 4
  (~-) 2
  (~/) 10
  >>= (~~) >>= (~~)

main :: IO ()
main = do
  let (_, result) = runState op 0
  print result
```

### 🔑 洞见十一：这个计算器的独特设计

**关键观察**：每个操作符 `~+`、`~*` 等返回的不是普通值，而是 **`Double -> Double` 函数**——

```
(~+) 10 :: State Double (Double -> Double)
```

**这是什么意思？**

- `State Double` 表示"在 Double 状态上下文中"
    
- `(Double -> Double)` 是计算结果——一个函数
    
- 同时，状态 `s` 被更新为 `s + x`
    

**双重效果**：

1. **产出**：一个函数 `f = (+x)`——可以对任意输入应用
    
2. **状态更新**：`s` 变为 `s + x`——全局变量被修改
    

### 🔑 洞见十二：操作符的语义解析

以 `(~+)` 为例 ：

```
(~+) x = State $ \s -> ((+x), s + x)
```

**解读**：

- 输入状态 `s`（当前全局变量值）
    
- 产出 `(+x)`——一个"加 x"的函数
    
- 新状态 `s + x`——全局变量增加了 x
    

**运行示例**（初始状态为 0）：

```
runState ((~+) 10) 0
-- 产出：函数 (+10)
-- 新状态：0 + 10 = 10

runState ((~*) 4) 10
-- 产出：函数 (*4)
-- 新状态：10 * 4 = 40

runState ((~-) 2) 40
-- 产出：函数 (\y -> y - 2)  -- 注意：这里是 ((-) 2)，即 \y -> 2 - y
-- 新状态：40 - 2 = 38

runState ((~/) 10) 38
-- 产出：函数 (/10)，即 \y -> y / 10
-- 新状态：38 / 10 = 3.8
```

### 🔑 洞见十三：`(~~)` 的"重复应用"语义

```
(~~) :: (Double -> Double) -> State Double (Double -> Double)
(~~) f = State $ \s -> (f, f s)
```

**解读**：

- 输入一个函数 `f`
    
- 产出 `f` 本身
    
- 新状态 `f s`——把 `f` 应用到当前状态 `s`
    

**作用**：在 `op` 的最后，`>>= (~~) >>= (~~)` 连续两次把前面累积的函数应用到当前状态——

```
op = do
  (~+) 10   -- 状态变为 10，产出 (+10)
  (~*) 4    -- 状态变为 40，产出 (*4)
  (~-) 2    -- 状态变为 38，产出 (\y -> 2 - y)
  (~/) 10   -- 状态变为 3.8，产出 (/10)
  >>= (~~) >>= (~~)   -- 把累积的函数应用到状态
```

**注意**：`do` 块中的 `<-` 绑定在这里没有出现——因为每个操作符产出的函数被直接用于后续计算。最终 `>>= (~~) >>= (~~)` 把最后一个函数（`/10`）应用到状态 3.8 上。

### 对比：更传统的计算器设计

芝加哥大学讲义给出了另一种更直观的计算器 ：

```
type CalcState = State [Double]
```

**设计思路**：用一个 `Double` 列表作为栈，每次运算操作栈顶元素。

更传统的"全局变量"计算器：

```
import Control.Monad.State

-- 全局变量类型
type Calc = State Double

-- 加法：全局变量 += x
add :: Double -> Calc ()
add x = modify (+ x)

-- 减法：全局变量 -= x
subtract :: Double -> Calc ()
subtract x = modify (subtract x)

-- 乘法：全局变量 *= x
multiply :: Double -> Calc ()
multiply x = modify (* x)

-- 除法：全局变量 /= x
divide :: Double -> Calc ()
divide x = modify (/ x)

-- 获取当前值
getCurrent :: Calc Double
getCurrent = get

-- 组合运算
calculate :: Double -> Calc Double
calculate x = do
  add 10        -- 全局变量 +10
  multiply 4    -- 全局变量 *4
  subtract 2    -- 全局变量 -2
  divide 10     -- 全局变量 /10
  getCurrent    -- 返回当前值

main :: IO ()
main = do
  let result = evalState (calculate 0) 0
  print result  -- 3.8
```

### 🔑 洞见十四：两种计算器设计的对比

|维度|书中设计（`~+` 等）|传统设计（`add` 等）|
|---|---|---|
|操作符返回|`Double -> Double` 函数|`()`|
|全局变量|隐式更新|隐式更新|
|计算方式|产出函数，最后应用|直接修改状态|
|灵活性|高（函数可组合）|低（操作固定）|
|直观性|较低|较高|

**书中的设计更"函数式"**——每个操作符同时产出"转换函数"和"状态更新"，最后通过 `(~~)` 把函数应用到状态上。这种设计展示了 State 单子的表现力：**状态更新和值产出可以是分离的**。

### 🔑 洞见十五：State 单子让"全局变量"变得可控

传统命令式语言中的"全局变量"：

```
# Python 全局变量
total = 0

def add(x):
    global total
    total += x
    return total

def multiply(x):
    global total
    total *= x
    return total

# 问题：全局变量可以被任何地方修改，难以追踪
```

Haskell 的 State 单子：

```
-- 全局变量被封装在 State 单子中
type Calc = State Double

add :: Double -> Calc ()
add x = modify (+ x)

-- 优势：
-- 1. 全局变量的作用域被限制在 State 计算内
-- 2. 类型系统保证只有 Calc 计算能修改它
-- 3. 可以用 runState/evalState/execState 精确控制
-- 4. 纯函数式——相同输入总是产生相同输出
```

**这就是"受控的全局变量"**——既有全局变量的便利，又有纯函数的可预测性。

---

## 🧭 本章在知识体系中的位置

```
第十六章: Monad（>>= 的定义）
   ↓
第十七章: 列表单子（非确定性）
第十八章: Reader 单子（只读环境）
   ↓
第十九章: State 单子（可读写状态）  ← 你在这里
   ↓         ↓                ↓
第二十章: IO 单子实践    第二十四章: Monad Transformers
                        （StateT 组合）
```

**本章给你三件行李**：

1. **State 单子的本质**：`State s a = s -> (a, s)`——一个状态转换函数；`>>=` 让状态从左到右流过计算
    
2. **核心操作**：`get` 读取状态、`put` 替换状态、`modify` 用函数更新状态、`gets` 提取状态的投影、`state` 从状态转换函数构造 State
    
3. **State 单子的两大应用**：随机数生成（隐藏生成器状态传递）、简易计算器（建模全局变量）
    

---

## 📝 本章核心洞见总结

> **洞见一：State 单子建模"可读写状态"**。
> 
> `State s a = s -> (a, s)`——输入状态 `s`，输出计算结果 `a` 和更新后的状态 `s`。与 Reader 的 `r -> a` 相比，State 不仅读取状态，还能写入状态 。

> **洞见二：`>>=` 让状态"从左到右"流过计算**。
> 
> `m >>= f = \s -> let (a, t) = m s in f a t`——第一步计算在状态 `s` 下运行，产生新状态 `t`；第二步计算在 `t` 下运行（**不是**​ `s`）。状态自动传递和更新，你不需要手动管理 。

> **洞见三：get/put/modify 是 State 单子的三大原语**。
> 
> `get = State $ \s -> (s, s)` 读取状态；`put t = State $ \_ -> ((), t)` 替换状态；`modify f = State $ \s -> ((), f s)` 用函数更新状态 。这三者的组合覆盖了所有状态操作。

> **洞见四：`state` 函数从状态转换函数构造 State**。
> 
> `state :: MonadState s m => (s -> (a, s)) -> m a`——把纯函数 `s -> (a, s)` 提升为 State 计算。这在随机数生成中极有用：`roll = state $ uniformR (1, n)` 。

> **洞见五：evalState/execState/runState 的分工**。
> 
> `runState` 返回 `(计算结果, 最终状态)`；`evalState` 只取计算结果；`execState` 只取最终状态 。这三者让你精确控制需要哪部分信息。

> **洞见六：State 单子让随机数生成变得优雅**。
> 
> 纯函数式随机数本质是 `seed -> (value, newSeed)`——每次调用都必须传递新种子。State 单子用 `state $ uniformR (1, n)` 把生成器隐藏起来，`>>=` 自动完成种子传递，`do` 块中的所有随机数调用共享同一个隐式生成器 。

> **洞见七：纯函数式随机数是"伪随机"但可复现**。
> 
> 相同种子产生相同序列——这不是缺陷，而是特性：可调试、可测试、可回放 。生产环境用 `IO` 的 `randomIO` 获取真随机；测试/模拟用 `State StdGen` 获取可控伪随机。

> **洞见八：简易计算器的独特设计**。
> 
> 书中每个操作符 `~+`、`~*` 等返回 `Double -> Double` 函数，同时更新全局状态：`(~+) x = State $ \s -> ((+x), s + x)` 。这种设计把"状态更新"和"值产出"分离，展示了 State 单子的表现力。

> **洞见九：State 单子实现"受控的全局变量"**。
> 
> 传统全局变量可以被任何代码修改，难以追踪；State 单子把全局变量封装在计算上下文中，类型系统保证只有 State 计算能修改它，且相同输入总是产生相同输出 。

> **洞见十：replicateM 在 State 单子中自动序列化和状态传递**。
> 
> `replicateM n action` 在 State 上下文中运行 `action` n 次，每次都把更新后的状态传给下一次——这正是随机数序列生成的标准模式 。

---

## 🛠️ 实践建议

1. **用 State 单子实现计数器**：
    
    ```
    import Control.Monad.State
    
    counter :: State Int Int
    counter = do
      modify (+1)
      get
    
    -- 运行
    evalState (replicateM 5 counter) 0  -- [1,2,3,4,5]
    ```
    
2. **用 State 单子实现栈**：
    
    ```
    type Stack a = State [a]
    
    push :: a -> Stack a ()
    push x = modify (x:)
    
    pop :: Stack a (Maybe a)
    pop = do
      stack <- get
      case stack of
        [] -> return Nothing
        (x:xs) -> do
          put xs
          return (Just x)
    ```
    
3. **用 State 单子生成随机数序列**：
    
    ```
    import Control.Monad.State
    import System.Random
    
    rollDie :: State StdGen Int
    rollDie = state $ uniformR (1, 6)
    
    rollMany :: Int -> State StdGen [Int]
    rollMany n = replicateM n rollDie
    
    -- 运行
    evalState (rollMany 10) (mkStdGen 42)
    ```
    
4. **用 State 单子实现简单解释器**：
    
    ```
    type Interpreter = State Env
    
    data Env = Env { vars :: Map String Int, counter :: Int }
    
    evalExpr :: Expr -> Interpreter Int
    evalExpr (Var name) = do
      env <- get
      case Map.lookup name (vars env) of
        Just v -> return v
        Nothing -> error $ "Undefined variable: " ++ name
    evalExpr (Add e1 e2) = do
      v1 <- evalExpr e1
      v2 <- evalExpr e2
      return (v1 + v2)
    ```
    
5. **用 State 单子实现游戏状态**：
    
    ```
    data GameState = GameState
      { playerPos :: (Int, Int)
      , score :: Int
      , level :: Int
      }
    
    type Game = State GameState
    
    movePlayer :: (Int, Int) -> Game ()
    movePlayer delta = modify $ \s ->
      s { playerPos = addPos (playerPos s) delta }
    
    addScore :: Int -> Game ()
    addScore n = modify $ \s -> s { score = score s + n }
    ```
    
6. **验证 State Monad 定律**：
    
    ```
    stateLeftIdentity :: s -> a -> (a -> s -> (b, s)) -> Bool
    stateLeftIdentity s a f =
      runState (return a >>= (\x -> State (f x))) s ==
      runState (State (f a)) s
    
    stateRightIdentity :: s -> State s a -> Bool
    stateRightIdentity s m =
      runState (m >>= return) s == runState m s
    ```
    

---

## 🔮 承上启下

第二十章"IO 单子的工程实践"会在本章基础上往前推一步——**展示 State 单子如何与 IO 组合**：

- **StateT s IO a**：在 IO 计算中使用 State——既有状态传递，又有副作用
    
- **真实应用架构**：`type App = ReaderT Config (StateT AppState IO)`——配置用 Reader，应用状态用 State，副作用用 IO
    

更深远的连接：

- **第二十四章 Monad Transformers**：`StateT s m a` 是 Transformer 的最基础形式之一。本章学的 `State s a` 实际上就是 `StateT s Identity a`——把 State 换成 StateT 就能与其他 Monad 组合
    
- **第二十五章 实用的 Monad**：会深入 `MonadBaseControl`、`MonadUnliftIO` 等现代工具，StateT 在这些工具中扮演关键角色
    
- **解析器组合子**：Parsec 的内部实现就是 `ReaderT` + `StateT` + `Identity` 的组合——本章学的 State 概念是理解解析器内部机制的基础
    

**第十九章是你理解"Monad 作为状态传递"的关键一跃**。吃透 `State s a = s -> (a, s)`、理解 `>>=` 如何让状态从左到右流动、掌握 `get`/`put`/`modify` 的使用，后面的 StateT、Monad Transformers、真实应用架构就不再是"新的魔法"，而是"我们已经知道的知识的自然延伸"——你已经知道 State 单子建模"可读写状态"，第二十章只是告诉你"StateT 让 State 与 IO 组合"；你已经知道 `modify` 更新状态，第二十章只是把这种更新用于真实的副作用场景；你已经知道随机数生成用 State 隐藏生成器，第二十四章只是把这种"隐藏"扩展到多个 Monad 的同时使用。

> 💡 学习建议：读完本章后，在 GHCi 里做三个实验：
> 
> ```
> -- 1. 体验 State 的基本操作
> runState (do { modify (+10); modify (*4); modify (subtract 2); get }) 0
> -- (38, 38)  -- 结果和最终状态都是 38
> 
> -- 2. 体验随机数生成
> import Control.Monad.State
> import System.Random
> evalState (replicateM 10 (state $ uniformR (1,6))) (mkStdGen 42)
> -- [5,1,1,3,5,2,3,5,6,3]  -- 10 次掷骰结果，每次都用新种子
> 
> -- 3. 体验书上的计算器
> let op = do { (~+) 10; (~*) 4; (~-) 2; (~/) 10; >>= (~~) >>= (~~) }
> runState op 0
> -- (..., 3.8)  -- 最终全局变量值为 3.8
> ```
> 
> 亲眼看见"状态从左到右流动"、"生成器状态自动传递"、"全局变量被隐式更新"，是理解本章精髓的最佳方式。
> 
> 然后尝试自己实现一个小应用——比如"银行账户系统"（存款、取款、查询余额）或"游戏角色状态"（移动、升级、拾取物品），用 `State` 管理状态。你会发现，**State 单子让"全局状态"从"命令式语言的隐患"变成了"纯函数式的受控资源"**——既有命令式编程的直观性，又有函数式编程的可预测性。

