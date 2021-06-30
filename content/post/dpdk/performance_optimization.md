---
title: "DPDK的性能优化"
date: 2018-08-22T20:04:01+08:00
draft: false
tags: ["dpdk"]
categories: ["dpdk"]
---

## 轮询

DPDK采用轮询方式进行网卡报文收发，避免了常规报文处理方法因采用中断方式造成的上下文切换以及中断处理带来的额外开销，极大提升了收发性能。

## 大页内存

在系统内存管理中由虚拟内存和物理内存，内存管理端元MMU完成虚拟内存到物理内存的地址转换。内存管理单将进行转换的信息保存在页表中，为了提升页表查询速度，使用缓存保存查询结果，这块缓存就叫TLB，它保存了虚拟内存到物理内存的映射关系。如果在TLB中未发现有效的映射，也就是TLB Miss，需要再从页表中查找。页表的查找过程对性能影响很大，因此要减少页表的查找，即减少TLB Miss的发生。

调大内存分页大小，随之映射表就会减少，TLB存储的映射表比率就会增加，从而提升TLB的命中率，减少TLB Miss的发生。DPDK利用大页技术，所有的内存从HupePage里分配，实现对内存池的管理。

## NUMA

NUMA和SMP是两种典型的处理器对内存的访问架构。在多核处理器时代，NUMA架构广泛应用，它为每个处理器提供分离的内存和内存控制器，以避免SMP架构中多个处理器同时访问同一存储器产生的性能损耗。

在NUMA架构中，CPU访问本地内存要比访问远端内存更快，因为在访问远端内存时，需要跨越QPI总线，这个访问延迟会大幅度降低系统性能。DPDK提供一套指定在NUMA节点上创建memzone、ring，rte_malloc以及mempool的API，来避免远端内存这类问题。

由NUMA架构，在进行网络报文收发时，网卡所在的socket_id与绑定的cpu core以及创建的memppol其所在的socket_id应一致，使其性能达到最高。

## mempool

对于需要频繁申请释放的内存空间采用内存池mempool的方式预先动态分配一整块内存区域，统一进行管理，从而省去频繁的动态分配和释放过程，既提高了性能，同时也减少了内存碎片的产生。

在DPDK中，还考虑了地址对齐，以及CPU core、local cache等因素，同时还针对内存多通道将对象分配在不同的内存通道上，保证了在系统极端情况下需要大量内存访问时，尽可能地将内存访问任务均匀平滑，最大地提升性能。


## 副本复制

在多线程需要频繁访问共享变量或者跨NUMA进行内存读取时，可以采取复制一份该变量的副本到本地，以减少访问延迟。

## CPU亲和性

多个进程或线程在多核处理器的cpu核上不断交替执行，每次核的切换，读需要将处理器的状态寄存器保存在堆栈中，以恢复当前进程的状态信息，从而带来额外的开销。同时，核的切换也会导致缓存的命中率。

而利用CPU的亲和性，将某个进程或线程绑定到特定的cpu核上执行，而不被迁移到其他核上，从而保证程序性能。


## prefetch

比如在对大数组非连续位置的访问时，因cache miss导致cpu开销严重，使用prefetch指令可以将指定地址的内存预取到cache，从而提升性能。

简单来说，prefetch将原本串行工作的计算过程和读内存过程并行化了。

## lockless队列

DPDK实现了支持单/多生产者－单/多消费者的lockless队列librte_ring，其实现利用了CAS原子操作。同时支持bulk/burst出入队。

## 地址对齐

当变量的地址被它本身的长度整除时，存取该变量最高校。通常对于频繁访问的结构体，要将其按缓存行大小对齐，减少Cache Miss、False Sharing。

```
(uing64_t)&val % sizeof(var) == 0

struct {
    int elem;
    int elem2;
    ...
} __attribute__((aligned(64)));
```

## 循环展开

直接看代码，librte_ring的实现。

```
    static __rte_always_inline void
__rte_ring_enqueue_elems_64(struct rte_ring *r, uint32_t prod_head,
        const void *obj_table, uint32_t n)
{
    unsigned int i;
    const uint32_t size = r->size;
    uint32_t idx = prod_head & r->mask;
    uint64_t *ring = (uint64_t *)&r[1];
    const unaligned_uint64_t *obj = (const unaligned_uint64_t *)obj_table;
    if (likely(idx + n < size)) {
        for (i = 0; i < (n & ~0x3); i += 4, idx += 4) {
            ring[idx] = obj[i];
            ring[idx + 1] = obj[i + 1];
            ring[idx + 2] = obj[i + 2];
            ring[idx + 3] = obj[i + 3];
        }
        switch (n & 0x3) {
            case 3:
                ring[idx++] = obj[i++]; /* fallthrough */
            case 2:
                ring[idx++] = obj[i++]; /* fallthrough */
            case 1:
                ring[idx++] = obj[i++];
        }
    } else {
        for (i = 0; idx < size; i++, idx++)
            ring[idx] = obj[i];
        /* Start at the beginning */
        for (idx = 0; i < n; i++, idx++)
            ring[idx] = obj[i];
    }
}
```
