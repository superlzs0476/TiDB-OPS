---
title: AlertManager 安装 & 部署
date: 2018-03-23 15:34:01
updated: 2017-04-08 13:49:26
categories:
  - Monitor
tags:
  - Deploy
  - AlertManager
  - Proemtheus
  - Blackbox
---

# AlertManager 安装 & 部署

## TiDB 监控架构

![monitor](https://raw.githubusercontent.com/pingcap/docs-cn/master/media/syncer-monitor-scheme.png)

- [Grafana](https://github.com/grafana/grafana) 监控、度量分析仪表板工具，从 Prometheus 获取数据
- [Prometheus](https://github.com/prometheus/prometheus) 用于存放监控数据的时序数据库
- [push gateway](https://github.com/prometheus/pushgateway) push acceptor for ephemeral and batch jobs
- [Black-box](https://github.com/prometheus/blackbox_exporter) 黑盒测试工具，支持 HTTP, HTTPS, DNS, TCP and ICMP
- [Node-export](https://github.com/prometheus/node_exporter) 主机资源 agent 组件

### AlertManager 组件

- 使用 2018年3月22日以后的 [TiDB-Ansible](https://github.com/pingcap/tidb-ansible/blob/master/deploy.yml) 可以自动部署 AlertManager 组件
- 也可按照参考[通过 systemd 守护 syncer 进程启动](../Docs/180323-Systemd-Syncer) 方案手动部署 AlertManager 组件
  - AlertManager 下载地址[传送门](https://prometheus.io/download/)

---

## Prometheus 与 AlertManager

- Prometheus 配置文件地址 `{{deploy_dir}}/conf/prometheus.yml`

### AlertManager 相关参数

- 在 Prometheus 配置文件更新以下代码块，修改完成后重启后生效

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - '172.16.10.65:39093' # AlertManager 地址
```

### Black-box 相关参数

- 以下功能用于监控 TiDB cluster 组件业务端口与监控组件业务端口存活状态
  - 修改 Prometheus 配置文件并更新添加以下代码块，修改完成后重启生效

```yaml
- job_name: "port_test"
  scrape_interval: 30s
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
  - targets: ['172.16.10.64:2379', '172.16.10.65:2379', '172.16.10.66:2379']
    labels:
      group: 'pd'
  - targets: ['172.16.10.64:4000', '172.16.10.65:4000', '172.16.10.66:4000']
    labels:
      group: 'tidb'
  - targets: ['172.16.10.67:20160', '172.16.10.68:20160', '172.16.10.69:20160']
    labels:
      group: 'tikv'
  - targets: ['172.16.10.64:9100', '172.16.10.65:9100', '172.16.10.66:9100','172.16.10.67:9100', '172.16.10.68:9100', '172.16.10.69:9100']
    labels:
      group: 'node-export'
  - targets: ['172.16.10.67:3000', '172.16.10.67:9091', '172.16.10.67:9090']
    labels:
      group: 'monitor'
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 172.16.10.65:9115  # Blackbox 地址
```

- 以下代码块使用 ICMP 协议监控
  - 修改 Prometheus 配置文件并更新添加以下代码块，修改完成后重启生效[可选]

```YAML
  - job_name: 'ping_all'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [icmp]  #ping
    static_configs:
      - targets: ['172.16.10.64','172.16.10.65','172.16.10.66','172.16.10.67','172.16.10.68','172.16.10.69',]
        labels:
          group: 'node-ping'
    relabel_configs:
      - source_labels: [__address__]
        regex: (.*)(:80)?
        target_label: __param_target
        replacement: ${1}
      - source_labels: [__param_target]
        regex: (.*)
        target_label: ping
        replacement: ${1}
      - source_labels: []
        regex: .*
        target_label: __address__
        replacement: 172.16.10.65:9115  # Blackbox exporter.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 172.16.10.65:9115  # Blackbox exporter.
```

### Prometheus 告警

#### 新建 `port.rules.yml` 文件，并输入以下内容

```yaml
  - alert: TiDB_is_Down
    expr: probe_success{group="tidb"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="tidb"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiDB_is_Down

  - alert: TiKV_is_Down
    expr: probe_success{group="tikv"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="tikv"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiKV_is_Down

  - alert: PD_is_Down
    expr: probe_success{group="pd"} == 0
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: probe_success{group="pd"} == 0
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: PD_is_Down
```

#### 更新 Prometheus 配置文件

- Prometheus 配置文件中添加 `port.rules.yml` 字段，修改完成后重启生效

```yaml
rule_files:
  - 'node.rules.yml'
  - 'blacker.rules.yml'
  - 'bypass.rules.yml'
  - 'pd.rules.yml'
  - 'tidb.rules.yml'
  - 'tikv.rules.yml'
  - 'binlog.rules.yml'
  - 'port.rules.yml'
```

## AlertManager 配置

### AlertManager 配置文件

- [AlertManager](https://prometheus.io/docs/alerting/configuration/) 配置以及工具使用说明
- [AlertManager](https://github.com/pingcap/tidb-ansible/blob/master/conf/alertmanager.yml) 配置文件模板
- 安装后位置在 `{{deploy_dir}}/conf` 目录下

- 告警发送
  - 通过邮件方式发送告警，选用 db-alert-email 接收器
    - 需要 SMTP 服务器信息
  - 通过 SMS 或者 syslog 方式发送告警选用 webhook-pulgin 接收器
    - 需要提供 syslog binary 工具接收
    - 发送到 syslog binary 中的告警等级为 port ，由 route 中的 `level: port` 参数控制。

```yaml
global:
  # The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@example.org'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

  # The Slack webhook URL.
  # slack_api_url: ''

route:
  # A default receiver
  receiver: "db-alert-email"

  # The labels by which incoming alerts are grouped together. For example,
  # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
  # be batched into a single group.
  group_by: ['env','instance','type','group','job']

  # The labels by which incoming alerts are grouped together. For example,
  # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
  # be batched into a single group.
  group_wait:      30s

  # When the first notification was sent, wait 'group_interval' to send a batch
  # of new alerts that started firing for that group.
  group_interval:  3m

  # If an alert has successfully been sent, wait 'repeat_interval' to
  # resend them.
  repeat_interval: 3m

  routes:
  - match:
      env: test-cluster # 过滤条件，满足此条件的，转送至相应的 receiver
    receiver: db-alert-email # receiver 关键词，全局唯一，不能重复
    continue: true # 默认告警匹配成功第一个 receivers 会退出匹配，开启 continue 参数后会继续匹配 receivers 列表，直到再无 receivers 时或者下一个 receivers 中 continue fasle 的时候才会退出。(continue default false)
  #- match:
  #    level: port
  #    env: test-cluster
  #  receiver: webhook-pulgin
  #  continue: true
  #- match:
  #    env: test-cluster
  #  receiver: db-alert-port

receivers:
- name: 'db-alert-email'
  email_configs:
  - send_resolved: true # 告警回执，监控下由 Alert 返回为 OK 状态时，会发送一条 OK 状态的回执
    to: 'xxx@xxx.com'
#- name: 'webhook-pulgin'
#  webhook_configs:
#  - send_resolved: true
#    url: 'http://172.16.10.65:28082/v1/alertmanager'
#- name: 'db-alert-slack'
#  slack_configs:
#  - channel: '#alerts'
#    username: 'db-alert'
#    icon_emoji: ':bell:'
#    title:   '{{ .CommonLabels.alertname }}'
#    text:    'in {{ .CommonLabels.env}}:   {{ .GroupLabels.alertname }}  {{ .CommonAnnotations.description }}  http://172.16.10.65:9093/alerts'
```

### AlertManager 告警

- 通过 http://IP:9093/ 打开网页，可通过网页管理告警
  - 通过页面可以 silence 一条正在持续发送的告警

### AlertManager 运维

- 自动化部署的 AlertManager，启动停止脚本位于 `{{deploy_dir}}/scripts` 目录中
- 手动部署的，参考[通过 systemd 守护 syncer 进程启动](../Docs/180323-Systemd-Syncer)文档