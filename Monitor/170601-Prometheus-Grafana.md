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

![monitor](/Media/180323-syncer-monitor-scheme.png)

- [Grafana](https://github.com/grafana/grafana) 监控、度量分析仪表板工具，从 Prometheus 获取数据
- [Prometheus](https://github.com/prometheus/prometheus) 用于存放监控数据的时序数据库
- [Push Gateway](https://github.com/prometheus/pushgateway) push acceptor for ephemeral and batch jobs
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

1. Prometheus server 定期从静态配置的 targets 或者服务发现的 targets 拉取数据。
1. 当新拉取的数据大于配置内存缓存区的时候，Prometheus 会将数据持久化到磁盘（如果使用 remote storage 将持久化到云端）。
1. Prometheus 可以配置 rules，然后定时查询数据，当条件触发的时候，会将 alert 推送到配置的 Alertmanager。
1. Alertmanager 收到警告的时候，可以根据配置，聚合，去重，降噪，最后发送警告。
1. 可以使用 API， Prometheus Console 或者 Grafana 查询和聚合数据。

#### 注意

- Prometheus 的数据是基于时序的 float64 的值，如果你的数据值有更多类型，无法满足。
- Prometheus 不适合做审计计费，因为它的数据是按一定时间采集的，关注的更多是系统的运行瞬时状态以及趋势，即使有少量数据没有采集也能容忍，但是审计计费需要记录每个请求，并且数据长期存储，这个和 Prometheus 无法满足，可能需要采用专门的审计系统。

### Prometheus 基础概念

> [Prometheus 基础概念](https://songjiayang.gitbooks.io/prometheus/content/concepts/data-model.html)

## Grafana

为 Graphite，InfluxDB 和 Prometheus 等提供漂亮的监控和度量分析和仪表板工具

### 工具介绍

#### 用户权限

Grafana 的权限分为三个等级：Viewer、Editor 和 Admin，Viewer 只能查看 Grafana 已经存在的面板而不能编辑，Editor 可以编辑面板，Admin 则拥有全部权限例如添加数据源、添加插件、增加 API KEY。

#### 数据源（DataSource）

- Graphite
- Elasticsearch
- CloudWatch
- InfluxDB
- OpenTSDB
- Prometheus

#### 面板展示 (Dashbord)

- Dashbord 组成如下：
  - 一个 Dashbord 由多个 row 组成。
  - 一个 row 由 1~n 个 panel 组成。
  - 一个 panel 可以一个 Graph、Singlestat、Table 等。

![Grafna-dashboard 组成](/Media/170601-2-Grafna-dashboard.jpg)

#### 报警（Alert）

Grafana 在 4.0 版本后增加了报警功能，不过 Grafana 的报警属于数据源的后置查询，实时性不大能满足需求。

### 其他

#### 误差

> 该部分引用 [Grafana 的一些使用技巧](https://juejin.im/post/5a9f29e8f265da23783fccae)

这里有一个常见的 Grafana 误区，因为经常有用数值类型的 count_ps(每秒的数量) 来顺便获取每秒打点数量的情况，注意在这种情况下，一段时间内的打点总量需要使用 count_ps(每秒的数量) 的 avg 平均值来乘以这段时间的秒数来计算，而不是通过界面上的 Total 直接读取。

这是因为，在界面上一条曲线能够展示的点的数量是有限的，Grafana 会根据你的窗口宽度来决定返回的点数，因为像一天这样的时间段肯定没办法在界面上展示每一秒的点，毕竟总量为 86400 个点就算带鱼屏也不可能挤得下。对于无法展示的点，Grafana 默认是使用 avg 平均值的行为来修正返回点的值，举个栗子，如下图：

![Grafna](/Media/170601-2-Grafna.jpg)

上图时间范围是一天，上部分为曲线面板的值，下部分为 面饼图表的值，并且上部分图标的曲线为 count 类型（十秒聚一次），可以看到 avg 平均值为 683，那么总量应该为 682 乘以 6 （如果是 count_ps(每秒的数量) 这里则是 60） 乘以 60 （一小时 60 分钟）再乘以 24 （一天 24 小时）得到 589 万，与图片中下部分的 582 万相近，因此上部分 total 的 117 万是一个完完全全让人误解的值，可以认为它毫无意义进而直接无视掉。

我们计算出来的 589 万和界面上的 582 万其实也有一点误差，不过这是可以接受的，因为 statsd 一般情况下是 UDP 的形式（它其实有 TCP 的形式），所以如果想要完全正确的数据，那么最好把打点相关的数据也入库，从数据库里后置查询出来的才是完全可靠。

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
