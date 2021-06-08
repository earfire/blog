---
title: "内存屏障"
date: 2018-09-16T22:11:09+08:00
draft: false
tags: ["os"]
categories: ["os"]
---

## 前言

在看DPDK的源码librte_ring时看到了在实现lock-free队列时使用到了内存屏障，对缓存进行更深一步的理解。

```
static __rte_always_inline unsigned int
__rte_ring_move_prod_head(struct rte_ring *r, int is_sp,
		unsigned int n, enum rte_ring_queue_behavior behavior,
		uint32_t *old_head, uint32_t *new_head,
		uint32_t *free_entries)
{
    const uint32_t capacity = r->capacity;
    unsigned int max = n;
    int success;
    do {
        /* add rmb barrier to avoid load/load reorder in weak
         * memory model. It is noop on x86
         */
        rte_smp_rmb();

        ...
    } while (unlikely(success == 0));
    return n;
}

/* dpdk/lib/librte_eal/common/include/arch/x86/rte_atomic.h */
#define rte_smp_rmb() rte_compiler_barrier()
/* dpdk/lib/librte_eal/common/include/generic/rte_atomic.h */
#define rte_comliler_barrier() do {     \
    asm volatile ("" : : : "memory");   \ //""::: 表示这是个空指令。
} while (0)

/* dpdk/lib/librte_eal/common/include/arch/ppc_64/rte_atomic.h */
#define rte_rmb() asm volatile("lwsync" : : : "memory")
#define rte_smp_rmb() rte_rmb()

/* dpdk/lib/librte_eal/common/include/arch/arm/rte_atomic_64.h */
#define dmb(opt) asm volatile("dmb " #opt : : : "memory")
#define rte_smp_rmb() dmb(ishld)
```



##  缓存一致性协议

在谈内存屏障之前，先谈一下缓存一致性。缓存一致性协议用于管理多个CPU cache之间数据的一致性。常见的协议有MSI、MESI、MOSI等，这里介绍下MESI:

MESI协议将Cache的状态分为modify、exclusive、shared、invalid分别是修改、独占、共享、失效。

| 状态      	| 描述		|   
| ------------- |:---------------:|
| M(modify)	| 当前CPU刚修改完数据的状态，只有当前CPU拥有最新数据，其他CPU拥有失效数据，而且和主存数据不一致，可以直接写数据	|
| E(exclusive)	| 只有当前CPU中拥有数据，其他CPU中没有改数据，当前CPU的数据和主存的数据是一致的, 可以直接写数据	|
| S(shared)	| 当前CPU和其他CPU中都有共同的数据，并且和主存中的数据一致，不可以直接写数据，写时需要需要先通知其他CPU cache |
| I(invalid)	| 当前CPU中的数据失效，数据应该从主存中获取，其他CPU中可能有数据也可能无数据；当前CPU中的数据和主存中的数据被认为不一致 |


## 内存模型

有了缓存一致性协议，为什么还需要内存屏障这些操作? 因为不是所有的内存模型都是强一致性的，比如在上述dpdk的librte_ring源码中，对于x86就是一个空指令。

关于内存模型，更多参考
[weak-vs-strong-memory-models](https://preshing.com/20120930/weak-vs-strong-memory-models/ "weak-vs-strong-memory-models")

## 内存屏障


编译器和CPU可以保证输出结果一样的情况下对指令重排序，使性能得到优化，但有时会导致可见性问题。对于指令重排序和可见性，这里不过多作介绍。

这种情况下，插入一个内存屏障，相当于告诉CPU和编译器先于这个命令的必须先执行，后于这个命令的必须后执行，从而解决可见性问题。

内存屏障可以强制更新一次不同CPU的缓存。例如，一个写屏障会把这个屏障前写入的数据刷新到缓存，这样任何试图读取该数据的线程将得到最新值，而不用考虑到底是被哪个cpu核心或者哪颗CPU执行的。

内存屏障的执行效果：
- 对于read memory barrier指令，它只是约束执行CPU上的load操作的顺序，具体的效果就是CPU一定是完成read memory barrier之前的load操作之后，才开始执行read memory barrier之后的load操作。read memory barrier指令像一道栅栏，严格区分了之前和之后的load操作。
- 同样的，write memory barrier指令，它只是约束执行CPU上的store操作的顺序，具体的效果就是CPU一定是完成write memory barrier之前的store操作之后，才开始执行write memory barrier之后的store操作。
- 全功能的memory barrier会同时约束load和store操作，当然只是对执行memory barrier的CPU有效。

对于x86，写屏障Store Barrier，是”sfence“指令。读屏障Load屏障，是”lfence“指令。读写屏障Full屏障，是”mfence“指令，其复合了Store和Load屏障的功能。

## volatile

带有volatile修饰的变量，会多一个lock指令，lock指令实际相当于一个内存屏障。lock前缀的指令有三个功能:
- 确保指令重排序时不会把其后面的指令重排到内存屏障之前的位置，也不会把前面的指令排到内存屏障后面，即在执行到内存屏障这句指令时，前面的操作已经全部完成。
- 将当前处理器缓存行的数据写回到系统内存。
- 这个写回内存的操作会使在其他cpu里缓存了该内存地址的数据无效。


