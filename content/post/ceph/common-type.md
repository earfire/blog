---
title: "ceph基础数据结构"
date: 2020-02-09T19:43:05+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## Object

Object是ceph的底层对象，定义的数据结构所在的源文件如下：

| type     		| position              |   
| ------------- |:----------------------|
| object_t		| include/object.h		|
| sobject_t		| include/object.h		|
| hobject_t		| common/hobject.h		|
| ghobject_t	| common/hobject.h		|

### object_t

obejct_t结构体：


```
struct object_t {
    string name;
    ...

    void encode(bufferlist &bl) const {
        using ceph::encode;
        encode(name, bl);
    }
    void decode(bufferlist::const_iterator &bl) {
        using ceph::decode;
        decode(name, bl);
    }
};
```

object_t对应的是底层文件系统中的一个文件，name就对于文件名。
encode和decode函数对应的是序列化和反序列化操作。序列化就是将数据结构转换成二进制流；反之，就是反序列化。在ceph中，支持序列化/反序列化的数据结构，都会定义其对应的encode/decode函数。

### sobject_t

sobject_t就是在object_t基础上增加了snapshot相关信息，用于标识是否是快照对象。数据成员snap为快照对象的序号。如果一个对象是head对象，snap字段对应的值为CEPH_NOSNAP，snapdir对象为CEPH_SNAPDIR，否则为克隆对象。

```
struct sobject_t {
    object_t oid;
    snapid_t snap;
    ...
};
```

snapid_t其实就是一个uint64_t类型的值:

```
struct snapid_t {
    uint64_t val;

    snapid_t(uint64_t v=0) : val(v) {}
    snapid_t operator+=(snapid_t o) { val += o.val; return *this; }
    snapid_t operator++() { ++val; return *this; }
    operator uint64_t() const { return val; }   
};
```

### hobject_t

hobject_t代表的是hash object，其结构体如下：

```
struct hobject_t {
  public:
    static const int64_t POOL_META = -1;
    static const int64_t POOL_TEMP_START = -2; // and then negative
  public:
    object_t oid;
    snapid_t snap;

  private:
    uint32_t hash; //hash和key不能同时设置，hash值一般设置为就是pg的id值?
    bool max;
    uint32_t nibblewise_key_cache;
    uint32_t hash_reverse_bits;
  public:
    int64_t pool; //所在的pool的id
    string nspace; //一般为空，用来标识特殊的对象

  private:
    string key; //特殊对象的标记
};
```

### ghobject_t

ghobject_t比hobject_t基础上添加了一些字段，主要是generation和shard_id字段，主要用于ErasureCode模式下的PG。

```
struct ghobject_t { //for Erasure Code Mode of PG
    hobject_t hobj;
    gen_t generation; //对象的版本号。当PG为EC时，写操作需要区分写前后两个版本的object。写操作保存对象的上一个版本generation的对象，当EC写失败时，可以rollback到上一个版本。
    shard_id_t shard_id; //对象所在的osd在EC类型的PG中的序号。如果是Replicate类型的PG，该字段设置为NO_SHARD。
    bool max;
};
```

　　

## Buffer

buffer是一个命名空间，在这个命名空间下定义了Buffer相关的数据结构。

buffer::raw：对应一段真实的物理内存，负责维护这段物理内存的引用计数nref和释放操作。

buffer::ptr：对应Ceph中的一段被使用的内存，也就是某个bufferraw的一部分或者全部。

buffer::list：表示一个ptr的链表，相当于将N个ptr构成一个更大的虚拟的连续内存。

定义的数据结构所在的源文件如下：

| type          | position              |   
| ------------- |:----------------------|
| raw           | include/buffer_raw.h	|
| ptr		    | include/buffer.h		|
| list		    | include/buffer.h		|
| hash  	    | include/buffer.h		|


相关命名空间的定义：

```
#ifndef BUFFER_FWD_H
#define BUFFER_FWD_H

namespace ceph {
  namespace buffer {
    inline namespace v14_2_0 {
        class ptr;
        class list;
    }   
    class hash;
  }

    using bufferptr = buffer::ptr;
    using bufferlist = buffer::list;
    using bufferhash = buffer::hash;
}
#endif
```

### raw

raw数据的实际存储，data指针指向真正的数据，其类定义如下：

```
class raw {
  public:
    char *data; //数据指针
    unsigned len; //数据长度
    std::atomic<unsigned> nref { 0 }; //引用计数
    int mempool; //所属内存池，和data分配有关，如malloc、mmap
};
```

<!--
### raw_xxx

raw_xxx实现了不同的内存分配方式：

| type     		    | method                |   
| ------------------|:----------------------|
| raw_malloc		| malloc函数    		|
| raw_mmap_pages	| mmap内存映射  		|
| raw_posix_aligned | posix_memalign函数	|
| raw_hack_aligned  | 在系统不支持内存对齐申请的情况下自己实现的内存地址对齐 |
| raw_pipe          | 实现了pipe作为Buffer内存空间 |
| raw_char          | 使用C++的new操作来申请|
-->

### ptr

ptr基于raw结构体，代表raw的一段数据。_off是_raw的偏移量，_len是ptr的数据长度。

```
class CEPH_BUFFER_API ptr {
    raw *_raw; //指向raw的指针
    unsigned _off, _len; //数据偏移和数据长度
};
```

### list

list简单来说是ceph内部实现的单向链表，它由多个ptr组成一个更大的内存空间。

```
class CEPH_BUFFER_API list {
    class buffers_t {
        ptr_hook _root;
        ptr_hook* _tail;
        std::size_t _size;
        ...
    };
    
    buffers_t _buffers; //burrers_t相当于一个链表，包含所有的ptr

    unsigned _len; //所有ptr的总长度
    unsigned _memcopy_count; //当调用rebuild用来内存对齐时，需要拷贝的内存量
    mutable iterator last_p; //访问list的迭代器

    class CEPH_BUFFER_API iterator_impl {
        bl_t* bl; //指针，指向bufferlist
        list_t* ls;  //指针，指向bufferlist的成员_buffers
        list_iter_t p; // buffers_t迭代器，用来迭代遍历bufferlist中的buffers
        unsigned off;   // in bl, 当前位置在整个bufferlist的偏移量
        unsigned p_off; // in *p, 当前位置在对应的bufferptr的偏移量
    };
};
```

成员函数push_back用于bufferlist添加元素。


## ThreadPool和WorkQueue

线程池和工作队列是紧密相连的，基本流程就是将任务送入到对应的工作队列中，线程池中的线程从工作队列中取出任务并进行处理。

ceph的线程池实现了多种不同的工作队列。一般情况下，一个线程池对应一个类型的工作队列。在要求不高的情况下，也可以一个线程池对应多种类型的工作队列，让线程池处理不同类型的任务。

| type     		| position              |   
| ------------- |:----------------------|
| ThreadPool    | common/WorkQueue.h	|
| WorkQueue     | common/WorkQueue.h	|

### ThreadPool

```
class ThreadPool : public md_config_obs_t {
    CephContext *cct;
    std::string name; //线程池的名字
    std::string thread_name;
    std::string lockname; //锁的名字
    ceph::mutex _lock; //线程互斥的锁，也是工作队列互斥的锁
    ceph::condition_variable _cond; //锁对应的条件变量
    bool _stop; //线程池是否停止的标志
    int _pause; //暂停中止线程池的标志
    int _draining;
    ceph::condition_variable _wait_cond;

    unsigned _num_threads; //线程数量
    std::vector<WorkQueue_*> work_queues; //工作队列
    int next_work_queue = 0; //最后访问的工作队列
    std::set<WorkThread*> _threads; //工作线程集合
    std::list<WorkThread*> _old_threads;  //等待Join操作的旧线程集合
}
```

work_queues工作队列保存要处理的任务。_threads是WorkThread的集合，WorkThread的基类是Thread：

```
// threads
struct WorkThread : public Thread {
    ThreadPool *pool;
    // cppcheck-suppress noExplicitConstructor
    WorkThread(ThreadPool *p) : pool(p) {}
    void *entry() override {
        pool->worker(this);
        return 0;
    }   
};
```

线程的主函数是entry()，也就是调用ThreadPool::worker()函数。


### WorkQueue

工作队列WorkQueue，就是将线程池要处理的任务加入到其工作队列中，进行后续处理。
WorkQueue实现了入队出队等功能，_void_process和_void_process_finish是WorkQueue基类中定义的函数，真正定义WorkQueue的时候，需继承该基类，定义自己的_void_process和_void_process_finish函数，来完成相应任务的具体处理。

```
template<class T>
class WorkQueue : public WorkQueue_ {
    ThreadPool *pool;

    /// Add a work item to the queue.
    virtual bool _enqueue(T *) = 0;
    /// Dequeue a previously submitted work item.
    virtual void _dequeue(T *) = 0;
    /// Dequeue a work item and return the original submitted pointer.
    virtual T *_dequeue() = 0;
    virtual void _process_finish(T *) {}

    // implementation of virtual methods from WorkQueue_
    void *_void_dequeue() override {
        return (void *)_dequeue();
    }   
    void _void_process(void *p, TPHandle &handle) override {
        _process(static_cast<T *>(p), handle);
    }   
    void _void_process_finish(void *p) override {
        _process_finish(static_cast<T *>(p));
    }

    protected:
    /// Process a work item. Called from the worker threads.
    virtual void _process(T *t, TPHandle &) = 0;
}
```

#### **线程池的启动**

线程池的启动由ThreadPool::start()来执行，其调用start_threads()函数。

```
void ThreadPool::start_threads()
{
    ceph_assert(ceph_mutex_is_locked(_lock));
    while (_threads.size() < _num_threads) {
        WorkThread *wt = new WorkThread(this);
        ldout(cct, 10) << "start_threads creating and starting " << wt << dendl;
        _threads.insert(wt);

        wt->create(thread_name.c_str());
    }
}
```

函数主要判断当前启动的线程数是否小于所配置的线程数，如果是的话，则创建新的线程，加入到_threads集合中。然后通过wt->create()调用线程主函数ThreadPool::worker()。

#### **线程数动态调整**

线程池的线程数量支持动态增加或减少，通过修改配置进行动态调整。增加线程数时，也会调用start_threads函数，该函数不仅可以初始化所有工作线程的启动，还可以用于线程的动态增加。

```
void ThreadPool::handle_conf_change(const ConfigProxy& conf,
        const std::set <std::string> &changed)
{
    if (changed.count(_thread_num_option)) {
        char *buf;
        int r = conf.get_val(_thread_num_option.c_str(), &buf, -1);
        ceph_assert(r >= 0);
        int v = atoi(buf);
        free(buf);
        if (v >= 0) {
            _lock.lock();
            _num_threads = v;
            start_threads();
            _cond.notify_all();
            _lock.unlock();
        }
    }
}
```

线程数的减少，在线程主函数ThreadPool::worker实现,主线程会检测线程的个数, 如果当前线程数大于配置的数量_num_threads，就把当前线程从线程集合中删除,加入_old_threads队列中，然后break跳出while循环，该线程主函数返回退出，达到线程减少。join_old_threads函数主要检查是否存在退出的线程，如果存在则进行join，回收资源。

```
void ThreadPool::worker(WorkThread *wt)
{
    while (!_stop) {
    // manage dynamic thread pool
    join_old_threads();
    if (_threads.size() > _num_threads) {
        ldout(cct,1) << " worker shutting down; too many threads (" << _threads.size() << " > " << _num_threads << ")" << dendl;
        _threads.erase(wt);
        _old_threads.push_back(wt);
        break;

        ...
    }
}

void ThreadPool::join_old_threads()
{
    ceph_assert(ceph_mutex_is_locked(_lock));
    while (!_old_threads.empty()) {
        ldout(cct, 10) << "join_old_threads joining and deleting " << _old_threads.front() << dendl;
        _old_threads.front()->join();
        delete _old_threads.front();
        _old_threads.pop_front();
    }
}
```

#### **工作线程主函数**

在while循环中，_pause和work_quues.empty确保当前线程没有暂停并且工作队列不为空,就开始遍历work_queues，以next_work_queue为索引得到wq，然后通过wq->_void_dequeue()，从该工作队列中获取到item，如果存在，就调用工作队列的处理函数wq->_void_process和wq->_void_process_finish进行处理。

```
void ThreadPool::worker(WorkThread *wt)
{
    ...

    while (!_stop) {
        // manage dynamic thread pool
        join_old_threads();
        if (_threads.size() > _num_threads) {
            ldout(cct,1) << " worker shutting down; too many threads (" << _threads.size() << " > " << _num_threads << ")" << dendl;
            _threads.erase(wt);
            _old_threads.push_back(wt);
            break;
        }

        if (!_pause && !work_queues.empty()) {
            WorkQueue_* wq;
            int tries = 2 * work_queues.size();
            bool did = false; 
            while (tries--) {
                next_work_queue %= work_queues.size();
                wq = work_queues[next_work_queue++];

                void *item = wq->_void_dequeue();
                if (item) {
                    processing++;
                    ldout(cct,12) << "worker wq " << wq->name << " start processing " << item
                        << " (" << processing << " active)" << dendl;
                    TPHandle tp_handle(cct, hb, wq->timeout_interval, wq->suicide_interval);
                    tp_handle.reset_tp_timeout();
                    ul.unlock();
                    wq->_void_process(item, tp_handle);
                    ul.lock();
                    wq->_void_process_finish(item);
                    processing--;
                    ldout(cct,15) << "worker wq " << wq->name << " done processing " << item
                        << " (" << processing << " active)" << dendl;
                    if (_pause || _draining)
                        _wait_cond.notify_all();
                    did = true;
                    break;
                }
            }
            if (did)
                continue;
        }

        ...
    }
}
```

### TPHandle

TPHandle用于超时检查，每次线程函数执行时，都会创建一个TPHandle，并根据配置设置超时时间和自杀时间。如果当前任务的执行时间超过grace，cct->get_heartbeat_map()->is_healthy()就会返回false。如果当前任务的执行时间超过suicide_grace时，线程就会自杀。

```
class TPHandle {
    friend class ThreadPool;
    CephContext *cct;
    heartbeat_handle_d *hb; //心跳，有一个定时器会检测是否超时
    ceph::coarse_mono_clock::rep grace; //超时时间
    ceph::coarse_mono_clock::rep suicide_grace; //自杀超时
};
```

### ShardedThreadPool

ShardedThreadPool不同于ThreadPool，它用于处理执行顺序有要求的任务，即需要先处理任务A，任务A处理完毕后才能处理任务B。

ShardedThreadPool基本原理就是每一个线程对应一个队列，所有要求顺序执行的任务都放入到该队列里，然后全部由该线程顺序处理。 所要顺序执行的任务，需关联相同的thread_index。

```
class ShardedThreadPool {
    uint32_t num_threads;
    std::vector<WorkThreadSharded*> threads_shardedpool;
    void shardedthreadpool_worker(uint32_t thread_index);
};
```

## Finisher

类Finisher用来异步完成回调函数Context的执行，其内部有一个finisher_queue用来contexts的入队，有一个FinisherThread线程用来执行回调函数。

```
class Finisher {
    CephContext *cct;

    /// Queue for contexts for which complete(0) will be called.
    vector<pair<Context*,int>> finisher_queue; //任务队列

    string thread_name;

    void *finisher_thread_entry(); //线程主函数

    struct FinisherThread : public Thread {
        Finisher *fin;    
        explicit FinisherThread(Finisher *f) : fin(f) {}
        void* entry() override { return fin->finisher_thread_entry(); }
    } finisher_thread;
}
```

## Throttle

类Throttle用来限制消费的资源数量。

## SafeTimer 

类SafeTimer实现了定时器的功能。

