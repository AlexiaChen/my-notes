
# 第二十八章 Haskell 与数据库 —— 读书笔记

> 💡 如果说第二十七章让你用 WAI/Warp 搭起了 Web 服务，那么第二十八章就是给这个服务配上"持久化记忆"——**Haskell 的数据库生态有两个主角：persistent 负责"类型安全的表/实体定义与增删改查"，esqueleto 负责"类型安全的 SQL 查询（尤其是 JOIN）"**。表面看是两个库的介绍，实则在揭示一个深层真相：**Haskell 的数据库层把"编译器即验证器"的理念推到了极致**——表结构用 quasiquote 定义，字段名和类型在编译期就绑定到 Haskell 数据类型，查询操作符（`==.`、`>=.`、`=.`）在编译期就检查类型匹配，JOIN 的字段投影在编译期就验证存在性。这一章让你亲眼看见：第二十四章学的单子变换器栈，在这里化身 `SqlPersistT`——数据库事务就是跑在这个变换器里的 `ReaderT SqlBackend`；第二十七章学的 Web 服务，用 `runDB` 把 `SqlPersistT` 动作嵌入 Handler 单子。从"定义实体"到"类型安全的 JOIN"，是 Haskell 生产级数据层的完整图景。

韩冬在这一章安排了两个小节：**28.1 persistent**、**28.2 esqueleto**。表面是从"ORM 层"到"SQL DSL"的递进，实则在铺设一条主线：**persistent 解决"实体↔表"的映射与基础 CRUD，esqueleto 解决"类型安全的复杂 SQL"——两者互补，persistent 打底，esqueleto 在 persistent 的实体定义之上做类型安全的 SQL 查询**。

---

## 🎯 全局脉络：为什么 Haskell 的数据库层是分层的？

```
第二十四章: 单子变换器（SqlPersistT = ReaderT SqlBackend）
   ↓
第二十七章: 网络编程（Web 服务用 runDB 嵌入 SqlPersistT）
   ↓
第二十八章: Haskell 与数据库  ← 你在这里
   ↓
┌─────────────────────────────────────────┐
│ esqueleto: 类型安全的 SQL EDSL          │ ← 复杂查询、JOIN
│ persistent: 类型安全的实体与 CRUD       │ ← 表定义、增删改查、迁移
│ SqlBackend: SQLite/PostgreSQL/MySQL     │ ← 数据库后端
└─────────────────────────────────────────┘
```

**这一章要解决的核心问题**：如何让数据库访问既类型安全又可表达复杂查询？

persistent 的 Hackage 文档开宗明义 ：

> _"This library intends to provide an easy, flexible, and convenient interface to various data storage backends. Backends include SQL databases, like mysql, postgresql, and sqlite, as well as NoSQL databases, like mongodb and redis."_

esqueleto 的 Hackage 文档紧接着给出定位 ：

> _"Esqueleto is a bare bones, type-safe EDSL for SQL queries that works with unmodified persistent SQL backends... In particular, esqueleto is the recommended library for type-safe JOINs on persistent SQL backends. (The alternative is using raw SQL, but that's error prone and does not offer any composability.)"_

**🔑 洞见一：persistent 与 esqueleto 是互补而非竞争**

- **persistent**：定义实体、自动迁移、基础 CRUD（`insert`/`get`/`selectList`/`update`/`delete`）
    
- **esqueleto**：在 persistent 实体之上写类型安全的 SELECT/UPDATE/INSERT/DELETE，尤其是 JOIN
    
- **关系**：esqueleto 不需要修改 persistent 的后端，两者无缝协作
    

---

## 28.1 persistent：类型安全的实体与 CRUD

### 实体定义：quasiquote 驱动的代码生成

persistent 的核心魔法是用 quasiquote 定义实体，Template Haskell 自动生成 Haskell 数据类型、字段访问器、键类型和类型类实例 。

Yesod 书的经典示例 ：

```
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE OverloadedStrings #-}

import Database.Persist
import Database.Persist.Sqlite
import Database.Persist.TH

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    name String
    age Int Maybe
    deriving Show

BlogPost
    title String
    authorId PersonId
    deriving Show
|]
```

**🔑 洞见二：persistent 自动生成了什么？**

根据 Hackage 文档 ，上面的 quasiquote 会生成：

1. **Haskell 数据类型**：
    
    ```
    data User = User { userName :: String, userAge :: Maybe Int }
        deriving Show
    ```
    
2. **键类型**：`UserId`、`BlogPostId`（包裹底层后端的键）
    
3. **字段类型**：`EntityField User String`（即 `UserName`）、`EntityField User (Maybe Int)`（即 `UserAge`）
    
4. **实体类型**：`Entity User` = 键 + 值的组合
    
5. **迁移函数**：`migrateAll`
    

### 🔑 洞见三：Entity 类型是"键 + 值"的组合

Monday Morning Haskell 精辟地解释 ：

> _"An Entity refers to a row in our database, and it associates a database ID with the object itself... This allows our other code to use a more pure User type."_

```
-- 纯的 User 值（没有数据库 ID）
let pureUser = User "Alice" (Just 30)

-- 数据库中的一行（有 ID）
let entityUser = Entity (toSqlKey 1) (User "Alice" (Just 30))
```

**这种分离让"纯数据"和"持久化数据"在类型层面清晰区分**——这是 Haskell 类型安全哲学的延伸。

### 类型类层次：分层的数据库操作

persistent 的 Hackage 文档揭示了核心类型类层次 ：

|类型类|关键操作|用途|
|---|---|---|
|`PersistStoreRead`|`get`, `getMany`|基于键的基本读取|
|`PersistStoreWrite`|`insert`, `update`, `delete`, `replace`|基本修改操作|
|`PersistUniqueRead`|`getBy`, `existsBy`|唯一约束上的操作|
|`PersistUniqueWrite`|`insertUnique`, `upsert`, `deleteBy`|唯一约束的修改|
|`PersistQueryRead`|`selectList`, `count`, `exists`|条件查询|
|`PersistQueryWrite`|`updateWhere`, `deleteWhere`|条件修改|

**🔑 洞见四：类型类层次体现了"能力最小化"原则**

每个类型类只提供特定能力——这意味着你可以写只依赖 `PersistStoreRead` 的纯函数（只读不写），编译器会强制保证该函数不会修改数据库 。这是 Haskell 类型系统对副作用的精细控制。

### 查询操作符：编译期类型检查

persistent 的 Hackage 文档列出了查询操作符 ：

```
(==.) :: EntityField v typ -> typ -> Filter v       -- 等于
(!=.) :: EntityField v typ -> typ -> Filter v       -- 不等于
(>.)  :: EntityField v typ -> typ -> Filter v       -- 大于
(<.)  :: EntityField v typ -> typ -> Filter v       -- 小于
(>=.) :: EntityField v typ -> typ -> Filter v       -- 大于等于
(<=.) :: EntityField v typ -> typ -> Filter v       -- 小于等于
(=.)  :: EntityField v typ -> typ -> Update v       -- 赋值
(+=.) :: EntityField v typ -> typ -> Update v       -- 加法赋值
(-=.) :: EntityField v typ -> typ -> Update v       -- 减法赋值
(*=.) :: EntityField v typ -> typ -> Update v       -- 乘法赋值
(/=.) :: EntityField v typ -> typ -> Update v       -- 除法赋值
```

**🔑 洞见五：操作符的类型确保字段与值的类型匹配**

```
-- ✅ 正确：UserAge 是 Maybe Int，传入 Int 值
selectList [UserAge >=. 18] []

-- ❌ 编译错误：类型不匹配
-- selectList [UserAge ==. "18"] []  -- Int vs String
```

persistent 教程明确指出 ：

> _"Attempting invalid queries results in compile errors: invalidQuery = selectList [UserAge ==. '18'] [] -- Type mismatch: Int vs Text"_

### SqlPersistT 单子：数据库动作的载体

Monday Morning Haskell 给出了关键洞察 ：

> _"The SqlPersistT Monad... All the query functions return actions in this monad. The monad transformer has to live on top of a monad that is MonadIO."_

```
-- SqlPersistT 本质是 ReaderT SqlBackend
type SqlPersistT m a = ReaderT SqlBackend m a

-- 运行数据库动作
runAction :: ConnectionString -> SqlPersistT IO a -> IO a
runAction connStr action = 
  runStdoutLoggingT $ withPostgresqlConn connStr $ \backend ->
    runReaderT action backend
```

**🔑 洞见六：SqlPersistT 是第二十四章单子变换器思想的体现**

这正是 `ReaderT SqlBackend m a`——把数据库连接作为"只读环境"注入。这意味着：

- 在 `SqlPersistT IO` 中可以做 IO 操作（通过 `liftIO`）
    
- 在 Web 框架（如 Yesod）中，`SqlPersistT` 被嵌入更大的变换器栈
    
- 用 `transaction` 开启数据库事务
    

### 实际应用：完整的 CRUD

基于 Hackage 文档 和 Monday Morning Haskell 教程 ：

```
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE OverloadedStrings #-}

import Database.Persist
import Database.Persist.Postgresql
import Database.Persist.TH
import Control.Monad.IO.Class (liftIO)
import Control.Monad.Logger (runStdoutLoggingT)
import Data.Text (Text)

-- 1. 实体定义
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Person
    name String
    age Int Maybe
    deriving Show

BlogPost
    title String
    authorId PersonId
    deriving Show
|]

-- 2. 连接字符串
connStr :: ConnectionString
connStr = "host=localhost dbname=test user=test password=test port=5432"

-- 3. 运行数据库动作
runAction :: SqlPersistT IO a -> IO a
runAction action = 
  runStdoutLoggingT $ withPostgresqlPool connStr 10 $ \pool ->
    runSqlPool action pool

-- 4. 迁移
migrateDB :: IO ()
migrateDB = runAction $ runMigration migrateAll

-- 5. 创建（Create）
createPerson :: IO ()
createPerson = runAction $ do
  johnId <- insert $ Person "John Doe" (Just 35)
  janeId <- insert $ Person "Jane Doe" Nothing
  insert $ BlogPost "My fr1st p0st" johnId
  insert $ BlogPost "One more for good measure" johnId
  liftIO $ putStrLn $ "Inserted John with id: " ++ show johnId

-- 6. 读取（Read）
readExamples :: IO ()
readExamples = runAction $ do
  -- 按 ID 查询
  johnId <- insert $ Person "John" (Just 30)
  mJohn <- get johnId
  liftIO $ print (mJohn :: Maybe Person)  -- Just (Person "John" (Just 30))
  
  -- 条件查询
  oneJohnPost <- selectList [BlogPostAuthorId ==. johnId] [LimitTo 1]
  liftIO $ print (oneJohnPost :: [Entity BlogPost])
  
  -- 计数
  count' <- count [PersonAge >=. Just 18]
  liftIO $ putStrLn $ "Adults: " ++ show count'

-- 7. 更新（Update）
updateExample :: IO ()
updateExample = runAction $ do
  johnId <- insert $ Person "John" (Just 30)
  update johnId [PersonAge =. 31]
  updateWhere [PersonName ==. "John"] [PersonAge +=. 1]

-- 8. 删除（Delete）
deleteExample :: IO ()
deleteExample = runAction $ do
  janeId <- insert $ Person "Jane" Nothing
  johnId <- insert $ Person "John" (Just 30)
  
  -- 按 ID 删除
  delete janeId
  
  -- 条件删除
  deleteWhere [BlogPostAuthorId ==. johnId]
```

### 🔑 洞见七：自动迁移是 persistent 的杀手锏

Yesod 书指出 ：

> _"Type-Driven Migrations: Persistent generates database migrations based on type definitions... The migration system: Detects schema differences, Generates appropriate DDL, Preserves existing data, Maintains type consistency."_

```
-- 运行迁移
runMigration migrateAll

-- 生成的 SQL（以 User 为例）：
-- CREATE TABLE "user" (
--   id SERIAL PRIMARY KEY,
--   name TEXT NOT NULL,
--   age INT NULL
-- );
```

**这意味着**：你修改 Haskell 实体定义，persistent 自动检测差异并生成相应的 ALTER TABLE 语句——增加字段是安全的，类型变更会做兼容性检查 。

### 关系建模：外键与关联

persistent 用 `UserId` 这种键类型表达外键 ：

```
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    name String
    email String
    UniqueEmail email
    deriving Show

Profile
    bio String
    website String Maybe
    userId UserId
    UniqueUserId userId  -- 一对一关系
    deriving Show

BlogPost
    title String
    content Text
    authorId UserId       -- 多对一关系
    deriving Show
|]
```

**🔑 洞见八：关系在类型层面的表达**

- `UserId` 作为 `Profile.userId` 的类型 → 编译器保证只能传入有效的用户键
    
- `UniqueUserId userId` → 数据库唯一约束，保证一对一
    
- `BlogPost.authorId` → 外键，指向 User 表
    

---

## 28.2 esqueleto：类型安全的 SQL EDSL

### 为什么需要 esqueleto？

Yesod 书明确指出了 persistent 的局限 ：

> _"Persistent strives to be backend-agnostic. The advantage of this approach is code which easily moves from different backend types. The downside is that you lose out on some backend-specific features. Probably the biggest casualty is SQL join support. Fortunately, thanks to Felipe Lessa and Chris Allen, you can have your cake and eat it too. The Esqueleto library provides support for writing type safe SQL queries, using the existing Persistent infrastructure."_

**persistent 的条件查询（`selectList`）不足以表达 JOIN**——这是 esqueleto 存在的根本原因。

### 核心语法：from + where_ + return

esqueleto 的 Hackage 文档给出了基本模式 ：

```
-- 基础 SELECT
putPersons :: SqlPersist m ()
putPersons = do
  people <- select $ from $ \person -> do
    return person
  liftIO $ mapM_ (putStrLn . personName . entityVal) people

-- 带 WHERE 的 SELECT
select $ from $ \p -> do
  where_ (p ^. PersonName ==. val "John")
  return p
```

**🔑 洞见九：esqueleto 的 SQL 映射几乎是直译**

|esqueleto 语法|生成的 SQL|
|---|---|
|`from $ \p -> ...`|`FROM Person`|
|`where_ (p ^. PersonName ==. val "John")`|`WHERE Person.name = 'John'`|
|`p ^. PersonName`|`Person.name`（字段投影）|
|`val "John"`|`'John'`（Haskell 值提升到 SQL）|
|`orderBy [asc (p ^. PersonName)]`|`ORDER BY Person.name ASC`|
|`limit 10` / `offset 20`|`LIMIT 10 OFFSET 20`|

esqueleto 的设计目标之一就是"容易被翻译成 SQL" 。

### 字段投影：(^.) 与 (?)

esqueleto 的 Hackage 文档详细说明 ：

```
-- (^.) 用于非空字段投影
select $ from $ \p -> do
  where_ (p ^. PersonName ==. val "John")
  return p

-- (?) 用于可空字段投影（LEFT OUTER JOIN 的结果）
select $ from $ \(p `LeftOuterJoin` mb) -> do
  on (just (p ^. PersonId) ==. mb ?. BlogPostAuthorId)
  orderBy [asc (p ^. PersonName), asc (mb ?. BlogPostTitle)]
  return (p, mb)
```

**🔑 洞见十：Maybe 类型的字段需要 just/?. 处理**

由于 `Person.age` 是 `Maybe Int`，在 SQL 表达式中需要用 `just` 提升：

```
-- age 是 Maybe Int，val 18 是 Int
-- 必须用 just (val 18) 提升为 SqlExpr (Value (Maybe Int))
select $ from $ \p -> do
  where_ (p ^. PersonAge >=. just (val 18))
  return p
-- 生成: SELECT * FROM Person WHERE Person.age >= 18
```

### JOIN：esqueleto 的真正威力

esqueleto 的 Hackage 文档展示了三种 JOIN ：

**1. 隐式 JOIN（逗号分隔的元组）**​ ：

```
select $ from $ \(b, p) -> do
  where_ (b ^. BlogPostAuthorId ==. p ^. PersonId)
  orderBy [asc (b ^. BlogPostTitle)]
  return (b, p)
-- 生成: 
-- SELECT BlogPost.*, Person.* 
-- FROM BlogPost, Person 
-- WHERE BlogPost.authorId = Person.id 
-- ORDER BY BlogPost.title ASC
```

**2. LEFT OUTER JOIN**​ ：

```
select $ from $ \(p `LeftOuterJoin` mb) -> do
  on (just (p ^. PersonId) ==. mb ?. BlogPostAuthorId)
  orderBy [asc (p ^. PersonName), asc (mb ?. BlogPostTitle)]
  return (p, mb)
-- 生成:
-- SELECT Person.*, BlogPost.*
-- FROM Person LEFT OUTER JOIN BlogPost ON Person.id = BlogPost.authorId
-- ORDER BY Person.name ASC, BlogPost.title ASC
```

**3. 多表 JOIN**​ ：

```
select $ from $ \(p1 `InnerJoin` f `InnerJoin` p2) -> do
  on (p2 ^. PersonId ==. f ^. FollowFollowed)
  on (p1 ^. PersonId ==. f ^. FollowFollower)
  return (p1, f, p2)
-- 注意 on 子句的顺序是反向的！
```

**🔑 洞见十一：JOIN 的字段投影保证编译期类型安全**

`b ^. BlogPostAuthorId` 中的 `BlogPostAuthorId` 是 persistent 生成的字段类型——如果 `b` 不是 `BlogPost` 实体，编译器立即报错。这是 esqueleto 相比原始 SQL 的最大优势：**JOIN 的字段引用在编译期验证**。

### Experimental 语法：更现代的类型安全

esqueleto 3.6+ 引入了 Experimental 模块 ，采用 `table @Entity` 和 `:&` 模式匹配：

```
-- 导入 Experimental 语法
import Database.Esqueleto.Experimental

-- INNER JOIN
select $ do
  (people :& blogPosts) <- from $ table @Person
    `innerJoin` table @BlogPost
    `on` (\(people :& blogPosts) -> 
            people ^. PersonId ==. blogPosts ^. BlogPostAuthorId)
  where_ (people ^. PersonAge >. just (val 18))
  pure (people, blogPosts)

-- LEFT OUTER JOIN
select $ do
  (people :& blogPosts) <- from $ table @Person
    `leftJoin` table @BlogPost
    `on` (\(people :& blogPosts) -> 
            just (people ^. PersonId) ==. blogPosts ?. BlogPostAuthorId)
  where_ (people ^. PersonAge >. just (val 18))
  pure (people, blogPosts)
```

DeepWiki 文档指出 ：

> _"Experimental Syntax (Database.Esqueleto.Experimental)... Type-safe JOINs... Advanced Features: Common Table Expressions, Set operations (UNION, EXCEPT), Lateral joins, Type-safe aliasing."_

**🔑 洞见十二：Experimental 语法是 esqueleto 的未来**

DeepWiki 文档的架构说明 ：

> _"Future (v4.0.0.0): Experimental becomes default. Legacy syntax deprecated."_

新项目应该直接导入 `Database.Esqueleto.Experimental`，获得更好的类型安全和更丰富的功能（CTE、UNION、LATERAL JOIN 等）。

### UPDATE 和 DELETE

esqueleto 的 Hackage 文档 ：

```
-- UPDATE
update $ \p -> do
  set p [PersonName =. val "João"]
  where_ (p ^. PersonName ==. val "Joao")

-- DELETE
delete $ from $ \p -> do
  where_ (p ^. PersonAge <. just (val 18))
```

### 实际应用：博客系统的复杂查询

基于 esqueleto 示例项目 ：

```
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DerivingStrategies #-}

import Database.Esqueleto
import Database.Esqueleto.Experimental  -- 使用新版语法
import Database.Persist
import Database.Persist.Postgresql
import Database.Persist.TH
import Control.Monad.IO.Class (liftIO)

-- 1. 实体定义
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Person
    name String
    age Int Maybe
    deriving Show

BlogPost
    title String
    authorId PersonId
    deriving Show

Follow
    follower PersonId
    followed PersonId
    deriving Show
|]

-- 2. 查询：获取所有名为 "John" 的人
getJohns :: SqlPersistT IO [Entity Person]
getJohns = select $ from $ \p -> do
  where_ (p ^. PersonName ==. val "John")
  return p

-- 3. 查询：博客文章及其作者（INNER JOIN）
getBlogPostsByAuthors :: SqlPersistT IO [(Entity BlogPost, Entity Person)]
getBlogPostsByAuthors = select $ from $ \(b, p) -> do
  where_ (b ^. BlogPostAuthorId ==. p ^. PersonId)
  orderBy [asc (b ^. BlogPostTitle)]
  return (b, p)

-- 4. 查询：所有人及其博客文章（LEFT OUTER JOIN）
getAuthorMaybePosts :: SqlPersistT IO [(Entity Person, Maybe (Entity BlogPost))]
getAuthorMaybePosts = select $ from $ \(p `LeftOuterJoin` mb) -> do
  on (just (p ^. PersonId) ==. mb ?. BlogPostAuthorId)
  orderBy [asc (p ^. PersonName), asc (mb ?. BlogPostTitle)]
  return (p, mb)

-- 5. 查询：互相关注关系（三表 JOIN）
followers :: SqlPersistT IO [(Entity Person, Entity Follow, Entity Person)]
followers = select $ from $ \(p1 `InnerJoin` f `InnerJoin` p2) -> do
  on (p2 ^. PersonId ==. f ^. FollowFollowed)
  on (p1 ^. PersonId ==. f ^. FollowFollower)
  where_ (p1 ^. PersonName ==. val "John")
  return (p1, f, p2)

-- 6. 查询：成年人（处理 Maybe 字段）
getAdults :: SqlPersistT IO [Entity Person]
getAdults = select $ from $ \p -> do
  where_ (p ^. PersonAge >=. just (val 18))
  return p

-- 7. 更新：改名
updateJoao :: SqlPersistT IO ()
updateJoao = update $ \p -> do
  set p [PersonName =. val "João"]
  where_ (p ^. PersonName ==. val "Joao")

-- 8. 删除：删除年轻人
deleteYoungsters :: SqlPersistT IO ()
deleteYoungsters = delete $ from $ \p -> do
  where_ (p ^. PersonAge <. just (val 18))

-- 9. 聚合查询：每个人的文章数
postsPerAuthor :: SqlPersistT IO [(Entity Person, Value Int)]
postsPerAuthor = select $ from $ \(p `InnerJoin` bp) -> do
  on (p ^. PersonId ==. bp ^. BlogPostAuthorId)
  groupBy (p ^. PersonId)
  let postCount = count (bp ^. BlogPostId)
  orderBy [desc postCount]
  return (p, postCount)

-- 10. 子查询：发布超过 10 篇文章的人
prolificAuthors :: SqlPersistT IO [Entity Person]
prolificAuthors = select $ from $ \p -> do
  where_ (p ^. PersonId `in_` 
    subSelect (from $ \bp -> do
      where_ (bp ^. BlogPostAuthorId ==. p ^. PersonId)
      groupBy (bp ^. BlogPostAuthorId)
      having ((count $ bp ^. BlogPostId) >. val 10)
      return (bp ^. BlogPostAuthorId)))
  return p
```

---

## 🧭 本章在知识体系中的位置

```
第二十四章: 单子变换器（ReaderT/StateT/IO）
   ↓
第二十七章: 网络编程（Web 服务）
   ↓
第二十八章: Haskell 与数据库  ← 你在这里
   ↓
┌──────────────────────────────────────────┐
│ esqueleto: 类型安全的 SQL EDSL          │
│   ↑ 在 persistent 实体之上做 JOIN       │
│   │                                     │
│ persistent: 实体定义 + CRUD + 迁移      │
│   ↑ SqlPersistT = ReaderT SqlBackend   │
│   │                                     │
│ SqlBackend: SQLite/PostgreSQL/MySQL    │
└──────────────────────────────────────────┘
```

**本章给你两件行李**：

1. **persistent 是类型安全的数据库 ORM**：用 quasiquote 定义实体，自动生成 Haskell 数据类型、字段访问器、键类型和迁移函数；通过分层类型类（`PersistStore`/`PersistQuery`/`PersistUnique`）提供基础 CRUD；`SqlPersistT` 单子把数据库连接作为"只读环境"注入
    
2. **esqueleto 是类型安全的 SQL EDSL**：在 persistent 实体之上写 SELECT/UPDATE/INSERT/DELETE，尤其是 JOIN；字段投影 `(^.)` 和 `(=?)` 在编译期验证字段存在性和类型匹配；Experimental 语法提供更强的类型安全和 CTE/UNION/LATERAL JOIN 等高级功能
    

---

## 📝 本章核心洞见总结

> **洞见一：persistent 与 esqueleto 是互补而非竞争**。
> 
> persistent 负责实体定义、自动迁移、基础 CRUD；esqueleto 负责类型安全的复杂 SQL 查询，尤其是 JOIN 。**esqueleto 不需要修改 persistent 后端，两者无缝协作**​ 。

> **洞见二：persistent 的 quasiquote 自动生成完整类型基础设施**。
> 
> `share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase| ... ]` 生成：Haskell 数据类型、字段访问器、键类型（`UserId`）、`EntityField` 类型、类型类实例、迁移函数 。**这让你写的 Haskell 代码与数据库表结构始终保持同步**。

> **洞见三：Entity 类型是"键 + 值"的组合**。
> 
> `Entity User = Entity (Key User) User` ——把"纯数据"（`User` 值）和"持久化数据"（数据库行）在类型层面清晰区分 。**这是 Haskell 类型安全哲学的延伸**。

> **洞见四：persistent 的查询操作符在编译期验证类型**。
> 
> `(==.)`, `(>=.)`, `(=.)` 等操作符要求字段类型与值类型匹配 。**错误的类型会导致编译错误，而非运行时 SQL 异常**：`selectList [UserAge ==. "18"] []` 会因 Int vs String 类型不匹配而编译失败 。

> **洞见五：persistent 的类型类层次体现"能力最小化"**。
> 
> `PersistStoreRead`/`PersistStoreWrite`/`PersistUniqueRead`/`PersistUniqueWrite`/`PersistQueryRead`/`PersistQueryWrite` 各自提供特定能力 。**你可以写只依赖 `PersistStoreRead` 的纯函数，编译器强制保证该函数不会修改数据库**。

> **洞见六：SqlPersistT 是第二十四章单子变换器思想的体现**。
> 
> `type SqlPersistT m a = ReaderT SqlBackend m a` ——把数据库连接作为"只读环境"注入 。**在 Web 框架中，`SqlPersistT` 被嵌入更大的变换器栈（如 `Handler`），用 `runDB` 运行数据库动作**​ 。

> **洞见七：自动迁移是 persistent 的杀手锏**。
> 
> 修改 Haskell 实体定义 → persistent 自动检测差异 → 生成相应的 ALTER TABLE 语句 → 保留现有数据 。**增加字段是安全的，类型变更会做兼容性检查**​ 。

> **洞见八：esqueleto 填补了 persistent 的 JOIN 空白**。
> 
> Yesod 书明确指出："Persistent strives to be backend-agnostic... the biggest casualty is SQL join support" 。**esqueleto 在 persistent 实体之上提供类型安全的 JOIN**——这是 Haskell 生态中处理关系查询的标准方案 。

> **洞见九：esqueleto 的语法几乎直译 SQL**。
> 
> `from` → FROM, `where_` → WHERE, `p ^. PersonName` → `Person.name`, `val "John"` → `'John'` 。**设计目标："容易翻译成 SQL"**​ ——你写的 esqueleto 查询与生成的 SQL 几乎一一对应 。

> **洞见十：Maybe 字段需要 just/?. 处理**。
> 
> 由于 `Person.age` 是 `Maybe Int`，在 SQL 表达式中必须用 `just (val 18)` 提升为 `SqlExpr (Value (Maybe Int))` 。**LEFT OUTER JOIN 的结果用 `(?.)` 投影，因为右侧实体可能为 `Nothing`**​ 。

> **洞见十一：JOIN 的字段投影保证编译期类型安全**。
> 
> `b ^. BlogPostAuthorId` 中的 `BlogPostAuthorId` 是 persistent 生成的字段类型——如果 `b` 不是 `BlogPost` 实体，编译器立即报错 。**这是 esqueleto 相比原始 SQL 的最大优势**。

> **洞见十二：Experimental 语法是 esqueleto 的未来**。
> 
> `Database.Esqueleto.Experimental` 采用 `table @Entity` 和 `:&` 模式匹配，提供更好的类型安全 。**支持 CTE、UNION、LATERAL JOIN 等高级功能**​ 。DeepWiki 文档指出："Future (v4.0.0.0): Experimental becomes default. Legacy syntax deprecated" ——新项目应直接使用 Experimental 语法。

> **洞见十三：persistent + esqueleto 构成完整的数据库层**。
> 
> 实体定义用 persistent 的 quasiquote，基础 CRUD 用 persistent 的 `insert`/`get`/`selectList`/`update`/`delete`，复杂查询（尤其是 JOIN）用 esqueleto 。**两者共享同一套实体定义和后端**——这是 Haskell 数据库编程的标准组合。

---

## 🛠️ 实践建议

1. **实体定义的最佳实践**：
    
    ```
    {-# LANGUAGE QuasiQuotes #-}
    {-# LANGUAGE TemplateHaskell #-}
    {-# LANGUAGE TypeFamilies #-}
    {-# LANGUAGE GADTs #-}
    {-# LANGUAGE FlexibleContexts #-}
    {-# LANGUAGE OverloadedStrings #-}
    {-# LANGUAGE DerivingStrategies #-}
    
    import Database.Persist
    import Database.Persist.Postgresql
    import Database.Persist.TH
    
    -- 用 persistLowerCase 获得一致的命名
    share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
    User
        name String
        email String
        age Int Maybe
        UniqueEmail email
        deriving Show Eq
    
    BlogPost
        title String
        content Text
        authorId UserId
        published Bool default=True
        createdAt UTCTime
        deriving Show Eq
    |]
    ```
    
2. **用 SqlPersistT 封装数据库操作**：
    
    ```
    import Database.Persist.Postgresql (ConnectionString, withPostgresqlPool)
    import Database.Persist.Sql (runSqlPool)
    import Control.Monad.Logger (runStdoutLoggingT)
    import Control.Monad.Reader (runReaderT)
    
    type DB a = SqlPersistT IO a
    
    -- 运行数据库动作
    runDB :: ConnectionString -> DB a -> IO a
    runDB connStr action = 
      runStdoutLoggingT $ withPostgresqlPool connStr 10 $ \pool ->
        runSqlPool action pool
    
    -- 事务
    runTransaction :: DB a -> DB a
    runTransaction = transaction
    ```
    
3. **基础 CRUD 操作**：
    
    ```
    -- 创建
    createUser :: String -> Maybe Int -> DB (Key User)
    createUser name age = insert $ User name age
    
    -- 读取
    getUser :: Key User -> DB (Maybe User)
    getUser = get
    
    -- 条件查询
    getAdults :: DB [Entity User]
    getAdults = selectList [UserAge >=. Just 18] [Asc UserName]
    
    -- 更新
    updateUserAge :: Key User -> Int -> DB ()
    updateUserAge userId newAge = update userId [UserAge =. Just newAge]
    
    -- 条件更新
    incrementAllAges :: DB ()
    incrementAllAges = updateWhere [] [UserAge +=. 1]
    
    -- 删除
    deleteUser :: Key User -> DB ()
    deleteUser = delete
    
    -- 条件删除
    deleteInactiveUsers :: DB ()
    deleteInactiveUsers = deleteWhere [UserAge ==. Nothing]
    ```
    
4. **用 esqueleto 写类型安全的 JOIN**：
    
    ```
    {-# LANGUAGE QuasiQuotes #-}
    {-# LANGUAGE OverloadedStrings #-}
    import Database.Esqueleto
    import Database.Esqueleto.Experimental
    
    -- 博客文章及其作者
    getPostsWithAuthors :: DB [(Entity BlogPost, Entity User)]
    getPostsWithAuthors = select $ from $ \(bp, u) -> do
      where_ (bp ^. BlogPostAuthorId ==. u ^. UserId)
      orderBy [desc (bp ^. BlogPostCreatedAt)]
      limit 10
      return (bp, u)
    
    -- 用户及其文章数（LEFT JOIN + 聚合）
    getUsersWithPostCount :: DB [(Entity User, Value Int)]
    getUsersWithPostCount = select $ from $ \(u `LeftOuterJoin` bp) -> do
      on (just (u ^. UserId) ==. bp ?. BlogPostAuthorId)
      groupBy (u ^. UserId)
      let postCount = count (bp ^. BlogPostId)
      orderBy [desc postCount]
      return (u, postCount)
    
    -- 三表 JOIN：互相关注
    getMutualFollowers :: Key User -> DB [(Entity User)]
    getMutualFollowers userId = select $ from $ \(p1 `InnerJoin` f1 
                                                 `InnerJoin` p2 `InnerJoin` f2) -> do
      on (f2 ^. FollowFollower ==. p2 ^. UserId)
      on (p2 ^. UserId ==. f1 ^. FollowFollowed)
      on (f1 ^. FollowFollower ==. p1 ^. UserId)
      where_ (p1 ^. UserId ==. val userId)
      where_ (p2 ^. UserId `in_` 
        subSelect (from $ \f -> do
          where_ (f ^. FollowFollowed ==. val userId)
          return (f ^. FollowFollower)))
      return p2
    ```
    
5. **在 Web 服务中集成数据库**：
    
    ```
    {-# LANGUAGE OverloadedStrings #-}
    import Network.Wai
    import Network.Wai.Handler.Warp (run)
    import Database.Persist.Postgresql
    import Database.Persist.TH
    
    -- 应用状态包含数据库连接池
    data AppState = AppState
      { appPool :: ConnectionPool
      , appConfig :: AppConfig
      }
    
    -- 应用单子
    type App a = ReaderT AppState IO a
    
    -- 运行数据库动作
    runDB :: SqlPersistT IO a -> App a
    runDB action = do
      pool <- asks appPool
      liftIO $ runSqlPool action pool
    
    -- Web 处理器
    getUserHandler :: Key User -> App Response
    getUserHandler userId = do
      mUser <- runDB $ get userId
      case mUser of
        Nothing -> return $ responseLBS status404 [] "User not found"
        Just user -> return $ responseLBS status200 
          [("Content-Type", "application/json")] 
          (encode user)
    ```
    
6. **迁移策略**：
    
    ```
    -- 开发环境：自动迁移
    migrateDev :: DB ()
    migrateDev = runMigration migrateAll
    
    -- 生产环境：手动控制迁移
    -- 1. 检查迁移差异
    -- 2. 生成迁移脚本
    -- 3. 审查并执行
    
    -- 用 persistent 的 migrateVersions 管理迁移历史
    ```
    

---

## 🔮 承上启下

第二十八章是整《Haskell 函数式编程入门》"生产级编程"部分的压轴之作。它把前面所有章节的知识熔于一炉：

- **第二十四章 单子变换器**：`SqlPersistT = ReaderT SqlBackend` 是单子变换器思想的体现
    
- **第二十五章 升格操作**：用 `MonadIO`、`liftIO` 在数据库单子中执行 IO
    
- **第二十六章 高效字符串**：实体的字段类型用 `Text`（来自 `text` 包）
    
- **第二十七章 网络编程**：Web 服务用 `runDB` 把 `SqlPersistT` 嵌入 Handler 单子
    

**第二十八章是你理解"Haskell 生产级数据层"的终极拼图**。吃透"persistent 的实体定义与 CRUD"、"esqueleto 的类型安全 SQL 查询"、"SqlPersistT 单子"、"自动迁移"，你就拥有了用 Haskell 构建真实数据库应用的完整能力。从这一刻起，你写的每一个 Haskell 数据库程序都不会再是"字符串拼接 SQL"——你会本能地用 persistent 定义实体，用 esqueleto 写类型安全的 JOIN，用 `SqlPersistT` 管理数据库事务。

> 💡 学习建议：读完本章后，做三个实验：
> 
> ```
> -- 1. 用 persistent 定义实体并做 CRUD
> {-# LANGUAGE QuasiQuotes #-}
> {-# LANGUAGE TemplateHaskell #-}
> import Database.Persist
> import Database.Persist.Sqlite
> import Database.Persist.TH
> 
> share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
> User
>     name String
>     age Int Maybe
>     deriving Show
> |]
> 
> main :: IO ()
> main = runSqlite ":memory:" $ do
>   runMigration migrateAll
>   userId <- insert $ User "Alice" (Just 30)
>   mUser <- get userId
>   liftIO $ print mUser
> 
> -- 2. 用 esqueleto 写类型安全的 JOIN
> import Database.Esqueleto
> import Database.Esqueleto.Experimental
> 
> getUsersWithPosts :: SqlPersistT IO [(Entity User, Entity BlogPost)]
> getUsersWithPosts = select $ from $ \(u, bp) -> do
>   where_ (bp ^. BlogPostAuthorId ==. u ^. UserId)
>   return (u, bp)
> 
> -- 3. 在 Web 服务中集成数据库
> -- 参考第二十七章的 WAI 应用，用 runDB 嵌入 SqlPersistT
> ```
> 
> 亲眼看见"persistent 的实体生成"、"esqueleto 的类型安全 JOIN"、"SqlPersistT 的事务管理"，是理解本章精髓的最佳方式。
> 
> 然后尝试构建一个完整的博客系统：用 persistent 定义 User/BlogPost/Comment 实体，用 esqueleto 写"获取用户及其所有博客文章"、"获取文章及其评论"、"获取互相关注的用户"等复杂查询，用 WAI 暴露 REST API。你会亲身体验到——**Haskell 的数据库层把"编译器即验证器"的理念推到了极致**：字段名拼写错误、类型不匹配、JOIN 字段不存在——所有这些错误都在编译期被捕获，而非在生产环境的 SQL 异常中爆发。记住 esqueleto Hackage 文档的智慧：**"Esqueleto is the recommended library for type-safe JOINs on persistent SQL backends. (The alternative is using raw SQL, but that's error prone and does not offer any composability.)"**——类型安全的 SQL 不仅是便利，更是 Haskell 工程化的必然选择。

