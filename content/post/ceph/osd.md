---
title: "ceph OSD"
date: 2020-08-23T20:53:34+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## OSD类

OSD内部驻留的公关服务组件主要由OSDService、各种类型的线程池、定时器等。

```
class OSD {
protected:
    Messenger   *cluster_messenger;
    Messenger   *client_messenger;
    Messenger   *objecter_messenger;
    MonClient   *monc;
    MgrClient   mgrc;

    ObjectStore *store;

    int whoami; //OSD在集群中的数字标识，这是一个从0开始编号的整数，全集群唯一，可以直接作为OSD的身份标识
    std::string dev_path, journal_path;

    int last_require_osd_release = 0;

    bool store_is_rotational = true;
    bool journal_is_rotational = true;

private:
    OSDSuperblock superblock;

    CompatSet osd_compat;

    std::atomic<int> state{STATE_INITIALIZING};

    ShardedThreadPool osd_op_tp; //线程池
    ThreadPool command_tp;

    Messenger *hb_front_client_messenger;
    Messenger *hb_back_client_messenger;
    Messenger *hb_front_server_messenger;
    Messenger *hb_back_server_messenger;

protected:
    OSDMapRef osdmap;

public:
    vector<OSDShard*> shards;
    uint32_t num_shards = 0;

protected:
    std::atomic<size_t> num_pgs = {0};

public:
    OSDService service;

};
```

### OSDService

OSDService组件用于提供OSD级别的服务，由于OSD是PG的载体，所以OSDService的服务对象也主要是PG。

```
class OSDService {
public:
    OSD *osd;
    CephContext *cct;
    ObjectStore::CollectionHandle meta_ch;
    const int whoami;
    ObjectStore *&store;
    LogClient &log_client;
private:
    Messenger *&cluster_messenger;
    Messenger *&client_messenger;
public:
    PerfCounters *&logger;
    PerfCounters *&recoverystate_perf;
    MonClient   *&monc;
    ClassHandler  *&class_handler;
private:
    OSDSuperblock superblock;
private:
    OSDMapRef osdmap;
private:
    OSDMapRef next_osdmap;
    ceph::condition_variable pre_publish_cond;
public:
    // -- Objecter, for tiering reads/writes from/to other OSDs --
    Objecter *objecter;
    int m_objecter_finishers;
    vector<Finisher*> objecter_finishers;

};
```
### 线程池

任何任务想要使用线程池中的线程资源时，必须先实例化线程池所支持的、某种抽象类型的工作队列，定义该任务的具体执行方式(例如入队、出队方法，真正的处理函数等)，然后再将工作队列与对应的线程池进行绑定。

### 定时器

定时器用于处理OSD内部一些需要周期性执行的任务，例如心跳检测、心跳伙伴刷新、Scrub调度等。定时器有需要全局osd_lock的SafeTimer tick_timer，有不需要全局锁的SafeTimer tick_timer_without_osd_lock。不需要osd_lock保护的任务占据了90%的比重。


