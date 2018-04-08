# alertmanager 组件安装部署运维手册

## 部署 alertmanager 

- [tidb-ansible](https://github.com/pingcap/tidb-ansible/blob/master/deploy.yml) 目前版本可以部署 alertmanager 组件
- 2018年3月22日之后的

## Prometheus 接入 alertmanager

- Prometheus 配置文件地址 `{{deploy_dir}}/conf/prometheus.yml`

### alertmanager 相关参数

- 在 Prometheus 配置文件更新以下代码块，修改完成后重启后生效

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - '172.16.10.65:39093' # alertmanager 地址
```

### blackbox 相关参数

- 在 Prometheus 配置文件更新以下代码块，修改完成后重启后生效
- 以下功能用于监控 TiDB cluster 组件业务端口

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
      group: 'node'
  - targets: ['172.16.10.67:3000', '172.16.10.67:9091', '172.16.10.67:9090']
    labels:
      group: 'monitor'
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 172.16.10.65:9115  # blackbox 地址
```

## Prometheus 告警规则

### 以下内容添加到 port.yml

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
  - alert: TiDB_query_duration
    expr: histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket[1m])) by (le, instance)) > 10
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: histogram_quantile(0.99, sum(rate(tidb_server_handle_query_duration_seconds_bucket[1m])) by (le, instance)) > 10
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiDB_query_duration
  - alert: TiDB_load_schema_failed
    expr: rate(tidb_server_schema_lease_error_counter[1m]) > 1
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: rate(tidb_server_schema_lease_error_counter[1m]) > 1
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: PD_is_Down
  - alert: TiKV_server_report_failed
    expr: sum(rate(tikv_server_report_failure_msg_total[1m])) by (type,instance,job,store_id) > 10
    for: 3m
    labels:
      env: test-cluster
      level: port
      expr: sum(rate(tikv_server_report_failure_msg_total[1m])) by (type,instance,job,store_id) > 10
    annotations:
      description: 'alert:{{ $labels.expr }} instance:
        {{ $labels.instance }} values: {{ $value }}'
      value: '{{ $value }}'
      summary: TiKV_server_report_failed
```

### 更新 Prometheus 配置文件

- 修改 Prometheus 配置文件中以下字段，修改完成后重启后生效

```yaml
rule_files:
  - 'node.rules.yml'
  - 'blacker.rules.yml'
  - 'bypass.rules.yml'
  - 'pd.rules.yml'
  - 'tidb.rules.yml'
  - 'tikv.rules.yml'
  - 'binlog.rules.yml'
```

- 替换后结果

```yaml
rule_files:
  - 'port.rules.yml'
```

## alertmanager 配置文件

- [alertmanager](https://prometheus.io/docs/alerting/configuration/) 官网
- [alert manager](https://github.com/pingcap/tidb-ansible/blob/master/conf/alertmanager.yml) 模板
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
      env: test-cluster
    receiver: db-alert-email
    continue: true
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
  - send_resolved: true
    to: 'xxx@xxx.com'
#- name: 'webhook-pulgin'
#  webhook_configs:
#  - send_resolved: true
#    url: 'http://10.0.3.6:28082/v1/alertmanager'
#- name: 'db-alert-slack'
#  slack_configs:
#  - channel: '#alerts'
#    username: 'db-alert'
#    icon_emoji: ':bell:'
#    title:   '{{ .CommonLabels.alertname }}'
#    text:    'in {{ .CommonLabels.env}}:   {{ .GroupLabels.alertname }}  {{ .CommonAnnotations.description }}  http://172.0.0.1:9093/alerts'

```

## alertmanager 运维

- alertmanager 启动脚本位于 `{{deploy_dir}}/scripts` 目录中