---
title: ROADMAP
date: 2017-06-01 21:40:52
updated: 2017-06-06 21:40:52
categories:
  - Docs
tags:
  - TiDB
  - ROADMAP
---
# ROADMAP

## Docs

* [X] 为 Syncer 选用守护进程工具
* [X] TiDB Cluster 修改组件 IP 地址
* [X] TiDB 环境变量解读
* [X] TiSpark 服务安装 部署 测试
* [ ] TiDB 业务场景问题收集与排查方法

### Case-by-case

* [X] TiKV 磁盘空间回收步骤记录
* [X] kill tcp connect

### Troubleshooting

* [ ] TiDB-Ansible
* [ ] TiDB-Binlog 日志文件
* [ ] Syncer-Loader 日志文件
* [ ] TiDB 日志文件
* [ ] TiKV 日志文件
* [ ] PD 日志文件

## Monitor

* [X] Node_exporter README
* [X] 使用 Blackbox_exporter 监测主机与服务状态
* [X] 告警如何炼成的
* [X] AlertManager 安装 & 部署
* [X] AlertManager 进阶操作 - 生产环境应用
* [X] Grafana & Prometheus 监控组件问题收集与排查思路
  * [ ] 监控 metrics 解读

## Test

* [X] TiDB AUTO_INCREMENT 功能测试
* [X] TiDB Cluster 使用域名部署测试
* [X] TiDB Online DDL 实操流程
* [ ] TiDB analyze 直方图收集与分析

## Software

* [X] Screen
* [X] Tmux
* [X] NextCloud
* [X] Megacli
* [X] Transfer
* [X] HAproxy
* [ ] Sourcegraph

----

## Plan

### Air Plan (空中加油)

> 未完待续，持续更新 (180802)

* [X] TiDB 安装部署 -- 参考 [TiDB Ansible 部署方案](https://github.com/pingcap/docs-cn/blob/master/op-guide/ansible-deployment.md)
* [X] [Grafana & Prometheus 监控架构](Monitor/170601-Prometheus-Grafana.md) -- 了解监控架构
  * [ ] TiDB 监控 metrics 解读 -- 学习如何使用监控
* [ ] TiDB 业务场景问题处理 -- 根据业务问题现象，如何使用监控/日志定位分析问题(案例)
  * [ ] TiDB 报错处理 -- TiDB 常见错误与处理方案 (案例)
  * [ ] TiDB 数据库 Log 解读 -- TiDB 日志内容解读，如果根据日志结合监控定位问题 (案例)
  * [ ] TiDB 数据库 SQL 业务优化 -- 慢 SQL 优化 (案例)
* [ ] TiDB 功能测试 analyze 、online DDL -- 功能测试与功能实现逻辑、对业务都有那些影响
  * [X] [TiDB AUTO_INCREMENT 功能测试](Test/180327-AutoIncrementTest.md)
  * [X] [TiDB Online DDL 实操测试](Test/171010-TiDB-Online-DDL.md)
  * [ ] analyze 直方图收集与分析 -- TiDB 1.0 引进，度过 2.0 的优化，预计 2.1 商用
