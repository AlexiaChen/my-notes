---
title: "第15章 查询优化的百科全书——EXPLAIN 详解"
book: "MySQL是怎样运行的：从根儿上理解MySQL"
chapter: 15
version_basis: "MySQL 8.0/8.4；不同格式和版本字段可能增加"
tags: [MySQL, EXPLAIN, 执行计划, 查询优化]
---

# 第15章 查询优化的百科全书——EXPLAIN 详解

## 本章主线

EXPLAIN 不是性能评分表，而是优化器选择的访问路径和估算结果。读执行计划要回答四个问题：访问了谁；怎么访问；估算了多少行；还有哪些额外工作。

## 15.1 执行计划输出中各列详解

### 15.1.1 table

表示当前计划节点访问的表、派生表或临时结果。多表查询中，行出现的顺序通常能帮助判断连接顺序，但要结合 id、嵌套和 JSON 结构。

### 15.1.2 id

表示 SELECT 查询块的标识。子查询、派生表和 UNION 可能有不同 id；相同 id 通常属于同一查询块。它不是表的唯一 ID，也不是执行耗时。

### 15.1.3 select_type

描述查询块类型，例如 SIMPLE、PRIMARY、SUBQUERY、DEPENDENT SUBQUERY、DERIVED、UNION 和 UNION RESULT。DEPENDENT 说明子查询依赖外层值，可能产生重复执行风险。

### 15.1.4 partitions

若使用分区，显示访问的分区。分区裁剪越充分，读取范围越小；没有裁剪时，分区数量并不会自动带来性能收益。

### 15.1.5 type

访问方法分类，常见有 system、const、eq_ref、ref、range、index、ALL 等。它只能描述路径形态，不能脱离表大小、rows、回表和返回列判断好坏。

### 15.1.6 possible_keys 和 key

possible_keys 是优化器认为可能使用的索引集合；key 是最终选择的索引。possible_keys 不代表一定会用，key 为 NULL 也可能是有意选择全表扫描。

### 15.1.7 key_len

表示实际使用的索引键长度。它可以帮助判断复合索引使用到了哪些前缀，但不能简单用字节数直接映射到用了几列，因为 NULL、字符集和类型会增加额外字节。

### 15.1.8 ref

说明索引列与什么比较：常量、另一个表的列或表达式。连接计划中 ref 是判断内表是否按外表值查找的关键线索。

### 15.1.9 rows

优化器估计需要检查的行数，不是实际返回行数。估计与实际偏差很大时，优先检查统计信息、数据倾斜和条件相关性。

### 15.1.10 filtered

估计经过表条件过滤后保留的百分比。下一层看到的行数可粗略看成 rows × filtered / 100。它也是估计值，不是运行时计数。

### 15.1.11 Extra

提供额外执行信息，例如 Using index、Using where、Using temporary、Using filesort、Using join buffer、Using index condition 等。Extra 不是坏事清单：

- Using index 可能表示覆盖索引；
- Using where 表示仍需应用过滤；
- Using temporary/filesort 是否有问题取决于数据量和频率；
- Using join buffer 说明连接未按理想索引查找。

实际诊断要将 Extra 与 rows、实际耗时和资源指标结合。

## 15.2 JSON 格式的执行计划

JSON 计划能表达更完整的嵌套结构、成本、行数、过滤、attached_condition、排序和连接顺序：

```sql
EXPLAIN FORMAT=JSON
SELECT ...;
```

它适合脚本解析和对比计划，但可读性低。建议先用传统格式建立总体路径，再用 JSON 定位成本和条件归属。

## 15.3 Extended EXPLAIN

Extended EXPLAIN 可以配合 SHOW WARNINGS 查看优化器重写后的查询形态。在复杂子查询、常量传播、外连接消除和谓词下推问题上，它能解释为什么计划和原 SQL 不一样。

版本差异很重要：字段、重写形式和可见信息会变化，不能把某个版本的输出格式当成 API。

## 15.4 总结

- EXPLAIN 描述计划，不直接给出最终性能分数；
- table/id/select_type 说明查询结构；
- type/key/ref 说明访问路径；
- rows/filtered 说明优化器估算；
- Extra 说明排序、临时表、覆盖和连接缓冲等额外工作；
- JSON 和 Extended EXPLAIN 用于深入解释计划，不替代真实执行验证。

英文延伸阅读：

- [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [Understanding the Query Execution Plan](https://dev.mysql.com/doc/refman/8.4/en/execution-plan-information.html)
- [EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.4/en/explain-output.html)

