---
title: 磁盘压力测试与场景优化
date: 2018-05-05 21:40:52
updated: 2018-05-06 21:40:52
categories:
  - Deploy
tags:
  - Benchmark
---
# 磁盘压力测试与场景优化

## FIO

其实按照我们实际情况，应该是两个 job 同时跑，一个是 write，另一个就是 randread

fio -ioengine=psync -bs=128k -fdatasync=1  -thread -rw=write -size=10G -filename=PingCAP-test -name="PingCAP" -runtime=60
fio -ioengine=psync -bs=128k -fdatasync=1  -thread -rw=randread -size=10G -filename=PingCAP-test -name="PingCAP" -runtime=60

fio -ioengine=psync -bs=256k -fdatasync=1  -thread -rw=write -size=10G -filename=PingCAP-test -name="PingCAP" -runtime=60
fio -ioengine=psync -bs=256k -fdatasync=1  -thread -rw=randread -size=10G -filename=PingCAP-test -name="PingCAP" -runtime=60
 
 lsblk -D 可以检查盘是不是做了trim   Disc-max那栏好像是0B 就是trim 过了

 
fio测试过程中，IO的引擎ioengine=psync/libaio理解，原来不是很明白
通常#mysql5.5后可以使用native aio(libaio)，但很多操作还是通过fake aio（类似psync）,同步io的方式进行


 libaio Linux native asynchronous I/O. This ioengine defines engine specific options.#linux下原生态的异常i/o，异步I/O的能够提高应用程序的性能,参考：http://www.ibm.com/developerworks/cn/linux/l-async/index.html


 psync  Basic pread(2) or pwrite(2) I/O.通过man pread得知
 pread, pwrite - read from or write to a file descriptor at a given offset #pread, pwrite - 在文件描述符给定的位置偏移上读取或写入数据
   描述
    pread() 从文件 fd 指定的偏移 offset (相对文件开头) 上读取 count 个字节到 buf 开始位置。文件当前位置偏移保持不变。
    pwrite() 把缓存区 buf 开头的 count 个字节写入文件描述符 fd offset 偏移位置上。文件偏移没有改变。
    fd 引用的文件必须是可定位的。

    1)pread与read的区别是，pread每次读取都要指定offset，这样防止了当有多个线程读取文件时，read之间的相互干扰，因为read时，对于在同一fd上的读写是共用的，那么会共用一个记录offset的结构，应该是File吧。
    2)由于lseek和read 调用之间，内核可能会临时挂起进程，所以对同步问题造成了问题，调用pread相当于顺序调用了lseek 和 read，这两个操作相当于一个捆绑的原子操作