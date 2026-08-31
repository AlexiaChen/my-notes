
# 第 6 章　Schema 设计与管理——读书笔记

> 💡 这一章的本质，不是教你"每个字段用什么类型"，而是建立一个**第一性原理**：Schema 是业务事实的"类型化表达"，其设计目标是在**存储成本、写入效率、查询性能、灵活性与可演化性**之间取得与你的工作负载匹配的权衡。每一个数据类型选择、每一处范式/反范式取舍、每一次 NULL 还是 NOT NULL 的决策，背后都是你对"数据长什么样、怎么被访问、怎么演化"的显式假设。
> 
> 正如 MySQL 官方优化指南开宗明义所说：**"Design your tables to minimize their space on the disk. This can result in huge improvements by reducing the amount of data written to and read from disk."**——更小的表意味着更少的内存占用、更小的索引、更快的查询。这是贯穿全章的工程主线。

---

## 🎯 本章核心脉络

Silvia Botros 和 Jeremy Tinley 作为 SRE 背景的作者，在这一章埋下了一条主线——**Schema 设计是可靠性工程的一部分，而不是画 ER 图的艺术**：

1. **用最小、最确定的类型表达数据**——整数选最小的、实数用 DECIMAL、字符串避免盲目 VARCHAR(255)
2. **为索引效率设计**——短主键、合适的列顺序、避免过多宽列
3. **警惕 Schema 反模式**——太多列、太多 JOIN、滥用 ENUM、恐惧 NULL
4. **Schema 变更要像代码一样管理**——版本化、可回滚、可演练

MySQL 8.4 官方文档的硬约束我们要先记住：**每张表硬上限 4096 列，但实际最大列数受行大小限制**；**InnoDB 表默认使用 DYNAMIC 行格式**；**8.4 中 BINARY 属性已废弃，应使用显式 _bin 排序规则**。这些是现代 Schema 设计的起点。

---

## 选择优化的数据类型

### 第一性原理视角

选择数据类型的本质问题只有一个：**"这个字段的数据，最小能用什么类型准确表达？"**​ MySQL 官方明确建议："Use the most efficient (smallest) data types possible"。

为什么"小"这么重要？

- **更小的数据类型 → 更小的行 → buffer pool 能装更多行 → 更高缓存命中率**（呼应第 4 章工作集概念）
- **更小的数据类型 → 更小的索引 → 索引缓存命中率更高 → 更少磁盘寻道**
- **更小的数据类型 → 更少的 CPU 解码开销**

📌 **洞见**：很多人以为"磁盘便宜，类型大一点无所谓"。但在 InnoDB 架构下，**类型大小直接决定了 buffer pool 的有效容量**。一张用 BIGINT 做主键的表，比用 INT 做主键的表，二级索引要大 2 倍——因为每个二级索引条目都包含主键副本。这是官方文档明确指出的："a short primary key saves considerable space if you have many secondary indexes"。

### 整数类型

书中从 TINYINT / SMALLINT / MEDIUMINT / INT / BIGINT 的存储占用展开。MySQL 官方补充了一个常被忽略的点：**MEDIUMINT 比 INT 少用 25% 的空间**。

|类型|字节|有符号范围|无符号范围|
|---|---|---|---|
|TINYINT|1|-128 ~ 127|0 ~ 255|
|SMALLINT|2|±3.2万|0 ~ 6.5万|
|MEDIUMINT|3|±830万|0 ~ 1677万|
|INT|4|±21亿|0 ~ 42亿|
|BIGINT|8|±922亿亿|0 ~ 1844亿亿|

📌 **洞见**：工业界有个常见误区——"主键一律用 BIGINT UNSIGNED AUTO_INCREMENT"。但对很多小表（如字典表、配置表），**MEDIUMINT UNSIGNED 足够支撑 1677 万行，能省下 25% 的主键空间和相应的二级索引空间**。一个真实案例：某 SaaS 平台的 `countries` 字典表用 INT 主键，实际上只有 200+ 行，MEDIUMINT 完全够用，全库二级索引累计节省数百 MB。

**主键设计的现代视角**：

- 单表单机：INT 或 MEDIUMINT 自增
- 分布式系统：BIGINT（雪花 ID）或 UUIDv7（时序 UUID，对索引友好）
- **避免使用 UUIDv4 字符串作主键**——32 字符的字符串主键会让所有二级索引膨胀严重，且插入是随机 IO
- **避免用业务字段做主键**（如身份证号、邮箱）——业务字段会变，而主键变更代价极高

### 实数类型

书中的核心观点：**DECIMAL 用于精确计算（金融），FLOAT/DOUBLE 用于近似计算（科学测量）**。

📌 **洞见**：这是金融系统的生命线。DECIMAL(10,2) 精确存储货币，但**计算时仍要注意**：

- DECIMAL 的加减乘除在 MySQL 内部是高精度计算，CPU 开销高于 FLOAT
- 对于"金额 = 单价 × 数量"这类计算，建议：**存储用 DECIMAL，但计算时也用 DECIMAL**，避免隐式转换为 FLOAT 引入误差
- 8.4 中处理 DECIMAL 时要注意：**所有返回 JSON 的函数（JSON_OBJECT、JSON_ARRAY）对 key 名称强制去重**——如果老代码依赖重复 key，升级 8.4 后行为会改变

**工业界最佳实践**：金额存储用 DECIMAL(18,4) 或 DECIMAL(18,2)，单位用"分"存储整数（BIGINT）也是一种常见方案——彻底避免浮点问题。

### 字符串类型

书中从 CHAR/VARCHAR/TEXT/BLOB 展开。MySQL 8.4 官方文档给了几个关键更新：

**1. 字符集与排序规则**

- MySQL 8.4 默认字符集是 utf8mb4，默认排序规则是 utf8mb4_0900_ai_ci
- **8.4 中 `BINARY` 属性已废弃**，应使用显式 _bin 排序规则
- 字符类型的长度以**字符**为单位，二进制类型以**字节**为单位

**2. CHAR vs VARCHAR 的现代选择**

📌 **洞见**：传统观点认为"定长用 CHAR，变长用 VARCHAR"。但在 UTF8MB4 时代，这个规则变了：

- CHAR(N) 在 utf8mb4 下，**实际存储是 N 到 N×4 字节的可变范围**（compact 系列行格式的优化）
- 这意味着 CHAR 不再有传统的"定长优势"
- **工业界共识：几乎总是用 VARCHAR**，除非存储的是真正的定长代码（如国家代码 CHAR(2)、MD5 CHAR(32)）

**3. VARCHAR 长度设置的科学**

书中批判了"VARCHAR(255) 到处用"的反模式。这是个关键洞见：

> ⚠️ **VARCHAR(255) 不是"安全选择"，而是"懒惰选择"**。它的问题：
> 
> - 误导优化器：MySQL 在内存中排序时，可能按 255 字符分配临时表
> - 破坏索引前缀优化：长 VARCHAR 上的索引必须用前缀索引，而前缀长度选择需要精细权衡
> - 浪费行空间估算：optimizer 会用最大长度估算临时表内存

**正确做法**：VARCHAR 长度应该**反映业务真实上限**。用户名 VARCHAR(50)、邮箱 VARCHAR(255)（RFC 5321 规定 254 字符）、电话号码 VARCHAR(20)、地址 VARCHAR(200)。

**4. TEXT/BLOB 的现代处理**

📌 **洞见**：当表中包含 TEXT/BLOB 大字段时，InnoDB 的行格式（DYNAMIC）会把大字段存储在溢出页，主行只保留 20 字节指针。这意味着：

- 即使表有 LONGTEXT 列，只要不查询它，buffer pool 不会被它污染
- **但如果你经常查询大字段，就应该考虑垂直拆分**——把大字段放到单独的"详情表"

这正是书中"太多列"陷阱的解药：**主表 + 详情表的垂直拆分模式**。

**5. 前缀索引的智慧**

MySQL 官方明确建议："If it is very likely that a long string column has a unique prefix on the first number of characters, it is better to index only this prefix"。这背后是数学：前缀索引 `INDEX(col(10))` 比 `INDEX(col)` 小得多，且"require less disk space... give you more hits in the index cache"。

### 日期和时间类型

书中从 DATE/DATETIME/TIMESTAMP/YEAR 展开。这里有几个 8.4 时代的关键点：

**1. TIMESTAMP vs DATETIME 的抉择**

|特性|TIMESTAMP|DATETIME|
|---|---|---|
|存储|4 字节|5-8 字节|
|范围|1970-2038|1000-9999|
|时区|随会话时区转换|不转换（8.0+ 可带时区）|
|默认值|可 `DEFAULT CURRENT_TIMESTAMP`|8.0+ 同样支持|

📌 **洞见**：**2038 年问题（Y2K38）是真实存在的**。任何新系统都应默认用 DATETIME，除非你有强烈的存储占用考虑且确定数据不会跨越 2038 年。TIMESTAMP 的"自动时区转换"特性在全球化系统中反而是个坑——同一个时间戳在不同时区会话中读出不同值。

**2. 不要使用 '0000-00-00 00:00:00' 作为"无日期"**

书中专门批评了这个反模式。MySQL 5.7+ 默认开启 `STRICT_TRANS_TABLES`，**不允许 '0000-00-00' 这种"魔术零值"**。这是 8.4 的默认行为。正确做法是：

- 如果日期确实可能为空 → 用 NULL
- 如果需要默认值 → 用 `DEFAULT CURRENT_TIMESTAMP`

**3. 时间精度**

DATETIME(6) 支持微秒精度，占用 8 字节。对于金融交易、分布式追踪等场景，微秒精度是必要的。

### 位压缩数据类型

书中讲 BIT 类型。这里有个重要的工业界洞见：

📌 **BIT 类型是陷阱**：

- BIT(1) 在 InnoDB 中实际存储为 1 字节（不是 1 比特）
- BIT 类型在应用层读取时，不同语言/驱动表现不一致（有些返回字符串 "0"/"1"，有些返回整数）
- **工业界共识：用 TINYINT(1) 或 BOOL 替代 BIT(1)**，语义更清晰，存储开销相同

**BIT 的唯一合理使用场景**：BIT(8)、BIT(16) 这种打包多个布尔标志到一个整数的场景——但即便如此，TINYINT UNSIGNED 也通常更清晰。

### JSON 数据类型

这是书中内容与 2026 年现状差距最大的一节，需要重点补充。

**MySQL 8.4 的 JSON 能力**（官方文档）：

1. **Native JSON 类型**：二进制存储格式，支持快速元素访问，自动验证 JSON 合法性
2. **丰富的函数集**：JSON_EXTRACT、JSON_SET、JSON_MERGE_PATCH（RFC 7396）、JSON_SCHEMA_VALID 等
3. **部分更新优化**：使用 JSON_SET()、JSON_REPLACE()、JSON_REMOVE() 时，InnoDB 做**原地部分更新**，而非整体重写
4. **生成列 + 索引**：JSON 列不能直接索引，但可以在生成列上建索引
5. **多值索引**：8.0.17+ 支持对 JSON 数组建多值索引
6. **JSON_STORAGE_SIZE() / JSON_STORAGE_FREE()**：可观测 JSON 列的存储占用和空闲空间

📌 **洞见**：JSON 类型在 MySQL 中的正确使用场景是：

|场景|推荐方案|
|---|---|
|结构化业务数据，高频查询|传统列 + 索引|
|半结构化属性（如商品属性、用户偏好）|JSON 列 + 生成列索引|
|EAV 模型的现代替代|JSON 列 + 多值索引|
|日志、审计、原始负载|JSON 列|
|需要跨字段事务一致性|传统列（JSON 不支持事务内跨字段约束）|

**工业界实践**：

某电商平台用 JSON 列存储商品的"动态属性"（不同品类商品属性差异巨大），配合生成列索引关键属性：

```
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    category_id INT,
    attrs JSON NOT NULL,
    -- 生成列 + 索引
    attrs_brand VARCHAR(50) AS (attrs->>'$.brand'),
    attrs_price DECIMAL(10,2) AS (attrs->>'$.price'),
    INDEX idx_brand (attrs_brand),
    INDEX idx_price (attrs_price)
);
```

> ⚠️ **JSON 类型的局限**：
> 
> - JSON 列上的查询无法使用传统索引（除非通过生成列）
> - JSON 文档大小受 `max_allowed_packet` 限制
> - JSON 函数的计算开销高于传统列访问
> - 部分更新只能用 JSON_SET/REPLACE/REMOVE，直接赋值 `SET col = '{"a":1}'` 不走部分更新

### 选择标识符

这是 Schema 设计中**最关键且最容易被低估**的决策。

**第一性原理**：标识符（主键/外键）的选择，决定了：

- 二级索引的大小（因为 InnoDB 二级索引包含主键副本）
- 聚簇索引的物理行顺序
- 插入模式（自增 vs 随机）

📌 **洞见**：书中建议"用整数做标识符"，这在 8.4 时代需要细化：

**最优主键的特征**（按重要性排序）：

1. **短小**：INT/MEDIUMINT/BIGINT，不用 CHAR(36) UUID 字符串
2. **单调递增**：自增 ID 或雪花 ID，避免随机 IO
3. **永远不变**：主键变更 = 全表数据物理重排
4. **无业务含义**：避免"业务字段作主键"——业务规则会变

**分布式系统的主键方案对比**：

|方案|优点|缺点|
|---|---|---|
|AUTO_INCREMENT|简单、紧凑、递增|分布式冲突、单点|
|雪花 ID (BIGINT)|分布式、递增、紧凑|依赖时钟、ID 可预测|
|UUIDv4 (CHAR(36))|全球唯一|**32 字节字符串主键，二级索引严重膨胀**​|
|UUIDv7 (BIGINT/二进制)|分布式、时序递增|MySQL 无原生支持，需应用层生成|

**工业界最佳实践**：分布式系统用**雪花 ID 存为 BIGINT**，兼顾了紧凑性和分布式唯一性。UUIDv4 字符串作主键是反模式——它对 InnoDB 的打击是双重的：主键本身 32 字节 + 所有二级索引膨胀 32 字节/条目。

### 特殊数据类型

书中讨论了 IP 地址、MAC 地址等特殊类型。

📌 **洞见**：

- **IP 地址**：IPv4 用 INT UNSIGNED（INET_ATON/INET_NTOA 转换），占用 4 字节；IPv6 用 VARBINARY(16) 或 BINARY(16)
- **不要**用 VARCHAR(15) 存 IPv4——浪费空间且无法直接比较大小
- **MAC 地址**：BIGINT UNSIGNED 或用两个 INT 存储
- **地理位置**：POINT 类型 + SPATIAL 索引（MySQL 8.0+ 支持 GIS）
- **加密哈希**：UNHEX 后存 BINARY(16) 或 BINARY(32)，而非 CHAR(32)/CHAR(64) 字符串

---

## MySQL Schema 设计中的陷阱

这一节是本书最具"洞见密度"的部分。结合 2026 年的工业实践重新梳理：

### 太多的列

书中解释了底层机制：MySQL 服务器层和存储引擎层通过 row buffer 传递数据，需要解码成列。**InnoDB 的可变长度行格式总是需要转换，且转换成本取决于列数**。

📌 **洞见**：极宽表（数百列）的解码成本很高，**即使查询只用少数几列**。这是 InnoDB 架构的硬约束。

**判定标准**：**如果一张表超过 50-100 列，就该质疑设计的合理性**。

**解决方案**：

1. **垂直拆分**：主表存高频访问字段，详情表存低频大字段
    
    ```
    CREATE TABLE articles (
        id BIGINT PRIMARY KEY,
        title VARCHAR(200),
        summary VARCHAR(500),
        author_id BIGINT,
        created_at DATETIME(6),
        -- 高频字段
        INDEX idx_author (author_id)
    ) ENGINE=InnoDB;
    
    CREATE TABLE article_details (
        article_id BIGINT PRIMARY KEY,
        content LONGTEXT,
        source_url VARCHAR(500),
        original_link VARCHAR(500)
    ) ENGINE=InnoDB;
    ```
    
2. **EAV 模式的反击**：传统 EAV（实体-属性-值）需要大量自连接，且 MySQL 限制单查询最多 61 个表 JOIN。现代替代方案是 **JSON 列 + 生成列索引**（见前节）

### 太多的联接

书中给出经验法则：**高并发场景下，单个查询最好控制在 12 个表以内 JOIN**。

📌 **洞见**：这不是 MySQL 的软限制（硬限制是 61 个表），而是性能悬崖。JOIN 的本质是：

- 每个 JOIN 可能引入随机 I/O
- 优化器的 JOIN 顺序搜索空间随表数指数增长
- 执行计划可能劣化而优化器无法感知

**应对之道**：

1. **适度反范式**：把高频一起查询的字段冗余到主表
    
    ```
    -- 规范化：需要 JOIN
    SELECT m.message_text, u.user_name 
    FROM message m 
    JOIN user u ON m.user_id = u.id 
    WHERE u.account_type = 'premium' 
    ORDER BY m.published DESC 
    LIMIT 10;
    
    -- 反范式：单表 + 复合索引 (account_type, published)
    SELECT message_text, user_name 
    FROM user_messages 
    WHERE account_type = 'premium' 
    ORDER BY published DESC 
    LIMIT 10;
    ```
    
2. **应用层 JOIN**：对于超多表场景，分步查询 + 应用层组装，效率往往更高
3. **物化视图/汇总表**：预计算高频 JOIN 的结果

### 全能的枚举

书中批判了用 ENUM 存储数字代码的反模式：

```
-- 反模式
CREATE TABLE t_countries (
    country ENUM('', '1', '2', ..., '45')
);
```

📌 **洞见**：ENUM 的正确使用场景是**值集合极小且不常变化**（如性别、状态机状态）。ENUM 的问题：

1. 新增枚举值需要 ALTER TABLE（5.0 是阻塞操作，5.1+ 末尾追加也需要 ALTER TABLE）
2. ENUM 的内部表示是整数，但排序规则可能与预期不符
3. 应用程序代码中处理 ENUM 容易出错

**现代替代方案**：

- **稳定小集合**​ → ENUM（性别、订单状态）
- **可能变化的集合**​ → TINYINT UNSIGNED + 字典表 + 外键
- **多值属性**​ → JSON 数组 + 多值索引

### 变相的枚举

书中指出：SET('Y','N') 几乎肯定是应该用 ENUM('Y','N')——除非值可以同时为 Y 和 N。

📌 **洞见**：ENUM vs SET 的根本区别：

- **ENUM**：单选（一个值）
- **SET**：多选（位的 OR）

误用 SET 代替 ENUM 是常见的设计错误。判定标准：这个字段的值是否可以同时为多个？如果是 → SET；如果不是 → ENUM。

### NULL 不是虚拟值

书中强调了一个反直觉的观点：**不要为了避免 NULL 而使用魔术常数**。

📌 **洞见**：这是极具深度的洞见。常见的错误：

```
-- 反模式：用 '0000-00-00 00:00:00' 表示"无日期"
dt DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00'

-- 反模式：用 -1 表示"未知整数"
age INT NOT NULL DEFAULT -1
```

这些"魔术常数"的问题：

1. **增加代码复杂度**：每次查询都要处理特殊值
2. **引入 bug**：`WHERE age > 18` 会意外包含 age = -1
3. **MySQL 会对 NULL 建索引**（Oracle 不会）——所以 NULL 在索引层面是"一等公民"

**正确做法**：

- 当字段确实可能"无值"时 → 用 NULL
- 8.4 默认 `STRICT_TRANS_TABLES` 已禁止 '0000-00-00' 这种零值
- 只有业务语义确定"永远有值"时，才用 NOT NULL + 默认值

但 MySQL 官方也建议："Declare columns to be NOT NULL if possible"——因为 NULL 列有轻微存储开销（每列 1 bit）且可能影响索引使用效率。**权衡的核心是：语义正确性 > 微小的性能差异**。

### 范式与反范式的现代视角

书中讨论了规范化与反规范化。结合 2026 年的工业实践：

|维度|规范化|反规范化|
|---|---|---|
|写效率|快（无冗余）|慢（需同步多处）|
|表大小|小|大|
|JOIN 需求|高|低|
|数据一致性|强|弱（需应用维护）|

📌 **洞见**：现代工业界的主流做法是**"规范化的基础 + 针对性的反范式加速"**：

- 基础 Schema 严格 3NF
- 对确认的高频查询路径，做**受控的反范式**（冗余字段 + 应用层或触发器维护一致性）
- 对超高频的聚合查询，建**汇总表**（如每小时销售额汇总）

---

## Schema 管理

这是 8.4 时代变化最大的领域。书中可能还在讲"pt-online-schema-change"，但 2026 年的工业实践已经演进到完整的 CI/CD 集成。

### 现代 Schema 管理的核心原则

**1. Schema 即代码（Schema as Code）**

所有 Schema 变更必须：

- 以 SQL 迁移脚本形式存在
- 纳入 Git 版本控制
- 每次变更都有"up"脚本和"down"脚本
- 通过 Flyway、Liquibase、Atlas 等工具管理历史

**2. 迁移的安全性闸门**

Mydbops 在 2026 年的 CI/CD 指南中提出**5 项预检**：

1. **分类变更**：新增 nullable 列、新索引等附加型变更 → 自动放行；DROP/RENAME/类型收窄/外键变更 → 人工审查
2. **确认应用兼容性**：新旧应用版本都能在扩展后的 Schema 上运行
3. **演练生产路径**：在接近生产的克隆环境上执行完全相同的迁移
4. **设置安全限制**：活跃 OLTP 表设置 5 秒锁上限，超过即停止
5. **证明恢复路径**：针对部分状态的回滚/恢复脚本必须测试过

**3. Expand-Contract 模式（零停机迁移的标准范式）**

这是 2026 年工业界处理破坏性 Schema 变更的标准做法：

```
-- Phase 1: EXPAND（扩展）
ALTER TABLE customers ADD COLUMN new_customer_status VARCHAR(32) NULL;
-- 旧应用版本继续安全运行

-- Phase 2: MOVE（应用迁移）
-- 部署新应用版本，双写 old_status 和 new_customer_status
-- 后台批处理迁移历史数据
-- 验证两边数据一致

-- Phase 3: CONTRACT（收缩）
ALTER TABLE customers DROP COLUMN customer_status;
ALTER TABLE customers RENAME COLUMN new_customer_status TO customer_status;
```

**4. Schema 变更必须在应用部署之前**

CI/CD 的最佳实践：**先迁移 Schema，再部署新应用代码**。因为：

- 新应用期望的列若不存在 → 部署失败
- 但旧应用在 Schema 扩展后仍能运行（只是忽略新列）
- 这种排序天然兼容滚动部署

**5. 大表迁移的工具链**

|工具|机制|适用场景|
|---|---|---|
|**gh-ost**​|影子表 + binlog 回放 + 原子切换|大表、零停机|
|**pt-online-schema-change**​|触发器同步 + 原子重命名|Percona 工具包，久经考验|
|**MySQL 8 Instant DDL**​|原地元数据变更（有限操作）|新增 nullable 列（无数据拷贝）|

📌 **洞见**：MySQL 8.4 的 Instant DDL 已经能处理很多原先需要 pt-osc/gh-ost 的场景（如新增 nullable 列、新增索引）。但**对于修改列类型、删除列等操作，仍需在线 DDL 工具**。

**6. 迁移的自动化流水线**

Mydbops 推荐的六步检查点：

1. 验证迁移历史
2. 测试应用
3. 在代表性数据上演练
4. 记录运行时、锁、复制行为
5. 对高风险变更设闸门
6. 在生产部署前迁移，再部署依赖的应用版本

**7. 回滚策略**

每一次迁移都必须有经过测试的回滚路径：

- **补偿性迁移**：对于 DROP 操作，回滚意味着"撤销效果"而非真正恢复
- **前向修复**：有时回滚比前向修复风险更大
- **时间点恢复**： catastrophic 故障的最后手段

### 作为数据存储平台一部分的 Schema 管理

📌 **洞见**：在现代数据平台架构中，MySQL Schema 不再是孤立的——它需要与以下系统协同：

1. **CDC（变更数据捕获）**：Debezium、Canal 等工具通过 binlog 同步 Schema 变更到下游（数据仓库、搜索引擎、缓存）
2. **多区域复制**：Schema 变更必须在所有区域按序执行
3. **Schema 注册表**：对于使用 Protobuf/JSON 的下游消费者，Schema 变更需要版本化管理
4. **数据库分支**：PlanetScale 等现代数据库平台提供"数据库分支"概念，Schema 变更先在分支上测试，再合并到生产

---

## 小结

这一章表面讲的是"怎么设计表"，深层传递的是**SRE 视角的 Schema 哲学**：

**1. 最小类型原则是第一性原理**

不是"磁盘便宜随便用"，而是"类型大小直接决定 buffer pool 有效容量"。MEDIUMINT 比 INT 省 25%，更小的主键让所有二级索引更小——这些累积起来就是数量级的性能差异。

**2. Schema 设计是权衡，不是教条**

- NULL vs NOT NULL：语义正确性优先
- 范式 vs 反范式：规范化为基础，针对性反范式加速
- ENUM vs 字典表：稳定小集合用 ENUM，可变集合用字典表
- JSON vs 传统列：结构化用列，半结构化用 JSON + 生成列索引

**3. 标识符选择是架构决策**

主键的"短、单调递增、不变、无业务含义"四大特征，决定了整个数据库的索引效率和插入模式。**雪花 ID 存为 BIGINT 是分布式系统的现代最佳实践**，UUIDv4 字符串作主键是反模式。

**4. 宽表是性能杀手**

超过 50-100 列的表应该垂直拆分。InnoDB 的 row buffer 解码成本与列数成正比，这是架构层面的硬约束。

**5. Schema 管理已经进化到 CI/CD 级别**

2026 年的工业标准是：

- Schema 即代码，Git 版本控制
- 5 项预检闸门（分类、兼容性、演练、安全限制、回滚）
- Expand-Contract 模式实现零停机迁移
- gh-ost / pt-osc / Instant DDL 工具链
- Schema 变更先于应用部署

**6. MySQL 8.4 的新现实**

- 默认 DYNAMIC 行格式，默认 utf8mb4_0900_ai_ci
- BINARY 属性已废弃，用显式 _bin 排序规则
- JSON 类型支持部分更新、生成列索引、多值索引
- Instant DDL 能处理更多在线变更场景
- 8.0 将于 2026 年 4 月 30 日 EOL，新项目应直接用 8.4 LTS

> 💡 **贯穿全章的洞见**：Schema 设计的本质，是用类型系统对业务事实进行**精确的、可演化的编码**。每一个类型选择都是你对"数据是什么、怎么被访问、怎么随时间变化"的假设。当这些假设与真实工作负载匹配时，Schema 是透明的——你感觉不到它的存在；当假设错误时，Schema 就成为瓶颈——每一次查询都在为错误的类型付出代价。
> 
> 一个真实工业级 Schema 设计工作流：
> 
> 1. **建模阶段**：严格 3NF，明确每个实体的边界
> 2. **类型选择**：最小类型原则 + 标识符最优化（BIGINT 雪花 ID）
> 3. **索引规划**：基于真实查询模式设计索引，避免过度索引
> 4. **反范式决策**：识别高频查询路径，做受控冗余
> 5. **迁移管理**：Expand-Contract + CI/CD + 在线 DDL 工具
> 6. **持续演进**：监控慢查询，识别 Schema 层面的瓶颈，迭代优化
> 
> 这套方法论，让 Schema 设计从"画 ER 图的艺术"变成了"工程的科学"——这正是本书 SRE 视角的精髓：**用数据驱动决策，用自动化保障安全，用可观测性闭环反馈**。

📌 **给读者的实践清单**：

- ✅ 整数类型选最小的能容纳数据的（MEDIUMINT 优先于 INT）
- ✅ 主键用 BIGINT 雪花 ID（分布式）或 INT 自增（单机）
- ✅ 禁止 UUIDv4 字符串作主键
- ✅ 金额用 DECIMAL，禁止 FLOAT/DOUBLE
- ✅ 时间用 DATETIME(6)，避开 2038 年问题
- ✅ 半结构化属性用 JSON + 生成列索引
- ✅ 超过 50 列的表做垂直拆分
- ✅ 单查询 JOIN 控制在 12 个表以内
- ✅ ENUM 仅用于稳定小集合
- ✅ 需要表达"无值"时用 NULL，不要魔术常数
- ✅ Schema 变更走 CI/CD，使用 Expand-Contract 模式
- ✅ 大表迁移用 gh-ost 或 pt-online-schema-change
- ❌ 不要盲目 VARCHAR(255)
- ❌ 不要用 BIT(1)，用 TINYINT(1)
- ❌ 不要用 '0000-00-00' 表示无日期
- ❌ 不要把 NULL 视为敌人
- ❌ 不要在生产环境直接手动 ALTER TABLE

**Schema 是数据库的骨架，骨架搭好了，肌肉（索引）和血液（数据）才能高效流动。一个优秀的 Schema 设计，是后续所有性能优化的基石——没有好的 Schema，再精妙的 SQL 优化和服务器调优都是空中楼阁。这正是为什么本书把"Schema 设计与管理"放在"查询性能优化"（后续章节）之前的第一性原理：先有好的结构，再有好的查询。**