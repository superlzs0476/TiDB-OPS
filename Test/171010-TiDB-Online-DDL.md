---
title: TiDB Online DDL 测试
date: 2017-10-10 21:40:52
updated: 2017-10-10 21:40:52
categories:
  - TiDB
tags:
  - TiDB
  - Online DDL
---
# TiDB Online DDL 测试

- [DDL](https://github.com/pingcap/tidb/blob/master/model/ddl.go) 代码块

## DDL 测试

### 支持的 DDL type

- [DDL type](https://github.com/pingcap/tidb/blob/master/ddl/ddl_worker.go#L344)

```go
	switch job.Type {
	case model.ActionCreateSchema:
		ver, err = d.onCreateSchema(t, job)
	case model.ActionDropSchema:
		ver, err = d.onDropSchema(t, job)
	case model.ActionCreateTable:
		ver, err = d.onCreateTable(t, job)
	case model.ActionDropTable:
		ver, err = d.onDropTable(t, job)
	case model.ActionAddColumn:
		ver, err = d.onAddColumn(t, job)
	case model.ActionDropColumn:
		ver, err = d.onDropColumn(t, job)
	case model.ActionModifyColumn:
		ver, err = d.onModifyColumn(t, job)
	case model.ActionAddIndex:
		ver, err = d.onCreateIndex(t, job)
	case model.ActionDropIndex:
		ver, err = d.onDropIndex(t, job)
	case model.ActionAddForeignKey:
		ver, err = d.onCreateForeignKey(t, job)
	case model.ActionDropForeignKey:
		ver, err = d.onDropForeignKey(t, job)
	case model.ActionTruncateTable:
		ver, err = d.onTruncateTable(t, job)
	case model.ActionRebaseAutoID:
		ver, err = d.onRebaseAutoID(t, job)
	case model.ActionRenameTable:
		ver, err = d.onRenameTable(t, job)
	case model.ActionSetDefaultValue:
		ver, err = d.onSetDefaultValue(t, job)
	case model.ActionShardRowID:
		ver, err = d.onShardRowID(t, job)
	case model.ActionModifyTableComment:
		ver, err = d.onModifyTableComment(t, job)
```

### DDL 工作过程

- DDL 是串行工作，每次工作前会先选择集群内任意 TiDB 作为 OWNER 节点执行 DDL Job ，集群内 TiDB OWNER 唯一
  - 使用客户端直连 TiDB-server，执行 `admin show ddl`，观察 OWNER 与 SELF_ID 是否一致，一致为 OWNER 节点
  - 只有当机器挂了或者网络断了才会重新选举 OWNER
  - 也可以使用 TiDB 配置文件中 run-ddl 参数控制该节点是否参与 OWNER 选举

  ```SQL
  mysql> admin show ddl;
  +------------+--------------------------------------+------+--------------------------------------+
  | SCHEMA_VER | OWNER                                | JOB  | SELF_ID                              |
  +------------+--------------------------------------+------+--------------------------------------+
  |         23 | 75be91ca-8242-4b35-bf6e-5690d20d4bef |      | 75be91ca-8242-4b35-bf6e-5690d20d4bef |
  +------------+--------------------------------------+------+--------------------------------------+
  1 row in set (0.00 sec)
  ```

  - 可以过滤每个 TiDB-server 节点的日志，日志内容如下

  ```log
  2017/08/03 23:37:35 owner_manager.go:176: [info] [ddl] /TiDB/ddl/bg/owner ownerManager d800281c-2abd-47ff-81bf-910800d5be0a etcd session is done, creates a new one
  ```

- DDL 由 OWNER 执行后，正式进入以下流程
  - DDL 执行的 4 个过程, Table Info 一直更新，相邻 Schema Version 之间可以共存；Public 状态后，DDL 工作完成
    - SchemaState:none
    - delete only(model.StateDeleteOnly)
    - write only(model.StateWriteOnly)
    - public(model.StatePublic)
  - InfoSchema 加载过程

  ```log
  2017/07/17 21:50:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 1432 to 1433, in 1.193129ms
  2017/07/17 21:50:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 1433 to 1434, in 1.103645ms
  2017/07/17 21:50:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 1434 to 1435, in 1.82274ms
  2017/07/17 21:50:23 domain.go:93: [info] [ddl] diff load InfoSchema from version 1435 to 1436, in 1.343882ms
  2017/07/21 17:14:00 domain.go:354: [info] [ddl] on DDL change, must reload
  ```

- 如果是 add index ，需要考虑表的大小，并且考虑 自增 ID 是否完全连续
- 如果是 create table / databases 时间长，需要检查 DDL job 是否出现排队
- drop / truncate DDL ，执行时间快，但是影响后续磁盘 IO，因为需要做 compaction

### DDL 查询与管理

- [admin](https://github.com/pingcap/docs-cn/blob/master/sql/admin.md) 管理命令介绍

- 使用 `admin show ddl` 查看当前运行 DDL 信息与 OWNER ID

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

- 使用 `admin show ddl jobs` 查看更多信息；改命令在每台机器执行结果相同

- 通过 job_id 查询 job 执行的语句内容
  - 该功能需要 2.0RC4 以上版本

  ```SQL
  mysql> admin show ddl job queries 44;
  +------------------------------------------------------------------------------------------+
  | QUERY                                                                                    |
  +------------------------------------------------------------------------------------------+
  | CREATE TABLE t3 (id INT NOT NULL AUTO_INCREMENT, age INT, PRIMARY KEY(id)) ENGINE=InnoDB |
  +------------------------------------------------------------------------------------------+
  1 row in set (0.01 sec)
  ```

- 取消当前 DDL
  - job_id 在 `admin show ddl jobs` 中获取
  - `ADMIN CANCEL DDL JOBS job_id;`

*****

## DDL FAQ

- 命令或者关键词
  - admin show ddl jobs; // 查看 DDL 历史记录
    - 可以查看到 ddl owner ，在 tidb.log 搜索 ddl owner ，有则是 owner 归属
    - 需要配合 API 查看 schema 信息
  - grep "finish DDL job ID:2092" tidb.log
    - 查看已经成功的 DDL ，需要先查到 ddl owner
  - CRUCIAL OPERATION  DDL 信息关键词 // 忘记了

- loading schema 时间过长

  ```log
  2018/04/09 17:40:38.079 domain.go:319: [warning] [ddl] loading schema takes a long time 7.564456596s
  ```

- loading schema 失败 [schemaValidator](https://github.com/pingcap/tidb/blob/master/domain/schema_validator.go)

  ```log
  the schema validator's latest schema version 944 is expired
  ```

## 各类型 DDL 语句测试记录

> 测试版本为 TiDB 2.0

- 创建 删除 Database
- 创建 删除 Table
- 添加 删除 修改 Column
- 创建 删除 Index
- 创建 删除 ForeignKey (目前版本不支持外键)
- Truncate Table
- Rename Table
- Modify Table Comment

### Create Table

```LOG
2017/08/15 16:11:35 ddl.go:421: [info] [ddl] start DDL job ID:34, Type:create table, State:none, SchemaState:none, SchemaID:29, TableID:33, RowCount:0, ArgLen:1, Query:
CREATE TABLE world.t2 (id INT, name VARCHAR(256), PRIMARY KEY(id)) ENGINE=InnoDB
2017/08/15 16:11:35 ddl_worker.go:253: [info] [ddl] run DDL job ID:34, Type:create table, State:none, SchemaState:none, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:11:35 domain.go:93: [info] [ddl] diff load InfoSchema from version 16 to 17, in 2.471353ms
2017/08/15 16:11:35 ddl_worker.go:359: [info] [ddl] wait latest schema version 17 changed, take time 52.091016ms
2017/08/15 16:11:35 ddl_worker.go:140: [info] [ddl] finish DDL job ID:34, Type:create table, State:synced, SchemaState:public, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:11:35 ddl.go:457: [info] [ddl] DDL job 34 is finished
2017/08/15 16:11:35 domain.go:365: [info] [ddl] on DDL change, must reload
```

### Drop Table

```LOG
2017/08/15 16:22:22 ddl.go:421: [info] [ddl] start DDL job ID:35, Type:drop table, State:none, SchemaState:none, SchemaID:29, TableID:33, RowCount:0, ArgLen:0, Query:
drop table world.t2
2017/08/15 16:22:22 ddl_worker.go:253: [info] [ddl] run DDL job ID:35, Type:drop table, State:none, SchemaState:none, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:22:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 17 to 18, in 1.187145ms
2017/08/15 16:22:22 ddl_worker.go:359: [info] [ddl] wait latest schema version 18 changed, take time 52.04002ms
2017/08/15 16:22:22 ddl_worker.go:253: [info] [ddl] run DDL job ID:35, Type:drop table, State:running, SchemaState:write only, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:22:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 18 to 19, in 999.369µs
2017/08/15 16:22:22 ddl_worker.go:359: [info] [ddl] wait latest schema version 19 changed, take time 51.84345ms
2017/08/15 16:22:22 ddl_worker.go:253: [info] [ddl] run DDL job ID:35, Type:drop table, State:running, SchemaState:delete only, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:22:22 domain.go:93: [info] [ddl] diff load InfoSchema from version 19 to 20, in 1.228868ms
2017/08/15 16:22:22 ddl_worker.go:359: [info] [ddl] wait latest schema version 20 changed, take time 51.969018ms
2017/08/15 16:22:22 delete_range.go:253: [info] [ddl] insert into delete-range table with key: (35,33)
2017/08/15 16:22:22 delete_range.go:93: [info] [ddl] add job (35,drop table) into delete-range table
2017/08/15 16:22:22 ddl_worker.go:140: [info] [ddl] finish DDL job ID:35, Type:drop table, State:synced, SchemaState:none, SchemaID:29, TableID:33, RowCount:0, ArgLen:0
2017/08/15 16:22:22 ddl.go:457: [info] [ddl] DDL job 35 is finished
2017/08/15 16:22:22 domain.go:365: [info] [ddl] on DDL change, must reload
```

### ALTER Table Modify Column

  ```LOG
  2018/01/05 10:31:37 ddl.go:410: [info] [ddl] start DDL job ID:540, Type:modify column, State:none, SchemaState:none, SchemaID:530, TableID:532, RowCount:0, ArgLen:3, Query:
  ALTER TABLE `ip_banner`
  MODIFY COLUMN `timestamp`  varchar(80) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '' AFTER `ip`
  2018/01/05 10:31:37 ddl_worker.go:262: [info] [ddl] run DDL job ID:540, Type:modify column, State:none, SchemaState:none, SchemaID:530, TableID:532, RowCount:0, ArgLen:0
  2018/01/05 10:31:38 domain.go:93: [info] [ddl] diff load InfoSchema from version 456 to 457, in 3.475215ms
  2018/01/05 10:31:38 ddl_worker.go:368: [info] [ddl] wait latest schema version 457 changed, take time 72.370341ms
  2018/01/05 10:31:38 ddl_worker.go:117: [info] [ddl] finish DDL job ID:540, Type:modify column, State:synced, SchemaState:public, SchemaID:530, TableID:532, RowCount:0, ArgLen:0
  2018/01/05 10:31:38 ddl.go:447: [info] [ddl] DDL job 540 is finished
  2018/01/05 10:31:38 domain.go:365: [info] [ddl] on DDL change, must reload
  ```