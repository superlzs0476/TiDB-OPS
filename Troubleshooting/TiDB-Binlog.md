---
title: TiDB Binlog 部署以及问题处理
date: 2017-10-02 11:40:52
updated: 2017-10-03 21:40:52
categories:
  - Troubleshooting
tags:
  - PUMP
  - Drainer
  - Backup
---
# TiDB Binlog 部署以及问题处理

> TiDB Binlog 安装部署可查看官方文档 [TiDB-Binlog 部署方案](https://github.com/pingcap/docs-cn/blob/master/tools/tidb-binlog-kafka.md "本文档介绍如何部署 Kafka 版本的 TiDB-Binlog")

- TiDB Binlog 架构

![tidb_binlog_kafka_architecture](/Media/tidb_binlog_kafka_architecture.png)

- TiDB Binlog 工作流
  - 支持全量 (mydumper to TiDB) + 增量备份 (Reparo 读取 PB)

![TiDB Binlog work flow](/Media/tidb_binlog_kafka_backup.png)

## TiDB Binlog 使用注意事项

- PUMP 服务部署前，需要确认 kafka 集群可用，kafka 集群可通过 [kafka-ansible](https://github.com/pingcap/thirdparty-ops/tree/master/kafka-ansible) 部署
- 从 TIDB 使用 mydumper 导出数据时，需要提前调整 [GC 时间](https://github.com/pingcap/docs-cn/blob/master/op-guide/gc.md)
- Drainer 同步支持 replicate do db
- binlog 与 kafka 关系
  - 一个 tidb 对应一个 pump，一个 pump 对应一个 topic
  - pump 的是根据 clusterID + 主机名 + 端口生成的 (topic name：6548960127462523450_tikv01_8250)
  - 2018年7月之后 drainer 支持写入 kafka ，drainer 按集群信息写入 topic

### Drainer 配置文件内参数解释

-disable-dispatch
    是否禁用拆分单个 binlog 的 sqls 的功能，如果设置为 true，则按照每个 binlog
    顺序依次还原成单个事务进行同步（下游服务类型为 mysql，该项设置为 False）

false 的时候，drainer 会并发执行获取到的 dml 语句，在执行之前会更具主键检测是否冲突  
如果是 true 的时候，drainer 是单线程执行 dml 语句，并且是严格按照事务 ID 顺序排序执行  
下游如果是 MySQL，该值设为 true 并没有什么问题，即使语句出现冲突，drainer 也能感知并排序修复。下游如果是 protobuf，一种静态文件方式存储，建议是使用 flase ，这样可以完整一致性回放该 binlog  
下游是 MySQL 的话，执行过程能保持原有顺序，设置为并发模式可以加快消费能力

### 通过 API 接口获取 binlog 信息

- 获取当前最新 binlog 信息
  - `curl http://Pump_IP:Port/status`

  ```BASH
  {"LatestBinlog":{"ip-172-16-1-24:8250":{"suffix":1925},"ip-172-16-1-24:8251":{},"ip-172-16-1-25:8251":{},"ip-172-16-1-26:8250":{"suffix":2071},"ip-172-16-1-26:8251":{}},"CommitTS":391861366903013392,"ErrMsg":""}
  ```

- 获取当前 Drainer 消费信息
  - `curl http://Drainer_IP:Port/status`

-----

## PUMP 日志

- PUMP 文件名
  - binlog-0000000000000000

- sock 文件被占用
  - /data1/ffej/deploy/status/pump.sock 文件被占用

  ```LOG
  2017/11/30 17:31:11 version.go:21: [info] pump Version: 2.0.0+git
  2017/11/30 17:31:11 version.go:22: [info] Git Commit Hash: 5893e217b104f025db924c877616baa9234a6712
  2017/11/30 17:31:11 version.go:23: [info] Build TS: 2017-10-27 07:47:47
  2017/11/30 17:31:11 version.go:24: [info] Go Version: go1.8
  2017/11/30 17:31:11 version.go:25: [info] Go OS/Arch: linuxamd64
  2017/11/30 17:31:11 server.go:119: [info] clusterID of pump server is 6485180256003903346
  2017/11/30 17:31:11 main.go:49: [error] pump server error, fail to start UNIX listener on /data1/ffej/deploy/status/pump.sock: listen unix /data1/ffej/deploy/status/pump.sock: bind: address already in use
  ```

- PUMP 正常创建 binlog file 日志

  ```LOG
  2017/08/06 04:03:16 binlogger.go:285: [info] segmented binlog file /data1/deploy/data.pump/clusters/6444812009423924287/binlog-0000000000000287 is created
  2017/08/06 08:21:59 binlogger.go:223: [info] GC binlog file: /data1/deploy/data.pump/clusters/6444812009423924287/binlog-0000000000000277
  2017/08/06 12:43:29 binlogger.go:285: [info] segmented binlog file /data1/deploy/data.pump/clusters/6444812009423924287/binlog-0000000000000288 is created
  2017/08/06 14:21:59 binlogger.go:223: [info] GC binlog file: /data1/deploy/data.pump/clusters/6444812009423924287/binlog-0000000000000278
  ```

- PUMP 会自己生成 faker binlog（程序 3s 生产一条 faker binlog 放置阻碍其他 PUMP ）

  ```LOG
  2017/08/03 23:59:31 server.go:375: [info] generate fake binlog successfully
  2017/08/03 23:59:34 server.go:375: [info] generate fake binlog successfully
  2017/08/03 23:59:37 server.go:375: [info] generate fake binlog successfully
  2017/08/03 23:59:40 server.go:375: [info] generate fake binlog successfully
  2017/08/03 23:59:43 server.go:375: [info] generate fake binlog successfully
  2017/08/03 23:59:46 server.go:375: [info] generate fake binlog successfully
  ```

- PUMP 推送 binlog 到 Drainer 服务

  ```LOG
  01:37:24 server.go:255: [error] gRPC: pullBinlogs send stream error, rpc error: code = Canceled desc = context canceled
  ```

- PUMP 空间不足
  - `no space left on device`
  - 会造成 TIDB 不可用，TiDB 会丢弃自己的业务端口（熔断机制）
    - TiDB 日志 `listener stopped, waiting for manual kill`

  ```LOG
  2018/01/05 11:50:14 binlogger.go:285: ^[[0;37m[info] segmented binlog file /data1/deploy/data.pump/clusters/64855565788039707
  22/binlog-0000000000000354 is created^[[0m
  2018/01/05 11:54:28 server.go:364: ^[[0;31m[error] generate forward binlog, write binlog err write /data1/deploy/data.pump/cl
  usters/6485556578803970722/binlog-0000000000000354: no space left on device^[[0m
  ```

-----

## Drainer 日志

- Drainer 启动时出现一次, 不影响正常使用

  ```log
  ERRO[0000] [pd] create tso stream error: rpc error: code = Canceled desc = context canceled
  ```

- PUMP binlog file 被删除了
  - PUMP binlog file 被 GC 了，注意 GC 时间
  - PUMP 一个文件 512M，遇见大事务时，会将一个事务写在同一个文件内
  - 文件名字格式 `binlog-0000000000000000`

  ```LOG
  2017/06/21 21:58:05 pump.go:386: [warning] [stream] node ip-172-16-1-24:8251, pos {Suffix:8 Offset:0}, error rpc error: code = 2 desc = binlogger: file not found
  ```

- Drainer 到 PUMP 网络通信
  - drainer 需要去 PUMP 读取消息，有时候因为下游慢，或者 PUMP 没有消息就会超时，
  - 断开重连一下，这个没关系，频率是被控制的

  ```LOG
  2017/06/21 22:02:20 pump.go:386: [warning] [stream] node ip-172-16-1-24:8250, pos {Suffix:5638 Offset:88966933}, error rpc error: code = 2 desc = unexpected EOF
  ```

- crc mismatch 数据检验损坏
  - 检查是否重启或者 kill -9 等操作
  - 处理办法目前只能跳过这段 Binlog 数据操作

  ```LOG
  2018/01/08 15:44:59 pump.go:386: [warning] [stream] node ip-10-0-70-42:8250, pos {Suffix:351 Offset:520960160}, error rpc error: code = Unknown desc = binlogger: crc mismatch
  2018/01/08 15:45:00 pump.go:386: [warning] [stream] node ip-10-0-70-40:8250, pos {Suffix:354 Offset:182450922}, error rpc error: code = Unknown desc = binlogger: crc mismatch
  ```

- 下游数据库 (MySQL)，出现了死锁
  - 检查下游数据库隔离级别与日志信息

  ```LOG
  2018/01/08 19:28:08 syncer.go:517: [fatal] Error 1213: Deadlock found when trying to get lock; try restarting transaction
  /home/pingcap/go/src/github.com/pingcap/tidb-binlog/drainer/executor/util.go:72:
  /home/pingcap/go/src/github.com/pingcap/tidb-binlog/drainer/executor/util.go:52:
  ```

- 同步错误退出
  - 上下游 schema 信息不完整
  - `[error] syncer exited, error schema 306 not found`

## TiDB Binlog FAQ

1. 更新下游的 SQL 是什么？为什么在下游的 MySQL 监控里看到同时有 delete 和 replace？
    - insert 会变成 replace
    - update 和 delete 会变成对应的 update 和 delete
2. 小流量数据的更新机制是什么？一定要凑满 txn*size 吗？有没有超时机制？
    - 和 syncer 处理机制一样， size + timeout
3. TiDB 的 binlog 的生成机制是什么？如果一台 TiDB 不可用 （比如磁盘满了），drainer 就会整个停掉吗？
    - 如果 TiDB 不可用，PUMP 会自己生成 faker binlog（也就是我说的无效数据流量），来推荐排序的进度，不会 block 其他有效 binlog 数据的进度；每 3 s 生成一个
4. binlog 有没有 purge 的机制，防止磁盘被占满？
    - binlog 有自己的 GC 机制以及熔断 (KAFKA 版本)
    - 如果 GC 比较慢，建议加磁盘容量报警
5. 日志里出现的 `error rpc error: code = Unknown desc = unexpected EOFESC`
    - rpc error / Drainer 需要去 PUMP 读取消息，有时候因为下游响应慢，或者 PUMP 没有消息就会超时，断开自动重试相应链接