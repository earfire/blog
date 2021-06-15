---
title: "ceph的上下文CephContext"
date: 2020-08-05T15:16:19+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## CephContext

```
class CephContext {
public:
    bool _finished = false;
private:
    // ref count!
    std::atomic<unsigned> nref;
public:
    void put();

    ConfigProxy _conf;
    ceph::logging::Log *_log;
    ...
};
```
