---
title: 定制一条告警规则与运维监控处理流程
date: 2017-06-05 15:34:01
updated: 2017-06-07 13:49:26
categories:
  - Monitor
tags:
  - Deploy
  - AlertManager
  - Proemtheus
---
# 定制一条告警规则与运维监控处理流程

## 告警设计流程

- Brain-storming
  - 现有监控架构
  - 现有监控数据
  - 补充监控数据
  - SRE 制定大纲
- 告警规则
  - 监控目标
  - 监控阈值
  - 自定义 labels
    - 优先级
    - 组件
- 发送告警
  - silences
  - 维护区间
  - 优先级
  - 告警链路
    - 邮箱
    - 微信
    - phone / sms
- 问题处理
  - 优先级
  - 问题处理
  - 问题升级
  - 问题记录
  - 工单关闭
- 告警重复出现
  - 是否可修复
    - 产品内修复
    - 产品外修复
  - 不可修复
    - 产品 bug
    - 考虑告警优化

### 告警等级设置

- env
  - test-cluster
- channels
  - alerts
- level
  - emergency  0     phone
    - 主要是某个组件挂掉了才会触发
  - critical   1     sms
    - 某个组件出现了问题，但是可以延后处理，比如 8 个小时
  - warning    2     wechat
    - 某个组件通讯或者延迟大于某个阈值，偶然现象或者频发现象，保留类告警，可以根据业务调整或取消
  - notice     3     slack
    - 提示类告警，就是通知下这块延迟高或者要问题了，你应该找个时间看下集群了。这块和业务有关系，不是 tidb 的业务，是 SQL 的业务
- service
  - TiDB
  - TiKV
  - PD
  - OS
- color
  - #2eb886

### 告警 - 5W1H

* [ ] 告警能否判断集群状态
  - 告警是否能帮助解决或预防问题
  - 在 POC 的时候，是否能(非精准)提醒集群瓶颈在哪里
* [ ] 告警能否有效的被执行
  - 可以设置多条规则监控同一个服务 // 单服务挂掉有多条告警，告警监控服务的不同路线
  - 当告警条件全部成功后，再进行优化、精准告警
* [ ] 告警误报的影响是什么
  - 告警可以误报，但是不可以不告警  // 结果出现有狼来了效应，产生怠慢消极状态
  - 在整体集群垮掉的时候，告警要暂时关闭 // 过多的重复告警，影响现有问题解决
* [ ] 我们还缺少那些告警判断

-----

## 告警设计

- [TiDB alert rule](https://github.com/pingcap/tidb-ansible/tree/master/roles/prometheus/files "TiDB alert rule")

### 主机监测

* [ ] 主机状态监测
  - 主机探活机制
* [ ] 主机资源监测
  - CPU
    - 单个进程 CPU
    - 主机整体 CPU
    - 机器 load 负载
  - RAM
    - 单个进程 RAM
    - 主机整体 RAM
      - SWAP 使用量 // 超过一定使用量告警
  - DISK
    - 磁盘 读写延迟
    - 磁盘 Utilization
    - 磁盘容量监测
    - 磁盘读写状态
    - 文件描述符占用量
    - 磁盘 inode 使用数量
  - 网络
    - 流量进出
  - 其他围观 [node_exporter.json](https://raw.githubusercontent.com/BigerCAP/tidb-ops/master/Media/node.json "Github Grafana node_exporter.json")

### 数据库监测

- 数据库组件状态监测
  - 组件探活
- 组件链路状态监测
  - TiDB PD TiKV 组件之间延迟
- 数据库业务
  - QPS 延迟
- 数据库组件运行状态监测
  - PD 调度
  - TiDB SQL 处理
  - TiKV 资源存储与调度
- 数据库资源占用监测
  - TiDB 内存使用量
  - TiKV 存储使用量、计算资源使用量

----

## 选择告警工具

#### TiDB 监控架构

![monitor](https://raw.githubusercontent.com/BigerCAP/tidb-ops/master/Media/180323-syncer-monitor-scheme.png)

- [Grafana](https://github.com/grafana/grafana) 监控、度量分析仪表板工具，从 Prometheus 获取数据
- [Prometheus](https://github.com/prometheus/prometheus) 用于存放监控数据的时序数据库
- [push gateway](https://github.com/prometheus/pushgateway) push acceptor for ephemeral and batch jobs
- [Black-box](https://github.com/prometheus/blackbox_exporter) 黑盒测试工具，支持 HTTP, HTTPS, DNS, TCP and ICMP
- [Node-export](https://github.com/prometheus/node_exporter) 主机资源 agent 组件

基于现有监控架构，目前可选有 Prometheus/AlertManger 与 Grafana，或者重新造个轮子

### 测试一个告警工具

- 未完待续

-----

## 引用

-[告警平台设计](http://os.51cto.com/art/201603/507858.htm)