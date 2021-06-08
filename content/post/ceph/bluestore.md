---
title: "ceph源码分析BlueStore"
date: 2020-10-27T15:16:19+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## 概述

Ceph使用BlueStore代替之前的FileStore，BlueStore可以绕过文件系统，直接管理裸设备，进行对象数据IO操作。

磁盘IO操作的最小单元为BlockSize，HDD为512B，SSD为4K。读写的数据即使小于BlockSize，磁盘IO的大小也是BlockSize。BlueStore的写策略结合运用了RMW(Read-Modify-Write)和COW(Copy-On-Write)两种策略，对于写入数据块对齐的部分，采用COW，其他部分采用RMW。


## 整体架构

整体架构如下：

![Local Picture](/images/ceph/bluestore/bluestore_arch.jpg "BlueStore Architecture")

主要模块：
- BlockDevice：最底层的块设备，BlueStore直接操作块设备，抛弃了本地文件系统。BlockDevice在用户态直接以linux系统实现的AIO直接操作块设备文件。
- RocksDB：存放所有的元数据，包括存储WAL、对象元数据、对象扩展属性omap、磁盘分配器元数据。
- BlueRocksEnv：这是RocksDB与BlueFS交互的接口；RocksDB提供了文件操作的接口EnvWrapper，用户可以通过继承实现该接口来自定义底层的读写操作，BlueRocksEnv就是继承自EnvWrapper实现对BlueFS的读写。
- BlueFS：BlueFS是BlueStore针对RocksDB开发的轻量级文件系统，相较于POSIX文件系统结构简单，且支持多种设备，性能更高，其用于存放RocksDB产生的.sst和.log等文件。
- Allocator：磁盘分配器，负责高效的分配磁盘空间。支持StupidAllocator和BitmapAllocator两种分配器，Stupid基于extent的方式实现。

BlueStore将所有存储空间从逻辑上分了3个层次：
- 慢速空间(SLOW)：存储对象数据，可以使用HDD，由BlueStore管理。
- 高速空间(DB)：存储RocksDB的.sst文件，可以使用SSD，由BlueFS管理。
- 超高速空间(WAL)：存储RocksDB的.log文件，可以使用NVME，由BlueFS管理。


## 数据结构

### ObjectStore

类ObjectStore是对象存储系统抽象操作接口。所有的对象存储引擎都要继承并实现它定义的接口。

```
class ObjectStore {
protected:
    string path;
public:
    CephContext* cct; 
    static ObjectStore *create(CephContext *cct, //根据type进行相应类型的实例化，如filestore、bluestore、memstore
            const string& type,
            const string& data,
            const string& journal,
            osflagbits_t flags = 0);

    struct CollectionImpl : public RefCountedObject { //同一Collection下的排队的事务会顺序处理，而不同Collection下的事务存在并行处理
        const coll_t cid;
    };
    typedef boost::intrusive_ptr<CollectionImpl> CollectionHandle;

    class Transaction { //用来实现相关的事务
    private:
        TransactionData data;

        map<coll_t, __le32> coll_index; //coll_t -> coll_id的映射关系
        map<ghobject_t, __le32> object_index; //ghobject -> object_id的映射关系

        __le32 coll_id {0}; //当前分配的coll_id的最大值
        __le32 object_id {0}; //当前分配的object_id的最大值

        bufferlist data_bl; //数据
        bufferlist op_bl; //元数据操作
        //一个事务中，对应如下三类回调函数，分别在事务不同的处理阶段调用
        list<Context *> on_applied; //事务应用完成之后的回调函数，在Finisher线程里异步调用执行
        list<Context *> on_commit; //事务提交完成之后调用的回调函数
        list<Context *> on_applied_sync; //事务应用完成之后的回调函数，同步调用执行
    };

    virtual int queue_transactions( //所有ObjectStore更新操作的接口，更新相关的操作(如创建一个对象，修改属性，写数据等)都是以事务的方式提交给ObjectStore。
            CollectionHandle& ch, vector<Transaction>& tls,
            TrackedOpRef op = TrackedOpRef(),
            ThreadPool::TPHandle *handle = NULL) = 0;

    virtual bool test_mount_in_use() = 0;
    virtual int mount() = 0; //加载objectstore相关的系统信息
    virtual int umount() = 0;
    virtual int mkfs() = 0;  // wipe //创建objectstore相关的系统信息
    virtual int mkjournal() = 0; // journal only

    virtual int getattr(CollectionHandle &c, const ghobject_t& oid, const char *name, bufferptr& value) = 0; //获取对象的扩展属性
    virtual int omap_get( //获取对象的omap信息
            CollectionHandle &c,     ///< [in] Collection containing oid
            const ghobject_t &oid,   ///< [in] Object containing omap
            bufferlist *header,      ///< [out] omap header
            map<string, bufferlist> *out /// < [out] Key to value map
            ) = 0;
};
```

### aio_t

对libaio操作的封装，IO的最小结构体。

```
struct aio_t {
#if defined(HAVE_LIBAIO)
    struct iocb iocb{};  // must be first element; see shenanigans in aio_queue_t //libio相关结构体
#elif defined(HAVE_POSIXAIO)
    //  static long aio_listio_max = -1;
    union {
        struct aiocb aiocb;
        struct aiocb *aiocbp;
    } aio;
    int n_aiocb;
#endif
    void *priv;
    int fd; 
    boost::container::small_vector<iovec,4> iov;
    uint64_t offset, length;
    long rval;
    bufferlist bl;  ///< write payload (so that it remains stable for duration)

    boost::intrusive::list_member_hook<> queue_item;

    aio_t(void *p, int f) : priv(p), fd(f), offset(0), length(0), rval(-1000) {
    }

    void pwritev(uint64_t _offset, uint64_t len); //对io_prep_pwrite的封装
    void pread(uint64_t _offset, uint64_t len); //对io_prep_pread的封装
};

```

### aio_queue_t 

用于初始化、提交、完成IO，一个裸设备一个aio队列。

```
struct aio_queue_t {
    int max_iodepth;
#if defined(HAVE_LIBAIO)
    io_context_t ctx;
#elif defined(HAVE_POSIXAIO)
    int ctx;
#endif

    typedef list<aio_t>::iterator aio_iter;

    explicit aio_queue_t(unsigned max_iodepth)
        : max_iodepth(max_iodepth),
        ctx(0) {
        }
    ~aio_queue_t() {
        ceph_assert(ctx == 0); 
    }

    int init(); //初始化，初始化io_context
    int submit_batch(aio_iter begin, aio_iter end, uint16_t aios_size,  //提交IO
            void *priv, int *retries);
    int get_next_completed(int timeout_ms, aio_t **paio, int max); //处理完成的IO
};

```

### IOContext

每个IO请求都会生成一个IO上下文，里面包含多个aio_t。
```
struct IOContext {
public:
    CephContext* cct;
    void *priv;
#ifdef HAVE_SPDK
    void *nvme_task_first = nullptr;
    void *nvme_task_last = nullptr;
    std::atomic_int total_nseg = {0};
#endif

#if defined(HAVE_LIBAIO) || defined(HAVE_POSIXAIO)
    std::list<aio_t> pending_aios;    ///< not yet submitted //待执行的aio
    std::list<aio_t> running_aios;    ///< submitting or submitted //正在执行的aio
#endif
    std::atomic_int num_pending = {0};
    std::atomic_int num_running = {0};
    bool allow_eio;
};

```

### BlockDevice

BlockDevice是抽象出来的基类类型，统一管理各种类型的设备，如Kernel, NVME, PMEM等，为裸盘的使用者(BlueFS/BlueStore)提供统一的操作接口。KernelDevice、NVMEDevice、PMEMDevice都继承于该BlockDevice基类。

```
class BlockDevice {
public:
    CephContext* cct;
    typedef void (*aio_callback_t)(void *handle, void *aio);
private:
    ceph::mutex ioc_reap_lock = ceph::make_mutex("BlockDevice::ioc_reap_lock");
    std::vector<IOContext*> ioc_reap_queue;
    std::atomic_int ioc_reap_count = {0};

protected:
    uint64_t size; //设备的总大小
    uint64_t block_size; //块的大小
    bool support_discard = false;
    bool rotational = true;
    bool lock_exclusive = true;

public:
    aio_callback_t aio_callback; //aio操作相关回调函数
    void *aio_callback_priv;
};
```

### KernelDevice

KernelDevice的继承于BlockDevice，对应的物理设备为HDD和Sata SDD，其定义如下：

```
class KernelDevice : public BlockDevice {
    std::vector<int> fd_directs, fd_buffereds; //分别存放以direct、buffered两种方式打开裸设备的fd
    bool enable_wrt = true;
    std::string path; //设备路径
    bool aio, dio; //是否启动libaio

    int vdo_fd = -1;      ///< fd for vdo sysfs directory
    string vdo_name;

    std::string devname;  ///< kernel dev name (/sys/block/$devname), if any

    std::atomic<bool> io_since_flush = {false};

    aio_queue_t aio_queue; //aio操作相关的队列
    aio_callback_t discard_callback;
    void *discard_callback_priv;
    bool aio_stop;
    bool discard_started;
    bool discard_stop;

    ceph::condition_variable discard_cond;
    bool discard_running = false;
    interval_set<uint64_t> discard_queued; //interval_set是offset+length, discard_queued存放需要做Discard的Extent。
    interval_set<uint64_t> discard_finishing; //discard_finishing和discard_queued交换值，存放完成Discard的Extent

    //aio线程，用于处理完成的IO
    struct AioCompletionThread : public Thread {
        KernelDevice *bdev;
        explicit AioCompletionThread(KernelDevice *b) : bdev(b) {}
        void *entry() override {
            bdev->_aio_thread();
            return NULL;
        }
    } aio_thread;


    struct DiscardThread : public Thread { //Discard线程，用于SSD的Trim
        KernelDevice *bdev;
        explicit DiscardThread(KernelDevice *b) : bdev(b) {}
        void *entry() override {
            bdev->_discard_thread();
            return NULL;
        }
    } discard_thread;

    //同步IO
    int read(uint64_t off, uint64_t len, bufferlist *pbl,
            IOContext *ioc,
            bool buffered) override;
    int write(uint64_t off, bufferlist& bl, bool buffered, int write_hint = WRITE_LIFE_NOT_SET) override;

    //异步IO
    int aio_read(uint64_t off, uint64_t len, bufferlist *pbl,
            IOContext *ioc) override;
    int aio_write(uint64_t off, bufferlist& bl,
            IOContext *ioc,
            bool buffered,
            int write_hint = WRITE_LIFE_NOT_SET) override;

    int flush() override;
    int discard(uint64_t offset, uint64_t len) override; //对SSD指定offset、len的数据做Trim
};
```

初始化设备

块设备由BlockDevice::create()实例化，其会根据根据不同的设备类型type来创建不同的设备，如new KernelDevice()。设备创建完成之后，需要对设备进行打开并初始化，KernelDevice::open()。

```

KernelDevice::KernelDevice(CephContext* cct, aio_callback_t cb, void *cbpriv, aio_callback_t d_cb, void *d_cbpriv)
    : BlockDevice(cct, cb, cbpriv),
    aio(false), dio(false),
    aio_queue(cct->_conf->bdev_aio_max_queue_depth),
    discard_callback(d_cb),
    discard_callback_priv(d_cbpriv),
    aio_stop(false),
    discard_started(false),
    discard_stop(false),
    aio_thread(this),
    discard_thread(this),
    injecting_crash(0)
{
    fd_directs.resize(WRITE_LIFE_MAX, -1); 
    fd_buffereds.resize(WRITE_LIFE_MAX, -1); 
}

int KernelDevice::open(const string& p)
{
    //以direct、buffered两种方式打开块设备
    for (i = 0; i < WRITE_LIFE_MAX; i++) {
        int fd = ::open(path.c_str(), O_RDWR | O_DIRECT);
        fd_directs[i] = fd;

        fd  = ::open(path.c_str(), O_RDWR | O_CLOEXEC);
        fd_buffereds[i] = fd;
    }

    ...

    /**
     * _aio_start()进行的操作:
     * 1. aio_queue.init()初始化aio_queue，调用io_setup()为当前进程初始化一个异步IO上下文
     * 2. 启动aio_thread
     */
    r = _aio_start();

    /* 启动discard_thread */
    _discard_start();

    ...
};

```

aio读写

libaio的读写请求都用io_submit下发。读请求KernelDevice::aio_read()和写请求KernelDevice::aio_write()分别调用pread和pwritev都io操作进行封装，然后将其添加到IOContext->pending_aios结构体中，最后调用aio_submit提交IO操作。

```
void KernelDevice::aio_submit(IOContext *ioc)
{
    //获取pending的aio
    list<aio_t>::iterator e = ioc->running_aios.begin();
    ioc->running_aios.splice(e, ioc->pending_aios);

    //提交aio
    r = aio_queue.submit_batch(ioc->running_aios.begin(), e,
    pending, priv, &retries);
}
```

aio线程

在KernelDevice::open()中的_aio_start()会启动aio_thread，用检查IO的完成情况，当真正完成的时候，执行回调函数通知调用方。

```
void KernelDevice::_aio_thread()
{
    while (!aio_stop) {
        //最终调用libaio的io_getevents api，检查io是否完成。
        int r = aio_queue.get_next_completed(cct->_conf->bdev_aio_poll_ms, aio, max);

        //执行回调函数
        if (ioc->priv) {
            if (--ioc->num_running == 0) {
                aio_callback(aio_callback_priv, ioc->priv);
            }
        }
    }
}


int aio_queue_t::get_next_completed(int timeout_ms, aio_t **paio, int max)
{
#if defined(HAVE_LIBAIO)
    io_event events[max];
#elif defined(HAVE_POSIXAIO)
    struct kevent events[max];
#endif
    struct timespec t = { 
        timeout_ms / 1000,
        (timeout_ms % 1000) * 1000 * 1000
    };  

    int r = 0;
    do {
#if defined(HAVE_LIBAIO)
        r = io_getevents(ctx, 1, max, events, &t);
#elif defined(HAVE_POSIXAIO)
        r = kevent(ctx, NULL, 0, events, max, &t);
        if (r < 0)
            r = -errno;
#endif
    } while (r == -EINTR);
};
```


Discard

在KernelDevice::open()中的discard_start()会启动discard_thread，不断的从discard_queued里面取出Extent，做Discard(Trim)。

```
void KernelDevice::_discard_thread()
{
    while (true) {
        if (discard_queued.empty()) { //空就等待
            if (discard_stop)
                break;
        } else {
            discard_finishing.swap(discard_queued);

            //对需要做Discard的Extent依次调用discard函数
            for (auto p = discard_finishing.begin();p != discard_finishing.end(); ++p) {
                discard(p.get_start(), p.get_len());
            }

            discard_callback(discard_callback_priv, static_cast<void*>(&discard_finishing));
            discard_finishing.clear();
        }
    }
}


int KernelDevice::discard(uint64_t offset, uint64_t len)
{
    if (support_discard) {
        r = BlkDev{fd_directs[WRITE_LIFE_NOT_SET]}.discard((int64_t)offset, (int64_t)len);
    }
    return r;
}

int BlkDev::discard(int64_t offset, int64_t len) const
{
    uint64_t range[2] = {(uint64_t)offset, (uint64_t)len};
    return ioctl(fd, BLKDISCARD, range);
}
```

### BlueFS

BlueFS负责对接RocksDB，是一个精简的文件系统，往下整合磁盘设备，往上给RocksDB提供一些必须的接口。

```
class BlueFS {
public:
    CephContext* cct;

    struct File : public RefCountedObject {};
    struct Dir : public RefCountedObject {};
    struct FileWriter {};
    struct FileReaderBuffer {};
    struct FileReader {};
    struct FileLock {};

    // cache
    mempool::bluefs::map<string, DirRef> dir_map;              ///< dirname -> Dir
    mempool::bluefs::unordered_map<uint64_t,FileRef> file_map; ///< ino -> File

    // map of dirty files, files of same dirty_seq are grouped into list.
    map<uint64_t, dirty_file_list_t> dirty_files;

    bluefs_super_t super;        ///< latest superblock (as last written)

    /*
     * There are up to 3 block devices:
     *
     *  BDEV_DB   db/      - the primary db device
     *  BDEV_WAL  db.wal/  - a small, fast device, specifically for the WAL
     *  BDEV_SLOW db.slow/ - a big, slow device, to spill over to as BDEV_DB fills
     */
    vector<BlockDevice*> bdev;                  ///< block devices we can use
    vector<IOContext*> ioc;                     ///< IOContexts for bdevs
    vector<interval_set<uint64_t> > block_all;  ///< extents in bdev we own
    vector<Allocator*> alloc;                   ///< allocators for bdevs
    vector<uint64_t> alloc_size;                ///< alloc size for each device
    vector<interval_set<uint64_t>> pending_release; ///< extents to release

    BlockDevice::aio_callback_t discard_cb[3]; //discard callbacks for each dev
};

```

实例化BlueFS时会设置discard回调函数

```
BlueFS::BlueFS(CephContext* cct) 
    : cct(cct),
    bdev(MAX_BDEV),
    ioc(MAX_BDEV),
    block_all(MAX_BDEV)
{
    discard_cb[BDEV_WAL] = wal_discard_cb;
    discard_cb[BDEV_DB] = db_discard_cb;
    discard_cb[BDEV_SLOW] = slow_discard_cb;
    asok_hook = SocketHook::create(this);
}
```

_open_db

BlueFS上电加载过程是通过open_db函数实现的。如果是创建osd，则调用_open_db(true)，如果是启动osd，则调用_open_db(false)。

```
int BlueStore::_open_db(bool create, bool to_repair_db, bool read_only)
{
    string fn = path + "/db"; //path = /var/lib/ceph/osd/osd-id/

    string kv_backend;
    std::vector<KeyValueDB::ColumnFamily> cfs;

    if (create) {
        kv_backend = cct->_conf->bluestore_kvbackend; //"rocksdb"
    } else {
        r = read_meta("kv_backend", &kv_backend); //先从block设备读取，读不到就从path/kv_backend文件读
    }

    bool do_bluefs;
    r = _is_bluefs(create, &do_bluefs); //如果是创建，就设置do_bluefs的值为cct->_conf->bluestore_bluefs，否则就read_meta("bluefs");

    if (do_bluefs) {
        r = _open_bluefs(create); //初始化BlueFS，RocksDB
  
        /**
         * RocksDB相关初始化
         * BlueRocksEnv继承rocksdb::EnvWrapper，BlueRocksEnv负责给rocksdb提供一些定制化的接口。
         * 如果是创建osd，则创建三个目录，名称分别是db、db.wal、db.slow。
         */
        env = new BlueRocksEnv(bluefs);

        if (create) {
            env->CreateDir(fn);
            env->CreateDir(fn + ".wal");
            env->CreateDir(fn + ".slow");
        }
    }

    // KeyValueDB::create()根据type(kv_backend)进行实例化，如type为rocksdb，则new RocksDBStore()。
    db = KeyValueDB::create(cct, kv_backend, fn, kv_options, static_cast<void*>(env));

    FreelistManager::setup_merge_operators(db);
    db->set_merge_operator(PREFIX_STAT, merge_op);
    db->set_cache_size(cache_kv_ratio * cache_size);

    if (kv_backend == "rocksdb") {
        options = cct->_conf->bluestore_rocksdb_options;

        map<string,string> cf_map;
        cct->_conf.with_val<string>("bluestore_rocksdb_cfs", get_str_map, &cf_map, " \t");
        for (auto& i : cf_map) {
            cfs.push_back(KeyValueDB::ColumnFamily(i.first, i.second)); //设置rocksdb的列簇，列簇名为L、M、P
        }
    }

    db->init(options); //初始化rocksdb的一些参数配置
    if (create) { //如果需要创建则create_and_open否则直接open
        if (cct->_conf.get_val<bool>("bluestore_rocksdb_cf")) {
            r = db->create_and_open(err, cfs);
        } else {
            r = db->create_and_open(err);
        }
    } else {
        // we pass in cf list here, but it is only used if the db already has
        // column families created.
        r = read_only ? db->open_read_only(err, cfs) : db->open(err, cfs);
    }
}


```

_open_bluefs

初始化BlueFS

```
int BlueStore::_open_bluefs(bool create)
{
    int r = _minimal_open_bluefs(create);
    if (r < 0) {
        return r;
    }
    if (create) {
        bluefs->mkfs(fsid);
    }
    r = bluefs->mount();
    if (r < 0) {
        derr << __func__ << " failed bluefs mount: " << cpp_strerror(r) << dendl;
    }
    return r;
}

```


_minimal_open_bluefs

```
int BlueStore::_minimal_open_bluefs(bool create)
{
    bluefs = new BlueFS(cct);

    /**
     * 在这里，会创建三个块设备block.db、block、block.wal
     * 
     * bluefs->add_block_device用于创建一个块设备，进行open初始化，添加此新建的块设备到bdev[id]=b, 
     * 创建IOcontext到ioc[id] = new IOContext(cct, NULL)。
     * 
     * block.db: 通过调用add_block_device将block.db设备加入到bluefs的管理之中，block.db负责存储BlueStore内部产生的元数据。
     * 会调用_check_or_set_bdev_label函数把一些label写入到block.db设备中，或从block.db设读取一些label。如果是创建osd，
     * 则还需要调用add_block_extent将这个磁盘设备中的空间加入到bluefs的管理之中，其空间为SUPER_RESERVED（8192字节）后的空间，
     * 前SUPER_RESERVED字节空间负责存放一些label信息和bluefs的super块信息。
     *
     * block: block设备与bluestore共享，在BlueStore::mkfs()的_open_bdev()中也创建了该设备。
     * 调用bluefs->add_block_device将block设备加入到bluefs的管理之中，block设备主要用于存储对象数据。
     * 并计算block设备中给bluefs使用的空间，还要将这些空间加入到bluefs_extents中，osd上电启动时，_open_fm函数会将
     * bluefs_extents中的空间使用记录持久化到rocksdb中，同时bluestore的allocator会将这些空间从空闲块btree中删除，以免重复分配。
     * 
     * block.wal: 调用bluefs->add_block_device将block.wal设备加入到bluefs的管理之中，调用_check_or_set_bdev_label函数把一些
     * label写入到block.db设备中，或从block.db设备中读取一些label。同时调用add_block_extent来将这个磁盘设备中的空间加入到bluefs
     * 的管理之中(默认从BDEV_LABEL_BLOCK_SIZE(4096)到最后)。
     *
     **/

     ...
}

```

mkfs

```
int BlueFS::mkfs(uuid_d osd_uuid)
{
    // 调用_init_alloc将每个设备(BDEV_WAL、BDEV_SLOW、BDEV_DB)的空闲空间加入到对应的stupidallocator中，空闲空间由前面的add_block_extent插入到block_all，stupidallocator是利用btree来保存空闲块的。
    _init_alloc();
    _init_logger();

    super.version = 1; 
    super.block_size = bdev[BDEV_DB]->get_block_size();
    super.osd_uuid = osd_uuid;
    super.uuid.generate_random();

    // init log
    // 调用_allocate函数给log file分配空间，log file的ino为1，优先选择block.wal设备存储，第一次分配的大小为4M，BlueFs的_allocate最终会调用StupidAllocator的allocate来分配具体的空间。最后_allocate将分配的pextent插入到log file的extents中
    FileRef log_file = new File;
    log_file->fnode.ino = 1; 
    log_file->fnode.prefer_bdev = BDEV_WAL;
    int r = _allocate(
            log_file->fnode.prefer_bdev,
            cct->_conf->bluefs_max_log_runway,
            &log_file->fnode);
    log_writer = _create_writer(log_file);

    // initial txn
    // 对block_all中的信息加日志，对每块空闲快，都执行log_t.op_alloc_add(bdev, q.get_start(), q.get_len());，其会在_flush_and_sync_log中持久化到日志中。
    log_t.op_init();
    for (unsigned bdev = 0; bdev < MAX_BDEV; ++bdev) {
        interval_set<uint64_t>& p = block_all[bdev];
        if (p.empty())
            continue;
        for (interval_set<uint64_t>::iterator q = p.begin(); q != p.end(); ++q) {
            log_t.op_alloc_add(bdev, q.get_start(), q.get_len());
        }
    }

    _flush_and_sync_log(l);

    // write supers
    // 写bluefs的super block，其是调用_write_super实现的。super block包含version、block_size、日志文件的fnode信息等。_write_super会把super block写到block.db设备的4096偏移处。
    super.log_fnode = log_file->fnode;
    _write_super(BDEV_DB);
    flush_bdev();

    // clean up
    // 做一些清理工作，比如清除block_all，_close_writer。因为会在_mount函数中继续从日志中读取信息，来进行回放操作
    super = bluefs_super_t();
    _close_writer(log_writer);
    log_writer = NULL;
    block_all.clear();
    _stop_alloc();
    _shutdown_logger();

    return 0;
}

```

mount

```

int BlueFS::mount()
{ 
    int r = _open_super(); //读取superblock

    block_all.clear();
    block_all.resize(MAX_BDEV);
    _init_alloc(); //为每块磁盘创建StupidAllocator实例
    _init_logger(); //回放文件系统日志，日志项即为事务OP，针对每个事务进行回放，文件系统的dir_map/file_map就会被更新，将分配的空间、创建的文件等恢复到内存

    r = _replay(false, false); //对bluefs进行还原

    // init freelist //初始化freelist，针对file_map中的每个文件，将分配给文件的空间从空闲空间中移除
    for (auto& p : file_map) {
        for (auto& q : p.second->fnode.extents) {
            alloc[q.bdev]->init_rm_free(q.offset, q.length); //对于_replay回放得出的文件，将文件已经占用的内容从allocator中删除
        }
    }

    // set up the log for future writes //创建log_writer, log_writer为日志文件的句柄，用于向日志中追加日志项，该日志文件的fnode.ino为1
    log_writer = _create_writer(_get_file(1)); 
    log_writer->pos = log_writer->file->fnode.size;
}

```

### FreelistManager

BlueStore直接管理裸设备，需要自行管理空间的分配和释放。Stupid和Bitmap分配器的结果是保存在内存中的，分配结果的持久化是通过FreelistManager来做的。

FreelistManager主要是一些接口的抽象。

```
class FreelistManager {
public:
    CephContext* cct;
    FreelistManager(CephContext* cct) : cct(cct) {}
    virtual ~FreelistManager() {}

    static FreelistManager *create( CephContext* cct, string type, string prefix);

    static void setup_merge_operators(KeyValueDB *db);

    virtual int create(uint64_t size, uint64_t granularity, KeyValueDB::Transaction txn) = 0;
    virtual int expand(uint64_t new_size, KeyValueDB::Transaction txn) = 0;

    virtual int init(KeyValueDB *kvdb) = 0;
    virtual void shutdown() = 0;

    virtual void dump(KeyValueDB *kvdb) = 0;

    virtual void enumerate_reset() = 0;
    virtual bool enumerate_next(KeyValueDB *kvdb, uint64_t *offset, uint64_t *length) = 0;

    virtual void allocate(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn) = 0;
    virtual void release(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn) = 0;

    virtual uint64_t get_size() const = 0;
    virtual uint64_t get_alloc_units() const = 0;
    virtual uint64_t get_alloc_size() const = 0;
};

```

BitmapFreelistManager

FreelistManager默认为bitmap实现。FreelistManager将block按一定数量组成段，每个段对应一个k/v键值对，key为第一个block在磁盘物理地址空间的offset，value为段内每个block的状态，即由0/1组成的位图，1为空闲，0为使用，这样可以通过与1进行异或运算，将分配和回收空间两种操作统一起来。


```
class BitmapFreelistManager : public FreelistManager {
    std::string meta_prefix, bitmap_prefix; //rocksdb key前缀：meta_prefix为 B，bitmap_prefix为 b
    std::shared_ptr<KeyValueDB::MergeOperator> merge_op; //rocksdb的merge操作：按位异或(xor)
    ceph::mutex lock = ceph::make_mutex("BitmapFreelistManager::lock");

    uint64_t size;            ///< size of device (bytes) //设备总大小
    uint64_t bytes_per_block; ///< bytes per block (bdev_block_size)
    uint64_t blocks_per_key;  ///< blocks (bits) per key/value pair
    uint64_t bytes_per_key;   ///< bytes per key/value pair
    uint64_t blocks;          ///< size of device (blocks, size rounded up) //设备总block数

    uint64_t block_mask;  ///< mask to convert byte offset to block offset
    uint64_t key_mask;    ///< mask to convert offset to key offset

    bufferlist all_set_bl;
    //遍历rocksdb key相关的成员
    KeyValueDB::Iterator enumerate_p;
    uint64_t enumerate_offset; ///< logical offset; position
    bufferlist enumerate_bl;   ///< current key at enumerate_offset
    int enumerate_bl_pos;      ///< bit position in enumerate_bl
};
```

_open_fm



### Allocator

Allocator是一个内存版本的磁盘空间分配器，管理已分配空间列表。类Allocator主要是一些接口的抽象。

```
class Allocator {
public:
    explicit Allocator(const std::string& name);
    virtual ~Allocator();

    /*  
     * Allocate required number of blocks in n number of extents.
     * Min and Max number of extents are limited by:
     * a. alloc unit
     * b. max_alloc_size.
     * as no extent can be lesser than alloc_unit and greater than max_alloc size.
     * Apart from that extents can vary between these lower and higher limits according
     * to free block search algorithm and availability of contiguous space.
     */
    virtual int64_t allocate(uint64_t want_size, uint64_t alloc_unit,
            uint64_t max_alloc_size, int64_t hint,
            PExtentVector *extents) = 0;

    int64_t allocate(uint64_t want_size, uint64_t alloc_unit,
            int64_t hint, PExtentVector *extents) {
        return allocate(want_size, alloc_unit, want_size, hint, extents);
    }

    /* Bulk release. Implementations may override this method to handle the whole
     * set at once. This could save e.g. unnecessary mutex dance. */
    virtual void release(const interval_set<uint64_t>& release_set) = 0;
    void release(const PExtentVector& release_set);

    virtual void dump() = 0;
    virtual void dump(std::function<void(uint64_t offset, uint64_t length)> notify) = 0;

    virtual void init_add_free(uint64_t offset, uint64_t length) = 0;
    virtual void init_rm_free(uint64_t offset, uint64_t length) = 0;

    virtual uint64_t get_free() = 0;
    virtual double get_fragmentation(uint64_t alloc_unit)
    {
        return 0.0;
    }
    virtual double get_fragmentation_score();
    virtual void shutdown() = 0;
    static Allocator *create(CephContext* cct, string type, int64_t size,
            int64_t block_size, const std::string& name = "");
private:
    class SocketHook;
    SocketHook* asok_hook = nullptr;
};
```

BitMapAllocator

BitMapAllocator为默认分配器，基于BitMap磁盘分配策略。

```
class BitmapAllocator : public Allocator,
public AllocatorLevel02<AllocatorLevel01Loose> {
    CephContext* cct;

public:
    BitmapAllocator(CephContext* _cct, int64_t capacity, int64_t alloc_unit, const std::string& name);
    ~BitmapAllocator() override {}


    int64_t allocate(uint64_t want_size, uint64_t alloc_unit, uint64_t max_alloc_size, int64_t hint, PExtentVector *extents) override;

    void release(const interval_set<uint64_t>& release_set) override;

    uint64_t get_free() override {return get_available();}

    void dump() override;
    void dump(std::function<void(uint64_t offset, uint64_t length)> notify) override;
    double get_fragmentation(uint64_t) override {return _get_fragmentation();}

    void init_add_free(uint64_t offset, uint64_t length) override;
    void init_rm_free(uint64_t offset, uint64_t length) override;

    void shutdown() override;
};
```

### Transaction


### cnode

PG对应的磁盘数据结构称为bluestore_cnode_t(简称cnode)，目前仅包含一个字段，用于指示执行stable_mod时PG的掩码位数。

在BlueStore中，每种类型的数据结构一般都有磁盘和内存两种格式，习惯上前者采用全部小写、以下划线"_"作为分隔符并且固定以字母t结尾的命名方式，而后者采用首字母大写的命名方式。

```
struct bluestore_cnode_t {
    uint32_t bits;   // 指示对象(通过stable_mod)映射至PG时，其32位全精度哈希值(从最低位开始)有多少位是有效的

    explicit bluestore_cnode_t(int b=0) : bits(b) {}

    DENC(bluestore_cnode_t, v, p) { 
        DENC_START(1, 1, p);
        denc(v.bits, p);
        DENC_FINISH(p);
    }
    void dump(Formatter *f) const;
    static void generate_test_instances(list<bluestore_cnode_t*>& o);
};

```

### extent

extent为逻辑段，每个extent都可以写成形如{offset、length、data}这样的三元组形式。因磁盘碎片化，无法保证为每个逻辑上连续的段(extent)分配物理上也是连续的一段空间，即逻辑段与磁盘上的物理空间段应该是一对多的关系。即extent.data在设计上主要包含一些物理段的集合，该物理段对应磁盘上的一块独立存储空间，称为bluestore_pextent_t。

extent是对象内的基本数据管理单元，很多扩展功能，例如数据校验、数据压缩、对象间的数据共享等，都是基于extent实现的。

```
template <typename OFFS_TYPE, typename LEN_TYPE>
struct bluestore_interval_t
{
    static const uint64_t INVALID_OFFSET = ~0ull;

    OFFS_TYPE offset = 0; 
    LEN_TYPE length = 0; 

    bluestore_interval_t(){}
    bluestore_interval_t(uint64_t o, uint64_t l) : offset(o), length(l) {}

    bool is_valid() const {
        return offset != INVALID_OFFSET;
    }
    uint64_t end() const {
        return offset != INVALID_OFFSET ? offset + length : INVALID_OFFSET;
    }

    bool operator==(const bluestore_interval_t& other) const {
        return offset == other.offset && length == other.length;
    }

};

/// pextent: physical extent
struct bluestore_pextent_t : public bluestore_interval_t<uint64_t, uint32_t> 
{
    bluestore_pextent_t() {}
    bluestore_pextent_t(uint64_t o, uint64_t l) : bluestore_interval_t(o, l) {}
    bluestore_pextent_t(const bluestore_interval_t &ext) :
        bluestore_interval_t(ext.offset, ext.length) {}

    DENC(bluestore_pextent_t, v, p) { 
        denc_lba(v.offset, p);
        denc_varint_lowz(v.length, p);
    }

    void dump(Formatter *f) const;
    static void generate_test_instances(list<bluestore_pextent_t*>& ls);
};
WRITE_CLASS_DENC(bluestore_pextent_t)
```


### blob



```
/// blob: a piece of data on disk
struct bluestore_blob_t {
private:
    PExtentVector extents;              ///< raw data position on device //磁盘上的物理段集合，单个extent的类型为bluestore_pextent_t
    uint32_t logical_length = 0;        ///< original length of data stored in the blob
    uint32_t compressed_length = 0;     ///< compressed length if any

public:
    enum {
        LEGACY_FLAG_MUTABLE = 1,  ///< [legacy] blob can be overwritten or split
        FLAG_COMPRESSED = 2,      ///< blob is compressed
        FLAG_CSUM = 4,            ///< blob has checksums
        FLAG_HAS_UNUSED = 8,      ///< blob has unused map
        FLAG_SHARED = 16,         ///< blob is shared; see external SharedBlob
    };
    static string get_flags_string(unsigned flags);

    uint32_t flags = 0;                 ///< FLAG_*

    typedef uint16_t unused_t;
    unused_t unused = 0;     ///< portion that has never been written to (bitmap)

    uint8_t csum_type = Checksummer::CSUM_NONE;      ///< CSUM_*
    uint8_t csum_chunk_order = 0;       ///< csum block size is 1<<block_order bytes

    bufferptr csum_data;                ///< opaque vector of csum data
};
```

### BlueStore

```
class BlueStore : public ObjectStore,
    public BlueFSDeviceExpander,
    public md_config_obs_t {

    /// cached buffer
    struct Buffer {
        //BufferSpace管理的基本单元
    };

    /// map logical extent range (object) onto buffers
    struct BufferSpace {
        //作为连接Blob(用户数据)和Cache的桥梁
        mempool::bluestore_cache_other::map<uint32_t, std::unique_ptr<Buffer>> buffer_map; //查找表
    };

    /// in-memory shared blob state (incl cached buffers)
    struct SharedBlob {
    };

    /// a lookup table of SharedBlobs
    struct SharedBlobSet {
    };

    /// in-memory blob metadata and associated cached buffers (if any)
    struct Blob {
        //Blob是Bluestore里引入的处理块设备数据映射的中间层
    };

    /// a logical extent, pointing to (some portion of) a blob
    typedef boost::intrusive::set_base_hook<boost::intrusive::optimize_size<true> > ExtentBase; //making an alias to avoid build warnings
    struct Extent : public ExtentBase {
        //Extent是实现object的数据映射的关键数据结构,每个Extent都会映射到下一层的Blob上，Extent会依据 block_size 对齐，没写的地方填充全零.
        uint32_t logical_offset = 0;      ///< logical offset //对应Object的逻辑偏移, 从0开始编址
        uint32_t blob_offset = 0;         ///< blob offset //对应Blob上的偏移
        uint32_t length = 0;              ///< length //数据段长度, 最小：block_size，最大：max_blob_size
        BlobRef  blob;                    ///< the blob with our data //对应Blob的指针，负责将逻辑段内的数据映射至磁盘。
    };

    /// a sharded extent map, mapping offsets to lextents to blobs
    struct ExtentMap {
        Onode *onode; //指向Onode指针
        extent_map_t extent_map;        ///< map of Extents to Blobs //Extents到Blobs的map, Extent的set集合,有序
        blob_map_t spanning_blob_map;   ///< blobs that span shards //跨越shards的blobs
        typedef boost::intrusive_ptr<Onode> OnodeRef;
    };

    /// an in-memory object 任何RADOS里的一个Object都对应Bluestore里的一个Onode(内存结构)
    struct Onode {
        std::atomic_int nref;  ///< reference count
        Collection *c; //对应PG的上下文

        ghobject_t oid;

        /// key under PREFIX_OBJ where we are stored
        mempool::bluestore_cache_other::string key;

        boost::intrusive::list_member_hook<> lru_item;

        bluestore_onode_t onode;  ///< metadata stored as value in kv store //元数据信息
        bool exists;              ///< true if object logically exists

        ExtentMap extent_map; //包含多个extent，每个extent负责管理一个逻辑段内的数据并关联一个blob，通过ExtentMap来查询Object数据到底层的映射lextents到blobs。
  };

  /// a cache (shard) of onodes and buffers
  struct Cache {
  };

  /// simple LRU cache for onodes and buffers
  struct LRUCache : public Cache {
  };

  // 2Q cache for buffers, LRU for onodes
  struct TwoQCache : public Cache {
  };

  //建立Onode与其归属Collection之间的联系
  struct OnodeSpace {
  private:
      Cache *cache; //表明自身术语哪个Cache实例

      /// forward lookups
      mempool::bluestore_cache_other::unordered_map<ghobject_t,OnodeRef> onode_map; //查找表
  };

  struct Collection : public CollectionImpl {
      BlueStore *store;
      OpSequencerRef osr; //每一个pg有一个osr，在bluestore层面保证读写事务的顺序和并发性
      Cache *cache;       ///< our cache shard
      bluestore_cnode_t cnode; //pg的磁盘结构, 目前仅包婚一个字段bits，用于指示执行stable_mod时PG的掩码位数
  };

  // --------------------------------------------------------
  // members
private:
  BlueFS *bluefs = nullptr;
  ...
};
```

## BlueStore

### mkfs、mount

在osd的部署过程中会对BlueStore进行初始化，即OSD::mkfs()操作:
1. store->set_fsid()设置fsid，fsid为osd的uuid
2. store->mkfs()，初始化bluestore，BlueStore::mkfs()
3. store->mount()，挂载bluestore，BlueStore::_mount()
4. 读取bluestore的superblock信息，如果已存在superblock，则检查相关信息；如果不存在，则先设置cluster_fsid、osd_fsid、whoami、compat_features信息，然后通过事务创建superblock。
5. 写fsid文件

BlueStore的mkfs过程:

![Local Picture](/images/ceph/bluestore/bluestore_mkfs.jpg "mkfs")

BlueStore的mount过程:

![Local Picture](/images/ceph/bluestore/bluestore_mount.jpg "mount")


### read


### write

