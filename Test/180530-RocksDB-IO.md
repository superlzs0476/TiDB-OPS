---
title: RocksDB 磁盘写入模拟测试
date: 2018-05-30 21:40:52
updated: 2018-06-01 21:40:52
categories:
  - Test
tags:
  - RocksDB
  - FIO
  - LSM
---
# RocksDB 磁盘写入模拟测试

## 理解 LSM 存储与 RocksDB 写入方式

- [RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)

> block_size -- RocksDB packs user data in blocks. When reading a key-value pair from a table file, an entire block is loaded into memory. Block size is 4KB by default. Each table file contains an index that lists offsets of all blocks. Increasing block_size means that the index contains fewer entries (since there are fewer blocks per file) and is thus smaller. Increasing block_size decreases memory usage and space amplification, but increases read amplification.

- [Direct-IO](https://github.com/facebook/rocksdb/wiki/Direct-IO)

```raft
  // Use O_DIRECT for user reads
  // Default: false
  // Not supported in ROCKSDB_LITE mode!
  bool use_direct_reads = false;

  // Use O_DIRECT for both reads and writes in background flush and compactions
  // When true, we also force new_table_reader_for_compaction_inputs to true.
  // Default: false
  // Not supported in ROCKSDB_LITE mode!
  bool use_direct_io_for_flush_and_compaction = false;
```

### direct_io

串行批量连续插入，这对 MySQL 来说就是连续顺序写磁盘，还没有网络消耗。
对于 tidb 来说，这个是磁盘要考虑 reigon 分裂，tidb 到 tikv 之间的网络消耗，事务大小限制等多个方面

uint64_t wal_bytes_per_sync = 0;

注意 direct_io 对 WAL 是不生效的

```bash
set_use_direct_io_for_flush_and_compaction

Use O_DIRECT for both reads and writes in background flush and compactions
use-direct-io-for-flush-and-compaction = false
```

## 模拟 IO 场景

- 大事务插入提交
- region blanch
  - region add peer
  - region delete
- delete limit
  - delete range
- drop table

## 使用 FIO 工具模拟测试

- [fio](https://linux.die.net/man/1/fio) 命令行参数、使用手册

- [pread, pwrite](https://linux.die.net/man/2/pread) read from or write to a file descriptor at a given offset

pread() reads up to count bytes from file descriptor fd at offset offset (from the start of the file) into the buffer starting at buf. The file offset is not changed.

pwrite() writes up to count bytes from the buffer starting at buf to the file descriptor fd at offset offset. The file offset is not changed.

The file referenced by fd must be capable of seeking.

### FIO engine

libaio 等异步 IO engine 中，会调用 io_setup 准备可以一次提交 iodepth 个 IO 的上下文，同时申请个 io 请求队列用于保持 IO。 在压测进行的时候，系统会生成特定的 IO 请求，往 io 请求队列里面扔，当队列里面的 IO 个数达到 iodepth_batch 值的时候，就调用 io_submit 批次提交请求，然后开始调用 io_getevents 开始收割已经完成的 IO。 每次收割多少呢？由于收割的时候，超时时间设置为 0，所以有多少已完成就算多少，最多可以收割 iodepth_batch_complete 值个。随着收割，IO 队列里面的 IO 数就少了，那么需要补充新的 IO。 什么时候补充呢？当 IO 数目降到 iodepth_low 值的时候，就重新填充，保证 OS 可以看到至少 iodepth_low 数目的 io 在电梯口排队着。

局部性是计算机中一种预测行为，通过缓存、内存中预取指令、处理器管道分支预测等技术来提高性能；更多参见《操作系统精髓与设计原理》。


### FIO 测试目标

```bash
➜  [20.1] ~ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       1.5T  1.1T  308G  78% /
devtmpfs         63G     0   63G   0% /dev
tmpfs            63G  4.0K   63G   1% /dev/shm
tmpfs            63G  4.0G   59G   7% /run
tmpfs            63G     0   63G   0% /sys/fs/cgroup
/dev/nvme0n1    344G   38G  289G  12% /data3
/dev/sdb        1.5T  101G  1.3T   8% /data2
/dev/sda1       190M  101M   75M  58% /boot
tmpfs            13G     0   13G   0% /run/user/1000
tmpfs            13G     0   13G   0% /run/user/1001
tmpfs            13G     0   13G   0% /run/user/1002
```

### FTO 测试脚本

- SSD 检查是否需要开启 trim

```conf
[global]
ioengine=psync
fdatasync=1
thread=1
runtime=60

[write128]
rw=write
bs=128k
size=10G
filename=PingCAP-write128
name="PingCAP-write-128"


[randread128]
rw=randread
bs=128k
size=10G
filename=PingCAP-randread128
name="PingCAP-randread-128"

[write256]
rw=write
bs=256k
size=10G
filename=PingCAP-write256
name="PingCAP-write-256"


[randread256]
rw=randread
bs=256k
size=10G
filename=PingCAP-randread256
name="PingCAP-randread-256"
```

### FIO SSD 测试

> FIO 并行测试

```bash
➜  [20.1] /data3 /home/pingcap/jeff-test/tidb-ansible/resources/bin/fio fio.conf

"PingCAP-randread-128": (g=0): rw=write, bs=128K-128K/128K-128K/128K-128K, ioengine=psync, iodepth=1
"PingCAP-randread-128": (g=0): rw=randread, bs=128K-128K/128K-128K/128K-128K, ioengine=psync, iodepth=1
"PingCAP-randread-256": (g=0): rw=write, bs=256K-256K/256K-256K/256K-256K, ioengine=psync, iodepth=1
"PingCAP-randread-256": (g=0): rw=randread, bs=256K-256K/256K-256K/256K-256K, ioengine=psync, iodepth=1
fio-2.16-4-g3544
Starting 4 threads
"PingCAP-randread-128": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-128": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-256": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-256": Laying out IO file(s) (1 file(s) / 10240MB)
Jobs: 1 (f=1): [W(1),_(3)] [97.0% done] [0KB/497.8MB/0KB /s] [0/3982/0 iops] [eta 00m:01s]
"PingCAP-randread-128": (groupid=0, jobs=1): err= 0: pid=39694: Wed May 30 17:49:50 2018
  write: io=10240MB, bw=330063KB/s, iops=2578, runt= 31769msec
    clat (usec): min=81, max=882, avg=108.69, stdev=14.36
     lat (usec): min=82, max=883, avg=110.79, stdev=14.54
    clat percentiles (usec):
     |  1.00th=[88],  5.00th=[   92], 10.00th=[   97], 20.00th=[  100],
     | 30.00th=[102], 40.00th=[  103], 50.00th=[  105], 60.00th=[  107],
     | 70.00th=[111], 80.00th=[  116], 90.00th=[  123], 95.00th=[  133],
     | 99.00th=[169], 99.50th=[  185], 99.90th=[  213], 99.95th=[  219],
     | 99.99th=[243]
    lat (usec) : 100=15.06%, 250=84.93%, 500=0.01%, 750=0.01%, 1000=0.01%
  cpu          : usr=1.15%, sys=36.81%, ctx=175872, majf=0, minf=297
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=81920/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-128": (groupid=0, jobs=1): err= 0: pid=39695: Wed May 30 17:49:50 2018
  read : io=10240MB, bw=746105KB/s, iops=5828, runt= 14054msec
    clat (usec): min=84, max=595, avg=170.51, stdev=67.17
     lat (usec): min=84, max=595, avg=170.58, stdev=67.17
    clat percentiles (usec):
     |  1.00th=[93],  5.00th=[   96], 10.00th=[  101], 20.00th=[  109],
     | 30.00th=[120], 40.00th=[  139], 50.00th=[  153], 60.00th=[  169],
     | 70.00th=[203], 80.00th=[  229], 90.00th=[  266], 95.00th=[  302],
     | 99.00th=[370], 99.50th=[  394], 99.90th=[  454], 99.95th=[  470],
     | 99.99th=[524]
    lat (usec) : 100=8.35%, 250=78.06%, 500=13.57%, 750=0.02%
  cpu          : usr=0.92%, sys=29.89%, ctx=81929, majf=0, minf=191
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=81920/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-256": (groupid=0, jobs=1): err= 0: pid=39696: Wed May 30 17:49:50 2018
  write: io=10240MB, bw=485879KB/s, iops=1897, runt= 21581msec
    clat (usec): min=162, max=973, avg=219.54, stdev=27.10
     lat (usec): min=164, max=976, avg=224.13, stdev=27.50
    clat percentiles (usec):
     |  1.00th=[171],  5.00th=[  181], 10.00th=[  187], 20.00th=[  193],
     | 30.00th=[205], 40.00th=[  215], 50.00th=[  221], 60.00th=[  227],
     | 70.00th=[231], 80.00th=[  239], 90.00th=[  249], 95.00th=[  262],
     | 99.00th=[298], 99.50th=[  318], 99.90th=[  366], 99.95th=[  398],
     | 99.99th=[470]
    lat (usec) : 250=90.13%, 500=9.86%, 750=0.01%, 1000=0.01%
  cpu          : usr=1.34%, sys=52.99%, ctx=324050, majf=0, minf=494
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=40960/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-256": (groupid=0, jobs=1): err= 0: pid=39697: Wed May 30 17:49:50 2018
  read : io=10240MB, bw=746636KB/s, iops=2916, runt= 14044msec
    clat (usec): min=179, max=775, avg=341.74, stdev=85.41
     lat (usec): min=179, max=775, avg=341.81, stdev=85.40
    clat percentiles (usec):
     |  1.00th=[193],  5.00th=[  211], 10.00th=[  229], 20.00th=[  262],
     | 30.00th=[290], 40.00th=[  314], 50.00th=[  338], 60.00th=[  366],
     | 70.00th=[386], 80.00th=[  414], 90.00th=[  454], 95.00th=[  490],
     | 99.00th=[556], 99.50th=[  580], 99.90th=[  644], 99.95th=[  668],
     | 99.99th=[716]
    lat (usec) : 250=15.21%, 500=80.87%, 750=3.92%, 1000=0.01%
  cpu          : usr=0.50%, sys=28.28%, ctx=81925, majf=0, minf=360
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=40960/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: io=20480MB, aggrb=1457.3MB/s, minb=746105KB/s, maxb=746636KB/s, mint=14044msec, maxt=14054msec
  WRITE: io=20480MB, aggrb=660125KB/s, minb=330062KB/s, maxb=485879KB/s, mint=21581msec, maxt=31769msec

Disk stats (read/write):
  nvme0n1: ios=163840/397493, merge=0/1051248, ticks=20469/34542, in_queue=55015, util=71.68%
```

> iostat

```bash
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00     0.00    1.00 56196.00     4.00 224784.00     8.00     0.63    0.01    0.00    0.01   0.01  62.65
sda               0.00    10.00    0.00    7.50     0.00    70.00    18.67     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.05    0.00    0.95    1.34    0.00   97.65

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00     8.00    0.00 56876.00     0.00 227534.00     8.00     0.59    0.01    0.00    0.01   0.01  59.35
sda               0.00     0.00    0.00    0.50     0.00     2.00     8.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

> dd 测试

```bash
➜  [20.1] /data3 dd bs=4k count=2560000 if=/dev/zero of=test oflag=direct
2560000+0 records in
2560000+0 records out
10485760000 bytes (10 GB) copied, 45.7644 s, 229 MB/s
```

### FIO SAS (RAID) 测试

> FIO 测试

```bash
➜  [20.1] /data2 /home/pingcap/jeff-test/tidb-ansible/resources/bin/fio /data3/fio.conf
"PingCAP-randread-128": (g=0): rw=write, bs=128K-128K/128K-128K/128K-128K, ioengine=psync, iodepth=1
"PingCAP-randread-128": (g=0): rw=randread, bs=128K-128K/128K-128K/128K-128K, ioengine=psync, iodepth=1
"PingCAP-randread-256": (g=0): rw=write, bs=256K-256K/256K-256K/256K-256K, ioengine=psync, iodepth=1
"PingCAP-randread-256": (g=0): rw=randread, bs=256K-256K/256K-256K/256K-256K, ioengine=psync, iodepth=1
fio-2.16-4-g3544
Starting 4 threads
"PingCAP-randread-128": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-128": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-256": Laying out IO file(s) (1 file(s) / 10240MB)
"PingCAP-randread-256": Laying out IO file(s) (1 file(s) / 10240MB)
Jobs: 2 (f=2): [_(1),r(1),_(1),r(1)] [40.8% done] [261.8MB/0KB/0KB /s] [1579/0/0 iops] [eta 01m:27s]
"PingCAP-randread-128": (groupid=0, jobs=1): err= 0: pid=39809: Wed May 30 17:53:18 2018
  write: io=10240MB, bw=208610KB/s, iops=1629, runt= 50265msec
    clat (usec): min=67, max=401, avg=106.03, stdev=19.99
     lat (usec): min=69, max=403, avg=108.07, stdev=20.27
    clat percentiles (usec):
     |  1.00th=[79],  5.00th=[   81], 10.00th=[   84], 20.00th=[   89],
     | 30.00th=[94], 40.00th=[  100], 50.00th=[  103], 60.00th=[  107],
     | 70.00th=[112], 80.00th=[  120], 90.00th=[  131], 95.00th=[  141],
     | 99.00th=[179], 99.50th=[  189], 99.90th=[  221], 99.95th=[  249],
     | 99.99th=[294]
    lat (usec) : 100=39.91%, 250=60.04%, 500=0.05%
  cpu          : usr=0.80%, sys=23.94%, ctx=191847, majf=0, minf=225
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=81920/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-128": (groupid=0, jobs=1): err= 0: pid=39810: Wed May 30 17:53:18 2018
  read : io=1986.9MB, bw=33909KB/s, iops=264, runt= 60001msec
    clat (usec): min=547, max=598302, avg=3773.02, stdev=6992.49
     lat (usec): min=547, max=598302, avg=3773.16, stdev=6992.52
    clat percentiles (usec):
     |  1.00th=[652],  5.00th=[  748], 10.00th=[  788], 20.00th=[  860],
     | 30.00th=[900], 40.00th=[  948], 50.00th=[ 1032], 60.00th=[ 1848],
     | 70.00th=[7072], 80.00th=[ 7648], 90.00th=[ 8768], 95.00th=[ 9408],
     | 99.00th=[10432], 99.50th=[10944], 99.90th=[20608], 99.95th=[104960],
     | 99.99th=[272384]
    lat (usec) : 750=5.45%, 1000=40.74%
    lat (msec) : 2=13.99%, 4=1.55%, 10=36.43%, 20=1.73%, 50=0.06%
    lat (msec) : 250=0.04%, 500=0.01%, 750=0.01%
  cpu          : usr=0.09%, sys=1.78%, ctx=15899, majf=0, minf=266
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=15895/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-256": (groupid=0, jobs=1): err= 0: pid=39811: Wed May 30 17:53:18 2018
  write: io=10240MB, bw=282057KB/s, iops=1101, runt= 37176msec
    clat (usec): min=144, max=860, avg=207.76, stdev=34.55
     lat (usec): min=146, max=863, avg=211.35, stdev=35.02
    clat percentiles (usec):
     |  1.00th=[161],  5.00th=[  169], 10.00th=[  173], 20.00th=[  179],
     | 30.00th=[185], 40.00th=[  191], 50.00th=[  197], 60.00th=[  211],
     | 70.00th=[229], 80.00th=[  239], 90.00th=[  251], 95.00th=[  262],
     | 99.00th=[298], 99.50th=[  322], 99.90th=[  422], 99.95th=[  516],
     | 99.99th=[708]
    lat (usec) : 250=89.25%, 500=10.69%, 750=0.05%, 1000=0.01%
  cpu          : usr=0.68%, sys=29.51%, ctx=347679, majf=0, minf=428
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=40960/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1
"PingCAP-randread-256": (groupid=0, jobs=1): err= 0: pid=39812: Wed May 30 17:53:18 2018
  read : io=1952.3MB, bw=33318KB/s, iops=130, runt= 60001msec
    clat (usec): min=487, max=607506, avg=7680.66, stdev=11143.04
     lat (usec): min=487, max=607507, avg=7680.89, stdev=11143.13
    clat percentiles (usec):
     |  1.00th=[1480],  5.00th=[ 1624], 10.00th=[ 1688], 20.00th=[ 1784],
     | 30.00th=[1864], 40.00th=[ 1960], 50.00th=[ 2160], 60.00th=[ 5088],
     | 70.00th=[14784], 80.00th=[15808], 90.00th=[17024], 95.00th=[17792],
     | 99.00th=[19584], 99.50th=[21632], 99.90th=[111104], 99.95th=[203776],
     | 99.99th=[610304]
    lat (usec) : 500=0.01%, 1000=0.01%
    lat (msec) : 2=41.91%, 4=17.20%, 10=2.56%, 20=37.52%, 50=0.67%
    lat (msec) : 250=0.09%, 500=0.01%, 750=0.01%
  cpu          : usr=0.06%, sys=1.85%, ctx=15623, majf=0, minf=483
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=7809/w=0/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: io=3939.2MB, aggrb=67226KB/s, minb=33317KB/s, maxb=33908KB/s, mint=60001msec, maxt=60001msec
  WRITE: io=20480MB, aggrb=417219KB/s, minb=208609KB/s, maxb=282057KB/s, mint=37176msec, maxt=50265msec

Disk stats (read/write):
  sdb: ios=31153/328842, merge=0/704425, ticks=117703/63555, in_queue=181149, util=99.83%
```

> iostat

```bash
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00     0.00    0.00    4.50     0.00    18.00     8.00     0.00    0.11    0.00    0.11   0.11   0.05
sda               0.00     0.00    0.00    0.50     0.00     2.00     8.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     4.50    1.00 26931.50     4.00 107744.00     8.00     0.69    0.03    0.00    0.03   0.03  69.15

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.04    0.00    0.70    0.00    0.00   99.26

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00     0.50    0.00    5.50     0.00    24.00     8.73     0.00    0.09    0.00    0.09   0.09   0.05
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.50 27016.00     2.00 108064.00     8.00     0.67    0.02    0.00    0.02   0.02  66.60

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.03    0.00    0.79    0.01    0.00   99.17

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00     0.00    0.00    3.00     0.00    12.00     8.00     0.00    0.00    0.00    0.00   0.00   0.00
sda               0.00     8.50    0.00   22.00     0.00   122.00    11.09     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    1.00 27259.00     4.00 109036.00     8.00     0.65    0.02    0.00    0.02   0.02  65.35
```

> dd 测试

```bash
➜  [20.1] /data2 dd bs=4k count=2560000 if=/dev/zero of=test oflag=direct
2560000+0 records in
2560000+0 records out
10485760000 bytes (10 GB) copied, 93.6281 s, 112 MB/s
```