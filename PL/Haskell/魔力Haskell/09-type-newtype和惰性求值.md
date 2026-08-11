
# 第九章 type、newtype 和惰性求值 —— 读书笔记

> 💡 如果说前八章一直在回答"如何定义类型"和"如何操作类型"，那么第九章就是在回答两个更底层的问题：**什么时候需要"新类型"？什么时候"类型"只是名字？以及——Haskell 程序到底是怎么跑起来的？**​ 这一章的三块内容（type / newtype / 惰性求值）看似松散，实则有一条金线贯穿：**Haskell 在编译期做尽可能多的类型区分，在运行期做尽可能少的无谓计算**。type 是"只编译期有意义的名字"，newtype 是"编译期新类型、运行期零开销的包装"，惰性求值则是"运行期按需计算"的策略——三者共同体现了 Haskell "编译期严格、运行期懒惰"的设计哲学。

韩冬在这一章安排了三个小节：**9.1 类型别名 `type`**、**9.2 新类型声明 `newtype`**、**9.3 惰性求值**（含 9.3.1 标记语义、常态和弱常态；9.3.2 `seq` 和 `deepseq`）。表面是语法 + 求值策略，实则在揭示 Haskell 类型系统与运行时模型的两个核心真相。

---

## 🎯 全局脉络：为什么是 type / newtype / 惰性求值？

```
第二章: data 创造新类型（运行时也有开销）
   ↓
第八章: 数字类型类（为现有类型追加行为）
   ↓
第九章: type / newtype / 惰性求值  ← 你在这里
   ↓         ↓              ↓
第十一章: Functor    第十三章: Applicative    第十六章: Monad
（newtype 是这些抽象的标准载体）
```

**这一章是整本书的"工程实践转折点"**。前八章你学会了用 `data` 创造类型、用类型类追加行为，但有两个问题一直没解决：

1. **类型膨胀问题**：`data Meter = Meter Double` 和 `data Second = Second Double` 在运行时都带了额外的构造子开销，但你其实只想让编译器区分它们，运行时它们就是 `Double`
    
2. **求值时机问题**：Haskell 程序到底什么时候真正计算？为什么 `take 5 (repeat 1)` 不会死循环？
    

第九章同时回答了这两个问题：**newtype 解决"零成本类型区分"，惰性求值解决"按需计算"**。两者结合，让 Haskell 既能有"类型级的安全"，又能有"运行时的效率"。

---

## 9.1 类型别名 `type`：只为可读性的"名字"

### 基本形态

```
type UserId = Int
type Deck = [Card]
type PhoneBook = [(String, String)]
```

### 🔑 洞见一：type 不创建新类型，只是别名

这是 type 最容易被误解的点。根据 Haskell 教程的明确说明：**`type` 只是为现有类型起一个新名字，不创建新类型**。`UserId` 和 `Int` **完全可互换**，编译器无法区分它们。

```
-- 以下两种写法完全等价
type UserId = Int
f :: UserId -> UserId
f x = x + 1

g :: Int -> Int
g x = x + 1

f 3 + g 5  -- 合法，因为 UserId 就是 Int
```

### 🔑 洞见二：type 的唯一价值是"文档化"

既然 type 不提供类型安全，它存在的唯一理由是**让类型签名更具可读性**：

```
-- 不清晰
process :: [(String, String)] -> String

-- 清晰
type PhoneBook = [(String, String)]
process :: PhoneBook -> String
```

调用者一眼就能看出这个函数处理的是"电话簿"而非"任意的字符串对列表"。

> ⚠️ **type 的局限**：你不能给 type 别名 `deriving` 类型类实例，也不能用它来区分两个不同的概念。如果你的意图是"让 `UserId` 和 `ProductId` 虽然底层都是 `Int` 但不能混用"——type 做不到，必须用 newtype。

### 实际应用

- **类型签名文档化**：`type Matrix = [[Double]]`
    
- **复杂类型的简写**：`type Parser a = String -> [(a, String)]`
    
- **领域概念命名**：`type Dollars = Double`、`type Euros = Double`
    

---

## 9.2 新类型声明 `newtype`：零成本的"真·新类型"

### 基本形态

```
newtype UserId = UserId Int
newtype Meter = Meter Double
newtype State s a = State { runState :: s -> (s, a) }
```

### 🔑 洞见三：newtype 的两条铁律

newtype 比 data 限制更严：

1. **只能有一个构造函数**
    
2. **构造函数只能有一个字段**
    

```
-- ✅ 合法
newtype Meter = Meter Double

-- ❌ 非法：多个构造子
newtype Bad = A Int | B Int

-- ❌ 非法：多个字段
newtype Bad2 = Bad2 Int String
```

这两条限制不是 bug，而是 feature——**它让 newtype 和底层类型在运行时"同构"**。

### 🔑 洞见四：newtype 是"编译期新类型，运行期零开销"

这是 newtype 最深刻的设计动机。Haskell Wiki 说得很透彻：

> _"after the type is checked at compile time, at run time the two types can be treated essentially the same, without the overhead or indirection normally associated with a data constructor"_

也就是说：

- **编译期**：`Meter` 和 `Double` 是**完全不同的类型**，不能混用
    
- **运行期**：`Meter` 的值就是 `Double` 的值，**没有任何额外的盒子/标签**
    

对比 data：

```
data DataMeter = DataMeter Double  -- 运行时有额外构造子开销
newtype NewMeter = NewMeter Double  -- 运行时就是 Double 本身
```

**为什么 newtype 能做到零开销？**​ 因为 newtype 只有唯一一个构造子、唯一一个字段，编译器在运行时**直接把构造子"擦除"**——它只是把底层值盒子上的"标签"从 `Double` 换成了 `Meter`，盒子里面装的东西完全没变。

### 🔑 洞见五：newtype 的三个核心用例

根据 Haskell 教程总结：

**① 区分相同底层类型的不同概念**

```
newtype UserId = UserId Int
newtype ProductId = ProductId Int
newtype Quantity = Quantity Int

createOrder :: UserId -> ProductId -> Quantity -> IO Order
-- 编译器现在能保证：你不可能把 ProductId 传给 UserId 参数
```

**② 为现有类型挂不同的类型类实例**

这是 newtype 在 Haskell 标准库里**最经典的使用场景**。比如 `Data.Monoid` 里的：

```
newtype Sum a = Sum a       -- 让 a 的 Monoid 实例用 (+) 做 mappend
newtype Product a = Product a  -- 让 a 的 Monoid 实例用 (*) 做 mappend
newtype Down a = Down a     -- 反转 Ord 实例的顺序
```

同一个 `Int`，通过 newtype 包装，可以拥有"求和 Monoid"和"求积 Monoid"两种不同实例。**这是 type 做不到的——type 不创建新类型，自然也不能有新实例**。

**③ 零成本抽象**

```
newtype Day = Day Int deriving (Show, Eq, Ord)
```

`Day` 在运行时就是 `Int`，但你得到了类型安全——编译器会阻止 `Day` 和 `Int` 的混用。

### 🔑 洞见六：newtype vs data vs type——三者的本质区别

|维度|`type`|`newtype`|`data`|
|---|---|---|---|
|创建新类型？|❌ 否（只是别名）|✅ 是|✅ 是|
|运行期开销|无|**零**（构造子被擦除）|有（额外构造子标签）|
|构造子数量|无|必须 1 个|任意|
|字段数量|无|必须 1 个|任意|
|可否 deriving 实例|❌ 否|✅ 是|✅ 是|
|严格性|同底层类型|**非严格**（模式匹配不强制求值）|**严格**（模式匹配强制到 WHNF）|
|典型用途|类型签名可读性|类型区分、挂不同实例|从零创造代数数据类型|

> 📌 **这张表是第九章最值得记住的内容**。选型准则：
> 
> - 只想让类型签名更易读 → `type`
>     
> - 想区分概念或挂不同类型类实例，且只有一个字段 → `newtype`
>     
> - 要从零创造数据类型（多构造子/多字段）→ `data`
>     

### 🔑 洞见七：newtype 与 data 的"严格性差异"——这是高级但极重要的点

这是 newtype 一个反直觉但极关键的特性。Haskell Wiki 明确指出：

> _"types declared with the data keyword are lifted - that is, they contain their own ⊥ value that is distinct from all the others... newtype types can only have one possible value constructor and one field"_

换句话说：

- **data 类型**是 **lifted**​ 的——`⊥ :: DataMeter` 与 `DataMeter ⊥ :: DataMeter` 是**两个不同的值**
    
- **newtype 类型**是 **unlifted**​ 的——`N ⊥` 与 `⊥` 是**同一个值**
    

这导致模式匹配时的行为差异：

```
data DataInt = D Int
newtype NewInt = N Int

case D undefined of D _ -> 1   -- 报错！因为要检查是 ⊥ 还是 D _
case N undefined of N _ -> 1   -- 返回 1！因为 N ⊥ = ⊥，模式匹配不强制求值
```

**为什么这个差异重要？**​ 因为它影响着递归定义和某些抽象的行为。比如在定义 Monad 的某些实例时，newtype 的非严格性可以避免无限循环。这也是为什么标准库里 `State`、`Reader`、`Writer` 等抽象都用 newtype 而非 data——它们需要这种"零成本 + 非严格"的组合特性。

### 实际应用

```
-- 1. 类型安全的单位系统
newtype Meter = Meter Double
newtype Second = Second Double

speed :: Meter -> Second -> Double
speed (Meter m) (Second s) = m / s

-- 2. 为现有类型挂不同的 Monoid 实例
import Data.Monoid

sumOfSquares = getSum $ foldMap (Sum . (^2)) [1..10]  -- 385
productOfNums = getProduct $ foldMap Product [1..5]   -- 120

-- 3. 零成本的状态包装（为后续 Monad 章节铺垫）
newtype State s a = State { runState :: s -> (a, s) }
```

---

## 9.3 惰性求值：Haskell 运行时的"心脏"

这是本章**最深奥也最重要**的一节。要理解惰性求值，必须先理解"程序即未求值的表达式图"。

### 9.3.1 标记语义、常态和弱常态

#### 程序即表达式

Haskell 把整个程序看作一个**巨大的未求值表达式**。运行程序的过程，就是不断对这个表达式做"归约"（reduction）的过程。

#### thunk：未求值的盒子

在内存中，未求值的表达式被表示为 **thunk（任务盒）**——一个记录"要计算什么"的数据结构。当它被需要时，thunk 才被"打开"，计算结果替换原来的 thunk。

```
let x = 1 + 2 * 3 in x + x
```

内存中最初 `x` 是一个 thunk：`1 + 2 * 3`。当第一个 `x + x` 需要 `x` 的值时，thunk 被求值得到 `7`，然后第二个 `x` 直接使用 `7`（因为共享）。

#### 弱常态（WHNF）与常态（NF）

**弱头常态（Weak Head Normal Form, WHNF）**：表达式的顶层构造子已确定，但构造子的参数可能仍未求值。

根据 Haskell 学术资料的精确定义：

> _"An expression is in weak head normal form if its type is a primitive type and the expression has been fully evaluated, or its type has been defined using data or newtype and the expression has been evaluated enough to determine its data constructor."_

举例：

```
(1+2) : []     -- 是 WHNF（顶层是 (:)，但第一个参数是 thunk）
1 : (2+3) : [] -- 是 WHNF
\x -> x + 1    -- 是 WHNF（lambda 已经是头常态）
```

**常态（Normal Form, NF）**：表达式完全求值，没有任何 thunk。

```
1 : 2 : 3 : []  -- 是 NF
```

#### 求值驱动：模式匹配

Haskell 的求值是由**模式匹配**驱动的。当函数需要对参数做模式匹配时，参数被求值到 WHNF：

```
f (x:xs) = x
```

调用 `f` 时，参数被求值到 WHNF——确认它是 `(:)` 构造子还是 `[]` 构造子。如果是 `(:)`，则 `x` 和 `xs` 仍然是 thunk，只在被真正需要时才进一步求值。

#### 🔑 洞见八：短路是惰性求值的自然结果

考虑 `(&&)` 的定义：

```
(&&) :: Bool -> Bool -> Bool
True && x = x
False && _ = False
```

求值 `True && undefined`：

1. 对第一个参数求值到 WHNF → `True`
    
2. 匹配第一个模式 `True && x` → 成功
    
3. 返回 `x`（即 `undefined`）作为 thunk——**但不强制求值**
    
4. 结果：`True && undefined` 本身是一个 thunk，只有在被强制求值时才会报错
    

而 `undefined && True` 会立即报错——因为第一个参数 `undefined` 被求值到 WHNF 时发现是 `⊥`。

**这正是惰性求值 + 模式匹配的自然行为**。它让 Haskell 拥有了"短路求值"能力，且不需要语言层面特殊处理——它只是惰性求值的自然推论。

#### 🔑 洞见九：无限列表之所以可能

回到第三章预告的 `repeat 1`：

```
repeat 1 = 1 : repeat 1
take 3 (repeat 1)  -- [1, 1, 1]
```

内存中 `repeat 1` 是一个 thunk。当 `take 3` 需要值时：

1. 对 `repeat 1` 求值到 WHNF → 得到 `1 : <thunk>`
    
2. `take 3` 匹配 `(:)` 模式，取出第一个 `1`
    
3. 对剩余的 `take 2 (repeat 1)` 递归——再次触发 `repeat 1` 求值的下一个 thunk
    
4. 如此三次后，`take 0 _` 匹配终止条件，返回 `[]`
    

**关键点**：`repeat 1` 在内存中从未完整展开。每次只展开一层，取出值后，剩下的部分仍然是个 thunk。这就是为什么 Haskell 能在有限内存中表示无限列表。

#### 🔑 洞见十：标记语义带来"引用透明"的威力

Haskell 的"标记语义"（denotational semantics）让**引用透明**（referential transparency）成为语言的硬保证：

> _"随时随地可以把一个绑定换成它对应的表达式，而不影响程序的求值"_

这带来三个深远后果：

1. **编译器可以自由做内联、化简、融合等变换**——因为语义不变
    
2. **你无法预测函数调用在什么时候发生**——求值时机由需求驱动
    
3. **equational reasoning 成立**——你可以像做数学证明一样推理 Haskell 程序
    

> 💡 这是 Haskell 区别于其他语言的核心特征。在 Java/C++ 里，函数调用时机是确定的（call-by-value）；在 Haskell 里，函数调用时机是不确定的（call-by-need），由消费者驱动。

### 9.3.2 `seq` 和 `deepseq`：打破惰性的"阀门"

惰性求值虽好，但也有代价——**thunk 堆积可能导致空间泄漏（space leak）**。

#### 问题：thunk 堆积

```
-- 危险：会堆积 1000000 个 thunk
foldl (+) 0 [1..1000000]
```

`foldl` 每次迭代都创建一个新的 thunk：`((((0+1)+2)+3)+...+1000000)`。这些 thunk 在计算最终结果前不会被求值，占用大量内存。

#### `seq`：强制求值到 WHNF

```
seq :: a -> b -> b
seq ⊥ b = ⊥
seq a b = b  (if a ≠ ⊥)
```

`seq a b` 强制 `a` 求值到 WHNF，然后返回 `b`。

```
-- 严格左折叠：用 seq 强制累积值求值
foldl' f z [] = z
foldl' f z (x:xs) = let z' = f z x in z' `seq` foldl' f z' xs
```

`foldl'` 来自 `Data.List`，它用 `seq` 强制每次累积值都求值到 WHNF，避免 thunk 堆积。

#### `deepseq`：完全求值

`seq` 只求值到 WHNF——对于嵌套结构，内部可能仍有 thunk。`deepseq` 来自 `Control.DeepSeq`，它**递归地完全求值**整个结构：

```
import Control.DeepSeq

deepseq :: NFData a => a -> b -> b
```

#### 🔑 洞见十一：seq 是"魔法"，不能在 Haskell 中实现

`seq` 是 Haskell 里少数几个**不能用 Haskell 表达**的函数。它的特殊性在于：它能区分 `⊥` 和 `N ⊥`（对于 newtype N）。正如 Haskell 资料所指出的：

> _"seq is magic, it cannot be implemented in Haskell"_

这也是为什么 newtype 的模式匹配是非严格的——`seq` 必须特殊处理 newtype，因为它要穿透 newtype 的构造子去检查底层值是否是 `⊥`。

#### 🔑 洞见十二：惰性求值的代价与收益

|收益|代价|
|---|---|
|无限数据结构|thunk 开销（内存+时间）|
|短路求值|空间泄漏风险|
|模块化（生产者/消费者解耦）|严格性分析困难|
|引用透明 → 自由的程序变换|性能预测困难|
|声明式编程|调试困难（:sprint 看 thunk）|

GHC 编译器通过**严格性分析（strictness analysis）**自动识别哪些参数必然被求值，并在适当位置插入 `seq`——这是 Haskell 能在保留惰性语义的同时获得良好性能的关键优化。

### 实际应用

```
-- 1. 使用严格折叠避免空间泄漏
import Data.List (foldl')

sum' :: [Int] -> Int
sum' = foldl' (+) 0

-- 2. 用 bang pattern 声明严格字段
data StrictPair = SP !Int !Int
-- 构造 SP 时，两个字段立即求值

-- 3. 用 $! 做严格应用
f $! x  -- 等价于 \x -> x `seq` f x

-- 4. 调试 thunk：用 :sprint 查看求值状态
-- ghci> let xs = [1..10] :: [Int]
-- ghci> :sprint xs
-- xs = _
-- ghci> xs !! 0
-- 1
-- ghci> :sprint xs
-- xs = 1 : _
```

---

## 🧭 本章在知识体系中的位置

```
第八章: 数字类型类（为现有类型追加行为）
   ↓
第九章: type / newtype / 惰性求值  ← 你在这里
   ↓         ↓              ↓
第十一章: Functor    第十三章: Applicative    第十六章: Monad
（newtype 是这些抽象的标准载体）
```

**本章给你三件行李**：

1. **type / newtype / data 的三元选择**——按需求选择：可读性用 type，类型区分/挂实例用 newtype，从零创造用 data
    
2. **newtype 的零成本抽象**——编译期新类型，运行期零开销，是 Haskell 抽象机制的标准载体
    
3. **惰性求值的完整图景**——thunk、WHNF、NF、seq/deepseq，理解 Haskell 程序如何运行
    

---

## 📝 本章核心洞见总结

> **洞见一：type 不创建新类型，只是别名**。
> 
> `type UserId = Int` 之后，`UserId` 和 `Int` 完全可互换。type 的唯一价值是让类型签名更可读，它不提供任何类型安全。

> **洞见二：newtype 是"编译期新类型，运行期零开销"的工程奇迹**。
> 
> 它创建编译期的新类型（与底层类型不可互换、可挂不同实例），但运行时构造子被擦除，值与底层类型完全一致。这正是"类型级区分，运行期零成本"的理想实现。

> **洞见三：newtype 的两条铁律（单构造子、单字段）是其零开销的代价，也是其威力的根源**。
> 
> 因为结构与底层类型同构，编译器能在运行时直接擦除构造子。这限制了 newtype 只能做"包装"，但也让它成为挂不同类型类实例、做零成本抽象的理想工具。

> **洞见四：newtype 与 data 严格性不同——这是高级但极重要的点**。
> 
> data 类型是 lifted（有独立的 `⊥`），模式匹配时强制求值到 WHNF；newtype 是 unlifted（`N ⊥ = ⊥`），模式匹配时不强制求值。这个差异影响着递归定义和 Monad 等抽象的行为。

> **洞见五：Haskell 程序是一个巨大的未求值表达式图**。
> 
> 整个程序由 thunk 组成，求值由模式匹配驱动。这是理解 Haskell 运行时模型的核心心智模型。

> **洞见六：WHNF 是 Haskell 求值的"最小单位"**。
> 
> 表达式求值到"顶层构造子已确定，但参数可能仍是 thunk"的状态，就是 WHNF。WHNF 让短路求值、无限列表、模块化成为可能。

> **洞见七：惰性求值是"引用透明"的工程后果**。
> 
> 因为表达式可以随时被替换而不改变语义，求值时机才能被推迟到"真正需要"的时刻。引用透明 ↔ 惰性求值，两者互为因果。

> **洞见八：seq/deepseq 是打破惰性的"紧急阀门"**。
> 
> 它们让你在 thunk 堆积导致空间泄漏时，能强制求值。但 seq 是语言层面的"魔法"，不能用 Haskell 实现——这反映了惰性求值对语言运行时的深层侵入。

> **洞见九：type/newtype/data 的选择，本质是"编译期 vs 运行期"的权衡**。
> 
> type 只在编译期有意义，newtype 在编译期创建新类型但运行期零开销，data 在编译期和运行期都创建新结构。理解这个光谱，你就能做出正确的工程选择。

---

## 🛠️ 实践建议

1. **type vs newtype vs data 的决策树**：
    
    ```
    只是想让类型签名更易读？
    ├─ 是 → type
    └─ 否 → 需要编译期类型区分？
            ├─ 是 → 只有一个字段需要包装？
            │       ├─ 是 → newtype
            │       └─ 否 → data
            └─ 否 → type
    ```
    
2. **用 newtype 构建类型安全的领域模型**：
    
    ```
    newtype Dollar = Dollar Double
    newtype Euro = Euro Double
    
    -- 编译器阻止 Dollar 和 Euro 混用
    exchangeRate :: Dollar -> Euro
    ```
    
3. **为现有类型挂不同的类型类实例**：
    
    ```
    import Data.Monoid
    -- Sum 和 Product 都是 newtype 包装，但 Monoid 实例行为不同
    ```
    
4. **永远警惕空间泄漏**：
    
    ```
    -- 危险：foldl 会堆积 thunk
    sum = foldl (+) 0
    
    -- 安全：foldl' 强制严格求值
    sum = foldl' (+) 0
    ```
    
5. **用 GHCi 的 :sprint 观察 thunk**：
    
    ```
    ghci> let xs = [1..5] :: [Int]
    ghci> :sprint xs
    xs = _
    ghci> xs !! 2
    3
    ghci> :sprint xs
    xs = 1 : 2 : 3 : _
    ```
    
    亲眼看见 thunk 如何被逐步求值，是理解惰性求值的最佳方式。
    
6. **记住 newtype 的非严格性**：
    
    ```
    newtype T = T Int
    case T undefined of T _ -> 1  -- 返回 1，不报错
    -- 对比 data：
    data D = D Int
    case D undefined of D _ -> 1  -- 报错
    ```
    

---

## 🔮 承上启下

第十章"模块语法以及 cabal、Haddock 工具"会把视野从语言特性拉到**工程组织**：

- **模块语法**：如何把 type/newtype/data 定义组织成模块，如何导出/隐藏
    
- **cabal**：如何管理依赖、构建项目——这是把前两部分的"类型和类型类"知识组装成真实应用的工具
    
- **Haddock**：如何从类型签名和注释自动生成文档
    

而第十一章开始的"Functor / Applicative / Monad"三部曲，会把本章 newtype 的价值彻底释放：

- **`Functor` 的 `fmap`**​ 本质是 `map` 的推广，而 `fmap` 的定律（identity、composition）可以通过 newtype 包装来验证
    
- **`Applicative` 的 `pure` 和 `<*>`**​ 在 `newtype ZipList` 上有不同于 `[]` 的行为——同一个底层列表，通过 newtype 挂不同的 `Applicative` 实例
    
- **`Monad` 的 `>>=`**​ 在 `newtype State`、`newtype Reader`、`newtype Writer` 上的实现，全部基于 newtype 的零成本包装
    

**第九章是你理解 Haskell "零成本抽象"和"运行时模型"的关键一章**。吃透 type/newtype/data 的三元选择，理解 thunk/WHNF/NF 的求值模型，后面的 Functor/Applicative/Monad 就不再是"魔法"——你已经知道 `newtype State s a = State (s -> (s, a))` 为什么用 newtype 而非 data（零开销 + 非严格性），也已经知道 `foldl` 为什么会空间泄漏、为什么需要 `foldl'`。

要继续第十章"模块语法以及 cabal、Haddock 工具"的读书笔记时，把目录发我，我会延续这条"语言特性 → 工程组织 → 高阶抽象"的脉络往下写。