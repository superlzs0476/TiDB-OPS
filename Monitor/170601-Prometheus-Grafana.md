---
title: Prometheus + Grafana = TiDB 监控架构
date: 2017-06-01 11:40:52
updated: 2017-06-02 21:40:52
categories:
  - Monitor
tags:
  - Prometheus
  - Grafana
  - TiDB
---
# Prometheus + Grafana = TiDB 监控架构

## TiDB 监控架构介绍

### Prometheus 资料片

- Prometheus 资料
  - https://prometheus.io/download/
  - https://prometheus.io/docs/prometheus/latest/querying/operators/
  - https://prometheus.io/docs/prometheus/latest/querying/functions/
  - https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus.yml
  - https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis

### Grafana 资料片

- Grafana 官方资料
  - https://github.com/grafana/grafana
  - http://docs.grafana.org
  - https://grafana.com/grafana/download
    - https://community.grafana.com/c/releases
  - https://grafana.com/plugins

### Prometheus 在 TiDB 的应用

![monitor](https://raw.githubusercontent.com/BigerCAP/tidb-ops/master/Media/180323-syncer-monitor-scheme.png)

- [Grafana](https://github.com/grafana/grafana) 监控、度量分析仪表板工具，从 Prometheus 获取数据
- [Prometheus](https://github.com/prometheus/prometheus) 用于存放监控数据的时序数据库
- [push gateway](https://github.com/prometheus/pushgateway) push acceptor for ephemeral and batch jobs
- [Black-box](https://github.com/prometheus/blackbox_exporter) 黑盒测试工具，支持 HTTP, HTTPS, DNS, TCP and ICMP
- [Node-export](https://github.com/prometheus/node_exporter) 主机资源 agent 组件

### 监控组件端口

服务 | 端口 | 组件介绍
---|---|-----
Grafana           | 3000 | 监控数据展示工具
Prometheus        | 9090 | 支持临时性 Job 主动推送指标的中间网关
Push Gatewagy     | 9091 | 监控数据 push 网关
node_exporter     | 9100 | 主机性能收集组件
blackbox_exporter | 9104 | 黑盒主动探测组件
mysql_exporter    | 9107 | MySQL 状态监测组件
kafka_exporter    | 9105 | kafka 状态监测组件

-----

## Prometheus

### Prometheus 介绍

- 多维数据模型（时序列数据由 metric 名和一组 key/value 组成）
- 在多维度上灵活的查询语言 (PromQl)
- 不依赖分布式存储，单主节点工作.
- 通过基于 HTTP 的 Pull 方式采集时序数据
- 可以通过 Push Gateway 进行时序列数据推送 (pushing)
- 可以通过服务发现或者静态配置去获取要采集的目标服务器
- 多种可视化图表及仪表盘支持

- Pull 方式
  - Prometheus 采集数据是用的 Pull 也就是拉模型, 通过 HTTP 协议去采集指标，只要应用系统能够提供 HTTP 接口就可以接入监控系统，相比于私有协议或二进制协议来说开发、简单。

- Push 方式
  - 对于定时任务这种短周期的指标采集，如果采用 Pull 模式，可能造成任务结束了，Prometheus 还没有来得及采集，这个时候可以使用加一个中转层，客户端推数据到 Push Gateway 缓存一下，由 Prometheus 从 Push Gateway Pull 指标过来。(需要额外搭建 Push Gateway，同时需要新增 job 去从 Gateway 采数据)

![Prometheus-Frame](/Media/170601-1-Prometheus-Frame.svg)




## Grafana


-----

## Prometheus FAQ

### Prometheus 升级

- Prometheus 升级
  - Prometheus 组件由 golang 编写, 直接替换 binary 重启服务即可

- TiDB 曾用版本
  - 1.6
  - 1.8
    - 1.8 与 2.0 数据不兼容
  - 2.0
    - https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis
    - alert.rule 格式为 yml

### Prometheus 监控数据异常

- Prometheus 监控数据异常
  - 数据短暂断崖式下降, 日志中打印 `Storage has entered rushed mode.` 信息
  - 相关 issue
    - https://github.com/prometheus/prometheus/issues/2542
    - https://github.com/coreos/prometheus-operator/issues/304
    - https://github.com/prometheus/prometheus/issues/2936
    - https://github.com/prometheus/prometheus/issues/2222

  ```LOG
  time="2017-10-24T10:08:30+08:00" level=warning msg="Storage has entered rushed mode." chunksToPersist=107071 maxChunksToPersist=524288 maxMemoryChunks=10485
  76 memoryChunks=1139421 source="storage.go:1660" urgencyScore=0.8663654327392578

  time="2017-10-24T10:05:39+08:00" level=info msg="Storage has left rushed mode." chunksToPersist=108398 maxChunksToPersist=524288 maxMemoryChunks=1048576 mem
  oryChunks=1121866 source="storage.go:1647" urgencyScore=0.6989479064941406

  time="2017-10-24T10:05:45+08:00" level=warning msg="Storage has entered rushed mode." chunksToPersist=112409 maxChunksToPersist=524288 maxMemoryChunks=10485
  76 memoryChunks=1268143 source="storage.go:1660" urgencyScore=1
  ```

- 额
  - http://www.jianshu.com/p/36f72490a2a0

- 内存问题?

  ```LOG
  time="2017-08-08T15:38:30+08:00" level=info msg="Starting prometheus (version=1.5.2, branch=master, revision=bd1182d29f462c39544f94cc822830e1c64cf55b)" source="main.go:75"
  time="2017-08-08T15:38:30+08:00" level=info msg="Build context (go=go1.7.5, user=root@a8af9200f95d, date=20170210-14:41:22)" source="main.go:76"
  time="2017-08-08T15:38:30+08:00" level=info msg="Loading configuration file /tidb-data/tidb/deploy/conf/prometheus.yml" source="main.go:248"
  time="2017-08-08T15:38:30+08:00" level=error msg="Error opening memory series storage: found existing files in storage path that do not look like storage files compatible with this version of Prometheus; please delete the files in the storage path or choose a different storage path" source="main.go:182"
  ```

#### 启动速度慢

- Prometheus 启动后，发现网页无法打开，查看日志后，发现 Prometheus 启动后一直在 GC
  - Prometheus 2.0 与 1.0.8 底层存储不一致，2.0 更换为了 RocksDB KV 存储
  - 或者使用 kill -SIGHUP PID 用来重新加载配置文件

  ```bahs
  [tidb@Jeff scripts]$ ./run_prometheus.sh
  level=info ts=2018-04-17T03:04:31.495865947Z caller=main.go:220 msg="Starting Prometheus" version="(version=2.2.0, branch=HEAD, revision=f63e7db4cbdb616337ca877b306b9b96f7f4e381)"
  level=info ts=2018-04-17T03:04:31.495932795Z caller=main.go:221 build_context="(go=go1.10, user=root@52af9f66ce71, date=20180308-16:40:42)"
  level=info ts=2018-04-17T03:04:31.495952002Z caller=main.go:222 host_details="(Linux 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 Jeff (none))"
  level=info ts=2018-04-17T03:04:31.495966987Z caller=main.go:223 fd_limits="(soft=1000000, hard=1000000)"
  level=info ts=2018-04-17T03:04:31.498497082Z caller=main.go:504 msg="Starting TSDB ..."
  level=info ts=2018-04-17T03:04:31.498536225Z caller=web.go:382 component=web msg="Start listening for connections" address=:9090
  level=info ts=2018-04-17T03:05:52.153499505Z caller=main.go:514 msg="TSDB started"
  level=info ts=2018-04-17T03:05:52.153580197Z caller=main.go:588 msg="Loading configuration file" filename=/data1/deploy/conf/prometheus.yml
  level=info ts=2018-04-17T03:05:52.177873304Z caller=main.go:491 msg="Server is ready to receive web requests."
  level=info ts=2018-04-17T03:05:57.320867461Z caller=compact.go:394 component=tsdb msg="compact blocks" count=1 mint=1523923200000 maxt=1523930400000
  level=info ts=2018-04-17T03:06:03.658563096Z caller=head.go:348 component=tsdb msg="head GC completed" duration=227.952838ms
  level=info ts=2018-04-17T03:06:06.403176186Z caller=head.go:357 component=tsdb msg="WAL truncation completed" duration=2.74453888s
  level=info ts=2018-04-17T03:06:06.666573558Z caller=compact.go:394 component=tsdb msg="compact blocks" count=3 mint=1523901600000 maxt=1523923200000
  ```

- 查看 metrics 数据目录大小

    ```bash
    du -sh *

    89G prometheus2.0.0.data.metrics
    ```

- 查看 Prometheus 具体版本信息
  - 查看官方 changelog 信息，在 Prometheus 2.0.2 对启动速度有优化

  ```bash
  [tidb@Jeff deploy]$ ./prometheus --version
  prometheus, version 2.2.0 (branch: HEAD, revision: f63e7db4cbdb616337ca877b306b9b96f7f4e381)
    build user:       root@52af9f66ce71
    build date:       20180308-16:40:42
    go version:       go1.10
  ```

### 监控数据快速清理

- 节点下线后，metric 由残留数据
  - 第一种方案：
    - 停止 Prometheus 与 Push Gateway 服务，然后删除 Prometheus 的所有监控数据，然后启动两个组件。
  - 第二种方案：
    - 人工修改 metric ，按个过滤掉所有已经被下线的节点信息
    - histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket{instance!="6a5114505e5f_4000"}[1m])) by (le, instance))
  - 第三种方案：
    - 目前只能通过 http api 删除历史数据，不能删除 instance 信息
    - 且版本必须高于 2.1.0 以上，官方文档 `https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis`
      - `curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/delete_series?match[]=tidb_server_handle_query_duration_seconds_bucket'`
      - `curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/clean_tombstones'`
