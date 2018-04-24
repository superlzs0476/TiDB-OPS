---
title: 主机信息收集与压力测试
date: 2017-06-01 21:40:52
updated: 2017-06-06 21:40:52
categories:
  - Deploy
tags:
  - Benchmark
---
# 主机信息收集与压力测试

## 主机信息收集

- [Linux 查看 CPU 信息，机器型号，内存等信息](https://my.oschina.net/hunterli/blog/140783)

### 查看系统信息

- centos-release
  - cat /etc/redhat-release
- dmidecode
- hostname
- issue
  - cat /etc/issue
- lsmod
- lsof
- lspci -t
- lspci -vvvvv
- modinfo shannon
- os-release
  - cat /etc/os-release
- ps aux
- ps aux --sort start_time
- pstree
- redhat-release -> centos-release
- system-release -> centos-release
- uname -a
- w

### 硬件信息

- block
- bus
- devices
- fs
- module

### perf

- iostat -dmx 1 5
- sar
- sar -A
- sar -r
- top -bc -d1 -n5
- vmstat 1 5

### numa

- numactl --hardware
- numactl --show
- numastat

### log

- dmesg
- kern.log
- messages

### disk

- df -h
- dmsetup status
- dmsetup table
- fstab
- fuser-df
- fuser-fio
- lvs
- mtab -> /proc/self/mounts
- pvs
- shannon-status
- shannon-status-fio
- vgs

### debug disk

- all blocks
- bad blocks
- badlun bitmap
- cold blocks
- command queue
- command queue raw data
- err blks
- free blocks
- gc state
- hot blocks
- l2p lunmap
- lun statistics
- mbr
- queue depth
- read count
- registers
- request queue
- rmw list
- statistics
- voltage temperature
- wait copy
- wait erased
- waitqueue

## Benchmark

### Vertica 压测工具

- [下载地址](https://my.vertica.com/download/vertica/client-drivers/)
- [官方文档](https://my.vertica.com/docs/9.0.x/HTML/index.htm)
  - [Vcpuperf](https://my.vertica.com/docs/8.1.x/HTML/index.htm#Authoring/InstallationGuide/scripts/vcpuperf.htm)
  - [vnetperf](https://my.vertica.com/docs/8.1.x/HTML/index.htm#Authoring/InstallationGuide/scripts/vnetperf.htm)
  - [vioperf](https://my.vertica.com/docs/8.1.x/HTML/index.htm#Authoring/InstallationGuide/scripts/vioperf.htm)

### netperf

- [netperf 与网络性能测量](https://www.ibm.com/developerworks/cn/linux/l-netperf/)

### FIO

- [Flexible I/O Tester](https://github.com/axboe/fio)

- fio 测试命令（10G 文件顺序读写）在待测数据目录下执行

```BASH
# 4k 基准连续顺序写
fio -ioengine=libaio -bs=4k -direct=1 -thread -rw=write -size=10G -filename=PingCAP-test -name="PingCAP" -iodepth=4 -runtime=60
# 4k 基准连续顺序读
fio -ioengine=libaio -bs=4k -direct=1 -thread -rw=read -size=10G -filename=PingCAP-test -name="PingCAP" -iodepth=4 -runtime=60
# 32k 基准随机读
fio -ioengine=libaio -bs=32k -direct=1 -thread -rw=randread -size=10G -filename=PingCAP-test -name="PingCAP" -iodepth=4 -runtime=60
```

### DD

- [dd](https://en.wikipedia.org/wiki/Dd_(Unix))

```BASH
# 4k 基准测试
dd bs=4k count=2560000 if=/dev/zero of=test conv=fdatasync
# 4k 基准测试
dd bs=4k count=2560000 if=/dev/zero of=test oflag=direct
# 4k 基准测试
dd bs=4k count=2560000 if=/dev/zero of=test oflag=dsync
```

- 指定数字的地方若以下列字符结尾，则乘以相应的数字:
  - b=512, c=1, k=1024, w=2, xm=number m
- if=file    # 输入文件名，缺省为标准输入。
- of=file    # 输出文件名，缺省为标准输出。
- ibs=bytes  # 一次读入 bytes 个字节 (即一个块大小为 bytes 个字节)。
- obs=bytes  # 一次写 bytes 个字节 (即一个块大小为 bytes 个字节)。
- bs=bytes   # 同时设置读写块的大小为 bytes ，可代替 ibs 和 obs 。
- cbs=bytes  # 一次转换 bytes 个字节，即转换缓冲区大小。
- skip=blocks   # 从输入文件开头跳过 blocks 个块后再开始复制。
- seek=blocks   # 从输出文件开头跳过 blocks 个块后再开始复制。(通常只有当输出文件是磁盘或磁带时才有效)。
- count=blocks  # 仅拷贝 blocks 个块，块大小等于 ibs 指定的字节数。
- -fsync 同样也是将数据已经写入磁盘，但是是在经过缓存后最后再写入硬盘
- -dsync 可以当成是模拟数据库插入操作，在 / dev/zone 中读出一条数据就立即写入硬盘
- dd 命令有一组参数 oflag 和 iflag, 控制源文件和目标文件的读写方式为 direct IO，即读或写文件时越过操作系统的读写 buffer。如果指定 oflag=direct,nonblock，写文件时忽略 cache 的影响；而如果指定 iflag=direct,nonblock，读文件时忽略 cache 的影响

----

## 检测运维工具

### 系统工具

#### cURL

#### top

> 监视系统进程使用 CPU/MEM / 等资源命令

- top
- top -p pid
  - 根据 进程 pid 查询资源占用

#### lsof

> 查看主机端口占用，进程占用文件

- lsof -i:80
  - 查看 80 端口占用情况
- lsof -c pd-server
  - 进程占用文件情况
- lsof -i@172.16.10.64
  - 列出 IP 占用情况

### 第三方组件

#### htop

#### iostat

- 检测磁盘性能
  - e.g: iostat -d -x -m 5

#### iotop

- 监视磁盘 I/O 使用状况，可以观察到进程占拥 IO 情况
  - e.g: iotop

#### vmstat


#### tcpdump

tcpdump -i bond0 -w /data/online.pcap -s 0 port 4000

其中 -i 指定网卡，心动机器都是 bond0，port 指定过滤端口，即只采集 4000 端口的流量，-w 指定保存文件，这里通过 mount 映射，实际数据保存在 Host 的 /data/pingcap/tcpdump/online.pcap，-s 指定 snapshot length，设为 0 表示用默认的 65536 bytes

sudo tcpdump -i any -s 0 -l -w - dst  port 3306 | strings

分别对两个驱动包抓包看了下
你们用的那个驱动包，抓出来的数据，转成 string 看起来是没有 timestamp 的数据，我没有看 16 进制编码，不知道是不是变成了什么不可见的字符

用我发的那个驱动包，抓出来的数据，是全的， timestamp 列的数据也是可见的

### sar

yum install sysstat

第一次执行提示错误
Cannot open /var/log/sa/sa22: No such file or directory
22 是指当天的日期

生成相应文件
sar -o 2 3
在对应的 / var/log/sa / 目录下就有对应的日志文件了

### 文本处理

- grep
  - -o
  - -A
  - -B
  - -C
  - -R
- more
- less
- tail
  - tailf == tail -f
- sed
- awk

----

## 优化

### 磁盘优化

- [调度器说明](http://blog.163.com/digoal@126/blog/static/1638770402015117214118/)
- [PCIE-SSD](https://stackoverflow.com/questions/27664334/selecting-the-right-linux-i-o-scheduler-for-a-host-equipped-with-nvme-ssd)

- 查看 linux 文件系统调度器
  - `cat /sys/block/sdb/queue/scheduler`
- 临时修改文件调度器
  - `echo deadline > /sys/block/sde/queue/scheduler`
- 开启
  - `echo "never" >  /sys/kernel/mm/transparent_hugepage/enabled`
- 文中的 elevator=cfg 替换为 elevator=deadline
  - `vi /etc/default/grub`
- 修改挂载参数
  - `mount  data=ordered,nodiratime,noatime`

### 内核优化

- [sysctl.conf](https://wiki.archlinux.org/index.php/sysctl) 内容介绍

net.core.rmem_default
net.core.wmem_default

net.core.rmem_max
net.core.wmem_max

net.ipv4.tcp_rmem
net.ipv4.tcp_wmem
