---
title: "DPDK的性能优化"
date: 2018-08-22T20:04:01+08:00
draft: false
tags: ["dpdk"]
categories: ["dpdk"]
---

## 轮询

DPDK采用轮询方式进行网卡报文收发，避免了常规报文处理方法因采用中断方式造成的响应延迟上下文切换，减少中断带来的CPU开销，极大提升了收发性能。

## 大页




## 地址对齐

当变量的地址被它本身的长度整除时，存取该变量最高校。通常对于频繁访问的结构体，要将其按缓存行大小对齐，减少Cass Miss、False Sharing。

```
(uing64_t)&val % sizeof(var) == 0

struct {

} __attribute__((aligned(64)));
```
