---
title: "第8章 数据的家——MySQL 的数据目录"
book: "MySQL是怎样运行的：从根儿上理解MySQL"
chapter: 8
version_basis: "MySQL 8.0/8.4；8.0 已采用数据字典，文件不等于完整元数据"
tags: [MySQL, datadir, 文件系统, 数据目录, 运维]
---

# 第8章 数据的家——MySQL 的数据目录

## 本章主线

数据目录是 MySQL 把逻辑对象映射到操作系统资源的地方，但不是“打开目录就能手工管理数据库”的文件夹。MySQL 8.0 之后，数据字典、表空间和文件系统之间由服务器维护关系，直接移动或删除文件可能破坏恢复。

## 8.1 数据库和文件系统的关系

逻辑层的数据库、表、索引和表空间，最终要落到文件系统的目录、文件和块上。这个映射受存储引擎影响：

```text
schema → database directory
InnoDB table → tablespace / .ibd
redo/undo → log or undo tablespace files
metadata → data dictionary and system schemas
```

文件系统提供持久化字节、目录和权限；MySQL 提供事务语义、页校验、日志恢复和对象生命周期。文件系统不会理解“这两个页必须原子提交”，所以 InnoDB 需要自己做 WAL、校验和崩溃恢复。

## 8.2 MySQL 数据目录

### 8.2.1 数据目录和安装目录的区别

安装目录放程序、插件和共享资源；数据目录放运行时数据、系统表、日志、表空间和状态文件。升级程序不应覆盖用户数据目录，备份数据目录也不能替代逻辑备份、物理备份和恢复测试。

### 8.2.2 如何确定 MySQL 中的数据目录

运行中的实例：

```sql
SELECT @@datadir, @@basedir;
SHOW VARIABLES LIKE 'datadir';
```

启动前可以查看 mysqld --verbose --help 的输出，或检查服务管理器实际使用的 defaults-file。容器环境还要确认容器内路径和宿主机挂载路径。

## 8.3 数据目录的结构

### 8.3.1 数据库在文件系统中的表示

数据目录下的子目录通常对应用户 schema，也有 mysql、performance_schema、sys 等系统 schema。INFORMATION_SCHEMA 是服务器提供的虚拟接口，不一定对应同名文件夹。

官方文档把数据目录描述为保存数据库目录、日志、InnoDB 表空间和其他运行信息的目录（[The MySQL Data Directory](https://dev.mysql.com/doc/refman/8.4/en/data-directory.html)）。目录结构属于实现细节，备份和运维应优先使用官方工具和 SQL 元数据。

### 8.3.2 表在文件系统中的表示

InnoDB 默认使用 file-per-table 时，表数据和索引通常在 schema 目录下的 .ibd 文件中；使用系统表空间或通用表空间时，物理位置不同。MyISAM 传统上有数据文件和索引文件。

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE, TABLESPACE_NAME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'app';
```

不要因为看到一个 .ibd 就认为拷走它即可恢复。InnoDB 还需要数据字典、表空间 ID、undo/redo 和一致元数据；单表传输要走受支持的 transportable tablespace 流程。

### 8.3.3 其他的文件

还可能看到 redo 日志、undo 表空间、错误日志、慢查询日志、二进制日志、自动生成的 SSL/RSA 文件、进程 ID 文件和持久化变量文件。不同发行版可能把它们移到别的目录。

排障时要建立“文件用途—生成者—恢复依赖”的清单，而不是凭扩展名删除大文件。

## 8.4 文件系统对数据库的影响

文件系统影响数据库的延迟、耐久性和可运维性：

- fsync、写缓存和电源保护影响提交后的持久性；
- 大小写敏感性影响表名迁移和跨平台复制；
- 文件权限影响启动与密钥保护；
- 磁盘空间不足会阻塞写入、日志和临时表；
- SSD/HDD、RAID、网络盘和容器卷会改变 I/O 延迟分布；
- 在线复制文件不能得到一致备份。

第一性原理洞见：数据库只能在底层存储提供的耐久性上做承诺。若磁盘控制器谎报 flush 已完成，数据库配置再正确也无法凭空制造物理可靠性。

## 8.5 MySQL 系统数据库简介

| Schema | 作用 |
| --- | --- |
| mysql | 数据字典、权限、系统表、时区和持久化配置等 |
| information_schema | 以视图式接口暴露元数据，很多内容动态生成 |
| performance_schema | 采集运行时性能、等待、锁和内存信息 |
| sys | 对 Performance Schema 的可读封装，帮助排障 |

系统 schema 不是普通业务库。升级、备份、权限和恢复都要按版本文档操作；不要直接编辑其物理文件或内部表。

## 8.6 总结

- 数据目录是逻辑数据库和物理文件的交汇点；
- 安装目录与数据目录职责不同；
- .ibd、redo、undo 和数据字典必须按 InnoDB 规则共同理解；
- 文件系统的权限、大小写、flush 和 I/O 能力会影响数据库语义；
- 备份的目标不是复制文件，而是得到可验证、可恢复的一致状态。

英文延伸阅读：

- [The MySQL Data Directory](https://dev.mysql.com/doc/refman/8.4/en/data-directory.html)
- [Files Created by CREATE TABLE](https://dev.mysql.com/doc/refman/8.4/en/create-table-files.html)
- [File-Per-Table Tablespaces](https://dev.mysql.com/doc/refman/8.4/en/innodb-file-per-table-tablespaces.html)

