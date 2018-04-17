---
title: Node_exporter README
date: 2017-06-01 21:40:52
updated: 2017-06-06 21:40:52
categories:
  - Monitor
tags:
  - Node_exporter
---
# Node_exporter README

## Collector

- 通过 `http://IP:9100/metrics` 查看当前主机监控信息
- [Collector](https://github.com/prometheus/node_exporter/tree/master/collector)

- 以下内容引用官方 [README](https://github.com/prometheus/node_exporter/blob/master/README.md)

There is varying support for collectors on each operating system. The tables
below list all existing collectors and the supported systems.

Collectors are enabled by providing a `--collector.<name>` flag.
Collectors that are enabled by default can be disabled by providing a `--no-collector.<name>` flag.

### Enabled by default

Name     | Description | OS
---------|-------------|----
arp | Exposes ARP statistics from `/proc/net/arp`. | Linux
bcache | Exposes bcache statistics from `/sys/fs/bcache/`. | Linux
bonding | Exposes the number of configured and active slaves of Linux bonding interfaces. | Linux
conntrack | Shows conntrack statistics (does nothing if no `/proc/sys/net/netfilter/` present). | Linux
cpu | Exposes CPU statistics | Darwin, Dragonfly, FreeBSD, Linux
diskstats | Exposes disk I/O statistics. | Darwin, Linux
edac | Exposes error detection and correction statistics. | Linux
entropy | Exposes available entropy. | Linux
exec | Exposes execution statistics. | Dragonfly, FreeBSD
filefd | Exposes file descriptor statistics from `/proc/sys/fs/file-nr`. | Linux
filesystem | Exposes filesystem statistics, such as disk space used. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
hwmon | Expose hardware monitoring and sensor data from `/sys/class/hwmon/`. | Linux
infiniband | Exposes network statistics specific to InfiniBand and Intel OmniPath configurations. | Linux
ipvs | Exposes IPVS status from `/proc/net/ip_vs` and stats from `/proc/net/ip_vs_stats`. | Linux
loadavg | Exposes load average. | Darwin, Dragonfly, FreeBSD, Linux, NetBSD, OpenBSD, Solaris
mdadm | Exposes statistics about devices in `/proc/mdstat` (does nothing if no `/proc/mdstat` present). | Linux
meminfo | Exposes memory statistics. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
netdev | Exposes network interface statistics such as bytes transferred. | Darwin, Dragonfly, FreeBSD, Linux, OpenBSD
netstat | Exposes network statistics from `/proc/net/netstat`. This is the same information as `netstat -s`. | Linux
nfs | Exposes NFS client statistics from `/proc/net/rpc/nfs`. This is the same information as `nfsstat -c`. | Linux
nfsd | Exposes NFS kernel server statistics from `/proc/net/rpc/nfsd`. This is the same information as `nfsstat -s`. | Linux
sockstat | Exposes various statistics from `/proc/net/sockstat`. | Linux
stat | Exposes various statistics from `/proc/stat`. This includes boot time, forks and interrupts. | Linux
textfile | Exposes statistics read from local disk. The `--collector.textfile.directory` flag must be set. | _any_
time | Exposes the current system time. | _any_
timex | Exposes selected adjtimex(2) system call stats. | Linux
uname | Exposes system information as provided by the uname system call. | Linux
vmstat | Exposes statistics from `/proc/vmstat`. | Linux
wifi | Exposes WiFi device and station statistics. | Linux
xfs | Exposes XFS runtime statistics. | Linux (kernel 4.4+)
zfs | Exposes [ZFS](http://open-zfs.org/) performance statistics. | [Linux](http://zfsonlinux.org/)

### Disabled by default

Name     | Description | OS
---------|-------------|----
buddyinfo | Exposes statistics of memory fragments as reported by /proc/buddyinfo. | Linux
devstat | Exposes device statistics | Dragonfly, FreeBSD
drbd | Exposes Distributed Replicated Block Device statistics (to version 8.4) | Linux
interrupts | Exposes detailed interrupts statistics. | Linux, OpenBSD
ksmd | Exposes kernel and system statistics from `/sys/kernel/mm/ksm`. | Linux
logind | Exposes session counts from [logind](http://www.freedesktop.org/wiki/Software/systemd/logind/). | Linux
meminfo\_numa | Exposes memory statistics from `/proc/meminfo_numa`. | Linux
mountstats | Exposes filesystem statistics from `/proc/self/mountstats`. Exposes detailed NFS client statistics. | Linux
ntp | Exposes local NTP daemon health to check [time](./docs/TIME.md) | _any_
qdisc | Exposes [queuing discipline](https://en.wikipedia.org/wiki/Network_scheduler#Linux_kernel) statistics | Linux
runit | Exposes service status from [runit](http://smarden.org/runit/). | _any_
supervisord | Exposes service status from [supervisord](http://supervisord.org/). | _any_
systemd | Exposes service and system status from [systemd](http://www.freedesktop.org/wiki/Software/systemd/). | Linux
tcpstat | Exposes TCP connection status information from `/proc/net/tcp` and `/proc/net/tcp6`. (Warning: the current version has potential performance issues in high load situations.) | Linux

### Command line

```bash
usage: node_exporter [<flags>]

Flags:
  -h, --help                    Show context-sensitive help (also try --help-long and --help-man).
      --collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$"  
                                Regexp of devices to ignore for diskstats.
      --collector.filesystem.ignored-mount-points="^/(sys|proc|dev)($|/)"  
                                Regexp of mount points to ignore for filesystem collector.
      --collector.filesystem.ignored-fs-types="^(sys|proc|auto)fs$"  
                                Regexp of filesystem types to ignore for filesystem collector.
      --collector.megacli.command="megacli"  
                                Command to run megacli.
      --collector.netdev.ignored-devices="^$"  
                                Regexp of net devices to ignore for netdev collector.
      --collector.ntp.server="127.0.0.1"  
                                NTP server to use for ntp collector
      --collector.ntp.protocol-version=4  
                                NTP protocol version
      --collector.ntp.server-is-local  
                                Certify that collector.ntp.server address is the same local host as this collector.
      --collector.ntp.ip-ttl=1  IP TTL to use while sending NTP query
      --collector.ntp.max-distance=3.46608s  
                                Max accumulated distance to the root
      --collector.ntp.local-offset-tolerance=1ms  
                                Offset between local clock and local ntpd time to tolerate
      --path.procfs="/proc"     procfs mountpoint.
      --path.sysfs="/sys"       sysfs mountpoint.
      --collector.qdisc.fixtures=""  
                                test fixtures to use for qdisc collector end-to-end testing
      --collector.runit.servicedir="/etc/service"  
                                Path to runit service directory.
      --collector.supervisord.url="http://localhost:9001/RPC2"  
                                XML RPC endpoint.
      --collector.systemd.unit-whitelist=".+"  
                                Regexp of systemd units to whitelist. Units must both match whitelist and not match blacklist to be included.
      --collector.systemd.unit-blacklist=".+\\.scope"  
                                Regexp of systemd units to blacklist. Units must both match whitelist and not match blacklist to be included.
      --collector.systemd.private  
                                Establish a private, direct connection to systemd without dbus.
      --collector.textfile.directory=""  
                                Directory to read text files with metrics from.
      --collector.wifi.fixtures=""  
                                test fixtures to use for wifi collector metrics
      --collector.arp           Enable the arp collector (default: enabled).
      --collector.bcache        Enable the bcache collector (default: enabled).
      --collector.bonding       Enable the bonding collector (default: disabled).
      --collector.buddyinfo     Enable the buddyinfo collector (default: disabled).
      --collector.conntrack     Enable the conntrack collector (default: enabled).
      --collector.cpu           Enable the cpu collector (default: enabled).
      --collector.diskstats     Enable the diskstats collector (default: enabled).
      --collector.drbd          Enable the drbd collector (default: disabled).
      --collector.edac          Enable the edac collector (default: enabled).
      --collector.entropy       Enable the entropy collector (default: enabled).
      --collector.filefd        Enable the filefd collector (default: enabled).
      --collector.filesystem    Enable the filesystem collector (default: enabled).
      --collector.gmond         Enable the gmond collector (default: disabled).
      --collector.hwmon         Enable the hwmon collector (default: enabled).
      --collector.infiniband    Enable the infiniband collector (default: enabled).
      --collector.interrupts    Enable the interrupts collector (default: disabled).
      --collector.ipvs          Enable the ipvs collector (default: enabled).
      --collector.ksmd          Enable the ksmd collector (default: disabled).
      --collector.loadavg       Enable the loadavg collector (default: enabled).
      --collector.logind        Enable the logind collector (default: disabled).
      --collector.mdadm         Enable the mdadm collector (default: enabled).
      --collector.megacli       Enable the megacli collector (default: disabled).
      --collector.meminfo       Enable the meminfo collector (default: enabled).
      --collector.meminfo_numa  Enable the meminfo_numa collector (default: disabled).
      --collector.mountstats    Enable the mountstats collector (default: disabled).
      --collector.netdev        Enable the netdev collector (default: enabled).
      --collector.netstat       Enable the netstat collector (default: enabled).
      --collector.nfs           Enable the nfs collector (default: disabled).
      --collector.ntp           Enable the ntp collector (default: disabled).
      --collector.qdisc         Enable the qdisc collector (default: disabled).
      --collector.runit         Enable the runit collector (default: disabled).
      --collector.sockstat      Enable the sockstat collector (default: enabled).
      --collector.stat          Enable the stat collector (default: enabled).
      --collector.supervisord   Enable the supervisord collector (default: disabled).
      --collector.systemd       Enable the systemd collector (default: disabled).
      --collector.tcpstat       Enable the tcpstat collector (default: disabled).
      --collector.textfile      Enable the textfile collector (default: enabled).
      --collector.time          Enable the time collector (default: enabled).
      --collector.uname         Enable the uname collector (default: enabled).
      --collector.vmstat        Enable the vmstat collector (default: enabled).
      --collector.wifi          Enable the wifi collector (default: enabled).
      --collector.xfs           Enable the xfs collector (default: enabled).
      --collector.zfs           Enable the zfs collector (default: enabled).
      --collector.timex         Enable the timex collector (default: enabled).
      --web.listen-address=":9100"  
                                Address on which to expose metrics and web interface.
      --web.telemetry-path="/metrics"  
                                Path under which to expose metrics.
      --log.level="info"        Only log messages with the given severity or above. Valid levels: [debug, info, warn, error, fatal]
      --log.format="logger:stderr"  
                                Set the log target and format. Example: "logger:syslog?appname=bob&local=7" or "logger:stdout?json=true"
      --version                 Show application version.
```