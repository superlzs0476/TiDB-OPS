---
title: kill tcp connect
date: 2018-05-05 21:40:52
updated: 2018-05-06 21:40:52
categories:
  - Case
tags:
  - tcpkill
---
# kill tcp connect

- tcpkill
  - 用于切换该节点上的网络链接与流量
- 类似软件
  - iptables/ipset,
  - blackhole routing

## tcpkill

tcpkill 命令属于 Dsniff 嗅探工具包，Dsniff 本身是一个高级的网络嗅探器，Dsniff 可以将制造的数据包注入到网络，一般 Linux 系统中可以找到对应的 dsniff 软件包进行安装。

tcpkill 的工作原理是利用 libpcap 库监控符合过滤条件的 TCP 连接并等待，当该 TCP 连接上有数据传输时就会被 tcpkill 感知到，不过作为应用程序的 tcpkill 当然不能直接关闭 TCP 连接，此时就会构造一条 RST 报文发回去直接导致 TCP 连接异常关闭。

### install

`yum install -y dsniff`

### tcpkill command

tcpkill 与 tcpdump 使用语句格式类似

`tcpkill [-i interface] [-1...9] expression`

  - -i 指定网卡设备
  - [-1...9] 指定 "kill" 的强制等级，越高越强，默认为 3
  - expression 匹配需要 kill 的 tcp 连接通配表达式，语法与 tcpdump 过滤表达式参数一致，具体如何使用可以参考 tcpdump 的[帮助信息](http://www.tcpdump.org/tcpdump_man.html)

### Case-by-case

TiKV 双链接场景使用，TiKV 执行 `store weight 123 0.1 0.1` 后，磁盘空间继续增长，观察监控未看到调度。观察 PD 日志发现大量关于 scheduler Timeout 的日志。

- 在 TiKV 机器执行以下命令，观察 TiKV 出现双链接
  - ss -ntp | grep -E ':2379'

  ```bash
  ESTAB      0      0      10.13.135.78:17026              10.13.13.13:2379                users:(("tikv-server",pid=53945,fd=25911))
  ESTAB      0      0      10.13.135.78:40926              10.13.13.15:2379                users:(("tikv-server",pid=53945,fd=14148))
  ```

- 在相应 TiKV 机器执行以下命令，切断 TiKV 与 PD follower 之间网络链接 (PD leader 切换后，TiKV 未主动断开与之前 leader 的链接，导致出现双链接场景)
  - /usr/sbin/tcpkill -9 host 10.13.13.15 and port 2379 &>/dev/null
    - 以上命令会切断该主机到 10.13.13.45:2379 的 TCP 链接
  - tcpkill 命令需要使用 ctrl + c 中断，否则前台持续运行；如果需要批量远程执行，可以添加 `&>/dev/null`

### tcpkill原理分析

- 以下内容摘取至 [how-tcpkill-works.md](https://github.com/stanzgy/wiki/edit/master/network/how-tcpkill-works.md)

在使用tcpkill时，会发现一件奇怪的事情，运行tcpkill命令后并不会马上中断匹配的tcp连接，只有当该连接有新的tcp包发送接收时，tcpkill才会“kill”这个tcp连接。这个奇怪的现象燃起了我们的好奇心，于是探索一下tcpkill到底是如何工作的。

下面以Linux下的nc命令为例。

我们在两个主机hostA与hostB间通过nc命令建立一个tcp连接：

hostA在本地tcp 5555端口监听

    hostA$ nc -l -p 5555

hostB通过本地6666端口连接hostA的5555端口

    hostB$ nc hostA 5555 -p 6666

此时在hostA上已经可以观察到一条与hostB的ESTABLISHED连接

    hostA$ netstat -anp|grep 5555
    tcp        0      0 hostA:5555    hostB:6666     ESTABLISHED 19638/nc

在hostA上通过tcpdump也可以观察到3次握手已经完成

    hostA# tcpdump -i eth1 port 5555
    IP hostB.6666 > hostA.5555: Flags [S], seq 750827752, ...
    IP hostA.5555 > hostB.6666: Flags [S.], seq 1191909671, ack 750827753, ...
    IP hostB.6666 > hostA.5555: Flags [.], ack 1, win 115, ...

如果此时运行tcpkill命令尝试“kill”这个tcp连接

    hostA# tcpkill -1 -i eth1 port 5555
    tcpkill: listening on eth1 [port 5555]

会发现hostA与hostB上的nc命令并没有受到任何影响而退出，hostA上观察到该tcp连接还是ESTABLISHED状态，tcpdump与tcpkill也没有任何新的输出。

    hostA$ netstat -anp|grep 5555
    tcp        0      0 hostA:5555    hostB:6666     ESTABLISHED 19638/nc

运行tcpkill命令后，建立好的tcp连接并没有受到任何影响。

如果我们此时在hostB的nc上输入任意字符发送，则会发现这时tcp连接中断，nc发送失败退出。

    hostB$ nc hostA 5555 -p 6666
    a<CR>
    (exit)
    hostB$

hostA上的nc监听进程也因为连接中断而退出

    hostA$ nc -l -p 5555
    (exit)
    hostA$

netstat已经观察不到这个tcp连接，而tcpdump此时则捕获了一个新tcp rst包：

    hostA# tcpdump -i eth1 port 5555
    IP hostB.6666 > hostA.5555: Flags [S], seq 750827752, ...
    IP hostA.5555 > hostB.6666: Flags [S.], seq 1191909671, ack 750827753, ...
    IP hostB.6666 > hostA.5555: Flags [.], ack 1, win 115, ...
    IP hostA.5555 > hostB.6666: Flags [R], seq 1191909672, ...

此时tcpkill的输出

    hostA# tcpkill -1 -i eth1 port 5555
    tcpkill: listening on eth1 [port 5555]
    hostB:6666 > hostA:5555: R 1191909672:1191909672(0) win 0
    hostA:5555 > hostB:6666: R 750827755:750827755(0) win 0

相信看到这里，已经可以明白tcpkill的工作原理，实际上就是通过双向fake tcp rst包重置目标连接双方的网络连接，和某墙的原理一样。

而之所以tcpkill不会马上中断目标tcp连接，是因为伪造tcp rst包时，需要填入正确的sequence number，这需要通过拦截双方的tcp通信才能实时得到。所以运行tcpkill后，只有目标连接有新tcp包发送/接受才会导致tcp连接中断。

最后分析一下tcpkill第二个参数的具体作用。manpage里的说明比较模糊，只能看出和receive window有关：

> -1...9 Specify the degree of brute force to use in killing a connection. Fast connections may require a higher number in order to land a RST in the moving receive window. Default is 3.

直接看源代码(只有100多行)

  ```C
    ...

    int
    main(int argc, char *argv[])
    {
        ...

        /* 通过libpcap抓取所有符合条件的包，回调函数为tcp_kill_cb */
        pcap_loop(pd, -1, tcp_kill_cb, (u_char *)&sock);

        ...
    }

    static void
    tcp_kill_cb(u_char *user, const struct pcap_pkthdr *pcap, const u_char *pkt)
    {
        ...

        /* 只处理tcp包 */
        ip = (struct libnet_ip_hdr *)pkt;
        if (ip->ip_p != IPPROTO_TCP)
            return;

        /* 不处理tcp syn/fin/rst包 */
        tcp = (struct libnet_tcp_hdr *)(pkt + (ip->ip_hl << 2));
        if (tcp->th_flags & (TH_SYN|TH_FIN|TH_RST))
            return;

        /* 伪造ip包 */
        libnet_build_ip(TCP_H, 0, 0, 0, 64, IPPROTO_TCP,
                ip->ip_dst.s_addr, ip->ip_src.s_addr,
                NULL, 0, buf);

        /* 伪造tcp rst包 */
        libnet_build_tcp(ntohs(tcp->th_dport), ntohs(tcp->th_sport),
                0, 0, TH_RST, 0, 0, NULL, 0, buf + IP_H);

        /* fake tcp rst包的sequence number即为抓到的包的ack number */
        seq = ntohl(tcp->th_ack);

        ...

        /* 这里Opt_severity即为tcpkill的第二个参数 */
        win = ntohs(tcp->th_win);
        for (i = 0; i < Opt_severity; i++) {
            ip->ip_id = libnet_get_prand(PRu16);
            seq += (i * win);
            tcp->th_seq = htonl(seq);

            libnet_do_checksum(buf, IPPROTO_TCP, TCP_H);

            /* 发送伪造的tcp rst包 */
            if (libnet_write_ip(*sock, buf, sizeof(buf)) < 0)
                warn("write_ip");

            fprintf(stderr, "%s R %lu:%lu(0) win 0\n", ctext, seq, seq);
        }
    }
  ```

从上面可以看出，tcpkill的第二个参数，实际上就是沿tcp连接窗口滑动而发送的tcp rst包个数。将这个参数设置较大主要是应对高速tcp连接的情况。

参数的大小从中断tcp连接的原理上没有区别，只是发送rst包数量的差异，通常情况下使用默认值3已经完全没有问题了。所以使用tcpkill时请不要像网络上某些中文教程中一样不适当的使用参数 `-9` 。
