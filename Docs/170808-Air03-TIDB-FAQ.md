---
title: TiDB 业务场景问题收集与排查思路
date: 2017-08-01 21:40:52
updated: 2017-08-08 21:40:52
categories:
  - TiDB
tags:
  - TiDB
  - Online DDL
---
# TiDB 业务场景问题收集与排查思路

## 运维场景

### TiDB 忘记密码

- TiDB 忘了密码怎么办
  - TiDB [权限管理](https://github.com/pingcap/docs-cn/blob/master/sql/privilege.md)

- 在集群中任选一台 TiDB 节点，修改配置文件添加以下内容；使用 Linux 系统 root 用户启动该 TiDB 节点
  - 重启后使用 `mysql -h 127.0.0.1 -u root` 直连登陆该节点，执行语句修改密码

  ```toml
  [security]
  skip-grant-table = true
  ```

- 如果非 Root 用户启动，会出现以下 log
  - `level=error msg="TiDB run with skip-grant-table need root privilege."`

### TiDB 查看 table 占用磁盘大小

- 可以通过 tidb-api 查看相关信息，可以用最新的 master 来测试
- 通过 tidb-api 查到数据, 只是说明该表有数据在这些 region, 并不一定是占满该 region
- master 版本拥有 spilt-table，确保该 region 内没有其他 table 数据
  - [TiDB API 说明文档](https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md)
  - [TiDB API Code](https://github.com/pingcap/tidb/blob/master/server/http_status.go)

### 磁盘空间回收慢

- 空间回收工作
  - drop truncated delete
  - GC job [history-read](https://github.com/pingcap/docs-cn/blob/master/op-guide/history-read.md)
  - tikv compact
- drop table
  - 删除 schema 中 这个 table 的信息
  - 把 table 数据的 keyrange 记录在一个表中
  - 读取这个表，向 tikv 发送 gc 命令，用 delete in range 功能删除一大片数据
  - 最终删除数据用的是 rocksdb 的 delete range / delete file in range 功能

- 新集群(release 2.0+)部署，部署前打开该参数

    ```TOML
    [rocksdb.defaultcf]
    dynamic-level-bytes = true
    [rocksdb.writecf]
    dynamic-level-bytes = true
    ```

- 老集群可以使用 tikv-ctl 手动执行 compaction
  - tikv-ctl 手动 compaction 会造成磁盘利用率上涨
  - `tail -f /tikv/data/db/LOG` 看看是不是在不停的做 compact

    ```bash
    tikv-ctl compact 的具体操作执行
    ./tikv-ctl --host={tikv-ip:tikv-port} compact
    ./tikv-ctl --host={tikv-ip:tikv-port} compact -c write 过几秒然后 ctrl+c
    ./tikv-ctl --host={tikv-ip:tikv-port} compact -c default
    ```

- 并发 GC ， 需要升级到 1.0.8 或者 1.1 版本之后
  - `update mysql.tidb set VARIABLE_VALUE="8" where VARIABLE_NAME="tikv_gc_concurrency";`
- 其中 30m 代表仅清理 30 分钟前的数据，这可能会浪费一定额外的存储空间。
  - `update mysql.tidb set variable_value='30m' where variable_name='tikv_gc_life_time';`
- 查看当前 gc leader 在哪个 tidb 上
  - `select * from mysql.tidb  where variable_name='tikv_gc_leader_desc';`
- 到所在节点看日志是否有做 delete range
  - `cat tidb.log |grep -e ""gc worker"" |grep "ranges"`
- metrics
  - `tikv->GC->GC Worker Actions` 看是否有 delete range

## 问题收集场景

### TiDB 没有响应检查思路

- Sql 没有响应，是指客户端连接不上么？
- 还是执行 sql 没有响应？
- tidb 僵死
- 如果部署了 pump，并且 tidb 和 pump 之间链接断了，那么会出现这种状态
  - Pump 如果访问不上，tidb 是会进入保护状态，那执行 sql 应该会报错而不是 hang 住吧
  - 会出现 connection refused , 可以尝试创建一个新的连接
- 获取 tidb debug 信息
  - curl http://127.0.0.1:10080/debug/pprof/goroutine?debug=1 > tidb.goroutine.debug1
  - curl http://127.0.0.1:10080/debug/pprof/block?debug=1 > tidb.block.debug1
  - curl http://127.0.0.1:10080/debug/pprof/heap?debug=1 > tidb.heap.debug1

### 慢查询

- .99 延迟高可能性
  - 主机相关
    - CPU 占用过高
    - DISK dd 4k 测试在 60M 以下
    - 网络？
  - SQL 相关
    - 事务冲突
    - 查询或者 update 时候 plan 不合理导致消耗无用的资源 拖慢系统
    - 查询或者写入有明显的热点
- 检查慢查询思路
  - analyze
  - explain query / schema / sql query
  - int 与 varchar
- 如果查到慢查询语句
  - 每个 session 练上来的时候  会获取一些 session variable 值
  - `grep "slow-query" tidb.log -rnI | grep select`

  ```SQL
  2455102:2017/11/08 15:17:45.937 adapter.go:293: [warning] slow-query connectionId=0 costTime=304.700712ms sql=select HIGH_PRIORITY * from mysql.global_variables where variable_name in ('autocommit', 'sql_mode', 'max_allowed_packet', 'tidb_skip_utf8_check', 'tidb_index_join_batch_size', 'tidb_index_lookup_size', 'tidb_index_lookup_concurrency', 'tidb_index_serial_scan_concurrency', 'tidb_max_row_count_for_inlj', 'tidb_distsql_scan_concurrency') type=slow-query succ=true
  ```

- [analyze 执行](#线上业务手动-analyze)
- analyze 后思路
  - analyze 结束，进行 explain。
  - 想办法让客户的 sql explain 走对，比如先 analyze 对应的 index，如果执行这步就 ok 了，咱们再慢慢推升级 ananlyze 的问题
  - 调高并行做 analyze 的参数，比如 200，执行一会取消或者 oom，有了直方图，sql 的 explain 就对了。
  - analyze isShow,notRate,type,disputeCount 这四个 index

### 热点数据处理

- 集群中某 tikv coprocessor 进程占用 CPU 比其他节点
  - 检查各 tidb.log
  - 搜索 COP_TASK / regions()
  - `grep TIME_COP_TASK tidb.log | awk '{print $7}' | grep region | awk -F"(" '{print $2}' | sort | uniq -c | sort -n | less`
- 需要的是 sql 比如 connect id 或者 handleid 与 kv region 的映射关系
- 这个问题是，首先需要命中 热点 region,根据热点region 当前日志解析 startkey 获取 tid,然后根据 tid 反查表信息，根据表信息再去 tidb 捞 日志
