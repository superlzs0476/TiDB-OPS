---
title: TiDB 环境变量解读
date: 2018-04-11 13:40:52
updated: 2017-04-12 17:07:52
categories:
  - TiDB
tags:
  - variable
---
# TiDB 环境变量解读

- docs-cn
  - [TiDB 系统变量](https://github.com/pingcap/docs-cn/blob/master/sql/variable.md)
  - [TiDB 专用系统变量和语法](https://github.com/pingcap/docs-cn/blob/master/sql/tidb-specific.md)
- 代码块
  - TiDB 已支持的所有环境变量 [sysvar.go](https://github.com/pingcap/tidb/blob/master/sessionctx/variable/sysvar.go)
  - TiDB 系统环境变量 [tidb_vars.go](https://github.com/pingcap/tidb/blob/master/sessionctx/variable/tidb_vars.go)

## TiDB system variable

### Session only

- tidb_snapshot
  - 默认不开启
  - `set @@session.tidb_snapshot = "2017-11-11 20:20:20";` 读取历史数据
- tidb_import_data
  - 布尔值，默认 false
  - `set @@session.tidb_import_data = true;`
- tidb_opt_agg_push_down
  - 布尔值，默认 false
  - 优化器规则下推
- tidb_opt_insubquery_unfold
  - 布尔值，默认 false
  - 是否启用展开子查询中的优化器执行计划
- tidb_build_stats_concurrency
  - int，默认 4
  - 用于加速 `ANALYZE` 语句，当一个表有多个索引时，可以同时扫描这些索引，以提高系统性能影响的成本。
- tidb_checksum_table_concurrency
  - int，默认 4
  - 用于加速 `ADMIN CHECKSUM TABLE;`，当一个表有多个索引时，可以同时扫描这些索引，以提高系统性能影响的成本。
- tidb_current_ts
  - 只读变量
  - `select @@tidb_current_ts;` , 用户获取当前 transaction 时间戳
- tidb_config
  - 只读变量
  - `select @@tidb_config;` , 查看当前 TiDB 服务配置文件信息
- tidb_batch_insert
  - 布尔值，默认 false
  - 自动分割插入数据
- tidb_batch_delete
  - 布尔值，默认 false
  - 自动分割删除数据
- tidb_dml_batch_size
  - int，默认 20000
  - 需要先开启 tidb_batch_insert 或者 tidb_batch_delete，当 DML 操作 20000 行，有可能打破 100MB 的事务限制，此时应该调小参数避免破坏 100MB 事务限制。
- tidb_max_chunk_size
  - int，默认 1024
  - 用于在查询执行期间控制最大 chunk 大小
- tidb_general_log
  - 布尔值，默认 false
  - 开启后，记录 TiDB 所执行的每条 SQL 原始语句
- tidb_enable_streaming
  - 默认 false
  - 可通过配置文件中 `enable-streaming` 参数修改默认值

### Session SQL 语句内存限制参数

- values 默认属性
  - 单位大小 Bytes
  - 默认 32GB
- tidb_mem_quota_query
- tidb_mem_quota_hashjoin
- tidb_mem_quota_mergejoin
- tidb_mem_quota_sort
- tidb_mem_quota_topn
- tidb_mem_quota_indexlookupreader
- tidb_mem_quota_indexlookupjoin
- tidb_mem_quota_nestedloopapply

### Session and Global

- tidb_distsql_scan_concurrency
  - int，默认 15
  - 用于设置 distsql 扫描任务的并发性
- tidb_index_join_batch_size
  - int，默认 25000
- tidb_index_lookup_size
  - int，默认 20000
- tidb_index_lookup_concurrency
  - int，默认 4
- tidb_index_lookup_join_concurrency
  - int，默认 4
- tidb_index_serial_scan_concurrency
  - int，默认 1
  - 控制索引扫描操作的并发性
- tidb_skip_utf8_check
  - 布尔值，默认 false
  - 跳过 UTF8 验证过程, 为了节省资源

## 案例演示

- 提高 select count(*) 语句运算速度
  - `set @@session.tidb_distsql_scan_concurrency = 300;`  调整不同的大小

- 记录当前会话下 TiDB 所执行的每条 SQL 语句
  - `set @@tidb_general_log = 1;`