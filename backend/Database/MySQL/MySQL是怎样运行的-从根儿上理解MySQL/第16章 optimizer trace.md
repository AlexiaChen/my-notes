---
title: "第16章 神兵利器——optimizer trace 的神奇功效"
book: "MySQL是怎样运行的：从根儿上理解MySQL"
chapter: 16
version_basis: "MySQL 8.0/8.4；trace 只针对当前会话，输出属于诊断信息"
tags: [MySQL, optimizer_trace, 优化器, 诊断]
---

# 第16章 神兵利器——optimizer trace 的神奇功效

## 16.1 optimizer trace 简介

EXPLAIN 告诉你最终选了什么，optimizer trace 更像一段决策日志：哪些候选被枚举、哪些被剪掉、统计信息估了多少、成本为什么更高、规则改写何时发生。

典型用法：

```sql
SET optimizer_trace = 'enabled=ON';
SELECT ...;
SELECT TRACE
FROM information_schema.OPTIMIZER_TRACE;
SET optimizer_trace = 'enabled=OFF';
```

官方文档强调 trace 只能看到当前会话执行的语句，不能读取其他连接的 trace（[Typical Usage](https://dev.mysql.com/doc/refman/8.4/en/optimizer-tracing-typical-usage.html)）。因此它适合隔离实验，不应在生产上无控制地长期开启。

## 16.2 通过 optimizer trace 分析查询优化器的具体工作过程

推荐按以下顺序读：

1. **确认输入**：记录版本、系统变量、表结构、索引和统计更新时间；
2. **先看 EXPLAIN**：确定最终连接顺序和访问方法；
3. **再看改写阶段**：观察条件化简、子查询转换和外连接处理；
4. **查看 range optimizer**：比较每个索引的可用范围和预计行数；
5. **查看 join optimization**：分析连接顺序候选、条件过滤和成本；
6. **核对 discarded plans**：理解为什么一个看似合理的计划被淘汰；
7. **用真实执行验证**：比较实际行数、耗时、I/O 和锁等待。

trace 的价值在于把“优化器错了”改写成可检验的问题：

```text
是统计信息错？
是隐式转换让索引不可用？
是回表成本被低估？
是连接顺序的中间结果估错？
是某个规则改写改变了预期？
```

trace 可能很大，尤其是多表连接和重复子查询。可以按功能开关减少输出；查询参数、用户数据和表名也可能出现在诊断日志中，生产环境要注意敏感信息。

英文参考：[Tracing the Optimizer](https://dev.mysql.com/doc/refman/8.4/en/optimizer-tracing.html)。

