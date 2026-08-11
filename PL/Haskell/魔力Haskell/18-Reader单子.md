
# 第十八章 Reader 单子 —— 读书笔记

> 💡 如果说第十七章的列表单子展示了 Monad 在"非确定性搜索"中的威力，那么第十八章就是把 Monad 用在了一个日常更常见的场景——**"只读环境的隐式传递"**。表面看是 `(->) r` 的 Monad 实例和 `Reader` 新类型，实则在揭示一个深层真相：**函数 `r -> a` 本身就是一个 Monad——它把"环境 `r`"隐式地穿过整个计算过程**。当你有一堆函数都需要访问同一个配置、同一个上下文、同一份变量绑定时，Reader 单子让你**不再手动把参数层层传递**，而是把它"升格"进 Monad 上下文，用 `do` 语法糖隐式地在线程中流动。这一章让你亲眼看见：Monad 不只是 IO 和列表——它是"计算上下文"的统一抽象，而"只读环境"只是其中一种。

韩冬在这一章安排了三个小节：**18.1 `(->) a` 的单子实例声明**、**18.2 模板渲染**、**18.3 `Reader` 新类型**。表面是从"函数即单子"到"模板渲染的应用"再到"`Reader` 新类型的封装"，实则在铺设一条主线：**`(->) r` 天然是 Monad，但用 `newtype Reader r a` 包装后，能获得更清晰的语义、更好的类型类支持和后续接入 `ReaderT` 的能力**。

---

## 🎯 全局脉络：为什么 `(->) r` 是 Monad？

```
第十三章: Applicative（参数独立的应用）
   ↓
第十六章: Monad（>>= 让后续依赖前文）
   ↓
第十七章: 列表单子（非确定性计算）  ← Monad 实例 #1
   ↓
第十八章: Reader 单子（只读环境传递）  ← 你在这里  ← Monad 实例 #2
   ↓
第十九章: Writer / State 单子（日志累积 / 可读写状态）
```

**这一章是 Monad 抽象"第二次震撼亮相"**。第十七章你看到列表 `[]` 是 Monad——但它的"非确定性"语义有点特殊。而本章要揭示的是：**最普通的函数 `r -> a`，也是 Monad**。

Haskell wikibook 与 Stack Overflow 上的高赞回答都明确指出 ：

```
instance Monad ((->) r) where
  return = const
  f >>= k = \r -> k (f r) r
```

**🔑 洞见一：函数即"依赖于环境 r 的计算"**

一个函数 `r -> a` 可以读作：

- **动词视角**："给定一个环境 `r`，产出一个 `a`"
    
- **名词视角**："一个被环境 `r` 参数化的计算"
    
- **Monad 视角**："Reader 单子中的一个计算，它读取环境 `r`，产生 `a`"
    

这三种视角是同构的。而 Monad 实例让我们可以用 `do` 语法糖把这些"环境依赖的计算"组合起来，**环境 `r` 被自动穿线过每一步**。

---

## 18.1 `(->) a` 的单子实例声明

### 类型推导：从 Monad 定律到具体实现

Dave Square 的教程给出了精彩的推导过程 ：

```
-- Monad 的 (>>=) 类型
(>>=) :: m a -> (a -> m b) -> m b

-- 代入 m = (->) r
(>>=) :: (r -> a) -> (a -> r -> b) -> (r -> b)
```

**如何实现一个类型是 `(r -> a) -> (a -> r -> b) -> (r -> b)` 的函数？**​ 唯一的自然方式：

```
instance Monad ((->) r) where
  return x = \_ -> x        -- const x，忽略环境
  f >>= g = \r ->
    let a = f r             -- 从环境中提取 a
    in g a r                -- 用 a 调用 g，再把同一个环境 r 传给结果
```

化简后得到标准形式 ：

```
instance Monad ((->) r) where
  return = const
  f >>= k = \r -> k (f r) r
```

### 🔑 洞见二：`>>=` 的本质是"环境 r 的自动穿线"

`f >>= k = \r -> k (f r) r` 做了三件事 ：

1. **提取**：`f r` 在环境 `r` 下执行 `f`，得到 `a`
    
2. **绑定**：`k a` 得到一个新计算 `r -> b`
    
3. **穿线**：把**同一个**环境 `r` 传给 `k a`，得到最终的 `b`
    

**关键洞察**：环境 `r` 在 `>>=` 链中是**共享**的——`f` 和 `k a` 看到的是**同一个**​ `r`。这正是"Reader"名称的由来：**所有计算都读取同一个只读环境**。

### 🔑 洞见三：`return = const` 的直觉

`return x = \_ -> x`——忽略环境，直接返回 `x`。这意味着 `return` 构造的计算**不读取环境**，它只是一个"常量计算"。

对比 ：

```
return 42 :: Num a => (r -> a)  -- 对任意环境 r，都返回 42
(\r -> 42)                      -- 等价写法
```

### 一个具体例子：showPerson

Dave Square 的经典例子 ：

```
data Person = Person { name :: String, age :: Int, address :: String }

-- 传统写法：手动传递 person 参数
showPersonOld :: Person -> String
showPersonOld person = 
  let x = "Name: " ++ name person
      y = "Age: " ++ show (age person)
      z = "Address: " ++ address person
  in unlines [x, y, z]

-- Reader 单子写法：用 (->) Person 作为 Monad
showPerson :: Person -> String
showPerson = do
  x <- ("Name: " ++) . name
  y <- ("Age: " ++) . show . age
  z <- ("Address: " ++) . address
  return (unlines [x, y, z])
```

**🔑 洞见四：do 语法糖消除了"参数透传"**

在 `showPerson` 的 do 块中：

- `x <- ("Name: " ++) . name` 脱糖为 `("Name: " ++) . name >>= \x -> ...`
    
- 环境 `Person` 被自动传给 `("Name: " ++) . name`，提取出 `name person`
    
- 后续的 `y` 和 `z` 绑定也自动获得**同一个**​ `Person` 环境
    

**这就是 Reader 单子的核心价值**：你不再需要把 `person` 参数手动传给每个子函数——`do` 语法糖和 `>>=` 自动完成了"环境穿线"。

### 🔑 洞见五：`(->) r` 与 `Reader r a` 的同构关系

Stack Overflow 上的经典回答指出 ：

> _"They are in fact exact same. We can make this more formal by mapping between them: toArrow :: Reader r a -> r -> a and toReader :: (r -> a) -> Reader r a"_

```
-- 两者完全同构
toArrow :: Reader r a -> (r -> a)
toArrow = runReader

toReader :: (r -> a) -> Reader r a
toReader = Reader
```

**`Reader r a` 只是 `(->) r a` 的 newtype 包装**——底层机制完全相同，但 `Reader` 新类型提供了：

1. **更清晰的语义**：`Reader r a` 明确表达了"在环境 `r` 下的计算"
    
2. **更好的类型类支持**：可以为 `Reader` 定义 `MonadReader` 类
    
3. **后续扩展**：`ReaderT r m a` 是 Monad Transformer，可以与 `IO`、`Maybe` 等组合
    

---

## 18.2 模板渲染：Reader 单子的经典应用

### 问题场景

All About Monads 给出了一个经典案例 ：

> _"Consider the problem of instantiating templates which contain variable substitutions and included templates. Using the Reader monad, we can maintain an environment of all known templates and all known variable bindings."_

具体问题：

- 有一组**模板**（可能互相包含）
    
- 每个模板中有**变量占位符**（如 `{{name}}`）
    
- 需要**变量绑定环境**（如 `name -> "Alice"`）
    
- 渲染时：遇到变量就用 `asks` 查环境，遇到模板包含就用 `local` 在修改后的环境中执行子模板
    

### 🔑 洞见六：Reader 单子如何建模模板渲染

**环境 `r` 的设计**：

```
type TemplateEnv = (Map String String, Map String Template)
--               变量绑定          已知模板
```

**核心操作**：

```
-- 查找变量值
lookupVar :: String -> Reader TemplateEnv String
lookupVar name = do
  (vars, _) <- ask
  case Map.lookup name vars of
    Just v  -> return v
    Nothing -> return ""  -- 或报错

-- 简化版：用 asks
lookupVar' :: String -> Reader TemplateEnv String
lookupVar' name = asks (\(vars, _) -> fromMaybe "" $ Map.lookup name vars)
```

**🔑 洞见七：`ask` 和 `asks` 的区别**

All About Monads 给出了精确定义 ：

```
ask :: MonadReader e m => m e
asks :: MonadReader e m => (e -> a) -> m a
```

- **`ask`**：获取整个环境 `e`
    
- **`asks`**：获取环境的某个投影 `a`（`asks f = ask >>= return . f`）
    

**模板渲染中的使用**：

```
-- 用 ask 获取整个环境
renderWithAsk :: Reader TemplateEnv String
renderWithAsk = do
  (vars, templates) <- ask
  -- 手动操作 vars 和 templates
  
-- 用 asks 直接获取需要的部分
renderWithAsks :: Reader TemplateEnv String
renderWithAsks = do
  varValue <- asks (\(vars, _) -> Map.lookup "name" vars)
  template <- asks (\(_, tmpls) -> Map.lookup "header" tmpls)
  -- ...
```

### 🔑 洞见八：`local` 的威力——临时修改环境

这是本节**最精妙的部分**。All About Monads 指出 ：

> _"When a template is included with new variable definitions, we can use the local function to resolve the template in a modified environment that contains the additional variable bindings."_

`local` 的类型 ：

```
local :: MonadReader e m => (e -> e) -> m a -> m a
```

**语义**：`local f action` 在"修改后的环境 `f e`"中执行 `action`，但**不影响外部环境**。

**模板渲染中的应用**：

```
-- 渲染一个包含子模板的模板
renderTemplate :: Template -> Reader TemplateEnv String
renderTemplate (Variable name) = 
  asks (\(vars, _) -> fromMaybe "" $ Map.lookup name vars)

renderTemplate (Include templateName localVars) = do
  templates <- asks snd
  case Map.lookup templateName templates of
    Just tmpl -> 
      -- 关键：用 local 把 localVars 合并到环境中，渲染子模板
      local (addLocalVars localVars) (renderTemplate tmpl)
    Nothing -> return ""

addLocalVars :: Map String String -> TemplateEnv -> TemplateEnv
addLocalVars localVars (vars, tmpls) = 
  (Map.union localVars vars, tmpls)  -- localVars 优先级更高
```

**🔑 洞见九：local 避免了"参数透传的噩梦"**

Stack Overflow 上的精彩回答解释了 local 的真正价值 ：

> _"Imagine an extreme scenario where instead of getLength we have a very complex action, possibly also invoking other actions, so that in total we touch a few thousand lines of code. Then we find a local f action we would like to replace. In such case, we would need to rewrite all of the code of action and its dependencies, just to insert f in all the right places. local avoids that."_

**如果没有 local**，要实现"在修改后的环境中执行子计算"，你需要：

1. 手动把修改后的环境传给子计算
    
2. 子计算内部再手动传给它的子计算
    
3. **整个调用链都要改写**
    

**有了 local**，你只需要：

```
local f action  -- f 修改环境，action 是原封不动的计算
```

`action` 内部的**所有**​ `ask` 调用，都会自动看到修改后的环境 `f e`——你**不需要修改 `action` 的任何代码**。

### 完整示例

```
import Control.Monad.Reader
import qualified Data.Map as M

type VarBindings = M.Map String String
type Templates = M.Map String Template
type Env = (VarBindings, Templates)

data Template = Variable String
              | Include String VarBindings  -- 模板名 + 局部变量
              | Concat [Template]

-- 简化版渲染器
render :: Template -> Reader Env String
render (Variable name) = 
  asks (\(vars, _) -> M.findWithDefault "" name vars)

render (Concat parts) = do
  strings <- mapM render parts
  return (concat strings)

render (Include templateName localVars) = do
  templates <- asks snd
  case M.lookup templateName templates of
    Just tmpl -> 
      local (\(vars, tmpls) -> (M.union localVars vars, tmpls)) 
            (render tmpl)
    Nothing -> return ""

-- 使用示例
main :: IO ()
main = do
  let vars = M.fromList [("user", "Alice"), ("lang", "Haskell")]
      templates = M.fromList 
        [("greeting", Concat [Variable "user", Variable "lang"])
        ,("page", Include "greeting" (M.singleton "user" "Bob"))]
      env = (vars, templates)
      
  putStrLn $ runReader (render (Variable "user")) env
  -- "Alice"
  
  putStrLn $ runReader (render (Include "greeting" M.empty)) env
  -- "AliceHaskell"
  
  putStrLn $ runReader (render (Include "page" M.empty)) env
  -- "BobHaskell"  -- 注意：local 修改了 user 为 "Bob"
```

**🔑 洞见十：Reader 单子让"隐式环境"成为显式类型**

对比传统写法：

```
-- 传统：手动传递 env 参数
render :: Template -> Env -> String
render (Variable name) env = 
  case M.lookup name (fst env) of
    Just v -> v
    Nothing -> ""
render (Include name localVars) env =
  let newEnv = (M.union localVars (fst env), snd env)
  in render template newEnv  -- 手动传递新环境
```

**Reader 单子的优势**：

1. **环境参数隐式化**：`render :: Template -> Reader Env String` 比 `render :: Template -> Env -> String` 更简洁
    
2. **do 语法糖自然**：`x <- render template1; y <- render template2` 自动共享环境
    
3. **local 组合性强**：`local f action` 比手动构造新环境再传递更优雅
    
4. **类型清晰**：`Reader Env a` 明确告诉读者"这是一个依赖环境 Env 的计算"
    

---

## 18.3 Reader 新类型：为什么需要包装？

### 现代 Haskell 的定义

Stackage 的源代码显示 ：

```
type Reader r = ReaderT r Identity

reader :: (r -> a) -> Reader r a
runReader :: Reader r a -> r -> a
```

**🔑 洞见十一：现代 `Reader` 是 `ReaderT r Identity` 的别名**

这意味着：

- `Reader r a` = `ReaderT r Identity a` = `r -> Identity a` ≈ `r -> a`
    
- 与 `(->) r a` **完全同构**
    
- 但作为 newtype，它有独立的类型类实例
    

### 为什么不直接用 `(->) r`？

**1. 语义清晰度**：

```
-- 模糊：这是什么意思？
f :: Person -> String
f = ...

-- 清晰：明确这是一个 Reader 计算
g :: Reader Person String
g = ...
```

`Reader Person String` 明确表达了"在 `Person` 环境下的计算"，而 `Person -> String` 只是一个普通函数。

**2. 类型类实例**：

`Reader` 新类型可以定义 `MonadReader` 类 ：

```
class Monad m => MonadReader r m | m -> r where
  ask :: m r
  local :: (r -> r) -> m a -> m a
  asks :: (r -> a) -> m a

instance MonadReader r (Reader r) where
  ask = Reader id
  local f (Reader r) = Reader (r . f)
  asks f = Reader f
```

**3. 后续扩展为 Transformer**：

```
-- 单纯 Reader
type App = Reader Config

-- Reader + IO
type AppIO = ReaderT Config IO

-- Reader + IO + Maybe（出错时）
type AppIOMaybe = ReaderT Config (MaybeT IO)
```

`ReaderT` 是 Monad Transformer——它让 `Reader` 的能力**叠加**到其他 Monad 上 。这是 `(->) r` 无法直接做到的。

### 🔑 洞见十二：`Reader` vs `(->) r` 的工程取舍

|维度|`(->) r`|`Reader r a`|
|---|---|---|
|底层机制|完全相同|newtype 包装|
|语义表达|模糊（只是函数）|清晰（Reader 计算）|
|类型类|仅有基础 Monad|有 `MonadReader`|
|Transformer|不支持|支持（`ReaderT`）|
|使用场景|简单场景、教学|生产代码、库|

**实践建议**：

- **教学和简单示例**：直接用 `(->) r`，展示 Monad 实例的推导
    
- **生产代码**：用 `Reader r a`，获得更好的语义和扩展性
    

### Reader 的核心 API

```
-- 运行 Reader 计算
runReader :: Reader r a -> r -> a

-- 获取整个环境
ask :: Reader r r

-- 获取环境的投影
asks :: (r -> a) -> Reader r a
asks f = fmap f ask  -- 等价于 ask >>= \r -> return (f r)

-- 在修改后的环境中执行计算
local :: (r -> r) -> Reader r a -> Reader r a
local f (Reader r) = Reader (r . f)

-- 构造函数
reader :: (r -> a) -> Reader r a
reader f = Reader f
```

### 🔑 洞见十三：`asks` 与 `Reader` 构造函数的关系

Kwang's Haskell Blog 指出 ：

> _"asks can be very elegantly implemented in terms of fmap: asks f = fmap f ask. This simplicity hints at a deeper observation: asks is effectively the constructor Reader."_

**类型对比**：

```
Reader  :: (e -> a) -> Reader e a
asks    :: (e -> a) -> Reader e a
-- 它们是同一个函数！
```

**这意味着**：`asks f` 和 `reader f` 是完全等价的——`asks` 只是 `Reader` 构造函数的"语义化别名"。

### 🔑 洞见十四：`local` 的实现本质

```
local :: (r -> r) -> Reader r a -> Reader r a
local f (Reader r) = Reader (r . f)
```

**脱糖理解**：

- `Reader r` 中的 `r` 是函数 `r -> a`（这里变量名冲突，实际是 `env -> a`）
    
- `local f (Reader action) = Reader (action . f)`
    
- 含义：在执行 `action` 之前，先用 `f` 修改环境
    

**直观图示**：

```
正常流程：env -> action -> result
local 流程：env -> f -> modifiedEnv -> action -> result
```

**关键**：`local` **不修改原始环境**——它只在 `action` 的执行期间临时替换环境，`action` 结束后，外部环境保持不变 。

### 完整工程示例

```
{-# LANGUAGE FlexibleContexts #-}

import Control.Monad.Reader
import qualified Data.Map as M

-- 应用配置
data Config = Config
  { dbHost :: String
  , dbPort :: Int
  , debugMode :: Bool
  } deriving Show

-- 应用状态（累积的日志）
type App = Reader Config

-- 业务逻辑
getUserById :: Int -> App String
getUserById userId = do
  cfg <- ask
  if debugMode cfg
    then return $ "[DEBUG] Fetching user " ++ show userId 
                ++ " from " ++ dbHost cfg ++ ":" ++ show (dbPort cfg)
    else return $ "User " ++ show userId

listUsers :: [Int] -> App [String]
listUsers ids = mapM getUserById ids

-- 在修改配置的情况下运行
debugRun :: App a -> a
debugRun app = runReader app debugConfig
  where
    debugConfig = Config "localhost" 5432 True

prodRun :: App a -> a
prodRun app = runReader app prodConfig
  where
    prodConfig = Config "db.example.com" 5432 False

main :: IO ()
main = do
  let ids = [1, 2, 3]
  
  putStrLn "=== Debug Mode ==="
  mapM_ putStrLn $ debugRun (listUsers ids)
  
  putStrLn "\n=== Production Mode ==="
  mapM_ putStrLn $ prodRun (listUsers ids)
  
  -- 临时切换 debug 模式
  putStrLn "\n=== Temporary Debug ==="
  let app = local (\cfg -> cfg { debugMode = True }) (listUsers ids)
  mapM_ putStrLn $ runReader app prodConfig
```

**运行结果**：

```
=== Debug Mode ===
[DEBUG] Fetching user 1 from localhost:5432
[DEBUG] Fetching user 2 from localhost:5432
[DEBUG] Fetching user 3 from localhost:5432

=== Production Mode ===
User 1
User 2
User 3

=== Temporary Debug ===
[DEBUG] Fetching user 1 from db.example.com:5432
[DEBUG] Fetching user 2 from db.example.com:5432
[DEBUG] Fetching user 3 from db.example.com:5432
```

**🔑 洞见十五：Reader 单子让"配置依赖"成为类型的一部分**

对比传统写法：

```
-- 传统：手动传递 config
getUserById :: Config -> Int -> String
getUserById cfg userId = ...

listUsers :: Config -> [Int] -> [String]
listUsers cfg ids = map (getUserById cfg) ids

main = do
  let cfg = ...
  mapM_ putStrLn (listUsers cfg [1,2,3])
```

**Reader 单子的优势**：

1. **签名更简洁**：`getUserById :: Int -> App String` vs `getUserById :: Config -> Int -> String`
    
2. **do 语法糖自然**：`do cfg <- ask; ...` 比 `let cfg = ...` 更符合 Monad 风格
    
3. **local 组合性**：`local (\cfg -> cfg { debugMode = True }) app` 比手动构造新 config 再传递更优雅
    
4. **类型即文档**：`App a = Reader Config a` 明确告诉读者"这个计算依赖 Config"
    

---

## 🧭 本章在知识体系中的位置

```
第十六章: Monad（>>= 的定义）
   ↓
第十七章: 列表单子（非确定性）  ← Monad 实例 #1
   ↓
第十八章: Reader 单子（只读环境）  ← 你在这里  ← Monad 实例 #2
   ↓         ↓                ↓
第十九章: Writer/State      第二十四章: Transformer
（日志/状态）             （ReaderT 组合）
```

**本章给你三件行李**：

1. **`(->) r` 天然是 Monad**：`return = const`，`f >>= k = \r -> k (f r) r`——环境 `r` 自动穿线过 `>>=` 链
    
2. **Reader 单子的核心 API**：`ask` 获取环境、`asks` 获取环境的投影、`local` 在修改后的环境中执行、`runReader` 运行计算
    
3. **`Reader r a` 是 `(->) r a` 的 newtype 包装**：语义更清晰、支持 `MonadReader` 类型类、可扩展为 `ReaderT`
    

---

## 📝 本章核心洞见总结

> **洞见一：`(->) r` 是 Monad，因为环境 `r` 可以自动穿线**。
> 
> `f >>= k = \r -> k (f r) r`——`f` 和 `k (f r)` 共享同一个环境 `r`。这意味着你可以用 `do` 语法糖把"环境依赖的计算"组合起来，环境自动流过每一步 。

> **洞见二：`return = const` 的直觉**。
> 
> `return x = \_ -> x` 忽略环境，直接返回 `x`。这构造了一个"不读取环境"的计算——它是 `>>=` 的单位元 。

> **洞见三：Reader 单子消除了"参数透传"**。
> 
> 传统写法中，每个函数都需要手动接收和传递配置参数；Reader 单子让配置"隐式"存在于 Monad 上下文中，`do` 语法糖自动完成穿线。Dave Square 的 `showPerson` 例子完美展示了这种简化 。

> **洞见四：`ask` vs `asks` 的区别**。
> 
> `ask :: MonadReader e m => m e` 获取整个环境；`asks :: MonadReader e m => (e -> a) -> m a` 获取环境的某个投影。`asks f = fmap f ask` 。

> **洞见五：`local` 是 Reader 单子最精妙的操作**。
> 
> `local f action` 在"修改后的环境 `f e`"中执行 `action`，但**不影响外部环境**。`action` 内部的所有 `ask` 调用都会自动看到新环境——你不需要修改 `action` 的任何代码 。

> **洞见六：模板渲染是 Reader 单子的经典应用**。
> 
> 环境 = (变量绑定, 已知模板)；遇到变量用 `asks` 查环境；遇到模板包含用 `local` 在修改后的环境中执行子模板。All About Monads 的经典案例展示了这种模式的优雅 。

> **洞见七：`Reader r a` 与 `(->) r a` 完全同构**。
> 
> `toArrow = runReader`，`toReader = Reader`——两者可以无损转换。但 `Reader` 新类型提供了更清晰的语义、更好的类型类支持，以及后续扩展为 `ReaderT` 的能力 。

> **洞见八：现代 Haskell 中 `Reader r a = ReaderT r Identity a`**。
> 
> `Reader` 是 `ReaderT` 的特化版本。这意味着 `Reader` 可以通过换成 `ReaderT` 轻松升级为"Reader + 其他 Monad"的组合，这是 `(->) r` 无法直接做到的 。

> **洞见九：`asks` 与 `Reader` 构造函数等价**。
> 
> `asks f` 和 `reader f` 类型完全相同，都是 `(e -> a) -> Reader e a`。`asks` 只是 `Reader` 构造函数的语义化别名 。

> **洞见十：Reader 单子 vs 显式参数传递的工程取舍**。
> 
> 简单场景用 `(->) r` 即可；生产代码用 `Reader r a` 获得更好的语义和扩展性。Reader 单子的价值在于：签名更简洁、do 语法糖自然、local 组合性强、类型即文档。

---

## 🛠️ 实践建议

1. **用 Reader 单子管理配置**：
    
    ```
    data Config = Config { dbHost :: String, dbPort :: Int }
    type App = Reader Config
    
    queryDB :: String -> App Result
    queryDB sql = do
      cfg <- ask
      -- 使用 cfg 中的 dbHost 和 dbPort
    ```
    
2. **用 `asks` 提取配置字段**：
    
    ```
    getHost :: App String
    getHost = asks dbHost
    
    getPort :: App Int
    getPort = asks dbPort
    ```
    
3. **用 `local` 临时修改环境**：
    
    ```
    -- 在 debug 模式下运行子计算
    withDebug :: App a -> App a
    withDebug app = local (\cfg -> cfg { debugMode = True }) app
    ```
    
4. **用 `runReader` 运行计算**：
    
    ```
    main :: IO ()
    main = do
      let cfg = Config "localhost" 5432
      let result = runReader (queryDB "SELECT ...") cfg
      print result
    ```
    
5. **Reader 与 do 语法糖的组合**：
    
    ```
    complexQuery :: App (String, Int, Bool)
    complexQuery = do
      host <- asks dbHost
      port <- asks dbPort
      debug <- asks debugMode
      result <- queryDB "SELECT ..."
      return (host, port, debug)
    ```
    
6. **验证 Reader Monad 定律**：
    
    ```
    readerLeftIdentity :: (r -> a) -> (a -> r -> b) -> r -> Bool
    readerLeftIdentity f g r = 
      (runReader (return a >>= k)) r == runReader (k a) r
      where a = f r
            k x = reader (g x)
    ```
    

---

## 🔮 承上启下

第十九章"Writer/State 单子"会在本章基础上往前推一步——**展示 Monad 不仅可以"读取"环境，还可以"写入"日志（Writer）和"读写"状态（State）**：

- **Writer 单子**：累积日志/输出（类似第十四章学的 Monoid，但包装在 Monad 中）
    
- **State 单子**：隐式传递可读写状态（如计数器、随机数生成器、解释器环境）
    
- **Monad Transformer**：`ReaderT`、`WriterT`、`StateT` 的组合——多个 Monad 能力叠加
    

更深远的连接：

- **第二十四章 Monad Transformers**：`ReaderT r m a` 是 Transformer 的最简单形式。本章学的 `Reader` 可以通过换成 `ReaderT` 升级为"Reader + IO"、"Reader + Maybe"等组合
    
- **第二十章 IO Monad 的工程实践**：`ReaderT Config IO` 是真实 Haskell 应用中最常见的 Monad 栈之一
    
- **第二十一章 解析器组合子**：Parsec 的 `ParsecT` 就是 `ReaderT` + `StateT` + `Identity` 的组合——本章学的 `Reader` 概念是理解解析器内部机制的基础
    

**第十八章是你理解"Monad 作为计算上下文"的关键一跃**。吃透 `(->) r` 的 Monad 实例、理解 `ask`/`asks`/`local` 的语义、明白 `Reader` 新类型与 `(->) r` 的同构关系，后面的 Writer/State 单子就不再是"新的魔法"，而是"我们已经知道的知识的自然延伸"——你已经知道 Monad 可以"读取"环境（Reader），第十九章只是告诉你 Monad 还可以"写入"日志（Writer）和"读写"状态（State）；你已经知道 `local` 临时修改环境，第十九章只是把这种"环境操作"扩展到"状态操作"；你已经知道 `Reader r a = ReaderT r Identity a`，第十九章只是揭示 `Writer w a = WriterT w Identity a` 和 `State s a = StateT s Identity a` 是完全对称的。

> 💡 学习建议：读完本章后，在 GHCi 里做三个实验：
> 
> ```
> -- 1. 体验 (->) r 的 Monad 实例
> let f = ("Hello, " ++) :: String -> String
> let g = (++ "!") :: String -> String
> (f >=> g) "World"  -- "Hello, World!"
> -- 等价于：\r -> g (f r) r
> 
> -- 2. 体验 Reader 的 ask 和 local
> import Control.Monad.Reader
> runReader ask "environment"  -- "environment"
> runReader (local ("Prefix " ++) ask) "env"  -- "Prefix env"
> 
> -- 3. 体验模板渲染
> let env = (M.fromList [("name", "Alice")], M.empty)
> runReader (asks (\(vars, _) -> M.lookup "name" vars)) env
> -- Just "Alice"
> ```
> 
> 亲眼看见"环境自动穿线"、"local 临时修改环境"、"asks 提取环境投影"，是理解本章精髓的最佳方式。
> 
> 然后尝试自己实现一个小应用——比如"配置驱动的邮件发送器"或"多环境部署工具"，用 `Reader Config` 管理配置，用 `local` 处理环境特定的覆盖。你会发现，**Reader 单子让"配置依赖"从"参数透传的噩梦"变成了"do 块中的一行 `ask`"**。

