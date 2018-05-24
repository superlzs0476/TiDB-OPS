---
title: TiDB Analyze
date: 2017-07-01 21:40:52
updated: 2017-07-07 21:40:52
categories:
  - TiDB
tags:
  - Analyze
---
# TiDB Analyze

- [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md)

## Analyze

- Analyze 相关命令，通过以下命令查看统计信息

命令 | 解释
---|---
`SHOW STATS_META`       | 来查看表的总行数以及修改的行数等信息
`SHOW STATS_HISTOGRAMS` | 来查看列的不同值数量以及 NULL 数量等信息
`SHOW STATS_BUCKETS`    | 来查看直方图每个桶的信息
`DROP STATS tableName`  | 可以删除统计信息

### 查看某个表直方图统计信息

- 通过以下命令查看某个表直方图统计信息
  - table_name 填这个 table 的名字，column_name 填索引的名字。

  ```SQL
  SHOW STATS_BUCKETS where table_name = "objects_tia";
  SHOW STATS_BUCKETS where table_name = "objects_tia" and column_name="PRIMARY";
  show stats_histograms where table_name=order_join_totalpay
  ```

### 线上业务手动 Analyze

- analzye 使用 Read Commited  隔离级别，时间戳是 max int64，调整扫描数据并发度
  - 关于 session 变量解释看 [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md)

  ```SQL
  set @@session.tidb_build_stats_concurrency=1;
  set @@session.tidb_distsql_scan_concurrency=4;
  set @@session.tidb_index_serial_scan_concurrency=4;
  analyze table tableName;
  ```

### Analyze 与 GC 问题

- Analyze 在 1.0.6 之前版本，受 GC 影响；1.0.6 之后版本已经修复; 以下为具体报错
  - 2018 年 1 月 8 日 fix

  ```SQL
  MySQL [lock_event_log]> analyze table mbk_lock_event;
  ERROR 1105 (HY000): start timestamp fall behind safe point
  ```