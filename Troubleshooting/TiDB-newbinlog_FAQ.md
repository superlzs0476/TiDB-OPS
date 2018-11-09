### new-binlog 部署完成以后，单独重启 pump 组件，导致 tidb 无法对外提供服务
#### 问题重现
##### 版本 
###### TiDB master ，PUMP 和 Drainer 通过 tidb-ansible 部署
##### 拓扑 
###### tidb-server → pump → drainer → binlog
##### 操作 
###### new-binlog 状态 running ，tidb 有频繁的 DML 事务操作。通过 tidb-ansible 完成 pump 集群的重启
##### 现象 
###### pump 的 data.pump 目录下 value 没有增量文件生成，tidb.log 出现大量的报错 "[error] failed to write binlog: [global:3]critical error no avaliable pump to write binlog" 和 "[error] listener stopped, waiting for manual kill."
#### 问题原因
###### tidb server 添加 enable_binlog 启动参数后，每次的事务 commit 前，需要确认事务是否已经写入到 tikv 和 pump ，如果 pump 没有完成 binlog 写入，那么 tidb 会在一个时间周期后，将 tidb server 监听端口踢出，所以报错为 "[error] listener stopped, waiting for manual kill."
###### tidb server 添加 enable_binlog 启动参数后，每次集群的启停需要先启动 pump servers ，然后再启动 tidb servers 。而关闭则需要先关闭 tidb servers ，再关闭 pump servers 。
这个机制实际上是为了保护 enable_binlog 启动模式下的数据一致性问题。
#### 解决办法及注意事项
###### pump servers 启停需要和 tidb servers 有关联关系，可以避免此报错。
###### 如果单独启停 pump servers ，需要重启 tidb servers。 
