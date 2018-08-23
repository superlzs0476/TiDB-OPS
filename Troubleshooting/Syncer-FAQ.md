---
title: Syncer-FAQ
date: 2018-01-01 21:40:52
updated: 2018-01-01 21:40:52
toc: true
categories:
  - Troubleshooting
tags:
  - Syncer
  - enterprise tools
---
# Syncer-FAQ

## Syncer 业务逻辑

- 留白

## 受限问题

### Case 1 ：上下游 schema 不一致场景

- 该问题主要出现于分库分表，上游执行 DDL 后，Syncer 出现上下游 schema 信息不一致。原因是上游分表同步进步不一致，没有有效的通过中控机制锁住所有分表的 DDL 执行状态。
- 日志输出如下：

```log
2018/07/18 19:21:17 main.go:79: [error] /home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/syncer.go:529: gen insert sqls failed: insert columns and data mismatch in length: 25 vs 23, schema: CRM, table: ORDER
/home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/syncer.go:165:
2018/07/18 19:21:17 syncer.go:752: [info] print status exits, err:context canceled
```

- 核验步骤
  - 通过 `show create table tablename` 核实上下游列宽
  - 还可以通过 mysqlbinlog 查看 binlog 内容，或者 syncer 开启 Debug 模式查看输出
    - mysqlbinlog 示例如下
      - `mysqlbinlog --base64-output=DECODE-ROWS  mysql-bin.002019 --start-position 430862799 --stop-position 430866627 -vv`

- 解决方案
  - 如果数据量小，可以从新来一遍全量，然后再坐增量
  - 第二种，联系 PingCAP 官方服务，官方会提供一个特殊版本 syncer，去掉检查上下游 schema 功能，此时可以继续同步数据。
    - 去掉检查上下游 schema 后，如果是先做 `alter table A add column`，然后 `alter table A delete column`，此时数据会有影响。

### Case 4 ：启动 Syner 失败，发现 Meta 文件中存在回车符，

- Meta 文件或者配置文件参数有回车符
  - `[error] Near line 3 (last key parsed ''): Strings cannot contain new lines.`

  ```LOG
  2017/12/14 08:36:20 main.go:52: [info] config: log-level:info log-file:/root/prega-syncer/log/card.log log-rotate:day status-addr::11105 server-id:999 worker-count:32 batch:100 meta-file:/root/prega-syncer/meta/card.meta do-tables:[] do-dbs:[~^mmm*] ignore-tables:[] ignore-dbs:[] from:DBConfig(host:10.1.1.14, user:root, port:3306, pass:<omitted>) to:DBConfig(host:10.1.1.25, user:syncer, port:5555, pass:<omitted>) skip-sqls:[] route-rules:[] enable-gtid:true safe-mode:false git-hash: ba3ca1271bb63d58f4d4eecf6d189e353b40440a utc-build-time:[2017-11-23 11:34:40] go-version:go1.9.2
  2017/12/14 08:36:20 main.go:75: [error] Near line 3 (last key parsed ''): Strings cannot contain new lines.
  /home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/meta.go:85:
  /home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/syncer.go:136:
  2017/12/14 08:36:20 metrics.go:107: [info] listening on :11105 for status and metrics report.
  ```

### Case 5 ：数据同步不正常或数据上下游不一致问题检查

- 自检 checklist [TODO]

- Syncer 正常运行，主库有插入数据，备库不同步
  - 检查 binlog 是否存在
  - 检查 binlog 格式
  - 检查 binlog 内容

    ```sql
    MySQL [(none)]> show master logs;
    +------------------+------------+
    | Log_name         | File_size  |
    +------------------+------------+
    | mysql-bin.000023 | 1073742014 |
    | mysql-bin.000024 | 1073742302 |
    | mysql-bin.000025 |  101832651 |
    +------------------+------------+
    ```

- 如何区分 binlog 内容格式为 row 模式
  - 使用 `show master logs;` 查到最新的 binlog file name 。
  - 使用 `show binlog events in 'mysql-bin.000025'  limit 10;`
  - 查看 Event_type 列是否有 `Write_rows` `Update_rows` 等值。

- `SET GLOBAL binlog_format = ROW;` 修改 binlog 格式后，下一个 binlog file 才会生效。
  - 执行 `flush logs;` 手动刷新生成 binlog file 。

- 查看 binlog 文件大小
  - 默认 binlog file size  `show global variables like '%max_binlog_size%';`

    ```sql
    +-----------------+------------+
    | Variable_name   | Value      |
    +-----------------+------------+
    | max_binlog_size | 1073741824 |
    +-----------------+------------+
    ```

------

## 已 fix

### Case 2 ：GTID 开关问题

- UUID 获取问题
  - Syncer 引入 UUID 库版本问题，即时没有开启 enable-gtid，也会解析 binlog 中 UUID 信息，解析数据时失败
  - 与 syncer.meta 文件内 `binlog-gtid = ""` 信息无关

  ```LOG
  2018/01/17 13:58:17 main.go:75: [error] uuid: invalid version number: %!s(uint8=101)
  ```

### Case 3 ：MySQL 上游发现由 Syncer 未关闭的长连接

- MySQL 上游出现由 Syncer 创建的大量长连接
  - 上游持续未插入数据，Syncer 会固定时间进行重连获取最新信息
  - syncer 每隔一个小时会获取一次新的数据，但在建立新连接的时候未关闭老的连接，导致 mysql 出现非常多的长连接。导致业务告警。
  - 主要原因是，syncer 建立连接未进行正确关闭。
  - 日志 tps 持续为 0 ，通过 sql 查询上下游同步正常

  ```LOG
  2017/12/30 00:17:40 syncer.go:803: [info] [syncer]total events = 102214510, total tps = 50, recent tps = 0, master-binlog = (mysql-bin.001244, 66802580), master-binlog-gtid=8c38f844-2985-11e7-a6f2-18ded76e567c:1-1838809963, syncer-binlog = (mysql-bin.001244, 66802580), syncer-binlog-gtid = 8c38f844-2985-11e7-a6f2-18ded76e567c:1-1838809963,b7ff0e7e-2985-11e7-a6f3-18ded76e34de:1-37
  ```

  - 通过 `show processlist` 查看到很多长连接，通过相邻 time 相减得到固定值 3614s ，确定每隔一段时间去检查一次信息。

  ```SQL
  849978  sync_tidb       10.1.1.2:49773        NULL    Binlog Dump GTID        401307  Master has sent all binlog to slave; waiting for binlog to be updated   NULL
  850140  sync_tidb       10.1.1.2:60822        NULL    Binlog Dump GTID        397693  Master has sent all binlog to slave; waiting for binlog to be updated   NULL
  850308  sync_tidb       10.1.1.2:43534        NULL    Binlog Dump GTID        394079  Master has sent all binlog to slave; waiting for binlog to be updated   NULL
  ```

  - 修复办法：杀掉 syncer 进程，然后登陆 MySQL 杀死所有长连接。

## 常规疑问/问题示例

### 通过开启 Syncer Debug 日志，查看 Syncer 运行内容

```LOG
2018/01/13 12:04:12 binlogsyncer.go:98: [info] create BinlogSyncer with config &{100 mysql rm-bp173ev089q70rp03.mysql.rds.aliyuncs.com 3306 g7pay_link_ro nxrwcQ47kdIn   false false <nil> false debug 0}
2018/01/13 12:04:12 metrics.go:107: [info] listening on :10073 for status and metrics report.
2018/01/13 12:04:12 binlogsyncer.go:274: [info] begin to sync binlog from position (mysql-bin.002019, 430862751)
2018/01/13 12:04:12 binlogsyncer.go:158: [info] register slave for master server rm-bp173ev089q70rp03.mysql.rds.aliyuncs.com:3306
2018/01/13 12:04:12 binlogsyncer.go:632: [info] rotate to (mysql-bin.002019, 430862751)
2018/01/13 12:04:12 syncer.go:521: [info] rotate binlog to (mysql-bin.002019, 430862751)
2018/01/13 12:04:12 syncer.go:676: [debug] gtid information: binlog (mysql-bin.002019, 430862799), gtid b470bfb4-bdb4-11e6-a7e5-70106fac1460:1-277917818
2018/01/13 12:04:12 syncer.go:525: [debug] source-db:cat_link table:link_cat; target-db:cat_link table:link_cat, RowsEvent data: [[E78D56BBBC2357PPP9687CACD6EBD2AC 区域经营 - 华南 118OO6009 华中 - 桂琼 - 柳州门店 118006009025OO1 200O6Q 200O6Q 广西TiDB储物流股份有份公司 陈王 120123ABCD149082 桂 A88888 0 2 梁王 18277238888 <nil> 0000-00-00 00:00:00 0 0 0000-00-00 00:00:00 小芳 lij 0 1 2018-01-12 21:10:09 <nil> 2  291464]]
2018/01/13 12:04:12 db.go:55: [debug] [query][sql]SHOW COLUMNS FROM `cat_link`.`link_cat`
2018/01/13 12:04:12 db.go:55: [debug] [query][sql]SHOW INDEX FROM `cat_link`.`link_cat`
2018/01/13 12:04:12 syncer.go:882: [info] flush all jobs meta = syncer-binlog = (mysql-bin.002019, 430862751), syncer-binlog-gtid = b470bfb4-bdb4-11e6-a7e5-70106fac1460:1-277917817
2018/01/13 12:04:12 meta.go:120: [info] save position to file, binlog-name:mysql-bin.002019 binlog-pos:430862751 binlog-gtid:b470bfb4-bdb4-11e6-a7e5-70106fac1460:1-277917817
2018/01/13 12:04:12 main.go:75: [error] /home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/syncer.go:556: gen insert sqls failed: insert columns and data mismatch in length: 27 vs 28, schema: cat_link, table: link_cat
/home/jenkins/workspace/build_tidb_enterprise_tools_master/go/src/github.com/pingcap/tidb-enterprise-tools/syncer/syncer.go:143:
2018/01/13 12:04:12 syncer.go:795: [info] print status exits, err:context canceled
2018/01/13 12:04:12 binlogsyncer.go:126: [info] syncer is closing...
2018/01/13 12:04:12 binlogstreamer.go:47: [error] close sync with err: sync is been closing...
2018/01/13 12:04:12 binlogsyncer.go:141: [info] syncer is closed
```

### Syncer 日志内的 TiDB 消息日志

- Syncer 是作为上游的服务端，获取数据转换后，作为下游的客户端插入数据，以下信息是在插入数据的时候遇见错误接受到的返回
- Syncer 日志内显示以下信息，实际是 Syncer 下游 TiDB 返回的错误信息

  ```LOG
  begin failed Error 1105: [try again later]: backoffer.maxSleep 5000ms is exceeded, errors:
  get timestamp failed: rpc error: code = Unknown desc = rpc error: code = Unavailable desc = not leader
  get timestamp failed: rpc error: code = Unavailable desc = grpc: the connection is unavailable
  get timestamp failed: rpc error: code = Unavailable desc = grpc: the connection is unavailable
  db.go:100: [warning] sql stmt_exec retry 90
  ```

### Syncer 运行状态

- 检查 syncer 是否还在运行
- 查看 syncer 最后 50 行日志，根据日志提示修复
- 提示 `2017/08/01 08:28:59 binlogsyncer.go:554: [error] connection was bad`
- tidb 链接 源库网络出现异常，tidb 会重试一段时间后退出