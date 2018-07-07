---
title: TiDB-ansible 问题收集处理
date: 2017-10-07 11:40:52
updated: 2017-10-08 21:40:52
categories:
  - Troubleshooting
tags:
  - ansible
  - inventory
  - deploy
  - tidb
---
# TiDB-ansible 问题收集处理

## 自定义信息修改

### 默认端口

- [端口默认值](https://github.com/pingcap/tidb-ansible/tree/master/group_vars)

- PD
  - pd_client_port: 2379
  - pd_peer_port: 2380

- Tikv
  - tikv_port: 20160

- TIDB
  - tidb_port: 4000
  - tidb_status_port: 10080

- PUMP
  - pump_port: 8250

- Monitor
  - node_exporter_port: 9100
  - prometheus_port: 9090
  - pushgateway_port: 9091
  - blackbox_exporter_port: 9115
  - kafka_exporter_port: 9308
  - grafana_port: 3000

### 默认目录信息及变量

- deploy_dir = /home/tidb/deploy
  - 服务部署目录

#### Local

- downloads_dir: "{{playbook_dir}}/downloads"
  - binary 下载目录
- resources_dir: "{{playbook_dir}}/resources"
  - 二进制文件目录
- fetch_log_dir: "{{playbook_dir}}/fetch_logfile"
  - ansible 获取主机 fetch 信息存放位置

#### supervise

- status_dir: "{{deploy_dir}}/status"
  - supervise 状态文件目录

- backup_dir: "{{deploy_dir}}/backup"
  - 检测到 binary 发生改变，备份 binary 目录

#### Tikv

- tikv_log_dir: "{{deploy_dir}}/log"
- tikv_data_dir: "{{deploy_dir}}/data"
  - tikv 数据目录，tikv --store 目录

#### PD

- pd_log_dir: "{{deploy_dir}}/log"
- pd_data_dir: "{{deploy_dir}}/data.pd"
  - PD 数据目录，PD --data-dir 目录

#### PUMP

- pump_socket: "{{status_dir}}/pump.sock"

- pump_log_dir: "{{deploy_dir}}/log"
- pump_data_dir: "{{deploy_dir}}/data.pump"
  - pump binlog 日志存放目录，pump --data-dir 目录

#### TiDB

- tidb_log_dir: "{{deploy_dir}}/log"

#### monitoring

- node_exporter_log_dir: "{{deploy_dir}}/log"

- pushgateway_log_dir: "{{deploy_dir}}/log"
- prometheus_data_dir: "{{deploy_dir}}/data.metrics"
  - prometheus 监控数据存放目录，来自于下层 node_export 与 tidb pd tikv 服务信息

- grafana_log_dir: "{{deploy_dir}}/log"
- grafana_data_dir: "{{deploy_dir}}/data.grafana"
  - grafana 存放数据目录

## 服务部署场景

### 单机多 TIKV 场景

> 物理机场景，多块磁盘，双路 CPU, 大于 128G 内存，千兆以上内网

1. 单机部署多 tikv，并设置 labels
    - labels 只有在第一次部署时生效，后续添加不生效。
2. tikv.toml 需要设置 `block-cache-size`，防止 tikv OOM
    - 假设机器配置 128G / 48U ,sql 常见为简单事务，多写 配置如下
      - block-cache-size
        - default 2G
        - write  4G
        - raft  1G
        - lock  1G

  ```yaml
  [tikv_servers]
  tikv-3-1 ansible_host=10.1.3.1 deploy_dir=/data1/deploy tikv_port=20171 labels="host=h1"
  tikv-3-2 ansible_host=10.1.3.2 deploy_dir=/data2/deploy tikv_port=20172 labels="host=h1"
  tikv-4-1 ansible_host=10.1.4.1 deploy_dir=/data1/deploy tikv_port=20171 labels="host=h2"
  tikv-4-2 ansible_host=10.1.4.2 deploy_dir=/data2/deploy tikv_port=20172 labels="host=h2"

  [pd_servers:vars]
  max-replicas = 3
  #location-labels = ["zone", "rack", "host"]
  location_labels = ["host"]
  ```

#### labels

> 配合多单机多 tikv 节点设置
> 后期可以通过 api 可以修改

1. labels 分层级读取与写入，假设层级为 `["zone", "rack", "host"]`
2. 逐级检测，数据不会存放在同性 (labels) 之内。当只有一种的时候，数据会存放随机存放
3. [传送门](https://github.com/pingcap/docs-cn/blob/master/op-guide/location-awareness.md)

## inventory.ini 特性开关

- enable_elk = False
  - 是否安装 ELK，目前暂不支持
- enable_firewalld = False
  - 是否开启防火墙，目前暂时不支持
- enable_ntpd = False
  - 是否部署 NTP 服务，目前暂时不支持
- machine_benchmark = True
  - 检测磁盘性能是否满足 tikv 服务读写, 默认 True
- set_hostname = False
  - 设置主机名，格式示例: ip-192-168-10-1
- enable_binlog = False
  - 是否部署 pump 服务，默认关闭，修改为 true 部署 pump 服务
  - 开启 pump 服务后，启动顺序为 PD >> TIKV >> PUMP >> TIDB

## ansible 错误说明

### 执行 bootstarp.yml

- 远程目标主机非 22 端口
  - 修改 playbook 同级 ansible.cfg 文件，修改 `#remote_port = 22` 参数并取消注释
- ansible -k 参数 与 ssh key 互信 场景
  - 当 ansible 主机与集群主机之间未建立 SSH key 互信时，执行 `ansible-playbook ***.yml -k`
  - 当 ansible 主机与集群主机之间已建立 SSH key 互信时，且 SSH key 无密码保护。执行 `ansible-playbook ***.yml`
- ansible 安装出现 `'the write speed of tikv deploy_dir disk is too slow: {{disk_write_speed.stdout}} MB/s < {{ min_write_speed }} MB/s'`
  - 磁盘性能达不到读写要求
  - inventory.ini 文件中 `machine_benchmark = True` 修改为 `machine_benchmark = False`

### 执行 deploy.yml

- ubuntu 主机，deploy 时提示 limit 小于 40960 时
  - 如果使用的 A 场景，请替换成 B 场景
  - 如果使用的 B 场景，请重启主机，ubuntu 修改 limit.conf 不能即时生效
- ansible 安装出现 `'The file system mounted at {{item.mount}} does not meet ext4 file system requirement'`
  - tikv deploy_dir 文件系统不是 ext4
  - 请修改 deploy_dir 文件系统为 ext4

### 执行 start.yml

- centos6 安装后启动失败或者启动后异常, 登录主机后只能看到 supervise 进程
  - `ansible-playbook local_prepare.yml` 默认下载适配 centos7 二进制文件
  - centos6 请下载 http://download.pingcap.org/tidb-latest-linux-amd64-centos6.tar.gz
- ansibel 启动时出现 wait up timeout
  - 逐步排查
  - status/pid
  - 查看日志是否还在运行
  - 随后 kill
- ubuntu 主机 提示 `run_node_exporter.sh: line 3: ulimit: open files: cannot modify limit: Operation not permitted`
  - 当前用户没有权限执行 `ulimit -n 1000000`
  - 使用当前用户执行 `ulimit -Hn` ，如果值为 1000000，可以注释脚本第三行。
- ubuntu 安装后启动，长时间卡在 wait up, 登录主机查看没有进程
- Grafana 导入 json 文件 ValueError: zero length field name in format
  - Python2(>=2.7) 和 Python3(>=3.1) 正常
  - Python2(<2.7) 和 Python3(<3.1) 有问题
    - http://ju.outofmemory.cn/entry/267078

### 执行 stop.yml

- ansible 停止进程出现 wait ** down ，长时间后返回失败
  - 检查主机进程日志
  - 无重要日志，手动 kill

### 执行 rolling_update.yml

- 使用 ansible 更新服务配置文件
  - 修改 {{playbook}}/conf/tikv.toml 文件
  - 取消参数注释，并修改参数为想要的值。
  - 执行 ansible-playbook rolling_update.yml 滚动升级

## 组件

### PD

- PD 是否可以更改 IP 地址
  - PD 无法更改 IP 或者更改启动目录。
  - PD 底层 etcd 会根据启动目录以及 IP 生成 cluster ID，当更换了 IP 或目录，PD 再次启动时就会认为是一个新的集群，此时 Tikv 无法辨认底层数据属于那个集群，同时无法读取数据。
  - TIDB 需要考虑分布式数据以及数据一致性，因此不能变动 cluster ID。

### TIKV

- 非金融证券企业 TIKV 参数调整
  - sync-log = false
- 单节点部署多 tikv, 必须修改，否则造成 OOM
  - 需要调整 block-cache-size
    - default 25%
    - write 15%
    - cf   2%
    - lock   2%
  - 总内存 * 50% == 所有 tikv 占用内存

### Grafana

dests.json 文件内容

```json
[
    {
        "name": "Test-Cluster",
        "url": "http://172.16.10.65:3000/",
        "titles": {
            "node": "Test-Cluster-Node-Export",
            "disk_performance": "Test-Cluster-Disk Performance",
            "overview": "Test-Cluster-Overview",
            "tikv": "Test-Cluster-TiKV",
            "pd": "Test-Cluster-PD",
            "tidb": "Test-Cluster-TiDB"
        },
        "user": "admin",
        "password": "admin",
        "datasource": "test-cluster"
    }
]
```

生成 dests.json 文件步骤在 tidb-ansible/start.yml 290 行

## FAQ

### ansible remote host No response

- [SSH works, but ansible throws unreachable error ](https://github.com/ansible/ansible/issues/15321)

- [paramiko](https://legacy.gitbook.com/book/yangdejie/paramiko/details)

  ```yml
  use paramiko as a workaround.

  ansible-playbook abc.yml -i development -c paramiko
  or add to ansible config

  [defaults]
  transport = paramiko
  ```

- 检查互信状态
  - Limiting ssh key permissions to 600 fixed this issue.
  - I was using a non-standard private key and it wasn't being found. It was 600 perms. ssh-add <path-to-private-key> fixed my issue.
  - Adding my key with ssh-copy-id to the remote server fixed the problem.


```yml
for me, this resolved on Ubuntu 16.04 with this added to ansible.cfg:

[ssh_connection] 
# for running on Ubuntu
control_path=%(directory)s/%%h-%%r
On related note, this resolved it when running on Mac host:

[ssh_connection] 
# for running on OSX
control_path = %(directory)s/%%C
```

```bash
Solved the issue by installing sshpass using command:

sudo apt-get install sshpass
```