---
title: Grafana & Prometheus 监控组件问题收集与排查思路
date: 2017-09-09 21:40:52
updated: 2017-09-09 21:40:52
categories:
  - Monitor
tags:
  - Grafana
  - Prometheus
  - Node_exporter
---
# Grafana & Prometheus 监控组件问题收集与排查思路

## 监控架构

![monitor](https://raw.githubusercontent.com/BigerCAP/tidb-ops/master/Media/180323-syncer-monitor-scheme.png)

- [Grafana](https://github.com/grafana/grafana) 监控、度量分析仪表板工具，从 Prometheus 获取数据
- [Prometheus](https://github.com/prometheus/prometheus) 用于存放监控数据的时序数据库
- [push gateway](https://github.com/prometheus/pushgateway) push acceptor for ephemeral and batch jobs
- [Black-box](https://github.com/prometheus/blackbox_exporter) 黑盒测试工具，支持 HTTP, HTTPS, DNS, TCP and ICMP
- [Node-export](https://github.com/prometheus/node_exporter) 主机资源 agent 组件

### 资源连接

- Prometheus 资料
  - https://prometheus.io/download/
  - https://prometheus.io/docs/prometheus/latest/querying/operators/
  - https://prometheus.io/docs/prometheus/latest/querying/functions/
  - https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus.yml
  - https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis

- Grafana 官方文档
  - http://docs.grafana.org/features/panels/graph/

### 清理监控数据

- 节点下线后，metric 由残留数据
  - 第一种方案：
    - 停止 Prometheus 与 Push Gateway 服务，然后删除 Prometheus 的所有监控数据，然后启动两个组件。
  - 第二种方案：
    - 人工修改 metric ，按个过滤掉所有已经被下线的节点信息
    - histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket{instance!="6a5114505e5f_4000"}[1m])) by (le, instance))
  - 第三种方案：
    - 目前只能通过 http api 删除历史数据，不能删除 instance 信息
    - 且版本必须高于 2.1.0 以上，官方文档 `https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis`
      - curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/delete_series?match[]=tidb_server_handle_query_duration_seconds_bucket'
      - curl -XPOST -g 'http://172.16.10.65:29090/api/v1/admin/tsdb/clean_tombstones'
