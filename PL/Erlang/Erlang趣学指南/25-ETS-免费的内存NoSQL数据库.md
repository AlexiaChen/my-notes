
# 第二十五章「ETS：免费的内存 NoSQL」读书笔记：当"进程状态"不够用时

前二十四章你走完了 Erlang/OTP 的完整抽象链：从不可变变量到并发原语，从 gen_server 到 supervisor，从 application 到 release，从套接字到 EUnit。但 Fred 在第二十五章"Bears, ETS, Beets: In-Memory NoSQL for Free!"开篇就抛出了一个此前刻意回避的尖锐问题：

> **"如果我们需要在许多进程之间共享状态怎么办？"**​

这个问题直击 Erlang 并发模型的"阿喀琉斯之踵"——第一章我们学到"变量不可变"，第十章学到"每个进程有自己的状态"，第十四章学到"gen_server 封装单一状态"。这些抽象都建立在"状态属于进程私有"的前提上。但现实世界里：

- **缓存**：几千个进程都需要查同一份配置
    
- **会话表**：连接进程死掉，用户的 session 不能丢
    
- **计数器**：全局请求数、限流器
    
- **路由表**：节点间路由信息需要共享
    

如果每次共享都用 `gen_server:call/2` 去问一个中心进程，那个中心进程会瞬间成为**瓶颈和单点故障**——这正是第十一章学到的"调用方阻塞"问题的规模化版本。

第二十五章要解决的根本问题是——

> **如何在"进程私有状态"和"全局共享状态"之间，找到一个高性能、Erlang 原生、符合"let it crash"哲学的平衡点？**

Fred 的答案是 **ETS（Erlang Term Storage）**——OTP 内置的内存 NoSQL 数据库，免费、高速、与进程模型深度融合 。

---

## 🎯 小节一：Why ETS —— 为什么需要"进程外的共享内存"

### 三个选项的权衡

Fred 在章节里直面这个设计困境：

|方案|优点|缺点|
|---|---|---|
|**gen_server 持有状态**​|符合 OTP 模式、易监督|所有读写都过单进程 → 瓶颈|
|**进程字典（process dictionary）**​|快|仅当前进程可见、破坏函数纯洁性|
|**ETS 表**​|内存级速度、多进程共享、与 BEAM 深度集成|不服从 GC、需手动管理生命周期|

📌 **洞见一：ETS 是"进程状态"与"全局数据库"之间的中间地带**

ETS 的核心定位 ：

> **"ETS 是 OTP 内置的、强大的、健壮的内存存储引擎，能够存储大量数据，并提供常量时间的数据访问。"**​

**关键特性**：

- **内存存储**：数据驻留 RAM，读写是 O(1) 或 O(log N)
    
- **项式存储**：可以存任意 Erlang term（原子、元组、列表、record、map）
    
- **进程拥有**：每张表由一个进程创建并"拥有"
    
- **跨进程共享**：其他进程可以读写（取决于访问模式）
    
- **内置于 OTP**：无需外部依赖，BEAM 自带
    

> 💡 **"免费"的含义**：ETS 不是外部数据库（如 Redis、Memcached），它是 BEAM 虚拟机内部的存储引擎——零网络开销、零序列化开销、与 Erlang 项无缝兼容。

### ETS 与 BEAM 的关系

```
┌─────────────────────────────────────────────┐
│                BEAM 虚拟机                    │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │           ETS 存储引擎                  │  │
│  │  ┌───────────────────────────────────┐  │  │
│  │  │ Table 1 (set, owned by P1)        │  │  │
│  │  │   {"alice", 42}                   │  │  │
│  │  │   {"bob", 17}                     │  │  │
│  │  └───────────────────────────────────┘  │  │
│  │  ┌───────────────────────────────────┐  │  │
│  │  │ Table 2 (bag, owned by P2)        │  │  │
│  │  │   {conn_1, data1}                 │  │  │
│  │  │   {conn_1, data2}                 │  │  │
│  │  │   {conn_2, data3}                 │  │  │
│  │  └───────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │ Proc A  │  │ Proc B  │  │ Proc C  │     │
│  │ (owner) │  │         │  │         │     │
│  └─────────┘  └─────────┘  └─────────┘     │
└─────────────────────────────────────────────┘
```

**这是 ETS 最反直觉的特性**——**表不属于任何进程的堆，它是 BEAM 内部的全局资源，但"逻辑上"由一个进程拥有**​ 。

---

## 📋 小节二：The Concepts of ETS —— 表类型与访问控制

### 四种表类型

这是 ETS 最关键的选型决策。ETS 官方文档明确定义了四种类型 ：

**1. set（默认）**

```
1> T = ets:new(t, [set]).
2> ets:insert(T, {1, a}).
3> ets:insert(T, {1, b}).     %% 覆盖
4> ets:insert(T, {1.0, c}).   %% 整数 1 和浮点 1.0 被视为不同键
5> ets:lookup(T, 1).
[{1, b}]
```

- 每个键对应**一个**对象
    
- 插入相同键 → **覆盖**
    
- 内部哈希实现，**O(1) 查找**
    
- 键匹配要求完全相同（1 和 1.0 是不同键）
    

**2. ordered_set**

```
1> T = ets:new(t, [ordered_set]).
2> ets:insert(T, {1, a}).
3> ets:insert(T, {1, b}).     %% 覆盖
4> ets:insert(T, {1.0, c}).   %% 整数 1 和浮点 1.0 被视为相等键！
5> ets:lookup(T, 1).
[{1.0, c}]    %% 注意返回的键是 1.0，不是 1
```

- 与 set 类似，但**按键的 Erlang 项顺序排序**
    
- **关键差异**：键的比较用 `==` 而非 `=:=`——**1 和 1.0 被视为相等**​
    
- 内部使用平衡树实现，**O(log N) 查找**
    
- 适合需要有序遍历的场景（如范围查询）
    

**3. bag**

```
1> T = ets:new(t, [bag]).
2> ets:insert(T, {1, a}).
3> ets:insert(T, {1, b}).
4> ets:insert(T, {1, b}).     %% 重复插入相同对象，被忽略
5> ets:lookup(T, 1).
[{1, a}, {1, b}]
```

- 每个键可以对应**多个对象**
    
- 但**同一对象在同一键下只能有一个实例**（对象完全匹配才算重复）
    
- 内部哈希实现
    
- 插入时需要与现有对象比较——**插入效率低于 duplicate_bag**
    

**4. duplicate_bag**

```
1> T = ets:new(t, [duplicate_bag]).
2> ets:insert(T, {1, a}).
3> ets:insert(T, {1, b}).
4> ets:insert(T, {1, b}).     %% 允许重复
5> ets:lookup(T, 1).
[{1, a}, {1, b}, {1, b}]
```

- 每个键可以对应多个对象
    
- **允许完全相同的对象重复存在**
    
- 内部哈希实现
    
- **插入效率最高**——不做去重检查
    

📌 **洞见二：表类型选择的决策树**

|需求|选择|
|---|---|
|一个键一个值，最快查找|`set`|
|一个键一个值，需要有序|`ordered_set`|
|一个键多个值，值不重复|`bag`|
|一个键多个值，值可重复|`duplicate_bag`|
|不确定|`set`（默认，性能最佳）|

### 三种访问控制

ETS 官方文档明确 ：

**1. protected（默认）**

```
%% 默认配置
1> T = ets:new(some_name, []).
2> ets:info(T).
[{read_concurrency, false}, {write_concurrency, false}, 
 {owner, <0.88.0>}, {heir, none}, {name, some_name},
 {size, 0}, {type, set}, {keypos, 1}, 
 {protection, protected}]    %% ← 默认 protected
```

- **拥有者进程**：可读可写
    
- **其他进程**：**只读**
    
- 这是默认设置，也是最常见的生产配置
    

**2. public**

```
1> T = ets:new(t, [public]).
```

- **任何进程**：可读可写
    
- 适合：计数器、全局配置（配合 `update_counter/3` 原子操作）
    

**3. private**

```
1> T = ets:new(t, [private]).
```

- **只有拥有者进程**：可读可写
    
- 其他进程完全不能访问
    
- 等价于"进程私有 + 内存共享"的混合体
    

📌 **洞见三：protected 是"读写分离"的 ETS 表达**

```
protected 表的典型架构：

        ┌─────────────────────────┐
        │   Owner Process         │
        │   (唯一写者)             │
        │                        │
        │   读请求 ──────┐        │
        │   写请求 ──────┤        │
        └───────────────┼────────┘
                        │
              ┌─────────▼─────────┐
              │   ETS Table        │
              │   (protected)     │
              └─────────┬─────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   Reader Process  Reader Process  Reader Process
     (只读)           (只读)           (只读)
```

**这正是"写集中、读分散"的高性能模式**——只有 owner 能写（避免竞态），所有读者可以直接读（避免 gen_server 瓶颈）。

### 关键选项详解

ETS 官方文档给出默认等价于 ：

```
[set, protected, {keypos, 1}, {heir, none}, 
 {write_concurrency, false}, {read_concurrency, false}]
```

**1. `{keypos, Pos}`**

指定元组中第几个元素作为键。默认是第一个元素（`Pos=1`）。

```
%% 存储 record 时很关键
-record(user, {name, age, email}).
%% 如果以 name 为键，需要 {keypos, #user.name}
1> T = ets:new(users, [set, {keypos, #user.name}]).
2> ets:insert(T, #user{name="alice", age=30, email="a@x.com"}).
```

**2. `{heir, Pid, HeirData}`**

设置继承者——当 owner 进程终止时，表的所有权转移给 heir 进程 。

```
1> T = ets:new(t, [set, {heir, HeirPid, "transfer_data"}]).
%% owner 崩溃 → 表自动转移给 HeirPid
%% HeirPid 收到 {'ETS-TRANSFER', T, FromPid, "transfer_data"}
```

**默认 `{heir, none}`**——owner 终止时**表被销毁**​ 。

> ⚠️ **这是 ETS 生产的 #1 陷阱**：如果 owner 进程崩溃，整张表消失。这正是下一节要解决的工程问题。

**3. `{write_concurrency, true|false|auto}`**

- `false`（默认）：写操作获得独占访问，阻塞并发写
    
- `true`：优化并发写——不同对象的写可以并行
    
- `auto`（OTP 25+）：运行时自动调整锁的数量
    

**4. `{read_concurrency, true}`**

优化并发读——读操作不阻塞彼此。

---

## 🔨 小节三：Creating and Deleting Tables —— 表的生命周期

### 创建表

```
1> T = ets:new(my_table, [set, protected, named_table]).
my_table    %% 因为 named_table，返回表名而非标识符
```

**named_table 选项的价值**：可以用名字代替标识符访问 。

```
%% 没有 named_table
1> T = ets:new(anon_table, [set]).
16402    %% 返回一个数字标识符
2> ets:insert(T, {key, val}).
true

%% 有 named_table
1> T = ets:new(named_table, [set, named_table]).
named_table    %% 返回原子名字
2> ets:insert(named_table, {key, val}).  %% 直接用名字
true
```

📌 **洞见四：named_table 是"全局注册"的 ETS 版本**

正如第十一章学的 `register/2` 注册进程名，named_table 让 ETS 表在节点内全局可寻址。这对于"配置表"、"全局计数器"等场景极其方便。

### 删除表

```
1> ets:delete(T).
ok
```

**删除的两种方式**：

1. **显式删除**：`ets:delete(T)`
    
2. **隐式删除**：owner 进程终止 → 表被销毁（除非设置了 heir）
    

> 💡 **工程启示**：ETS 表的生命周期绑定 owner 进程。这是优点（自动清理）也是陷阱（意外丢失）。

---

## 💾 小节四：Inserting and Looking Up Data —— 读写操作

### 插入数据

```
1> T = ets:new(t, [set, named_table]).
t
2> ets:insert(t, {<<"user:1">>, #{name => "Alice", age => 30}}).
true
3> ets:insert(t, {<<"user:2">>, #{name => "Bob", age => 25}}).
true
```

**`ets:insert/2` 返回 `true`**——它总是成功（除非参数错误）。

**`ets:insert_new/2`**——只在键不存在时插入：

```
4> ets:insert_new(t, {<<"user:1">>, #{name => "Alice2"}}).
false    %% user:1 已存在，插入失败
5> ets:insert_new(t, {<<"user:3">>, #{name => "Charlie"}}).
true     %% user:3 不存在，插入成功
```

### 查找数据

```
6> ets:lookup(t, <<"user:1">>).
[#{age => 30, name => "Alice"}]    %% 返回列表！
```

📌 **洞见五：lookup 总是返回列表**

这是 ETS 与"key-value store"的关键差异：

- 即使 set 表每个键只有一个对象，`ets:lookup/2` 仍返回**列表**
    
- 这是因为 bag/duplicate_bag 表一个键可能对应多个对象
    
- 调用方必须处理列表（通常取 `[Obj]` 或 `[]`）
    

### 原子计数器

ETS 提供了原子性的计数器更新 ：

```
1> T = ets:new(counters, [set, public, named_table]).
counters
2> ets:update_counter(counters, request_count, 1).
1
3> ets:update_counter(counters, request_count, 1).
2
4> ets:update_counter(counters, request_count, {2, 10}).  %% 增量 2，下限 10
12
```

**`update_counter/3` 是原子的**——多个进程并发调用不会丢失更新。这是 ETS 解决"竞态条件"的标准武器 ：

```
%% 非原子操作（有竞态）
case ets:lookup(counters, key) of
    [{key, Val}] -> 
        ets:insert(counters, {key, Val + 1});  %% 可能导致丢失更新
    [] ->
        ets:insert(counters, {key, 1})
end.

%% 原子操作（推荐）
ets:update_counter(counters, key, 1).  %% 原子读-改-写
```

---

## 🔍 小节五：Meeting Your Match —— 模式匹配查询

ETS 的强大之处在于**类 SQL 的模式匹配查询**。

### 简单匹配

```
1> ets:match(T, '$1').  %% 匹配所有记录，返回键列表
[[{<<"user:1">>, #{age => 30, name => "Alice"}}],
 [{<<"user:2">>, #{age => 25, name => "Bob"}}]]
 
2> ets:match(T, {<<"user:1">>, '$1'}).  %% 匹配特定键，提取 value
[#{age => 30, name => "Alice"}]

3> ets:match(T, {'_', '$1'}).  %% '_' 匹配任意键
[#{age => 30, name => "Alice"}, #{age => 25, name => "Bob"}]
```

**匹配变量**：

- `$N`：绑定变量（如 `$1`、`$2`）
    
- `_`：通配符，不绑定
    
- 具体值：精确匹配
    

### 高级查询：select

```
%% 查找所有 age > 25 的用户
1> MatchSpec = [{{'$1', #{age => '$2', name => '$3'}}, 
                 [{'>=', '$2', 25}], 
                 ['$1', '$3']}].
2> ets:select(T, MatchSpec).
[<<"user:1">>, "Alice"]
```

📌 **洞见六：MatchSpec 是"ETS 的 SQL"**

MatchSpec 的结构：`[{Pattern, [Guard], [Result]}]`

- **Pattern**：匹配模板
    
- **Guard**：过滤条件
    
- **Result**：返回什么
    

**这是 ETS 最强大的特性**——它让你在 BEAM 内部执行"类查询"操作，而无需把数据传到 Erlang 进程里过滤。

---

## 💾 小节六：DETS —— 磁盘版 ETS

Fred 在章节里提到了 DETS（Disk ETS）：

> **"DETS 是 ETS 的磁盘版本，数据持久化到文件。"**

```
1> {ok, T} = dets:open_file(my_dets, [{type, set}, {file, "data.dets"}]).
{ok, my_dets}
2> dets:insert(my_dets, {key, value}).
ok
3> dets:lookup(my_dets, key).
[{key, value}]
4> dets:close(my_dets).
ok
```

**ETS vs DETS**：

|维度|ETS|DETS|
|---|---|---|
|存储介质|内存|磁盘文件|
|速度|极快（O(1)）|较慢（磁盘 I/O）|
|容量|受限于内存|受限于磁盘|
|持久化|否（owner 死则表亡）|是（文件持久）|
|表类型|set/bag/duplicate_bag/ordered_set|set/bag/duplicate_bag（**无 ordered_set**）|
|用途|缓存、共享状态|持久化存储、Mnesia 后端|

> 💡 **Mnesia 使用 DETS 作为磁盘存储后端**——这是 ETS/DETS 在工业级系统中的典型协作模式。

---

## 🏗️ 小节七：Implementation Details —— ETS 与 OTP 监督的集成

这是本章最关键的工程洞见，也是 Fred 隐含传达的设计纪律。

### 生产陷阱：Owner 进程崩溃

```
%% 反模式：在临时进程中创建 ETS 表
bad_pattern() ->
    T = ets:new(temp_table, [named_table, set]),
    ets:insert(T, {key, val}),
    %% 函数返回 → 当前进程继续运行，但...
    %% 如果这个进程是短暂的，表会随之消失
    T.
```

**问题**：ETS 表由创建它的进程拥有。如果该进程终止，表被销毁（除非设置了 heir）。

### 正确模式： supervised ETS owner

```
-module(my_ets_owner).
-behaviour(gen_server).
-export([start_link/0, init/1, handle_call/3, handle_cast/2, 
         handle_info/2, terminate/2, code_change/3]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% 在 gen_server 的 init 中创建表
    %% 表的 owner 是这个 gen_server 进程
    T = ets:new(user_cache, 
                [set, protected, named_table, 
                 {read_concurrency, true}]),
    %% 设置 heir 为 self（虽然没必要，因为 supervisor 会重启）
    {ok, #{table => T}}.

%% 客户端 API
lookup(UserKey) ->
    case ets:lookup(user_cache, UserKey) of
        [{UserKey, Val}] -> {ok, Val};
        [] -> not_found
    end.

insert(UserKey, Val) ->
    ets:insert(user_cache, {UserKey, Val}).
```

**监督树集成**：

```
-module(my_app_sup).
-behaviour(supervisor).

init([]) ->
    SupFlags = #{strategy => one_for_one, intensity => 3, period => 10},
    ChildSpecs = [
        #{id => ets_owner,
          start => {my_ets_owner, start_link, []},
          restart => permanent,
          shutdown => 5000,
          type => worker,
          modules => [my_ets_owner]}
    ],
    {ok, {SupFlags, ChildSpecs}}.
```

📌 **洞见七：ETS 表的生命周期 = Owner 进程的生命周期 = 监督策略**

```
supervisor (one_for_one, intensity=3)
    │
    ▼
my_ets_owner (gen_server)
    │
    ├── init/1: ets:new(user_cache, [...])
    │
    ├── 进程崩溃 → 表被销毁
    │       │
    │       ▼
    │   supervisor 重启 my_ets_owner
    │       │
    │       ▼
    │   init/1 重新创建表（空表）
    │
    └── 这是预期行为：崩溃 → 重启 → 全新状态
```

**关键工程决策**：

1. **ETS owner 必须是 supervised 进程**——否则表的生命周期不可控
    
2. **表的重建策略**：重启后是空表？还是从 DETS/Mnesia 恢复？
    
3. **public vs protected**：如果表是 public，多个进程可能并发写——需要用 `update_counter/3` 等原子操作
    

### ETS + GenServer 的混合模式

这是 WhatsApp 等系统的典型模式：

```
┌─────────────────────┐
                    │   GenServer         │
                    │   (写操作、失效逻辑) │
                    └──────────┬──────────┘
                               │ 拥有
                               ▼
                    ┌─────────────────────┐
                    │   ETS Table         │
                    │   (protected)       │
                    └──────────┬──────────┘
                               │ 直接读
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        Reader Process   Reader Process   Reader Process
```

**分工**：

- **GenServer**：处理写、失效策略、生命周期管理
    
- **ETS**：处理读密集型访问，多个调用方直接读
    
- **减少邮箱压力**：GenServer 不必处理每个读请求
    

---

## 🌐 全局脉络：ETS 在整本书中的位置

把第二十五章放进全书架构：

```
第一章：不可变 → 状态更新 = 创建新版本
第十章：并发原语 → 进程私有状态
第十四章：gen_server → 单一状态的服务
第十七章：supervisor → 容错骨架
第二十四章：EUnit → 正确性验证
第二十五章：ETS ← 你在这里
   ↓ 进程间共享内存、高性能 NoSQL
第二十六章及之后：分布式 Erlang
```

### 与前二十四章的深度连接

**与第一章（不可变）的连接**：

- ETS 表是**可变的**——插入相同键会覆盖
    
- 这是 Erlang "不可变"规则的例外，但是受控的例外
    
- ETS 操作是"原子"的——读-改-写要么完整执行，要么不执行
    

**与第十章（并发原语）的连接**：

- ETS 表由进程创建（`ets:new/2` 内部涉及进程通信）
    
- 表的访问控制类似于"进程权限"
    
- `ets:give_away/3` 可以转移表的所有权
    

**与第十一章（深入多重处理）的连接**：

- `ets:lookup/2` 返回列表——需要模式匹配处理结果
    
- MatchSpec 使用 `$1` 等变量——与 `receive` 的模式匹配同源
    
- 超时控制在 `ets:select/3` 中体现
    

**与第十四章（gen_server）的连接**：

|维度|gen_server 状态|ETS 表|
|---|---|---|
|存储位置|进程堆|BEAM 全局内存|
|访问方式|`gen_server:call/2`|`ets:lookup/2`|
|并发性|串行（单进程）|并行（多进程直读）|
|生命周期|进程存活期间|owner 进程存活期间|
|监督|由 supervisor 监督|owner 由 supervisor 监督|

**ETS 不是 gen_server 的替代品，是补充**——gen_server 管理"需要串行化的写操作"，ETS 承载"需要并行化的读操作"。

**与第二十章（分布式应用）的连接**：

- ETS 表是**节点本地**的——不跨节点共享
    
- 分布式场景需要 Mnesia（基于 ETS + DETS）
    
- 或每个节点维护自己的 ETS 副本
    

**与第二十二章（热升级）的连接**：

- ETS 表中的数据不参与代码热升级
    
- 升级时需要考虑：表数据如何迁移？
    
- 通常策略：升级前持久化到 DETS，升级后恢复
    

### 与工业级系统的连接

**WhatsApp 的 ETS 使用**（推测）：

```
WhatsApp 节点
    │
    ├── 连接状态表（ETS, set, public）
    │   ├── 键：UserID
    │   ├── 值：ConnectionPid, LastSeen, ...
    │   └── 用途：路由消息到正确连接
    │
    ├── 用户会话表（ETS, set, protected）
    │   ├── 键：SessionToken
    │   ├── 值：UserId, CreatedAt, ...
    │   └── 用途：鉴权、会话管理
    │
    ├── 消息计数器（ETS, set, public）
    │   ├── ets:update_counter/3 原子递增
    │   └── 用途：限流、统计
    │
    └── 配置缓存（ETS, bag, protected）
        ├── 键：ConfigKey
        ├── 值：多个配置项
        └── 用途：避免频繁读取远端配置
```

**35 人支撑 9 亿用户的 ETS 基础**：

- 每个节点的连接状态用 ETS 存储
    
- 多进程并发读 ETS（protected 模式）
    
- 只有 owner 进程（gen_server）写 ETS
    
- 避免所有读请求都过中心 gen_server
    
- **性能**：内存级 O(1) 查找，零网络开销
    

**RabbitMQ 的 ETS 使用**：

- 队列元数据用 ETS 存储
    
- 消息路由表用 ETS
    
- 节点内共享，跨节点用 Mnesia
    

---

## ✍️ 读书笔记的核心收获

1. **ETS 的定位**：OTP 内置的内存 NoSQL，免费、高速、与 BEAM 深度集成
    
2. **为什么需要 ETS**：gen_server 中心化状态是瓶颈；ETS 提供进程间共享内存
    
3. **四种表类型**：
    
    - **set**：一键一值，覆盖插入，O(1)，默认
        
    - **ordered_set**：一键一值，按键排序，O(log N)，1 和 1.0 视为相等
        
    - **bag**：一键多值，值不重复
        
    - **duplicate_bag**：一键多值，值可重复，插入最快
        
    
4. **三种访问控制**：
    
    - **protected**（默认）：owner 读写，其他进程只读
        
    - **public**：任何进程读写
        
    - **private**：仅 owner 读写
        
    
5. **默认选项等价于**：`[set, protected, {keypos,1}, {heir,none}, {write_concurrency,false}, {read_concurrency,false}]`
    
6. **关键选项**：
    
    - `{keypos, Pos}`：指定键位置（record 必备）
        
    - `{heir, Pid, Data}`：owner 死后表转移给 heir
        
    - `{write_concurrency, true}`：优化并发写
        
    - `{read_concurrency, true}`：优化并发读
        
    - `named_table`：用名字而非标识符访问
        
    
7. **表的生命周期绑定 owner 进程**：owner 终止 → 表销毁（除非设置 heir）
    
8. **lookup 总是返回列表**：即使 set 表也返回 `[Obj]` 或 `[]`
    
9. **原子计数器**：`ets:update_counter/3` 解决竞态条件
    
10. **模式匹配查询**：
    
    - `ets:match/2`：简单模式匹配
        
    - `ets:select/2`：MatchSpec，类 SQL 查询
        
    
11. **DETS**：磁盘版 ETS，持久化存储，Mnesia 的后端
    
12. **生产陷阱**：owner 进程崩溃 → 表消失
    
13. **正确模式**：在 supervised gen_server 的 init/1 中创建 ETS 表
    
14. **ETS + GenServer 混合架构**：
    
    - GenServer 处理写、失效、生命周期
        
    - ETS 处理读密集型访问
        
    - 减少 GenServer 邮箱压力
        
    
15. **ETS 是"内存加速层"，不是"数据源"**：数据持久化仍需 DETS/Mnesia/外部数据库
    

> 💡 **贯穿全书的隐喻再进化**：Erlang 的状态管理体系可以总结为——
> 
> **"用进程封装私有状态，用 ETS 封装共享状态，用 DETS/Mnesia 封装持久状态。"**
> 
> - **进程堆**：私有、不可变、随进程生死
>     
> - **ETS 表**：共享、可变、随 owner 进程生死、内存级速度
>     
> - **DETS 文件**：共享、可变、持久化、磁盘速度
>     
> - **Mnesia**：分布式、事务性、ETS + DETS 的组合
>     
> 
> **第二十五章的真正价值，是让你理解"Erlang 不是只有进程状态"**：
> 
> 1. **第一章~第五章**：变量不可变，状态更新 = 创建新版本
>     
> 2. **第十章~第十一章**：进程私有状态，通过消息传递共享
>     
> 3. **第十四章~第十六章**：gen_server 封装单一状态的服务
>     
> 4. **第二十五章：ETS ← 你在这里**：进程间共享内存，突破 gen_server 瓶颈
>     
> 5. **后续**：Mnesia 提供分布式、持久化、事务性的数据存储
>     
> 
> **ETS 是"let it crash"哲学在共享状态领域的精致落地**：
> 
> - 表由进程拥有 → 进程崩溃 → 表消失 → 监督者重启 → 表重建
>     
> - 这正是"受控的遗忘"——崩溃后从干净状态重新开始
>     
> - 如果需要持久化，用 DETS 或 Mnesia
>     
> 
> **ETS 的工程智慧**：
> 
> - **protected 模式**：写集中（owner 唯一写者），读分散（多进程并发读）
>     
> - **named_table**：全局注册，简化访问
>     
> - **heir 机制**：owner 崩溃时表的优雅转移
>     
> - **并发选项**：`read_concurrency`/`write_concurrency` 调优
>     
> - **与监督树集成**：ETS owner 必须是 supervised 进程
>     
> 
> **WhatsApp 35 人支撑 9 亿用户的 ETS 基础**：
> 
> - 连接状态表：路由消息到正确连接
>     
> - 会话表：鉴权、会话管理
>     
> - 计数器表：限流、统计（原子操作）
>     
> - 配置缓存：避免频繁读取远端
>     
> - **每个节点维护自己的 ETS 副本**
>     
> - **多进程并发读，单进程（gen_server）写**
>     
> - **owner 进程由 supervisor 监督，崩溃后表重建**
>     
> 
> 正如 Elixir School 的精辟总结 ：
> 
> **"ETS is a robust in-memory store for Elixir and Erlang objects that comes included. ETS is capable of storing large amounts of data and offers constant time data access. Tables in ETS are created and owned by individual processes. When an owner process terminates, its tables are destroyed."**
> 
> **这就是 ETS 的本质**：
> 
> - 它是"免费的"——BEAM 自带，零外部依赖
>     
> - 它是"NoSQL"——列式存储、无 schema、键寻址
>     
> - 它是"内存级"——O(1) 或 O(log N) 访问
>     
> - 它是"进程导向的"——由进程拥有，与"let it crash"完美融合
>     
> 
> 从第一章的不可变变量，到第二十五章的 ETS——**你看到了 Erlang 状态管理的完整谱系**：
> 
> - 不可变变量 → 函数式纯洁性
>     
> - 进程状态 → 并发隔离
>     
> - gen_server 状态 → OTP 行为封装
>     
> - **ETS → 共享内存，突破进程边界**​ ← 你在这里
>     
> - Mnesia → 分布式持久化（后续章节）
>     
> 
> **ETS 是 Erlang 性能工程的秘密武器**——它让你在"函数式纯洁性"和"高性能共享状态"之间找到了平衡点。WhatsApp 的 9 亿用户、RabbitMQ 的百万级消息吞吐、Ejabberd 的千万级并发连接——背后都有 ETS 的身影。
> 
> 正如 Fred 在章节标题"Bears, ETS, Beets"所暗示的幽默——**ETS 就像一只温暖的熊（bear），抱住你的数据（beets），免费提供内存级 NoSQL 服务**。它不是银弹，但是 Erlang 工程师工具箱里最锋利的刀之一。
> 
