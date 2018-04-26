# TiDB-ops

## Know

> That too late when it is the first time [.](https://www.google.com/ "Google")

- [TiDB Weekly](http://weekly.pingcap.com "Weekly update in TiDB")

### PingCAP Blog-cn

- [TiDB 的正确使用姿势](https://github.com/pingcap/blog-cn/blob/master/how-to-use-tidb.md "如果整篇文章你只想记住一句话，那就是数据条数少于 5000w 的场景下通常用不到 TiDB，TiDB 是为大规模的数据场景设计的。如果还想记住一句话，那就是单机 MySQL 能满足的场景也用不到 TiDB。")
- [三部曲-说存储](https://github.com/pingcap/blog-cn/blob/master/tidb-internal-1.md)
- [三部曲-说计算](https://github.com/pingcap/blog-cn/blob/master/tidb-internal-2.md)
- [三部曲-谈调度](https://github.com/pingcap/blog-cn/blob/master/tidb-internal-3.md)
- [SQL 的一生](https://github.com/pingcap/blog-cn/blob/master/tidb-source-code-reading-3.md)
- [Raft 的优化](https://github.com/pingcap/blog-cn/blob/master/optimizing-raft-in-tikv.md)
- [PD Scheduler](https://github.com/pingcap/blog-cn/blob/master/pd-scheduler.md)

### PingCAP Docs-cn

- [TiDB 历史读与 GC](https://github.com/pingcap/docs-cn/blob/master/op-guide/history-read.md)
- [analyze](https://github.com/pingcap/docs-cn/blob/master/sql/statistics.md)
- [TiDB 官方 FAQ](https://github.com/pingcap/docs-cn/blob/master/FAQ.md "TiDB 官方 FAQ")

## Main

- [ROADMAP](ROADMAP.md)

### Monitor

- [Node_exporter README](Monitor/170601-Node_exporter.md)
- [使用 Blackbox_exporter 监测主机与服务状态](Monitor/180401-blackbox_exporter.md)
- [告警如何炼成的](Monitor/171212-Alert-Mind.md)
- [AlertManager 安装 & 部署](Monitor/180323-AlertManager-Deploy.md)
- [AlertManager 进阶操作-生产环境应用](Monitor/180412-Alert.rules.md)
- [Grafana & Prometheus 监控组件问题收集与排查思路](Monitor/170909-Monitor-FAQ.md)

### Docs

- [为 Syncer 选用守护进程工具](Docs/180323-Systemd-Syncer.md)
- [TiDB 环境变量解读](Docs/180411-TiDB-vars.md)
- [TiSpark 服务安装 部署 测试](Docs/180416-TiSpark-deploy.md)
- [TiDB Cluster 修改组件 IP 地址](Docs/180327-TiDB-IP.md)
- [TiDB 业务场景问题收集与排查方法](Case/180315-TiDB-FAQ.md)

### Test

- [TiDB AUTO_INCREMENT 功能测试](Test/180327-AutoIncrementTest.md)
- [TiDB Cluster 域名部署测试](Test/180406-TiDB-Domain.md)
- [TiDB Online DDL 实操测试](Test/171010-TiDB-Online-DDL.md)

### SoftWare

- [Tmux 快捷键 & 速查表](SoftWare/tmux.md)
- [Screen 速查表](SoftWare/screen.md)
- [NextCloud 安装运维调测](SoftWare/nextcloud.md)

### API

- TiDB [http_status.go](https://github.com/pingcap/tidb/blob/master/server/http_status.go)
  - TiDB [API](https://github.com/pingcap/tidb/blob/master/docs/tidb_http_api.md) 文档解释
- PD [router](https://github.com/pingcap/pd/blob/master/server/api/router.go)
  - [PD-CTL](https://github.com/pingcap/docs-cn/blob/master/op-guide/horizontal-scale.md) 使用文档

## About

> These violent delights have violent ends [.](https://github.com/BigerCAP/tidb-ops "Westworld")

![TiDB SQL at SCALE](https://raw.githubusercontent.com/BigerCAP/tidb-ops/master/Media/about-logo.png "A Distributed HTAP database compatible with the MySQL protocol")
