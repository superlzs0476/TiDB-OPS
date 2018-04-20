---
title: TiDB Online DDL 实操测试
date: 2017-10-10 21:40:52
updated: 2017-10-10 21:40:52
categories:
  - TiDB
tags:
  - TiDB
  - Online DDL
---
# TiDB Online DDL 实操测试

## 查 DDL 问题

* 思路如下
  * 已知 DDL 是串行工作，且每次工作前会先选择 PD 下任意 tidb 作为 owner 左右 job woner ，集群内 owner 唯一，每个 tidb 会自动获取一个 owner
  * 如果是 add index ，需要考虑表的大小，并且考虑 自增 ID 是否完全连续
  * 如果是 create table / databases 时间长，应该是 tidb 自身问题或者 DDL job 排队
  * drop / truncate DDL ，执行时间块，但是影响后续磁盘 IO，因为需要做 compress
* 命令或者关键词
  * admin show ddl jobs; // 查看 DDL 历史记录
    * 可以查看到 ddl owner ，在 tidb.log 搜索 ddl owner ，有则是 owner 归属，
    * 需要配合 API 查看 schema 信息
  * grep "finish DDL job ID:2092" tidb.log
    * 查看已经成功的 DDL ，需要先查到 ddl owner ，
  * CRUCIAL OPERATION  DDL 信息关键词 // 忘记了

### 查看 tidb DDL 信息

* [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/admin.md)
* amdin show ddl jobs  是在每台机器执行是相同的

  ```SQL
  MySQL [test]> admin show ddl jobs;
  +********************************************************************************************************************+***********+
  | JOBS                                                                                                               | STATE     |
  +********************************************************************************************************************+***********+
  | ID:731, Type:rename table, State:synced, SchemaState:public, SchemaID:1, TableID:206, RowCount:0, ArgLen:0         | synced    |
  | ID:730, Type:set default value, State:cancelled, SchemaState:none, SchemaID:1, TableID:206, RowCount:0, ArgLen:0   | cancelled |
  | ID:729, Type:rename table, State:synced, SchemaState:public, SchemaID:1, TableID:206, RowCount:0, ArgLen:0         | synced    |
  | ID:728, Type:set default value, State:cancelled, SchemaState:none, SchemaID:1, TableID:724, RowCount:0, ArgLen:0   | cancelled |
  | ID:727, Type:rename table, State:synced, SchemaState:public, SchemaID:1, TableID:724, RowCount:0, ArgLen:0         | synced    |
  | ID:726, Type:set default value, State:cancelled, SchemaState:none, SchemaID:1, TableID:724, RowCount:0, ArgLen:0   | cancelled |
  | ID:725, Type:create table, State:synced, SchemaState:public, SchemaID:1, TableID:724, RowCount:0, ArgLen:0         | synced    |
  | ID:723, Type:set default value, State:cancelled, SchemaState:none, SchemaID:134, TableID:273, RowCount:0, ArgLen:0 | cancelled |
  | ID:722, Type:set default value, State:cancelled, SchemaState:none, SchemaID:134, TableID:718, RowCount:0, ArgLen:0 | cancelled |
  | ID:721, Type:set default value, State:cancelled, SchemaState:none, SchemaID:134, TableID:718, RowCount:0, ArgLen:0 | cancelled |
  +********************************************************************************************************************+***********+
  10 rows in set (0.19 sec)
  ```

* admin show ddl 在每台机器执行结果不一致
  * 当 OWNER 与 SELF_ID 一致时，就是实际执行 DDL 的 tidb*server
  * 只有当机器挂了或者网络断了才会重新选举 owner

```SQL
MySQL [test]> admin show ddl;
+************+**************************************+******+**************************************+
| SCHEMA_VER | OWNER                                | JOB  | SELF_ID                              |
+************+**************************************+******+**************************************+
|        552 | 43b6103f*63d1*4604*8558*b06eb1abc227 |      | 43b6103f*63d1*4604*8558*b06eb1abc227 |
+************+**************************************+******+**************************************+
1 row in set (0.00 sec)

MySQL [(none)]> admin show ddl;
+************+**************************************+******+**************************************+
| SCHEMA_VER | OWNER                                | JOB  | SELF_ID                              |
+************+**************************************+******+**************************************+
|        552 | 43b6103f*63d1*4604*8558*b06eb1abc227 |      | b56dda44*19ad*4544*9561*5f8cf936e053 |
+************+**************************************+******+**************************************+
1 row in set (0.00 sec)
```

- job 具体执行的语句内容

  ```SQL
  mysql> admin show ddl job queries 44;
  +------------------------------------------------------------------------------------------+
  | QUERY                                                                                    |
  +------------------------------------------------------------------------------------------+
  | CREATE TABLE t3 (id INT NOT NULL AUTO_INCREMENT, age INT, PRIMARY KEY(id)) ENGINE=InnoDB |
  +------------------------------------------------------------------------------------------+
  1 row in set (0.01 sec)
  ```

### 取消当前 DDL

* ADMIN CANCEL DDL JOBS job_id;
