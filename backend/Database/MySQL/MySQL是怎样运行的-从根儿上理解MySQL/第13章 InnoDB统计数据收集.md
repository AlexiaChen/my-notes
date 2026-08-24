---
title: "第13章 兵马未动，粮草先行——InnoDB 统计数据是如何收集的"
book: "MySQL是怎样运行的：从根儿上理解MySQL"
chapter: 13
version_basis: "MySQL 8.0/8.4；统计信息是估算输入，不是精确行数"
tags: [MySQL, InnoDB, 统计信息, 优化器, ANALYZE]
---

# 第13章 兵马未动，粮草先行——InnoDB 统计数据是如何收集的

## 本章主线

优化器要决定索引还是全表、先连哪张表，但它通常不会先完整执行查询。它依赖统计信息估算表大小、索引基数和条件选择性。因此统计数据是优化器的粮草：错了，后面成本计算再精细也会选错路。

## 13.1 统计数据的存储方式

InnoDB 统计信息大体有两个生命周期：

| 类型 | 存放位置 | 生命周期 | 优点 | 风险 |
| --- | --- | --- | --- | --- |
| 永久性统计数据 | mysql.innodb_table_stats、mysql.innodb_index_stats 等 | 重启后保留 | 计划更稳定 | 可能陈旧 |
| 非永久性统计数据 | 内存 | 重启或重新计算后变化 | 更新灵活 | 重启后计划可能漂移 |

第一性原理是：**统计信息不是数据本身，而是数据分布的压缩摘要**。摘要越小，计算越快；但它不可能保留所有细节。

## 13.2 基于磁盘的永久性统计数据

### 13.2.1 innodb_table_stats

innodb_table_stats 记录表级信息，例如行数估计、聚簇索引大小和修改计数。表名、数据库名和统计更新时间帮助 DBA 识别统计信息来源。

```sql
SELECT database_name, table_name, last_update,
       n_rows, clustered_index_size, sum_of_other_index_sizes
FROM mysql.innodb_table_stats
WHERE database_name = 'app';
```

n_rows 是估计值，不是 COUNT(*) 的承诺。优化器用它进行相对比较。

### 13.2.2 innodb_index_stats

innodb_index_stats 记录索引级统计，例如不同索引层级的页数、叶子页数和基数估计。索引基数帮助优化器判断某个条件能过滤多少行。

```sql
SELECT database_name, table_name, index_name,
       stat_name, stat_value, sample_size, last_update
FROM mysql.innodb_index_stats
WHERE database_name = 'app'
ORDER BY table_name, index_name, stat_name;
```

### 13.2.3 定期更新统计数据

统计数据需要在数据分布显著变化后更新。InnoDB 可以根据修改比例等条件自动重新估算，也可以由 ANALYZE TABLE 触发。

自动更新降低维护成本，但会带来计划变化。高峰期大批量加载、删除或分区切换后，统计更新可能和业务流量竞争 I/O。

### 13.2.4 手动更新 innodb_table_stats 和 innodb_index_stats 表

直接改系统统计表属于高级操作。它可以用于实验或临时校正，但绕过正常采样流程，升级、复制和后续 ANALYZE 都可能覆盖它。

更安全的常规做法是：

```sql
ANALYZE TABLE app.orders;
```

如果确实要人工修改，必须先备份原值、记录原因、在影子环境比较计划，并准备恢复原统计信息。

## 13.3 基于内存的非永久性统计数据

非永久性统计数据保存在内存中，实例重启后可能重新采样。它适合数据持续变化的场景，但会让重启后的计划与重启前不同。

生产上要把重启计划漂移纳入演练：保存关键 SQL 的 EXPLAIN、索引统计和基准延迟，重启后比较。

## 13.4 innodb_stats_method 的使用

NULL 值的处理会影响基数估计。innodb_stats_method 的不同设置可以表达：将所有 NULL 视为一个相同值；将不同 NULL 出现位置视为不同分组；或忽略 NULL。

选择应根据查询语义和数据分布，而不是为了让某个 EXPLAIN 好看。若 NULL 占比很高，方法差异会明显影响选择性估计。

## 13.5 总结

- 统计信息是优化器对真实数据的压缩观察；
- 永久统计提高计划稳定性，内存统计提高灵活性；
- table_stats 描述表级规模，index_stats 描述索引级分布；
- ANALYZE TABLE 是常规刷新手段，直接改系统表应极其谨慎；
- 统计信息的正确性要用实际行数、EXPLAIN 和延迟共同验证。

英文延伸阅读：

- [InnoDB Persistent Optimizer Statistics](https://dev.mysql.com/doc/refman/8.4/en/innodb-persistent-stats.html)
- [Configuring Non-Persistent Optimizer Statistics](https://dev.mysql.com/doc/refman/8.4/en/innodb-nonpersistent-stats.html)
- [ANALYZE TABLE Statement](https://dev.mysql.com/doc/refman/8.4/en/analyze-table.html)

