---
title: Megacli  组装 RAID
date: 2009-01-01 21:40:52
updated: 2009-01-01 21:40:52
toc: true
categories:
  - software
tags:
  - Megacli
  - RAID
---
# Megacli  组装 RAID

## 文档引用

- [创建一个 raid 10](https://serverfault.com/questions/519917/how-to-create-raid-10-with-megacli)
- [Dell Raid 卡命令行控制 megacli](http://www.gaizaoren.org/archives/tag/megacli)
- [常用 PC 服务器阵列卡、硬盘健康监控](http://imysql.com/tag/megacli)
- [Megacli 常用命令汇总](http://www.opstool.com/article/184)
- [MegaCli 监控 raid 状态](http://blog.chinaunix.net/uid-25135004-id-3139293.html)
- [cc 校验 / 初始化](http://w55554.blog.51cto.com/947626/1211844)
- [wiki](https://www.xargs.cn/doku.php/lsi:megacli%E5%AE%8C%E6%95%B4%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8C)
- [服务器阵列卡及硬盘健康监控](http://blog.phpdba.com/post/685)

## DISK Metrics

### IO 操作

- 读写 IO(Read/Write IO) 操作
- 单个 IO 操作
- 随机访问 (Random Access) 与连续访问 (Sequential Access)
- 顺序 IO 模式 (Queue Mode)/ 并发 IO 模式 (Burst Mode)
- 单个 IO 的大小 (IO Chunk Size)

### 磁盘指标

- IOPS (IO per Second)
- IO 响应时间 (IO Response Time)
- 传输速度 (Transfer Rate)/ 吞吐率 (Throughput)

## RAID (Redundant Array Of Inexpensive Disks)

RAID0，RAID1，RAID5，RAID6，RAID10

### RAID Level 介绍

- RAID0

　　RAID0 将数据条带化 (striping) 将连续的数据分散在多个磁盘上进行存取，系统发出的 IO 命令 (不管读 IO 和写 IO 都一样) 就可以在磁盘上被并行的执行，每个磁盘单独执行自己的那一部分请求，这样的并行的 IO 操作能大大的增强整个存储系统的性能。假设一个 RAID0 阵列有 n(n>=2) 个磁盘组成，每个磁盘的随机读写的 IO 能力都达到 140 的话，那么整个磁盘阵列的 IO 能力将是 140*n。同时如果在阵列总线的传输能力允许的话 RAID0 的吞吐率也将是单个磁盘的 n 倍。

- RAID1

　　RAID1 在容量上相当于是将两个磁盘合并成一个磁盘来使用了，互为镜像的两个磁盘里面保存的数据是完全一样的，因此在并行读取的时候速度将是 n 个磁盘速度的总和，但是写入就不一样了，每次写入都必须同时写入到两个磁盘中，因此写入速度只有 n/2。

- RAID5

　　我们那一个有 n(n>=3) 个磁盘的 RAID5 阵列来看，首先看看 RAID5 阵列的读 IO，RAID5 是支持并行 IO 的，而磁盘上的数据呈条带状的分布在所有的磁盘上，因此读 IO 的速度相当于所有磁盘速度的总和。不过这是在没有磁盘损坏的情况下，当有一个磁盘故障的时候读取速度也是会下降的，因为中间需要花时间来计算丢失磁盘上面的数据。

　　读取数据的情况相对就要复杂的多了，先来看下 RAID5 奇偶校验数据写入的过程，我们把写入的数据称为 D1，当磁盘拿到一个写 IO 的命令的时候，它首先会读取一次要入的地址的数据块中修改之前的数据 D0，然后再读取到当前条带中的校验信息 P0，接下来就根据 D0，P0，D1 这三组数据计算出数据写入之后的条带的奇偶校验信息 P1，最后发出两个写 IO 的命令，一个写入 D1，另一个写入奇偶校验信息 P1。可以看出阵列在实际操作的时候需要读、读、写、写一共 4 个 IO 才能完成一次写 IO 操作，也就是实际上的写入速度只有所有磁盘速度总和的 1/4。从这点可以看出 RAID5 是非常不适合用在要大批量写入数据的系统上的。

- RAID6

　　RAID6 和 RAID5 很类似，差别就在于 RAID6 多了一个用于校验的磁盘。就写 IO 速度上来说这两个是完全一样的，都是所有磁盘 IO 速度的总和。

　　在写 IO 上也很是类似，不同的是 RAID 将一个命令分成了三次读、三次写一共 6 次 IO 命令才能完成，也就是 RAID6 实际写入磁盘的速度是全部磁盘速度之和的 1/6。可以看出从写 IO 看 RAID6 比 RAID5 差别是很大的。

- RAID10

　　RAID0 读写速度都很好，却没有冗余保护; RAID5 和 RAID6 都有同样的毛病就是写入的时候慢，读取的时候快。那么 RAID1 呢? 嗯，这里要说的就是 RAID1，其实不管是 RAID10 还是 RAID01，其实都是组合大于 2 块磁盘时候的 RAID1，当先镜像后条带时候就称为 RAID10，先条带后镜像的时候称为 RAID01。从性能上看 RAID01 和 RAID10 都是一样的，都是 RAID1 嘛，但是 RAID10 在重建故障磁盘的时候性能比 RAID01 要快。

　　因为 RAID10 其实就是 RAID1，所以它的性能与 RAID1 也就是一样的了，这里不需要再做过多的讨论。

### RAID Cache 策略

- 常见磁盘 RAID Cache 策略
  1. WT (Write through
  2. WB (Write back)
  3. NORA (No read ahead)
  4. RA (Read ahead)
  5. ADRA (Adaptive read ahead)
  6. Cached
  7. Direct

- 缓存策略
  - 查看 RAID 卡缓存大小 (缓存为 0 可直接跳过此部分)
  - `/usr/local/hwtool/tool/MegaCli adpAllInfo aAll | grep "Memory Size"`

- 写策略: 二选一
  - WB - WriteBack 回写模式，使用缓存加速写入，提高写性能 (有缓存的默认模式) 直写模式，不使用缓存，数据直接写入硬盘 (无缓存的默认模式)
  - WT - WriteThrough

- 读策略: 三选一
  - RA - ReadAhead 预读模式，可提高顺序读取性能，但会降低随机读取性能 (有缓存的默认模式)
  - NoRA - ReadAheadNone 关闭预读，不使用缓存 (无缓存的默认模式)
  - AdRA - ReadAdaptive 自适应预读模式，仅当最近 2 个读请求访问顺序扇区时开启预读；如果随后是随机读请求，返回到无预读模式，继续监控

- cache 策略: 二选一 (与读策略不冲突)

  - Direct 使用 RAID 卡的缓存提高读命中率 (默认模式，一般性能比 Cached 好)
  - Cached 使用内存作 Cache 提高读命中率

- BBU 策略: 二选一 (BBU 是电池模块，给缓存供电的)
  - CachedBadBBU BBU 故障时仍使用 WB 模式，如果断电，有数据丢失风险
  - NoCachedBadBBU BBU 故障时切回 WT 模式，写性能下降，断电也不会丢失数据 (默认模式)

如果有缓存，推荐 HDD 采用 WB 模式; SSD 采用 WT 模式。

----

## MegaCli Software

### MegaCli Install

- 下载 MegaCli
  - http://www.lsi.com/support/Pages/download-results.aspx?keyword=MegaCLI

- 先安装 Lib_Utils-1.00-09.noarch.rpm，接着安装 MegaCli-8.07.14-1.noarch.rpm

  - rpm -ivh Lib_Utils-1.00-09.noarch.rpm
  - rpm -ivh MegaCli-8.07.14-1.noarch.rpm

- 安装完成后，产生以下文件夹，有日志信息和可执行命令

  - /opt/MegaRAID/MegaCli

- 建立链接，以更方便使用：

  - ln -s /opt/MegaRAID/MegaCli/MegaCli64 /usr/local/bin/megacli

### modinfo 查看设备

```bash
[root@tidb-online-tikv-1 ~]# modinfo nvme
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/drivers/block/nvme.ko
version:        1.0
license:        GPL
author:         Matthew Wilcox <willy@linux.intel.com>
rhelversion:    7.2
srcversion:     71E0CF0D222671148201A53
alias:          pci:v*d*sv*sd*bc01sc08i02*
depends:
intree:         Y
vermagic:       3.10.0-327.el7.x86_64 SMP mod_unload modversions
signer:         CentOS Linux kernel signing key
sig_key:        79:AD:88:6A:11:3C:A0:22:35:26:33:6C:0F:82:5B:8A:94:29:6A:B3
sig_hashalgo:   sha256
parm:           admin_timeout:timeout in seconds for admin commands (byte)
parm:           io_timeout:timeout in seconds for I/O (byte)
parm:           shutdown_timeout:timeout in seconds for controller shutdown (byte)
parm:           nvme_major:int
parm:           nvme_char_major:int
parm:           use_threaded_interrupts:int
```

### 使用 MegaCli 重组 RAID

- 查看状态
  - megacli -LdInfo -LALL -aAll
- 删掉已有 raid
  - megacli -cfglddel -L1 -a0
  - megacli -cfglddel -L2 -a0
  - megacli -cfglddel -L3 -a0
  - megacli -cfglddel -L4 -a0
  - megacli -cfglddel -L5 -a0
  - megacli -cfglddel -L6 -a0
  - megacli -cfglddel -L7 -a0
  - megacli -cfglddel -L8 -a0
  - megacli -cfglddel -L9 -a0
  - megacli -cfglddel -L10 -a0
  - 其中 L1 替换数字 1，2，3，4 一直到这些单盘 raid0 删干净
- 创建 raid 10
  - megacli -CfgSpanAdd -r50 -Array0[8:2,8:3,8:4,8:5,8:6] Array1[8:7,8:8,8:9,8:10,8:11] WB RA Direct CachedBadBBU -a0
- 查看进度
  - megacli -LDBI -ShowProg -LALL -aALL
- cc 校验进度
  - megacli -ldcc -progdsply -L1 -a0
- 查看 cc 校验计划
  - megacli -adpccsched -info -a0
- 快速初始化
  - megacli -LDInit  -start –L1  -a0
- 查看当前所有 raid
  - megacli -ldinfo -lall -aall
- 获取磁盘标识
  - megacli -PDList -aAll -NoLog | grep -Ei "(enclosure|slot)"
  - megacli -PDlist -aall | grep -e '^Enclosure Device ID:' -e '^Slot Number:'

### 常用命令

- megacli -LdPdInfo -aAll -NoLog 显示所有 raid 信息
- megacli -LDInfo -Lall -aALL 查 raid 级别
- megacli -AdpAllInfo -aALL 查 raid 卡信息
- megacli -PDList -aALL 查看硬盘信息
- megacli -AdpBbuCmd -aAll 查看电池信息
- megacli -FwTermLog -Dsply -aALL 查看 raid 卡日志
- megacli -adpCount 显示适配器个数
- megacli -AdpGetTime –aALL 显示适配器时间
- megacli -AdpAllInfo -aAll 显示所有适配器信息
- megacli -LDInfo -LALL -aAll 显示所有逻辑磁盘组信息
- megacli -PDList -aAll 显示所有的物理信息
- megacli -AdpBbuCmd -GetBbuStatus -aALL | grep 'Charger Status' 查看充电状态
- megacli -AdpBbuCmd -GetBbuStatus -aALL 显示 BBU 状态信息
- megacli -AdpBbuCmd -GetBbuCapacityInfo -aALL 显示 BBU 容量信息
- megazli -AdpBbuCmd -GetBbuDesignInfo -aALL 显示 BBU 设计参数
- megacli -AdpBbuCmd -GetBbuProperties -aALL 显示当前 BBU 属性
- megacli -cfgdsply -aALL 显示 Raid 卡型号，Raid 设置，Disk 相关信息

然后用   选项创建一个 -r10,  就是 raid 10
比较容易，但是 CfgSpanAdd 需要的参数要从  -cfgdsply -aALL  列出的参数获取
稍微查下 cfgspanadd 就知道了

### 磁盘缓存策略

- 查看磁盘缓存策略
  - /opt/MegaRAID/MegaCli/MegaCli64 -LDGetProp -Cache -L0 -a0
  - `Adapter 0-VD 1(target id: 1): Cache Policy:WriteBack, ReadAhead, Direct, No Write Cache if bad BBU`
- 默认磁盘缓存策略
  - 读取策略：自适应
  - 写策略：回写
  - 磁盘高速缓存策略：禁用

----

## FAQ

### 哪些场合适合使用大缓存的 RAID 卡

- 缓存替换策略也能影响缓存大小的选择：如果某个缓存替换策略足够优秀，跟其他缓存策略相比，即使缓存很小，依然能够维持较高的缓存命中率，达到更好的性能；

- 跟缓存策略也有关：如果是写透缓存，对于写请求来说，其实缓存是起不到加速性能的作用，且如果写请求相对于读请求很多，那么相当于实际上还是直接写磁盘了，如果读请求与写请求无关，那缓存的写数据基本上也没意义了；如果是写回缓存，那么缓存越大就能存储更多的写入数据，这些数据在满足一定的条件 (如时间超过某个阈值、缓存替换算法选择将其换出等) 才写到磁盘，增加了写合并的可能，可以提升性能；

- 跟 IO 请求的特征也有关：因为实际上对操作系统来说，一般 OS 对写都是异步的，而对读则是同步的，实际上真正导致 IO 性能的瓶颈是 OS 对读请求的响应能力。如果读请求没有规律，且也没有部分数据频繁读取的情况，那么大缓存与小缓存的性能其实差别不大（当然，你要是选择一个基本跟底层存储磁盘一样大的缓存那就另当别论了）。实际上，如果读取的数据具有空间和时间的局部性规律或者写入时缓存的策略是写回且写入的数据在不久的将来就会访问到，那么的确是缓存越大越好，但是如果应用不具备这样的特性，那么选择一个相对于底层存储磁盘来说大小有限的缓存实际上也没办法带来性能的显著提升。

总体来说，实际上是应用的 IO 特性决定了最合适的缓存大小。