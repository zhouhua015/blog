---
layout: post
title: NUMA 与 Linux
categories:
  - performance
  - linux
date: 2019-01-08
---

## 硬件

[NUMA](https://www.wikiwand.com/en/Non-uniform_memory_access) 从硬件架构上来讲，是一种为了支持可扩展的内存带宽的硬件系统设计。NUMA 是对称多处理架构(Symetric Multiprocessing)的延伸，每个 NUMA 节点都可以视作 SMP 的子集，包含 CPU\内存以及 I/O 总线。当前节点的 CPU 访问本节点的内存，速度更快，带宽更高，而跨节点访问内存时，性能随节点间的距离降低。跨节点的内存访问由硬件提供支持，通过节点间总线更新本节点 CPU cache，称为 ccNUMA (Cache Coherent NUMA)。

![ccNUMA](https://upload.wikimedia.org/wikipedia/commons/9/95/Hwloc.png?1546939997615)


## 操作系统软件

软件层面上，Linux 把 NUMA 的硬件节点抽象为 NUMA node，同样的，node 本地内存访问要比跨 node 访问快速高效的多。

X86 架构下，如果一个 NUMA 节点只有 CPU 没有内存，那么 Linux 会将此节点的 CPU 归到另外一个有内存的 node 上，而且在系统中不再显示没有内存的节点，所谓“隐藏”此节点。例如，NUMA 节点1上有一块8核双线程的 CPU 和32G 内存，而节点2上只有一块8核双线程的 CPU 而没有内存，在 Linux 操作系统看来，则只有一个 NUMA node #0节点，它有32个逻辑 CPU 和 32G 内存。因此，不要假设 Linux 下的同一个 NUMA node 上的所有逻辑 CPU 可以有同样的内存性能。

Linux 内核对于每个 NUMA node 会单独构建内存管理子系统，独立的可用内存页列表，在用的内存页列表等等。除此之外，内核还为每一类内存 zone（DMA, DMA32, NORMAL, HIGH_MEMORY, MOVABLE）构造可分配的 zone/node 有序列表，若当前 zone 无法满足内存需求时（称为"overflow"或者"fallback"），内核就会从列表中选择下一个 zone 分配内存。

每个 NUMA node 都可能包含多个 zone，当"fallback"分配内存时，内核就要在当前节点的其他 zone 分配还是其他节点的此类 zone 分配二者中选择。内核目前使用的是默认节点策略，也就是说，即使当前节点的某一内存 zone 无法满足分配需求，内核也会优先从本节点其他的 zone 分配内存。只有当本节点内存无法满足需求时，才会跨到最近的节点分配内存。

内核这种优先分配本地内存的策略，使得后续的内存访问更可能获得优异的性能，只要访问内存的线程不被调度到其他 NUMA node 上。然而，虽然调度器能够理解 NUMA 结构，但在调度时并没有考虑线程曾经的 NUMA 运行节点，这种将线程调度到另外的 NUMA node 的可能性仍然存在。[taskset](https://linux.die.net/man/1/taskset) 和 [numactl](https://linux.die.net/man/8/numactl) 两个命令行工具，以及 [sched_setaffinity](https://linux.die.net/man/2/sched_setaffinity) 函数可以通过设置 CPU 亲和性和/或绑定 NUMA 节点实现任务调度的 NUMA 本地化。


---
Links:
* [Kernel Documatation](https://www.kernel.org/doc/Documentation/vm/numa)
