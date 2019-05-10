---
layout: post
title: 高性能磁盘 I/O 编程
tags:
  - performance
  - linux
  - io
  - filesystem
  - disk
date: 2019-05-08
---

在网络编程中，我们通常使用 nonblocking I/O 配合 select()/poll()/epoll() 等系统调用实现高性能的服务。不过这套方法在面对磁盘 I/O 的时候就行不通了。由于普通的文件在 Linux 系统中总是可读又可写的，设置 O_NONBLOCK 标志位对于它们就失去了意义。此时，可以选择的就剩下：

1. 利用多个线程处理磁盘请求，在操作系统调度 I/O 请求时保持系统的响应状态
2. 使用异步 I/O，在等待磁盘完成的同时继续处理后续请求

多线程，甚至多进程，有时候会是不错的武器，但当请求数量过于庞大时，可能不是一个特别好的选择。因此，大多数情况下，我们求助于异步 I/O 机制解决磁盘操作与系统响应的矛盾。

## Linux aio

glibc 从2.1开始提供一套 aio_xxxx 的方法，允许调用方以异步方式处理磁盘 I/O 请求。但是，这套方法却是在用户态以线程模拟异步接口，当用户通过 aio_read/aio_write 提交读写请求时，glibc 将请求缓存至队列，而后台线程则从队列获取任务执行用户请求。像刚刚说的那样，这套线程模拟方法并不是一个好的解决方案，无法支持大规模请求场景下的平滑扩展。

其实，Linux 内核从2.5版本开始提供专用的[异步 I/O 接口](http://lse.sourceforge.net/io/aio.html)，实现真正的异步操作，又称 `KAIO`。这套接口还没有 glibc 的包装，但可以[通过 syscall() 或者 libaio 库调用](http://www.fsl.cs.sunysb.edu/~vass/linux-aio.txt)。

```cpp
#include <libaio.h>     // For aio wrappers
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{
    auto fname = "/tmp/xzseSA87a.tmp";
    int fd = open(fname, O_CREAT|O_EXCL|O_RDWR|O_DIRECT, 0600);
    if (fd < 0) {
        perror("open error");
        return -1;
    }
    unlink(fname);

    io_context_t ioctx = {};
    auto ret = io_setup(128, &ioctx);
    if (ret < 0) {
        perror("io_setup error");
        return -1;
    }

    auto buf = aligned_alloc(4096, 4096);
    iocb cb;
    io_prep_pwrite(&cb, fd, buf, 4096, 0);

    iocb *cbs[] = { &cb };
    cbs[0] = &cb;
    io_submit(ioctx, 1, cbs);
    if (ret != 1) {
        if (ret < 0)
            perror("io_submit error");
        else
            fprintf(stderr, "could not sumbit IOs\n");
        return  -1;
    }

    io_event event;
    auto n = io_getevents(ioctx, 1, 1, &event, nullptr);
    printf("got %d event(s)\n", n);

    io_destroy(ioctx);
    if (ret < 0) {
        perror("io_destroy error");
        return -1;
    }

    close(fd);
    return 0;
}
```
KAIO 有几点注意事项：

1. 使用 `O_DIRECT` 打开，提交的 I/O 请求参数，buffer 的地址和 length 的值，满足 `O_DIRECT` 的对齐要求
2. 操作对象是 [raw device](https://en.wikipedia.org/wiki/Raw_device)，或者支持 aio 的文件系统，像 `ext2`, `ext3`, `ext4`, `jfs`, `xfs`

然而，有了这些并不表示万事大吉。aio 在内核的支持相当有限，在很多情况下 `io_submit()` 会 block 调用线程。

## 潜在的性能瓶颈

如果不幸没有使用 `O_DIRECT` 打开文件，在 `ext2`, `ext3`, `ext4`, `jfs`, `xfs` 等文件系统中，`API` 并不返回错误，而是自动 fallback 到同步模式，也就是说，`io_submit()` 会等待操作完成之后才返回。（！！！）

按照 API 要求用 `O_DIRECT` 之后，在同时提交大量请求到内核，超过目标设备最大请求数 [nr_requests](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt) 时仍然会发生阻塞。

    nr_requests (RW)
    ----------------
    This controls how many requests may be allocated in the block layer for
    read or write requests. Note that the total allocated number may be twice
    this amount, since it applies only to reads or writes (not the accumulated
    sum).

    ......

或者数据块大小超过设备允许的最大请求大小 [max_sectors_kb](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt)，内核必须把数据切成符合要求的多个小块，此时也可能阻塞调用者。

    max_sectors_kb (RW)
    -------------------
    This is the maximum number of kilobytes that the block layer will allow
    for a filesystem request. Must be smaller than or equal to the maximum
    size allowed by the hardware.

CPU 也是个潜在的故障点。如果单个线程提交请求过于频繁，CPU 频率就成为瓶颈，这时候需要引入多线程分散工作量。不过，引入多个线程也要多加小心，多个线程共同操作同一个 `io_context_t` 会产生竞争，也可能引入阻塞，建议用一定的分配算法避免这种竞争，提高吞吐量。

阻塞还可能发生在文件系统，例如需要更新 `metadata` 才能完成 I/O 请求之类的[情况](https://github.com/littledan/linux-aio#performance-considerations)。众多的文件系统中，XFS 对于异步 I/O 的支持相对来说更完备，不过使用者仍然要仔细调优 I/O 请求以避免之前的设备繁忙、CPU 瓶颈等等问题。

另外，在提交 I/O 请求时，指定的文件 offset 值到磁盘物理位置的转换操作必须在提交异步 I/O 请求之前完成。追求极致性能的话，最好的方法是跳过文件系统，直接操作 [raw device](https://www.tldp.org/HOWTO/SCSI-2.4-HOWTO/rawdev.html) 向硬件发送 I/O 请求，这也是 KAIO 支持的场景（记得使用 `O_DIRECT` 打开 raw device）。

## 硬件

软件万事俱备之后，硬件方面也有一些可以优化的点。依据实际 I/O 请求的大小，适当调整 `stripe size`，让 RAID 卡更高效工作完成磁盘操作。此外，还有一些其他选项，如 `write policy`, `cache policy` 等等。在实践过程中，`write back` 优于 `write through` （牺牲一定程度的数据安全），以及视读取请求的具体分配，可以调配 `cache policy`。

最好，这篇小小的总结不是终极银弹，只是汇总各种可能的陷阱以及 dirty evil details。这里提到的各项参数也没有普适的最优解，收集实际测试数据，分析系统瓶颈，调优代码热点路径和系统设置，衡量各方案利弊，才能得到符合业务需求的最佳平衡点。

## 相关链接:
- [Kernel Asynchronous I/O (AIO) Support for Linux](http://lse.sourceforge.net/io/aio.html)
- [How to use the Linux AIO feature > Performance Consideration](https://github.com/littledan/linux-aio#performance-considerations)
- [Qualifying Filesystems for Seastar and ScyllaDB](https://www.scylladb.com/2016/02/09/qualifying-filesystems/)
- [Toward non-blocking asynchronous I/O](https://lwn.net/Articles/724198/)
- [xfs mailing list: sleeps and waits during io_submit](https://marc.info/?l=linux-xfs&m=144961792322329&w=2)
