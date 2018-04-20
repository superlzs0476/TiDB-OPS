---
title: 运用工具收集 TiDB 运行问题
date: 2017-06-01 21:40:52
updated: 2017-06-09 21:40:52
categories:
  - Tools
tags:
  - TiDB
  - Air
---
# 运用工具收集 TiDB 运行问题

## 通过监控看现象

- 未完待续

## 通过工具收集信息

### core 文件

- 使用 gdb 命令获取 core 文件信息
  -`gdb -c core-file-name bin/tikv-server`; core-file-name  这个替换成 core 文件名
    - 执行 `bt` ，输出文本发送至 TiDB 官方支持，或在相关项目创建 issue

### 获取 Go Profile 程度堆栈

- 通过 Go lang 语言自身特性 pporf 获取 Binary 瞬间运行状态
  - https://golang.org/pkg/net/http/pprof/

#### 第一种 Go tool 交互方式查看 pprof

安装 Go lang 环境，可以使用 yum 或者官方下载二进制文件方式；建议安装版本大于 **1.9.0**

- 通过 Yum 方式安装 Go
  - `yum install go -y`
  - `go version`

- 通过二进制方式安装
  - https://golang.org/dl/
  - 下载后解压到相关目录
  - 添加环境变量 [Getting Started](https://golang.org/doc/install)
    - `export PATH=$PATH:/usr/local/go/bin`
    - `export GOROOT=$HOME/go1.X`
    - `export PATH=$PATH:$GOROOT/bin`
  - `go version`

- 使用 Go tool 获取即时信息并分析
  - 获取内存信息
    - `go tool pprof http://127.0.0.1:10080/debug/pprof/heap`
  - 获取 profile
    - `go tool pprof 'http://127.0.0.1:10080/debug/pprof/profile'`
  - 交互命令执行 help 查看

#### 第二种 静态获取 pporf

- 静态保存相关信息
  - 获取 goroutine 计算资源信息
    - `curl http://127.0.0.1:10080/debug/pprof/goroutine?debug=1 > save.tidb.goroutine.debug1`
    - `curl http://127.0.0.1:10080/debug/pprof/goroutine?debug=2 > save.tidb.goroutine.debug2`
  - 获取 heap 内存信息
    - `curl http://127.0.0.1:10080/debug/pprof/heap?debug=1 > save.tidb.heap.debug1`
    - `curl http://127.0.0.1:10080/debug/pprof/heap?debug=2 > save.tidb.heap.debug2`

### 火焰图

- 使用时可阅读下 [Debug 利器之火焰图](https://pingcap.com/blog-cn/flame-graph/) 文档

- 下载并解压 https://github.com/brendangregg/FlameGraph/archive/master.zip
- mv FlameGraph-master FlameGraph
- 把下列脚本放到一个 sh (perf.sh)，然后执行 sh perf.sh $tikv-pid

  ```bash
  #!/bin/bash

  perf record -F 99 -p $1 -g -- sleep 60
  perf script > out.perf
  ./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
  ./FlameGraph/flamegraph.pl out.folded > kernel.svg
  ```

### 使用 pstack 查看 Binary 堆栈

观察 Binary 运行栈，pstack 查看 Binary 堆栈过程中，会中断 Binary 正常运行

- `pstack -p pid`
