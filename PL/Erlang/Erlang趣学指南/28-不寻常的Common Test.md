
# 第二十八章「不寻常的 Common Test」读书笔记：当"单元测试"不够用时

前二十七章你走完了 Erlang/OTP 的完整抽象链——第二十四章学了 EUnit 做单元测试，第二十六、二十七章进入分布式 Erlang 与分布式 OTP 应用。但 Fred 在第二十八章"Unusual Common Test"开篇就点出一个边界：

> **"EUnit 适合单元测试，但当你需要测试'系统'——跨多个应用、跨节点、跨外部目标系统（数据库、FTP、Telnet、SNMP）时，EUnit 力不从心。这时候需要 Common Test。"**​

更精确地说，Erlang 官方文档对 Common Test 的定位是：

> **"Common Test 框架是一个支持实现和自动化执行针对任何类型目标系统的测试用例的工具。它是 Erlang/OTP 系统开发和维护中所有测试和验证活动的主要工具。"**​

第二十八章要解决的根本问题是——

> **如何用 Common Test 组织"套件（Suite）— 用例（Case）— 配置函数（Config Function）— 测试规范（Spec）"的完整体系，来测试集成级、系统级乃至分布式的 Erlang 系统？**

---

## 🎯 小节一：Why Common Test —— 它和 EUnit 的根本分野

### 第二十四章 EUnit 的边界

EUnit 擅长：

- 函数级单元测试
    
- 模块内私有函数测试
    
- 快速反馈、低语法开销
    
- 开发阶段频繁运行
    

EUnit 不擅长：

- 跨应用集成测试
    
- 跨节点分布式测试
    
- 外部系统（数据库、FTP、Telnet、SNMP）测试
    
- 复杂的准备/清理逻辑
    
- 测试报告与日志管理
    

### Common Test 的定位

Erlang 官方文档明确 ：

> **"Common Test 还提供分布式测试模式，具有集中控制和日志。此功能允许在一个公共会话中独立测试多个系统。这在运行自动化大规模回归测试时很有用。"**

📌 **洞见一：Common Test 是"系统测试的操作系统"**

|维度|EUnit|Common Test|
|---|---|---|
|测试粒度|函数级|套件级、用例级、组级|
|测试对象|Erlang 模块|任何目标系统（含外部）|
|准备/清理|有限的 setup/teardown|套件级、组级、用例级三层配置函数|
|分布式|不支持|原生支持，集中控制|
|日志|简单文本|HTML 概览 + 详细日志|
|测试规范|无|.spec 文件详述运行方式|
|外部协议|无|内置 RPC/SNMP/FTP/Telnet 等支持模块|
|适用场景|单元测试|集成测试、系统测试、回归测试|

> 💡 Fred 用"不寻常（Unusual）"形容 Common Test——因为它不像传统 xUnit 框架那样"每个测试是一个函数"，而是一整套**有生命周期、有配置层级、有规范文件**的测试体系。

---

## 📋 小节二：测试套件的结构 —— `*_SUITE.erl` 的契约

### 套件模块的基本形态

Erlang 官方文档严格规定 ：

> **"测试套件是一个普通的 Erlang 模块，其中包含测试用例。建议模块的名称采用 `*_SUITE.erl` 的形式。否则，Common Test 中的目录和自动编译功能将无法找到它（至少默认情况下不能）。还建议在所有测试套件模块中包含 `ct.hrl` 头文件。"**

**最小套件示例**（来自官方文档）：

```
-module(db_data_type_SUITE).
-include_lib("common_test/include/ct.hrl").

%% 测试服务器回调
-export([suite/0, all/0, init_per_suite/1, end_per_suite/1, 
         init_per_testcase/2, end_per_testcase/2]).

%% 测试用例
-export([string/1, integer/1]).

-define(CONNECT_STR, "DSN=sqlserver;UID=alladin;PWD=sesame").

%% COMMON TEST 回调
suite() -> [{timetrap, {minutes, 1}}].

init_per_suite(Config) ->
    {ok, Ref} = db:connect(?CONNECT_STR, []),
    TableName = db_lib:unique_table_name(),
    [{con_ref, Ref}, {table_name, TableName} | Config].

end_per_suite(Config) ->
    Ref = ?config(con_ref, Config),
    db:disconnect(Ref),
    ok.

init_per_testcase(Case, Config) ->
    %% 每个测试用例前的初始化
    [{case_name, Case} | Config].

end_per_testcase(_Case, _Config) ->
    ok.

%% 测试用例
string(Config) ->
    Ref = ?config(con_ref, Config),
    TableName = ?config(table_name, Config),
    %% 测试字符串类型的插入/查询
    ok = db:insert(Ref, TableName, "hello", "world"),
    "world" = db:select(Ref, TableName, "hello"),
    ok.

integer(Config) ->
    Ref = ?config(con_ref, Config),
    TableName = ?config(table_name, Config),
    %% 测试整数类型的插入/查询
    ok = db:insert(Ref, TableName, 42, 100),
    100 = db:select(Ref, TableName, 42),
    ok.
```

### 必导出与选导出

**必须导出**：

- `all/0`：返回该套件中所有测试用例组和测试用例的列表
    

**可选导出（配置函数）**：

- `suite/0`：返回套件属性列表（如 timetrap）
    
- `init_per_suite/1` 和 `end_per_suite/1`：套件级准备/清理
    
- `init_per_testcase/2` 和 `end_per_testcase/2`：用例级准备/清理
    

📌 **洞见二：`all/0` 是套件的"目录"**

```
all() -> 
    [test_case_1, test_case_2, {group, group1}, test_case_3].
```

`all/0` 的返回值告诉 Common Test 执行哪些测试、按什么顺序。可以混用例和组 。

---

## 🔧 小节三：三层配置函数 —— 套件级、组级、用例级

这是 Common Test 最强大的特性，也是它与 EUnit 最显著的区别。

### 第一层：套件级（`init_per_suite/1`, `end_per_suite/1`）

官方文档明确 ：

> **"`init_per_suite/1` 在执行测试用例之前首先调用。它通常包含套件中所有测试用例共用的初始化，这些初始化只需执行一次。`init_per_suite` 建议在被测系统（SUT）或 Common Test 主机节点上设置和验证状态和环境。`end_per_suite/1` 在测试套件执行的最后阶段调用，用于清理。"**

**典型操作**​ ：

- 打开到 SUT 的连接
    
- 初始化数据库
    
- 运行安装脚本
    

**关键语义**：

- `init_per_suite/1` 的参数是 `Config`（键值列表）
    
- 可以修改 `Config` 并作为返回值，传递给所有测试用例
    
- **如果 `init_per_suite/1` 失败，套件中所有测试用例被自动跳过（auto skipped）**​
    

### 第二层：用例级（`init_per_testcase/2`, `end_per_testcase/2`）

> **"如果 `init_per_testcase/2` 存在，它会在套件中每个测试用例之前被调用。"**​

**典型操作**：

- 为每个测试用例准备干净的环境
    
- 重置状态
    
- 记录用例名称
    

### 第三层：组级（Group）

测试用例可以分组，组有执行属性 ：

```
groups() ->
    [{group1, [parallel], [test1a, test1b]},
     {group2, [shuffle, sequence], [test2a, test2b, test2c]}].
```

**组执行属性**​ ：

|属性|含义|
|---|---|
|`parallel`|组内用例并行执行|
|`sequence`|按顺序执行|
|`shuffle`|随机顺序执行|
|`repeat` / `repeat_until_*`|重复执行|

**`all/0` 中引用组**：

```
all() -> [testcase1, {group, group1}, {group, group2}].
```

📌 **洞见三：三层配置 = "测试隔离"的精细控制**

```
init_per_suite/1          ← 套件开始：连接数据库、建表
    │
    ├── init_per_testcase/2   ← 用例1前：清空表
    │   test_case_1/1
    │   end_per_testcase/2    ← 用例1后：验证清理
    │
    ├── init_per_testcase/2   ← 用例2前：清空表
    │   test_case_2/1
    │   end_per_testcase/2    ← 用例2后：验证清理
    │
    └── end_per_suite/1       ← 套件结束：断开连接、删表
```

**这正是 Arrange-Act-Assert 模式在系统测试中的工程化表达**。

---

## 🎭 小节四：测试用例的语义 —— "返回即成功，崩溃即失败"

Common Test 官方文档明确：

> **"如果测试用例返回给调用者，无论返回什么值，都认为测试成功。然而，一些返回值有特殊含义：`{skip, Reason}` 表示跳过测试用例；`{comment, Comment}` 在日志中打印注释；`{save_config, Config}` 让 Common Test 服务器将 Config 传递给下一个测试用例。测试用例失败被指定为运行时错误（崩溃），无论终止原因是什么。"**

### 成功与失败的语义

```
%% 成功——利用模式匹配
session(_Config) ->
    {started, ServerId} = my_server:start(),
    {clients, []} = my_server:get_clients(ServerId),
    MyId = self(),
    connected = my_server:connect(ServerId, MyId),
    {clients, [MyId]} = my_server:get_clients(ServerId),
    disconnected = my_server:disconnect(ServerId, MyId),
    {clients, []} = my_server:get_clients(ServerId),
    stopped = my_server:stop(ServerId).
```

**这正是官方文档推荐的风格**：

> **"如果你有效使用 Erlang 模式匹配，你可以利用这个属性。结果是简洁、可读的测试用例函数，看起来更像脚本而非实际程序。"**​

### 特殊返回值

```
%% 跳过
test_known_bug(_Config) ->
    {skip, "Bug #1234 not yet fixed"}.

%% 带注释的成功
test_with_comment(_Config) ->
    do_something(),
    {comment, "Performance improved by 30%"}.

%% 保存配置给下一个用例
test_first(_Config) ->
    State = complex_setup(),
    {save_config, [{state, State}]}.

test_second(Config) ->
    State = ?config(state, Config),
    %% State 来自 test_first 的 save_config
    ok.
```

### 跳过测试用例的三种方式

官方文档列出 ：

1. **测试规范中使用 `skip_suites` 和 `skip_cases` 项**
    
2. **从 `init_per_testcase/2` 或 `init_per_suite/1` 返回 `{skip, Reason}`**
    
3. **从测试用例执行子句返回 `{skip, Reason}`**
    

---

## 📜 小节五：测试规范（Test Specification）—— `.spec` 文件

这是 Fred 在"不寻常的 Common Test"中重点强调的内容。文章 1（LYSE 第二十八章译本）明确：

> **"与其每次运行测试时都手动指定所有内容，不如看看一种叫做'测试规范'的东西。测试规范是特殊的配置文件，允许你详细说明如何运行测试，并且适用于 Erlang shell 和命令行。规范文件将包含 Erlang 元组，就像一个咨询文件。"**​

### 核心规范项

**1. `{include, IncludeDirectories}`**

> **"当 Common Test 自动编译套件时，此选项允许你指定它应该在哪里查找包含文件。"**​

**2. `{logdir, LoggingDirectory}`**

> **"在记录时，所有日志都应移动到 LoggingDirectory。请注意，目录必须在运行测试之前存在，否则 Common Test 会报错。"**​

**3. `{suites, Directory, Suites}`**

> **"在 Directory 中查找给定的套件。Suites 可以是原子（some_SUITE）、原子列表，或者原子 `all`，表示运行目录中的所有套件。"**​

**4. `{skip_suites, Directory, Suites, Comment}`**

> **"此选项将从先前声明的套件列表中减去一系列套件并跳过它们。Comment 参数是一个字符串，解释了为什么要跳过它们。此注释将被添加到最终的 HTML 日志中。"**​

**5. `{groups, Directory, Suite, Groups}`**

> **"此选项用于从给定套件中选择一些组。Groups 变量可以是单个原子（组名）或 `all`。"**​

**6. `{groups, Directory, Suite, Groups, {cases, Cases}}`**

> **"允许你指定要包含在测试中的一些测试用例。"**​

### 完整示例

```
%% demo.spec
{alias, demo, "./demo/"}.
{logdir, "./logs/"}.
{suites, demo, all}.
{skip_cases, demo, demo_SUITE, test_skipped, "this test fails on purpose"}.
```

**运行测试**：

```
$ ct_run -spec demo.spec
```

或在 Erlang shell 中：

```
1> ct:run_test(["-spec", "demo.spec"]).
```

📌 **洞见四：测试规范 = "测试运行的声明式配置"**

它让你：

- 精确选择运行哪些套件、哪些组、哪些用例
    
- 跳过已知的失败用例
    
- 指定日志目录
    
- 复用配置，避免每次手动输入
    

---

## 🌐 小节六：分布式测试 —— Common Test 的"超能力"

Common Test 官方文档明确：

> **"Common Test 还提供分布式测试模式，具有集中控制和日志。此功能允许在一个公共会话中独立测试多个系统。"**​

### 为什么分布式测试重要？

回想第二十六章（分布式 Erlang）和第二十七章（分布式 OTP 应用）：

- 你的系统可能跨多个节点
    
- 你需要验证 failover/takeover 是否正确
    
- 你需要模拟网络分区
    
- 你需要在多个目标节点上并行执行测试
    

**Common Test 天然支持**：

```
┌──────────────────────┐
                    │   CT Master Node      │
                    │   (集中控制 + 日志)    │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ CT Node A │    │ CT Node B │    │ CT Node C │
        │ (测试 SUT1)│    │ (测试 SUT2)│    │ (测试 SUT3)│
        └──────────┘    └──────────┘    └──────────┘
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ Target 1  │    │ Target 2  │    │ Target 3  │
        └──────────┘    └──────────┘    └──────────┘
```

**这正是测试分布式 Erlang 系统的理想架构**。

---

## 🔌 小节七：支持模块 —— `ct_*` 系列

Common Test 官方文档明确：

> **"Common Test 应用程序还包括其他名为 `ct_*` 的模块，这些模块提供各种支持，主要是简化通信协议（如 RPC、SNMP、FTP、Telnet 等）的使用。"**​

### 常用支持模块

|模块|用途|
|---|---|
|`ct_rpc`|跨节点 RPC 调用|
|`ct_snmp`|SNMP 协议操作|
|`ct_ftp`|FTP 客户端|
|`ct_telnet`|Telnet 客户端|
|`ct_ssh`|SSH 客户端|
|`ct_http`|HTTP 请求|

### 示例：使用 `ct_rpc` 测试分布式应用

```
distributed_app_test(_Config) ->
    %% 在主节点启动应用
    ok = ct_rpc:call(cp1@cave, application, start, [myapp]),
    
    %% 验证应用在主节点运行
    [myapp | _] = ct_rpc:call(cp1@cave, application, which_applications, []),
    
    %% 杀掉主节点
    ct_rpc:call(cp1@cave, init, stop, []),
    
    %% 等待 failover
    timer:sleep(6000),
    
    %% 验证应用在备节点运行
    [myapp | _] = ct_rpc:call(cp2@cave, application, which_applications, []),
    
    ok.
```

**这正是第二十七章"分布式 OTP 应用"的理想测试方式**——用 Common Test 自动化验证 failover/takeover。

---

## 🌐 全局脉络：Common Test 在整本书中的位置

把第二十八章放进全书架构：

```
第二十四章：EUnit（单元测试）
第二十五章：ETS（节点本地共享内存）
第二十六章：分布式 Erlang（节点通信）
第二十七章：分布式 OTP 应用（节点级容错）
第二十八章：Common Test ← 你在这里
   ↓ 集成测试、系统测试、分布式测试
第二十九章及之后：Mnesia（分布式数据库）
```

### 与前二十七章的深度连接

**与第二十四章（EUnit）的连接**：

|维度|EUnit|Common Test|
|---|---|---|
|测试文件|`*_tests.erl` 或内嵌 `_test()`|`*_SUITE.erl`|
|发现机制|命名约定 `_test()` / `_test_()`|`all/0` 显式声明|
|准备/清理|`?setup` / `?foreach`|`init_per_suite/1` 等三层配置函数|
|运行方式|`eunit:test(Module)`|`ct:run_test/1` 或 `ct_run`|
|报告|简单文本|HTML 概览 + 详细日志|
|分布式|不支持|原生支持|

**两者互补**：EUnit 做单元测试（开发时频繁运行），Common Test 做集成/系统测试（CI/CD 中运行）。

**与第二十三章（套接字编程）的连接**：

Common Test 可以测试 TCP/UDP 服务器：

```
tcp_server_test(_Config) ->
    {ok, Socket} = gen_tcp:connect("localhost", 2345, [binary, {active, false}]),
    ok = gen_tcp:send(Socket, <<"hello">>),
    {ok, <<"HELLO">>} = gen_tcp:recv(Socket, 0),
    gen_tcp:close(Socket),
    ok.
```

**与第二十七章（分布式 OTP 应用）的连接**：

Common Test 是验证 failover/takeover 的理想工具——正如上面分布式测试示例所示。

### 与工业级系统的连接

**WhatsApp 的测试体系**（推测）：

```
test/
├── unit/                    %% EUnit 单元测试
│   ├── conn_worker_tests.erl
│   └── message_router_tests.erl
│
├── integration/             %% Common Test 集成测试
│   ├── conn_handshake_SUITE.erl
│   ├── message_delivery_SUITE.erl
│   └── failover_SUITE.erl
│
├── system/                  %% Common Test 系统测试
│   ├── cluster_SUITE.erl
│   ├── netsplit_SUITE.erl
│   └── load_test_SUITE.erl
│
└── spec/
    ├── ci.spec              %% CI 运行所有测试
    ├── smoke.spec           %% 冒烟测试
    └── regression.spec      %% 回归测试
```

**35 人支撑 9 亿用户的测试基础**：

- **EUnit**：每个函数、每个模块都有单元测试
    
- **Common Test 集成测试**：验证跨模块协作
    
- **Common Test 系统测试**：验证分布式行为
    
- **Common Test 分布式测试**：跨多节点验证 failover
    
- **测试规范**：精确控制 CI 运行哪些测试
    
- **HTML 日志**：失败时快速定位问题
    

---

## ✍️ 读书笔记的核心收获

1. **Common Test 的定位**：Erlang/OTP 的主要测试工具，支持任何类型目标系统的自动化测试
    
2. **与 EUnit 的区别**：
    
    - EUnit：单元测试，函数级，开发时频繁运行
        
    - Common Test：集成/系统测试，套件级，CI/CD 中运行
        
    
3. **套件结构**：
    
    - 模块名 `*_SUITE.erl`
        
    - 必须导出 `all/0`
        
    - 建议包含 `ct.hrl`
        
    
4. **三层配置函数**：
    
    - 套件级：`init_per_suite/1` + `end_per_suite/1`
        
    - 用例级：`init_per_testcase/2` + `end_per_testcase/2`
        
    - 组级：通过 `groups/0` 定义，支持 `parallel`/`sequence`/`shuffle`/`repeat`
        
    
5. **测试用例语义**：
    
    - 返回即成功（利用模式匹配）
        
    - 崩溃即失败
        
    - 特殊返回：`{skip, Reason}`、`{comment, Comment}`、`{save_config, Config}`
        
    
6. **测试规范（.spec 文件）**：
    
    - `{include, Dir}`：指定包含文件目录
        
    - `{logdir, Dir}`：指定日志目录（必须预先存在）
        
    - `{suites, Dir, Suites}`：选择套件
        
    - `{skip_suites, Dir, Suites, Comment}`：跳过套件
        
    - `{groups, Dir, Suite, Groups}`：选择组
        
    - `{groups, Dir, Suite, Groups, {cases, Cases}}`：选择用例
        
    
7. **分布式测试**：Common Test 原生支持集中控制的多节点测试
    
8. **支持模块 `ct_*`**：
    
    - `ct_rpc`：跨节点 RPC
        
    - `ct_snmp`：SNMP 操作
        
    - `ct_ftp`：FTP 客户端
        
    - `ct_telnet`：Telnet 客户端
        
    
9. **`init_per_suite/1` 失败 → 整个套件自动跳过**（auto skipped）
    
10. **HTML 日志**：主要日志（概述）+ 次要日志（用例级详情）
    
11. **配置数据通过 Config 键值列表传递**：`init_per_suite` → 测试用例 → `end_per_suite`
    
12. **`?config(Key, Config)` 宏**：从 Config 中提取配置值
    
13. **`suite/0` 返回套件属性**：如 `{timetrap, {minutes, 1}}`
    
14. **运行方式**：
    
    - `ct_run -spec test.spec`
        
    - `ct:run_test(["-spec", "test.spec"])`
        
    - `ct:run_test(["-dir", "test/", "-suite", "my_SUITE"])`
        
    
15. **Common Test 是"系统测试的操作系统"**：
    
    - 套件组织：测试目录 + 数据目录
        
    - 支持库：通用支持 + 特定区域支持 + 自定义支持
        
    - 黑盒测试：通过标准 O&M 和 CLI 协议连接目标系统
        
    - 白盒测试：直接调用 Erlang 函数
        
    

> 💡 **贯穿全书的隐喻再进化**：Erlang 的测试体系可以总结为——
> 
> **"用 EUnit 封装单元测试，用 Common Test 封装集成与系统测试，用测试规范封装测试运行配置。"**
> 
> - **EUnit**：函数级正确性验证（第二十四章）
>     
> - **Common Test**：套件级系统验证（第二十八章）← 你在这里
>     
> - **ct_rpc 等支持模块**：分布式系统测试
>     
> - **.spec 文件**：测试运行的声明式配置
>     
> 
> **第二十八章的真正价值，是让你理解"单元测试"与"系统测试"的本质差异**：
> 
> 1. **第二十四章 EUnit**：验证"函数做对了"——输入确定、输出确定、无副作用
>     
> 2. **第二十八章 Common Test**：验证"系统做对了"——跨模块、跨进程、跨节点、跨外部系统
>     
> 
> **Common Test 的设计哲学**：
> 
> - **套件即模块**：`*_SUITE.erl` 是普通 Erlang 模块，符合 Erlang 哲学
>     
> - **配置即生命周期**：init/end 函数精确控制准备与清理
>     
> - **规范即声明**：`.spec` 文件声明式地描述"运行什么、跳过什么、日志在哪"
>     
> - **分布式即原生**：集中控制多节点测试，验证分布式行为
>     
> - **协议即支持**：`ct_*` 模块内置对 RPC/SNMP/FTP/Telnet 的支持
>     
> 
> **从第一章到第二十八章，你看到了 Erlang 工程质量的完整保障体系**：
> 
> - 不可变变量 → 函数式纯洁性
>     
> - 并发原语 → 进程模型
>     
> - 监督树 → 进程级容错
>     
> - 分布式 Erlang → 节点透明通信
>     
> - 分布式 OTP 应用 → 节点级容错
>     
> - EUnit → 单元正确性
>     
> - **Common Test → 系统集成正确性**​ ← 你在这里
>     
> - Mnesia → 分布式数据一致性（后续）
>     
> 
> **Common Test 是 Erlang/OTP 自身的测试工具**——OTP 团队用它来验证整个 Erlang/OTP 系统 。这意味着：
> 
> - 你使用的工具，正是构建 Erlang/OTP 的工具
>     
> - 它的成熟度经过 30+ 年的生产验证
>     
> - 它与 OTP 深度集成（如 `ct_rpc` 用于分布式测试）
>     
> 
> **WhatsApp 35 人支撑 9 亿用户的测试基础**：
> 
> - **EUnit**：每个 PR 自动运行单元测试
>     
> - **Common Test 集成测试**：验证跨模块协作
>     
> - **Common Test 系统测试**：验证集群行为
>     
> - **Common Test 分布式测试**：跨多节点验证 failover
>     
> - **.spec 文件**：CI 精确控制测试范围
>     
> - **HTML 日志**：生产事故复盘的依据
>     
> 
> 正如 Erlang 官方文档所言 ：
> 
> **"Common Test is the main tool being used in all testing- and verification activities that are part of Erlang/OTP system development and maintenance."**
> 
> **这正是 Common Test 在工程实践中的价值**——它不仅是测试框架，更是 Erlang 系统的"验证基础设施"。从单元测试到分布式系统测试，从简单函数到外部协议，Common Test 提供了一致的、声明式的、可重复的测试体验。
> 
> **第二十八章是"EUnit 单元测试"与"Mnesia 分布式数据"之间的桥梁**：
> 
> - 它告诉你如何测试集成级、系统级、分布式系统
>     
> - 它告诉你如何通过测试规范精确控制测试运行
>     
> - 它告诉你如何利用 `ct_*` 支持模块测试外部系统
>     
> - 它为后续更复杂的系统验证（Mnesia 分布式事务、网络分区处理）奠定基础
>     
> 
> 正如 Fred 在章节标题"Unusual Common Test"所暗示的——**Common Test 不寻常，因为它不像传统测试框架**。它不是"每个测试一个函数"的简单模型，而是"套件—用例—配置函数—测试规范"的完整体系。这种"不寻常"正是为了应对 Erlang 系统的"不寻常"——并发、分布式、容错、热升级。
> 
> 从第一章的不可变变量，到第二十八章的 Common Test——**你走完了 Erlang/OTP 工程质量的完整路径**。每一条抽象都建立在前一条之上，最终构成了一个"默认可靠"的系统构建与验证框架：
> 
> - 不可变变量 → 函数式纯洁性
>     
> - 并发原语 → 进程模型
>     
> - 行为 → 通用/特定分离
>     
> - 监督树 → 容错骨架
>     
> - 应用 → 生命周期管理
>     
> - 发布 → 可部署单元
>     
> - 热升级 → 不停机演化
>     
> - ETS → 节点本地共享内存
>     
> - 分布式 Erlang → 跨节点位置透明
>     
> - 分布式 OTP 应用 → 节点级容错
>     
> - EUnit → 单元正确性
>     
> - **Common Test → 系统集成正确性**​ ← 你在这里
>     
> 
> **Common Test 不是测试的终点，是"系统可信度"的起点**。它让 Erlang 的"let it crash"哲学有了完整的验证闭环——系统既能从故障中恢复（监督树、分布式 failover），又能确保业务逻辑正确（EUnit + Common Test）。
> 
> 正如 Fred 在章节里展示的——**当"单元测试"升级为"系统测试"**，你需要的不只是断言，而是：
> 
> - 套件级的生命周期管理（init_per_suite/end_per_suite）
>     
> - 用例级的隔离（init_per_testcase/end_per_testcase）
>     
> - 组的执行控制（parallel/sequence/shuffle/repeat）
>     
> - 测试规范的声明式配置（.spec 文件）
>     
> - 分布式测试的原生支持（集中控制 + 日志）
>     
> - 外部协议的内置支持（ct_rpc/ct_snmp/ct_ftp/ct_telnet）
>     
