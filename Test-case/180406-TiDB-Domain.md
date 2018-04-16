---
title: TiDB cluster 使用域名部署测试
date: 2018-04-06 21:40:52
updated: 2017-04-06 21:40:52
categories: 
  - deploy
tags:
  - domain
---
# TiDB Cluster 使用域名部署测试

## 场景&问题

- 部署场景
  - 使用 tidb-ansible 部署 binary 方式，启动使用 supervise
  - inventory.ini 文件中直接填写域名
  - 测试场景域名未经过 DNS 解析，直接使用 `/etc/hosts` 文件解析

- 测试场景
  - 域名部署未影响实际功能，因此只测试个组件之间网络延迟与通讯是否正常
  - PD 更换 IP 地址映射，用于修改主机 IP 或者迁移时场景
  - TiDB 到 PD 之间通讯
  - TiKV 到 PD 之间通讯
  - PD 与 PD 之间通讯

- 问题
  - PD 启动时， client-urls 与 peer-urls 参数不能使用域名方式，只能使用 IP 或者 0.0.0.0
  - TiKV 启动时，不应该将 --addr 设置为域名，可以设置为 0.0.0.0
  - 测试 TiDB 域名与 IP 行为一致，无特殊影响

### 部署架构

域名 | 服务 | 初始IP | 修改后
---|----|------|----
pd.p.cc | PD | 127.0.0.1 | 172.16.10.64
kv.p.cc | PD | 127.0.0.1 | 172.16.10.64
db.p.cc | PD | 127.0.0.1 | 172.16.10.64

### PD 启动参数调整

```BASH
exec bin/pd-server \
    --name="pd1" \
    --client-urls="http://0.0.0.0:2379" \
    --advertise-client-urls="http://pd.p.cc:2379" \
    --peer-urls="http://0.0.0.0:2380" \
    --advertise-peer-urls="http://pd.p.cc:2380" \
    --data-dir="/data1/domain/data.pd" \
    --initial-cluster="pd1=http://pd.p.cc:2380" \
    --config=conf/pd.toml \
    --log-file="/data1/domain/log/pd.log" 2> "/data1/domain/log/pd_stderr.log"
```

### TiKV 参数启动示例

```bash
exec bin/tikv-server \
    --addr "0.0.0.0:20160" \
    --advertise-addr "http://tikv.p.cc:20160" \
    --pd "http://pd.p.cc:2379" \
    --data-dir "/data1/domain/data" \
    --config conf/tikv.toml \
    --log-file "/data1/domain/log/tikv.log" 2> "/data1/domain/log/tikv_stderr.log"
```

## 测试结果

- TiDB 到 PD 之间网络通信
  - PD 使用域名启动，如果经过 DNS 解析，TiDB 获取 TSO 会慢 200us 上下
  - PD cluster 置于 haproxy 后面，实际通讯时 tidb -- haproxy -- 某个 etcd(etcd proxy) -- pd leader ，会比直接连接 PD 慢 600us - 1ms 左右
- PD 到 PD
  - 部署单节点，未达到测试目的
- TiKV 到 PD
  - 部署单节点，未参与 PD 切换 leader 时对 TiKV 的影响
- TiKV 到 TiKV
  - 多实例部署未受到影响
  - 域名造成通讯延迟

#### 修改 IP

- 修改 PD
  - 正常启动
- 修改 TiDB
  - TiDB 无状态，正常使用
- 修改 TiKV
  - ok ，需要重启
  - TiKV 是将 ip:port 注册到 pd etcd 信息中，如果重复，已有的必须是 tomestone ， 否则出现以下错误，地址被占用，无法生成新的 id

  ```log
  2018/03/20 15:50:14.436 util.rs:315: [ERROR] fail to request: Grpc(RpcFailure(RpcStatus { status: Unknown, details: Some("duplicated store address: id:1001 address:\"kv.p.cc:20160\" , already registered by id:1 address:\"kv.p.cc:20160\" ") }))
  2018/03/20 15:50:55.313 tikv-server.rs:263: [ERROR] failed to start node: Pd(Other(StringError("[src/pd/util.rs:323]: fail to request")))
  ```