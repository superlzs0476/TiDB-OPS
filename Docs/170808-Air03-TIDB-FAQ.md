---
title: TiDB 业务场景问题收集与排查思路
date: 2017-06-01 21:40:52
updated: 2017-06-06 21:40:52
categories:
  - TiDB
tags:
  - TiDB
  - Online DDL
---
# TiDB 业务场景问题收集与排查思路

## 步骤

- tidb 对于 range scan，大范围扫描的查询有优化么？

- 支持 binary 方式写入

- 使用 goldengate 将数据从 oracle 复制到 tidb,
  - ogg 执行登陆代码： dblogin sourcedb 时报错 ERROR OGG-00768 The Value '2' of system variable lower_case_table_names is not supported. SQL error (0). 不知道怎么解决？
  - 同样命令配置在 mysql5.5 没问题，数据可以成功同步。
  - tidb 针对 lower_case_table_names 无法修改，因此失败

- multi-statement 的区别
  - 事务内是 insert into xxx values(),(),(),..... ;
  - 还是 insert into xxx ; insert into xxx ;
    - create table xx (id int,age int) engine=InnoDB;
    - insert into xx values(1,1);insert into xx values(2,2);insert into xx values(b,3);

- 使用 docker-compose 启动一个 tidb 集群测试
  - 该集群内包含 `tidb - 1、tikv - 3 、pd - 3 、Prometheus - 1 、Grafana - 1、 pushgateway - 1`
  - 可以使用 mac pro 中高配以上或相同配置机器启动。
  - https://github.com/pingcap/tidb-docker-compose
    - 出现 ulimit 问题，应该添加 ulimit 模块
    - ulimit

## 文档连接

- 火焰图
  - https://pingcap.com/blog-cn/flame-graph/

- release note
  - https://github.com/pingcap/tidb/releases

- spark 使用相关文档
  - https://github.com/pingcap/docs-cn/blob/master/tispark/tispark-user-guide.md

- tidb-http api
  - https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md
  - https://github.com/pingcap/tidb/blob/master/server/http_status.go

- tidb 所有变量
  - https://github.com/pingcap/tidb/blob/master/sessionctx/variable/tidb_vars.go

- REGION 自动平衡
  - pd-ctl
  - region 能控制自动调度吗，一般这样的调度都放到凌晨后
  - https://github.com/pingcap/docs-cn/blob/master/tools/pd-control.md

- analyze
  - 可以查看到非精确值的 count
  - https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md


- 这个错误的原因，是事务执行时间，超过 GC 间隔时间 (10m 钟)
  - https://github.com/pingcap/docs-cn/blob/master/op-guide/history-read.md

- alter table hot_spot shard_row_id_bits = 15 (分片数量 32768)
  - https://github.com/pingcap/tidb/commit/04ef7d79928ac27f6fab53c944efbb523f896755#diff-caef8063af52c86ddcdb638aff3e5bd9

***

## TiDB 忘记密码

- tidb 忘了密码怎么办？
  - https://github.com/pingcap/docs-cn/blob/master/sql/privilege.md

- 选一台 TiDB 节点，修改配置文件添加以下内容；使用 Linux 系统 root 用户启动该 TiDB 节点
  - 重启后使用 `mysql -h 127.0.0.1 -u root` 登陆

  ```toml
  [security]
  skip-grant-table = true
  ```

- 如果非 Root 用户启动，会出现以下 log
  - `level=error msg="TiDB run with skip-grant-table need root privilege."`

## 关闭 delete tange

- 这个是 rocksdb delete range 的一个问题，tikv 现在默认开了 delete range
- 目前的解决方案是先把 use-delete-range 设为 false，然后用 tikv-ctl 进行 compact

1. 修改 tidb-ansible/conf/tikv.yml 文件中字段
  * use-delete-range = false
2. 执行 ansible-playbook rolling_update.yml --tags=tikv 滚动升级 tikv 组件(通过这个步骤更新配置文件与重启 tikv)
3. 在 tidb-ansible/resource/bin 下找到 tikv-ctl
  * 执行 ./tikv-ctl --host={tikv-ip:tikv-port} compact

***

## 场景

- 还有一点和大家同步一下，TiDB 只支持大小写敏感的字符集校验，SQL 里面的值要和表内数据大小写完全一致才能筛选出数据，所以这里只要注意一下 SQL 里面传值的大小写就好了，举个例子比如表 a 字段值有 ABC Abc abc 三条 ，SQL 里用 where a='Abc' 就只能筛出来一条，where a like '%abc%' 也是一样区分大小写的

### Tidb 查看 table 占用磁盘大小

- 可以通过 tidb-api 查看相关信息，可以用最新的 master 来测试，1.0 的功能不全
- 另外这个查到数据, 只是说明该表有数据在这些 region, 并不一定是占满该 region . 该表在 region 内占用的数据内容是不确定的.
- master 版本拥有 spilt-table，确保该 region 内没有其他 table 数据
  - https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md
  - https://github.com/pingcap/tidb/blob/master/server/http_status.go

### optimizer

- Optimizer Hint 文档 [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/tidb-specific.md)
- optimizer 的目前只有两个，TIDB_SMJ(table_name1, table_name2...) 和 TIDB_INLJ(table_name1, table_name2...)

  ```SQL
  explain select /*+ TIDB_INLJ(job_exposure_detail) */ * from job_exposure_detail left join TiResumeScore on job_exposure_detail.user_id = TiResumeScore.userId where job_no = 'CC595492326J90257235000' order by TiResumeScore.totalScore ;
  ```

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

- 如果是新集群，打开该参数就可以了

    ```TOML
    [rocksdb.defaultcf]
    dynamic-level-bytes = true
    [rocksdb.writecf]
    dynamic-level-bytes = true
    ```

- 如果是老集群，暂时不要打开该参数，以防引发太多的 compaction 影响线上业务。
- 对于已有的老集群，如果有用户空间问题比较严重，可以使用 tikv-ctl 先把所有 tikv 的数据全量 compact 一下之后打开该参数。
- tail -f /tikv/data/db/LOG 看看是不是在不停的做 compact

    ```bash
    tikv-ctl compact 的具体操作执行
    ./tikv-ctl --host={tikv-ip:tikv-port} compact
    ./tikv-ctl --host={tikv-ip:tikv-port} compact -c write 过几秒然后 ctrl+c
    ./tikv-ctl --host={tikv-ip:tikv-port} compact -c default
    ```
- 疑问
  - release 108 支持 tikv-ctl 手动 compaction 么？ // 支持
  - dynamic-level-bytes  与 tikv-ctl 手动 compaction 是否有关系？ // 没有关系
  - tikv-ctl 手动 compaction 与 tikv 自动 compaction 逻辑一致么？ // 逻辑一致，一个是自动，一个是手动
  - tikv-ctl 手动 compaction 效果咋样，影响是啥 // 磁盘利用率会上涨

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

### 热点数据统计

- coprocessor 进程占用 CPU 异常之高
  - 检查各 tidb.log
  - 搜索 COP_TASK / regions()
  - `grep TIME_COP_TASK tidb.log | awk '{print $7}' | grep region | awk -F"(" '{print $2}' | sort | uniq -c | sort -n | less`

### MVCC 获取

- MVCC
  - http://10.30.1.24:10080/tables/mbk_pay/mbk_refund/regions
    - 这个可以查到 table id 和 index id
  - http://tidb.host:10080/mvcc/key/mbk_pay/mbk_refund/24936763
  - http://tidb.host:10080/mvcc/key/{db}/{table}/{recordID}
    - 可以看到 这条 handle 的 mvcc 信息
  - http://10.30.1.32:10080/tables/mbk_pay/mbk_refund/regions
    - 那个 region 的 start key 是 table 199 的 row，end key 是 table 212 的 index

***

## 隔离级别

- [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/transaction-isolation.md)

tidb set 支持 SESSION 与 GLOBAL

SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED

GLOBAL 的话 所有的隔离级别都是这个低级别的了
建议只有只读的用这个隔离级别
因为低级别不一定能满足你们业务的要求
这个可以自行评估
不是  我的意思是看你的业务对隔离级别的要求
比如 想统计一下当前有多少人在线  那么用 RC 没什么问题
但是如果是银行转账，那么 RC 隔离级别肯定不够
嗯 这个场景主要是用户订阅 其实数据快慢没有特别实时的要求

## 直方图

- [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md)
- 目前无法自行运行 analyze，因 auto analyze 条件苛刻
  - 5 * lease 会定时更新 STATS_META , 连续 4 次没变化 , 执行 auto analyze table tablename
  - lease 为 tidb.toml 文件中 `-stats-lease` 参数
- analyze 相关命令，通过以下命令查看统计信息
  - 通过 SHOW STATS_META 来查看表的总行数以及修改的行数等信息。
  - 通过 SHOW STATS_HISTOGRAMS 来查看列的不同值数量以及 NULL 数量等信息。
  - 通过 SHOW STATS_BUCKETS 来查看直方图每个桶的信息。
  - 通过 DROP STATS tableName 可以删除统计信息
    - 删除统计信息，操作会非常块
    - 删除后 auto analyze 不对这个表有操作. 需要对这张表重新手动执行后才会 auto analyze
- analyze 更新信息
    `show stats_histograms 里有 update time`

### 查看某个表直方图统计信息

- 通过以下命令查看某个表直方图统计信息
  - table_name 填这个 table 的名字，column_name 填索引的名字。

  ```SQL
  SHOW STATS_BUCKETS where table_name = "objects_tia";
  SHOW STATS_BUCKETS where table_name = "objects_tia" and column_name="PRIMARY";
  show stats_histograms where table_name=order_join_totalpay
  ```

### 线上业务手动 analyze

- analzye 使用 Read Commited  隔离级别，时间戳是 max int64，调整扫描数据并发度
  - 关于 session 变量解释看 [docs-cn](https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md)

  ```SQL
  set @@session.tidb_build_stats_concurrency=1;
  set @@session.tidb_distsql_scan_concurrency=4;
  set @@session.tidb_index_serial_scan_concurrency=4;
  analyze table tableName;
  ```

### analyze 与 GC 问题

- analyze 在 1.0.6 之前版本，受 GC 影响；1.0.6 之后版本已经修复; 以下为具体报错
  - 2018 年 1 月 8 日 fix

  ```SQL
  MySQL [lock_event_log]> analyze table mbk_lock_event;
  ERROR 1105 (HY000): start timestamp fall behind safe point
  ```

***

## 其他

- stale 这个是 region 在一直分裂吧，你 grep -v 之后，看看 takes 的 log
  - grep takes tikv.log | grep scheduler
- 因为现在 batch get 是没有 hot read 调度的，所以我认为一个可行的办法就是 grep 下最近一段时间的 batch get 慢日志，把里面的 region 给弄出来，然后找到几个热点 region，手动调度
- 但是 scheduler pending commands 总是堆在一个 tikv 上

### Goroutine

- goroutine 数量越多越快，join-concurrency 有没有相应的 环境变量 来设置？
- statsLease 参数的在自动更新统计信息的时候，是不是对内存开销很大？分析的时候是不是要对全表扫描？ 和 analyze table 有关系么？
- tcp-keep-alive 是布尔值么？如果不是，那么与上层 LB tcp keepalive 值是否需要设置成一样？
- 也看数据量和数据分布情况，一定情况下是越多越好。没有相应的环境变量
- 内存应该不是大问题，但是要全表扫。analyze table 是用户主动分析这个表的统计信息
- 是布尔值

### 修改 tidb 到 tikv grpc 时间

- tidb RC 级别的版本，新版本拥有流控，不会出现类似问题
  - 怀疑遇见了大 region ，然后 tikv 收集 reigon 信息超过了 30s ，因此失败
  - 新版本时获取到一批数据，发送一批数据，而不是整体获取完成后再发送相关信息
- 这个变量，现在是 60s，把这个改长
  - store/tikv/client.go
  - ReadTimeoutMedium = 60 * time.Second

  ```LOG
  ** (mydumper:11077): CRITICAL **: Could not read data from ty2_lb1_net_ip_4.i_ip_url_2: [try again later]: backoffer.maxSleep 20000ms is exceeded, errors:
  send tikv request error: rpc error: code = DeadlineExceeded desc = context deadline exceeded, ctx: region_id:17793 region_epoch:<conf_ver:3 version:2752> peer:<id:17795 store_id:4 > , try next peer later
  stale_epoch:<>
  send tikv request error: rpc error: code = DeadlineExceeded desc = context deadline exceeded, ctx: region_id:17793 region_epoch:<conf_ver:3 version:2752> peer:<id:17795 store_id:4 > , try next peer later
  ```