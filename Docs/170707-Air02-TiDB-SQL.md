---
title: TiDB 数据业务优化
date: 2017-07-01 21:40:52
updated: 2017-07-07 21:40:52
categories:
  - TiDB
tags:
  - TiDB
  - Online DDL
  - Air
---
# TiDB 数据业务优化

## 特性

- `alter table hot_spot shard_row_id_bits = 15` (分片数量 32768)
  - [TiDB PR](https://github.com/pingcap/tidb/commit/04ef7d79928ac27f6fab53c944efbb523f896755#diff-caef8063af52c86ddcdb638aff3e5bd9)
  - [SHARD_ROW_ID_BITS](https://github.com/pingcap/docs-cn/blob/master/sql/tidb-specific.md#shard_row_id_bits)
- `_tidb_rowid` 隐藏列
  - [_tidb_rowid](https://github.com/pingcap/docs-cn/blob/master/sql/tidb-specific.md#_tidb_rowid)