---
title: 为 Syncer 选用守护进程工具
date: 2018-03-23 21:40:52
updated: 2018-04-06 21:40:52
categories: 
  - Deploy
tags:
  - Syncer
  - Systemd
---
# 为 Syncer 选用守护进程工具

## 测试案例

- 背景
  - Syncer 遇到以下操作会自动退出，目前采用的 Supervise 守护进程，当进程退出后自动拉起服务。但 Supervise 无法控制拉起次数，无法让进程彻底停止
    - 网络闪断 (这个需要被重试拉起)
    - tidb 或者 tikv 繁忙导致 server is busy
    - 不支持的 DDL
    - 不支持的 DML 语法
- 需求
  - 需要守护进程工具拥有重试最大次数功能
    - 可选方案有 Supervisord 与 Systemd
      - Supervisord 有 Startretries 参数可以实现，但需要部署 Supervisord binary
      - Systemd 是 CentOS 系统自带服务，无需单独部署，官方参数暂无 startretries 类似功能。通过 Google 查到其他方式实现相关功能

### Systemd 配置文件

- cat /etc/systemd/system/syncer-port.service
  - 注册 systemd 服务
  - 修改 service 后需要执行 `systemctl daemon-reload`
- Service
  - RestartSec：自动重启当前服务间隔的秒数
  - StartLimitInterval=, StartLimitBurst=
    - 限制该服务的启动频率。默认值是每10秒内不得超过5次(StartLimitInterval=10s StartLimitBurst=5)
    - [引用 简书](http://www.jinbuguo.com/systemd/systemd.service.html)
    - [引用 CSDN](http://blog.csdn.net/yuesichiu/article/details/51485147)
    - [引用 阮一峰](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)

  ```c#
  [Unit]
  Description=syncer service
  After=syslog.target network.target remote-fs.target nss-lookup.target

  [Service]
  LimitNOFILE=1000000
  User=root
  ExecStart=/root/systemd-syncer/run_syncer.sh
  Restart=always
  RestartSec=15s
  StartLimitInterval=300
  StartLimitBurst=5

  [Install]
  WantedBy=multi-user.target
  ```

### 启动脚本

- cat run_syncer.sh
  - 服务启动脚本
  - 启动脚本内容所涉及到的路劲均需要指定绝对路劲

  ```c#
  #!/bin/bash

  svr=syncer-job
  work_dir="/root/systemd-syncer"
  bin_dir="${work_dir}/bin/syncer"
  conf_dir="${work_dir}/conf"
  log_dir="${work_dir}/log"
  nohup_dir="${work_dir}/nohup"

  ${bin_dir} --config ${conf_dir}/${svr}.toml --enable-gtid  --auto-fix-gtid  --log-file ${log_dir}/${svr}.log --status-addr :11105 &>>${nohup_dir}/${svr}.out
  ```

- systemd-syncer 目录结构

  ```c#
  [root@ip-172-16-10-65 systemd-syncer]# tree ./
  ./
  ├── bin
  │   └── syncer
  ├── conf
  │   └── syncer-job.toml
  ├── log -> /data1/systemd-syncer.log
  │   └── syncer-job.log
  ├── meta
  │   └── syncer-job.meta
  ├── nohup
  │   └── syncer-job.out
  └── run-syncer
      └── run_syncer-job.sh
  ```

### Syncer 模拟 DDL 同步失败测试

- MySQL
  - `create index idx on hello (id,status,size) lock=none`
  - 利用早期 TiDB 版本不支持 lock=none 来复现 Syncer 不支持语法退出
- Syncer
  - Syncer 持续重启，因为 TiDB 无法执行该 DDL
  - 进程退出，然后被 Systemd 重试拉起，5 次后，进程自动退出

## 测试验证记录

### Systemd 记录

- systemctl start syncer-port
  - 启动该服务
- systemctl stop syncer-port
  - 停止该服务
- systemctl restart syncer-port
  - 重启该服务
- systemctl status syncer-port -l
  - 查看服务状态，-l 可以查看更多信息

  ```bash
  [root@ip-172-16-10-65 bin]# systemctl status syncer-port -l
  ● syncer-port.service - syncer service
    Loaded: loaded (/etc/systemd/system/syncer-port.service; disabled; vendor preset: disabled)
    Active: failed (Result: start-limit) since Fri 2018-02-09 16:43:52 CST; 9min ago
    Process: 393938 ExecStart=/root/systemd-syncer/run_syncer.sh (code=exited, status=0/SUCCESS)
  Main PID: 393938 (code=exited, status=0/SUCCESS)

  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: start request repeated too quickly for syncer-port.service
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: Failed to start syncer service.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: Unit syncer-port.service entered failed state.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: syncer-port.service failed.
  ```

- `journalctl -f -u syncer-port`
  - 使用 journalctl 获取该服务更多的日志信息

  ```LOG
  Feb 09 16:33:42 ip-172-16-10-65 systemd[1]: Unit syncer-port.service entered failed state.
  Feb 09 16:33:42 ip-172-16-10-65 systemd[1]: syncer-port.service failed.
  Feb 09 16:42:36 ip-172-16-10-65 systemd[1]: Started syncer service.
  Feb 09 16:42:36 ip-172-16-10-65 systemd[1]: Starting syncer service...
  Feb 09 16:42:51 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:42:51 ip-172-16-10-65 systemd[1]: Started syncer service.
  Feb 09 16:42:51 ip-172-16-10-65 systemd[1]: Starting syncer service...
  Feb 09 16:43:06 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:43:06 ip-172-16-10-65 systemd[1]: Started syncer service.
  Feb 09 16:43:06 ip-172-16-10-65 systemd[1]: Starting syncer service...
  Feb 09 16:43:22 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:43:22 ip-172-16-10-65 systemd[1]: Started syncer service.
  Feb 09 16:43:22 ip-172-16-10-65 systemd[1]: Starting syncer service...
  Feb 09 16:43:37 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:43:37 ip-172-16-10-65 systemd[1]: Started syncer service.
  Feb 09 16:43:37 ip-172-16-10-65 systemd[1]: Starting syncer service...
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: syncer-port.service holdoff time over, scheduling restart.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: start request repeated too quickly for syncer-port.service
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: Failed to start syncer service.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: Unit syncer-port.service entered failed state.
  Feb 09 16:43:52 ip-172-16-10-65 systemd[1]: syncer-port.service failed.
  ```

### 通过 syncer 日志查看 syncer 重启记录

  ```BASH
  [root@ip-172-16-10-65 bin]# grep "status exits" s.log
  2018/02/09 16:42:36 syncer.go:795: [info] print status exits, err:context canceled
  2018/02/09 16:42:51 syncer.go:795: [info] print status exits, err:context canceled
  2018/02/09 16:43:06 syncer.go:795: [info] print status exits, err:context canceled
  2018/02/09 16:43:22 syncer.go:795: [info] print status exits, err:context canceled
  2018/02/09 16:43:37 syncer.go:795: [info] print status exits, err:context canceled
  ```

## 告警

- 按照官方文档配置告警信息 [Syncer](https://github.com/pingcap/docs-cn/blob/master/tools/syncer.md) 传送门

- 进程退出告警，可添加 [Blackbox](../Monitor/180323-AlertManager-安装部署.md) 组件，监听业务端口状态。

---

## Systemd 的一些文档

- systemd及系统初始化
  - https://open-doc.dingtalk.com/docs/doc.htm?treeId=412&articleId=107236&docType=1

- Systemd 入门教程：命令篇
  - http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html

- systemd 服务脚本的编写
  - https://www.cnblogs.com/sysk/p/4969368.html

- 编写 systemd service 文件，内容质量较差
  - http://blog.csdn.net/djskl/article/details/46671453

- systemd 启动遇见 `(code=exited, status=1/FAILURE)`
  - 需要查看多方日志

- systemd restart // How to set up a systemd service to retry 5 times on a cycle of 30 seconds
  - https://stackoverflow.com/questions/39284563/how-to-set-up-a-systemd-service-to-retry-5-times-on-a-cycle-of-30-seconds
  - This worked worked for me for a service that runs a script using 'Type=idle'. Note that 'StartLimitInterval' must be greater than 'RestartSec * StartLimitBurst' otherwise the service will be restarted indefinitely.