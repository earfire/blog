---
title: "ceph读写流程"
date: 2020-12-24T09:07:18+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## 概述

OSD是RADOS集群的基本存储单元。

PG(placement group)从名字可理解为放置策略组，它是对象的集合，该集合里的所有对象都具有相同的放置策略：对象的副本都分布在相同的OSD列表上。一个对象只能属于一个PG，一个PG对应于放置在其上的OSD列表。一个OSD上可以分布多个PG。处理来自客户端的读写请求是PG的基本功能。

Pool是整个集群层面定义的一个逻辑的存储池，它规定了数据冗余的类型以及对应的副本分布策略。目前实现了两种pool类型：replicated类型和Erasure Code类型。一个pool由多个PG构成。

## 数据结构

对象有两个关键属性，分别用于保存对象的基本信息和快照信息，通常为OI(Object Info)和SS(Snap Set)属性。

### pg

```
struct pg_t {
  uint64_t m_pool; //所在的pool
  uint32_t m_seed; //pg的序号
};

struct spg_t {
  pg_t pgid;
  shard_id_t shard; //代表该PG所在的OSD在对应的OSD列表的序号。在Erasure Code模式下，该字段保存了每个分片的序号，该序号在EC的数据encode和decode过程很关键。对于副本模式，该字段没有意义。
};
```

### OSDMap

类OSDMap定义了Ceph整个集群的全局信息。它由Monitor实现管理，并以全量或者增量的方式向整个集群扩散。每个epoch对应的OSDMap都需要持久化保存在meta下对应对象的omap属性中。类OSDMap的内部类Incrementtal以增量的形式保存了OSDMap新增的信息。包含了四类信息：集群的信息、pool相关的信息、临时PG相关的信息、OSD的状态信息。

```
class OSDMap {
  class Incremental {
     //为了减小传输负担，通常情况下，除了第一张OSDMap以外，后续所有需要传递OSDMap的场景，都可以只传递前后两个OSDMap有差异的部分，这种增量形式的OSDMap称为Incremental。
  };
private:
  uuid_d fsid; //进群UUID
  epoch_t epoch;        // what epoch of the osd cluster descriptor is this //本OSDMap对应的版本号，总是单调递增
  utime_t created, modified; // epoch start time
  int32_t pool_max;     // the largest pool num, ever

  uint32_t flags;

  int num_osd;         // not saved; see calc_num_osds
  int num_up_osd;      // not saved; see calc_num_osds
  int num_in_osd;      // not saved; see calc_num_osds

  int32_t max_osd; //集群最大的OSD个数，从0开始编号，所以max_osd总是指向下一个尚未分配的osd编号。此外，为了避免不必要的数据迁移，实现OSD无缝替换，Monitor不会主动回收被删除的OSD编号，因此max_osd并不总是等于集群当前实际的OSD个数
  vector<uint32_t> osd_state;

  mempool::osdmap::map<int32_t,uint32_t> crush_node_flags; // crush node -> CEPH_OSD_* flags
  mempool::osdmap::map<int32_t,uint32_t> device_class_flags; // device class -> CEPH_OSD_* flags

  utime_t last_up_change, last_in_change;	

  std::shared_ptr<addrs_s> osd_addrs; //OSD地址列表（Monitor）需要收集每个OSD包括公共地址、集群地址、心跳地址在内的多个通信地址

  entity_addrvec_t _blank_addrvec;

  mempool::osdmap::vector<__u32>   osd_weight;   // 16.16 fixed point, 0x10000 = "in", 0 = "out"
  mempool::osdmap::vector<osd_info_t> osd_info;
  std::shared_ptr<PGTempMap> pg_temp;  // temp pg mapping (e.g. while we rebuild)
  std::shared_ptr< mempool::osdmap::map<pg_t,int32_t > > primary_temp;  // temp primary mapping (e.g. while we rebuild)
  std::shared_ptr< mempool::osdmap::vector<__u32> > osd_primary_affinity; ///< 16.16 fixed point, 0x10000 = baseline //亲和性列表，数值越大，对应的OSD在执行PG映射过程中被选为Primary的概率越大

  // remap (post-CRUSH, pre-up)
  mempool::osdmap::map<pg_t,mempool::osdmap::vector<int32_t>> pg_upmap; ///< remap pg
  mempool::osdmap::map<pg_t,mempool::osdmap::vector<pair<int32_t,int32_t>>> pg_upmap_items; ///< remap osds in up set

  mempool::osdmap::map<int64_t,pg_pool_t> pools; //集群已创建的存储池列表, pool的id -> 类pg_pool_t
  mempool::osdmap::map<int64_t,string> pool_name; //集群已创建的存储池名称列表, pool的id -> pool名字
  mempool::osdmap::map<string,map<string,string> > erasure_code_profiles; //pool的EC相关的信息
  mempool::osdmap::map<string,int64_t> name_pool; //pool名字 -> pool_id

  std::shared_ptr< mempool::osdmap::vector<uuid_d> > osd_uuid;
  mempool::osdmap::vector<osd_xinfo_t> osd_xinfo;

  mempool::osdmap::unordered_map<entity_addr_t,utime_t> blacklist;
};

```

### object_info_t和ObjectState

object_info_t是对象OI属性的磁盘结构，保存对象快照之外的元数据。ObjectState是object_info_t的内存版本，在其基础上增加了一个exists字段，用于指示对象逻辑上是否存在。

```
struct object_info_t {
  hobject_t soid; //对应的对象，如果soid.snap为CEPH_NOSNAP,说明为head对象；如果为CEPH_SNAPDIR,说明为snapdir对象；否则为克隆对象，此时siod.snap为克隆对象关联的最新快照序列号
  eversion_t version, prior_version; //上一次修改版本时PG生成的版本号；再上一次修改本对象时，PG生成的版本号，如果为空，表示上次修改动作为创建或克隆，即对象从无到有
  version_t user_version; //客户端用户可见的对象版本号
  osd_reqid_t last_reqid; //上一次客户端修改本对象时，由其生成的唯一请求标识

  uint64_t size; //对象的大小
  utime_t mtime;
  utime_t local_mtime; // local mtime
  
  flag_t flags; //用于指定对象当前是否包含数据校验和、是否使用了omap、是否包含omap校验和

  map<pair<uint64_t, entity_name_t>, watch_info_t> watchers; //watchers记录了客户端监控信息，一旦对象的状态发生了变化，需要通知客户端

  // opportunistic checksums; may or may not be present
  __u32 data_digest;  ///< data crc32c //数据校验和
  __u32 omap_digest;  ///< omap crc32c

  // alloc hint attribute
  uint64_t expected_object_size, expected_write_size; //由特定应用(客户端)下发的对象大小提示，写请求大小提示
  uint32_t alloc_hint_flags; //由特定应用(客户端)下发的对象特征及访问提示，例如连续读写，随机读写，仅进行追加写，是否建议进行压缩等等
};


struct ObjectState {
  object_info_t oi; 
  bool exists;         ///< the stored object exists (i.e., we will remember the object_info_t) //标记对象是否存在.因为object_info_t可能是从缓存的attrs[OI_ATTR]获取，并不确定对象是否存在

  ObjectState() : exists(false) {}

  ObjectState(const object_info_t &oi_, bool exists_)
    : oi(oi_), exists(exists_) {}
};

```

### SnapSet和SnapSetContext

SnapSet是对象SS属性的磁盘结构，保存对象快照及克隆信息。SnapSetContext是SnapSet的内存版本，主要增加了引用计数机制，便于SS属性在head对象与克隆对象之间共享。

```
struct SnapSet {
  snapid_t seq; //最新的快照序号
  vector<snapid_t> snaps;    // descending //所有的快照序号列表，降序
  vector<snapid_t> clones;   // ascending //对象关联的所有的clone对象,使用每个克隆对象关联的最新快照序号列表，升序
  map<snapid_t, interval_set<uint64_t> > clone_overlap;  // overlap w/ next newest //和上次clone对象之间的overlap的部分。重叠部分，用于在数据恢复阶段对象恢复的优化
  map<snapid_t, uint64_t> clone_size; //clone对象的size
  map<snapid_t, vector<snapid_t>> clone_snaps; // descending
};


struct SnapSetContext {
  hobject_t oid; //对象
  SnapSet snapset; //对象快照的相关记录
  int ref; //引用计数，SnapSetContext可能被同一个对象的多个相关对象(head对象、克隆对象、snapdir对象)关联
  bool registered : 1; //是否已经将SnapSet加入缓存
  bool exists : 1; //指示SnapSet是否存在

  explicit SnapSetContext(const hobject_t& o) :
    oid(o), ref(0), registered(false), exists(true) { } 
};
```

### SnapContext

包含的快照信息。

```
struct SnapContext {
  snapid_t seq;            // 'time' stamp //最新快照序列号
  vector<snapid_t> snaps;  // existent snaps, in descending order //当前存在的所有快照序号，降序排队, 亦即总有snaps[0] == seq
};
```

### ObjectContext

对象上下文保存了对象的OI和SS属性，此外，内部实现了一个属性缓存(主要用于缓存用于自定义属性对)和读写互斥锁机制，后者用于对来自客户端的请求进行保存。

```
struct ObjectContext { //对象在内存的一个管理类，保存了一个对象的上下文信息
  ObjectState obs; //对象状态，主要是object_info_t，描述了对象的状态信息 

  SnapSetContext *ssc;  // may be null  //SnapSet上下文，没有快照就空

  Context *destructor_callback; //析构函数

public:

  // any entity in obs.oi.watchers MUST be in either watchers or unconnected_watchers.
  map<pair<uint64_t, entity_name_t>, WatchRef> watchers;

  // attr cache
  map<string, bufferlist> attr_cache; //属性缓存

  struct RWState { //读写锁，用于对来自客户端的op进行保序，以保证数据一致性。
  }rwstate;
};
```

### Log

### OpContext

PG将所有来自客户端的请求和集群内部诸如执行数据恢复、Scrub等任务产生的请求都统称为Op。

引入OpContext的意义：单个op可能操作对个对象(指同一个head对象、克隆对象)，需要分别记录对应对象上下文的变化；如果op涉及写操作，那么会产生一条或者多条新的日志；如果op涉及异步操作，那么需要注册一
个或者多个回调函数；收集op相关的统计，例如读写次数、读写涉及的字节数等，并周期性地上传给Monitor，用于监控集群IOPS、带宽等。

```
struct OpContext {
    OpRequestRef op;
    osd_reqid_t reqid;
    vector<OSDOp> *ops;

    const ObjectState *obs; // Old objectstate //op执行之前，对象状态
    const SnapSet *snapset; // Old snapset //op执行之前，对象关联的SS属性

    ObjectState new_obs;  // resulting ObjectState //op执行之后，新的对象状态
    SnapSet new_snapset;  // resulting SnapSet (in case of a write) //op执行之后，新的SS属性
    //pg_stat_t new_stats;  // resulting Stats
    object_stat_sum_t delta_stats;

    bool modify;          // (force) modification (even if op_t is empty)
    bool user_modify;     // user-visible modification
    bool undirty;         // user explicitly un-dirtying this object
    bool cache_evict;     ///< true if this is a cache eviction
    bool ignore_cache;    ///< true if IGNORE_CACHE flag is set
    bool ignore_log_op_stats;  // don't log op stats
    bool update_log_only; ///< this is a write that returned an error - just record in pg log for dup detection

    // side effects
    list<pair<watch_info_t,bool> > watch_connects; ///< new watch + will_ping flag
    list<watch_disconnect_t> watch_disconnects; ///< old watch + send_discon
    list<notify_info_t> notifies;

    list<NotifyAck> notify_acks;

    uint64_t bytes_written, bytes_read;

    utime_t mtime;
    SnapContext snapc;           // writer snap context //快照上下文，每次收到op时，基于op或者PGPool更新
    eversion_t at_version;       // pg's current version pointer //如果op包含写操作，那么PG将为op生成一个PG内唯一的序列号。该序列号单调递增，用于后续(例如peering)对本次写操作进行追踪会回溯
    version_t user_at_version;   // pg's current user version pointer

    /// index of the current subop - only valid inside of do_osd_ops()
    int current_osd_subop_num;
    /// total number of subops processed in this context for cls_cxx_subop_version()
    int processed_subop_count = 0;

    PGTransactionUPtr op_t;
    vector<pg_log_entry_t> log; //op产生的所有日志集合。
    boost::optional<pg_hit_set_history_t> updated_hset_history;

    interval_set<uint64_t> modified_ranges; //如果op包含写操作，记录对象本次被改写的数据范围
    ObjectContextRef obc; //对象上下文
    ObjectContextRef clone_obc;    // if we created a clone //克隆对象上下文，仅在需要时加载
    ObjectContextRef head_obc;     // if we also update snapset (see trim_object)

    // FIXME: we may want to kill this msgr hint off at some point!
    boost::optional<int> data_off = boost::none;

    MOSDOpReply *reply;

    PrimaryLogPG *pg;

    int num_read;    ///< count read ops
    int num_write;   ///< count update ops

    mempool::osd_pglog::vector<pair<osd_reqid_t, version_t> > extra_reqids;
    mempool::osd_pglog::map<uint32_t, int> extra_reqid_return_codes;

    hobject_t new_temp_oid, discard_temp_oid;  ///< temp objects we should start/stop tracking

    //各类回调上下文，满足各自条件时执行
    //on_applied一般在所有副本的本地事务都写入日志盘后执行
    //on_committed一般在所有副本的本地事务都写入数据盘后执行
    //on_success与on_finish通常在事务已经完成(也就是on_committed已经执行)后执行
    //on_success一般可用于执行Watch/Notify相关的操作
    //on_finish通常用于删除OpContext
    list<std::function<void()>> on_applied;
    list<std::function<void()>> on_committed;
    list<std::function<void()>> on_finish;
    list<std::function<void()>> on_success;
    ...
};

```

### RepGather

RepGather，如果op包含写操作，通常情况下需要由Primary主导，在副本之间分发分布式写。RepGather用于(取代op)追踪该分布式写在副本之间的完成情况。

```
class RepGather {
    hobject_t hoid; //op管理的对象标识
    OpRequestRef op;
    xlist<RepGather*>::item queue_item; //负责将RepGather挂入PG全局的repop_queue
    int nref;

    eversion_t v;
    int r = 0;

    ceph_tid_t rep_tid;

    bool rep_aborted; //RepGather被异常终止，例如PG收到新的OSDMap并且需要切换至一个新的Interval
    bool all_committed; //所有副本已经将本次事务写入了磁盘

    utime_t   start;

    eversion_t  pg_local_last_complete;

    ObcLockManager lock_manager;

    //同OpContext中的
    list<std::function<void()>> on_committed;
    list<std::function<void()>> on_success;
    list<std::function<void()>> on_finish;
    ...
};
```

## 读写流程

读写流程大致分为以下几个阶段：
1. 客户端基于对象标识中的32位哈希值，通过stable_mod找到存储池中承载该对象的PGID，然后使用该PGID作为CRUSH输入，找到对应PG当前Primary所在的OSD并发送读写请求。
2. OSD收到客户端发送的读写请求，将其封装为一个op，并基于其携带的PGID将其转发至对应的PG。
3. PG收到op后，完成一些列检查，所有条件均满足后，开始真正执行op。
4. 如果op只包含读操作，那么直接执行同步读(对应多副本)或者异步读(对应纠删码)，等待读操作完成后由Primary向客户端应答。
5. 如果op包含写操作，则由Primary基于op生成一个针对原始对象操作的PG事务，然后将其提交至PGBackend，由后者按照备份策略转化为每个副本真正需要执行的本地事务，并进行分发。当Primary收到所有副本的写入完成应答之后，对应op执行完成，由Primary向客户端回应写入完成。

### 对象寻址

对象寻址参考[client写操作部分](http://earfire.me/post/ceph/client/#%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%93%8D%E4%BD%9C)。

### 消息接收与分发

读写请求都是从OSD::ms_fast_dispatch开始，它是接收读写消息message的入口。

```
void OSD::ms_fast_dispatch(Message *m)
{
  FUNCTRACE(cct);
  if (service.is_stopping()) {
    m->put();
    return;
  }

  // peering event?
  switch (m->get_type()) {
  case CEPH_MSG_PING:
    dout(10) << "ping from " << m->get_source() << dendl;
    m->put();
    return;
  case MSG_MON_COMMAND:
    handle_command(static_cast<MMonCommand*>(m));
    return;
  case ...
  }

  OpRequestRef op = op_tracker.create_request<OpRequest, Message*>(m); //把Message消息转换为OpRequest类型

  ...

  if (m->get_connection()->has_features(CEPH_FEATUREMASK_RESEND_ON_SPLIT) ||
      m->get_type() != CEPH_MSG_OSD_OP) {
    // queue it directly //直接enqueue_op处理
    enqueue_op(
      static_cast<MOSDFastDispatchOp*>(m)->get_spg(),
      std::move(op),
      static_cast<MOSDFastDispatchOp*>(m)->get_map_epoch());
  } else {
    // legacy client, and this is an MOSDOp (the *only* fast dispatch
    // message that didn't have an explicit spg_t); we need to map
    // them to an spg_t while preserving delivery order.
    auto priv = m->get_connection()->get_priv();
    if (auto session = static_cast<Session*>(priv.get()); session) { //获取session,OSD会为每个客户端创建一个独立的会话上下文
      std::lock_guard l{session->session_dispatch_lock};
      op->get();
      session->waiting_on_map.push_back(*op); //把请求加入到waiting_on_map列表里
      OSDMapRef nextmap = service.get_nextmap_reserved(); //获取nextmap，也就是最新的osdmap
      dispatch_session_waiting(session, nextmap); //该函数循环处理请求
      service.release_map(nextmap);
    }
  }
}

void OSD::dispatch_session_waiting(SessionRef session, OSDMapRef osdmap)
{
  ceph_assert(session->session_dispatch_lock.is_locked());

  auto i = session->waiting_on_map.begin();
  while (i != session->waiting_on_map.end()) {
    OpRequestRef op = &(*i);
    ceph_assert(ms_can_fast_dispatch(op->get_req()));
    const MOSDFastDispatchOp *m = static_cast<const MOSDFastDispatchOp*>(
      op->get_req());
    if (m->get_min_epoch() > osdmap->get_epoch()) { //osdmap版本不对应
      break;
    }
    session->waiting_on_map.erase(i++);
    op->put();

    spg_t pgid; 
    if (m->get_type() == CEPH_MSG_OSD_OP) {
      pg_t actual_pgid = osdmap->raw_pg_to_pg(
    static_cast<const MOSDOp*>(m)->get_pg());
      if (!osdmap->get_primary_shard(actual_pgid, &pgid)) { //获取pgid，该PG的主OSD
    continue;
      }     
    } else {
      pgid = m->get_spg();
    }
    enqueue_op(pgid, std::move(op), m->get_map_epoch()); //进行处理
  }

  if (session->waiting_on_map.empty()) {
    clear_session_waiting_on_map(session);
  } else {
    register_session_waiting_on_map(session);
  }
}

void OSD::enqueue_op(spg_t pg, OpRequestRef&& op, epoch_t epoch)
{
  const utime_t stamp = op->get_req()->get_recv_stamp();
  const utime_t latency = ceph_clock_now() - stamp;
  const unsigned priority = op->get_req()->get_priority();
  const int cost = op->get_req()->get_cost();
  const uint64_t owner = op->get_req()->get_source().num();

  op->osd_trace.event("enqueue op");
  op->osd_trace.keyval("priority", priority);
  op->osd_trace.keyval("cost", cost);
  op->mark_queued_for_pg();
  logger->tinc(l_osd_op_before_queue_op_lat, latency);
  op_shardedwq.queue( //入队
    OpQueueItem(
      unique_ptr<OpQueueItem::OpQueueable>(new PGOpItem(pg, std::move(op))),
      cost, priority, stamp, owner, epoch));
}
```

### 消息出队处理

```
void OSD::dequeue_op(
  PGRef pg, OpRequestRef op,
  ThreadPool::TPHandle &handle)
{ 
  FUNCTRACE(cct);
  OID_EVENT_TRACE_WITH_MSG(op->get_req(), "DEQUEUE_OP_BEGIN", false);
  
  utime_t now = ceph_clock_now();
  op->set_dequeued_time(now);
  utime_t latency = now - op->get_req()->get_recv_stamp();
  
  logger->tinc(l_osd_op_before_dequeue_op_lat, latency);
  
  auto priv = op->get_req()->get_connection()->get_priv();
  if (auto session = static_cast<Session *>(priv.get()); session) {
    maybe_share_map(session, op, pg->get_osdmap()); //调用函数进行osdmap的更新
  }
  
  if (pg->is_deleting()) //正在删除，直接返回
    return;
  
  op->mark_reached_pg();
  op->osd_trace.event("dequeue_op");

  pg->do_request(op, handle); //调用pg的do_request处理, PrimaryLogPG::do_request
  
  // finish
  dout(10) << "dequeue_op " << op << " finish" << dendl;
  OID_EVENT_TRACE_WITH_MSG(op->get_req(), "DEQUEUE_OP_END", false);
}
```

### do_request

主要进行PG级别的检查，处理流程：

![Local Picture](/images/ceph/pg/do_request.jpg "do_request")

### do_op

主要进行对象级别的检查和一些上下文的准备工作。处理流程：

![Local Picture](/images/ceph/pg/do_op.jpg "do_op")

### execute_ctx

![Local Picture](/images/ceph/pg/execute_ctx.jpg "execute_ctx")


prerare_transaction操作主要分为三个阶段：
1. 通过do_osd_ops生成原始op对应的PG事务。
2. 如果op针对head对象操作通过make_writeable检查是否需要预先执行克隆操作。
3. 通过finish_ctx生成操作原始对象的日志，并更新对象的OI和SS属性。


do_osd_ops:

对于读操作，在do_osd_ops()->do_read()中，如果是同步读直接调用pgbackend->objects_read_sync进行读操作。如果是异步读，先将op添加到pending_async_reads中，然后在execute_ctx()检测pending_async_reads队列时执行读操作。

写操作步骤如下：

![Local Picture](/images/ceph/pg/do_osd_ops.jpg "do_osd_ops")


make_writeable:

如果op针对head对象操作，通过make_writeable检查是否需要预先执行克隆操作。

![Local Picture](/images/ceph/pg/make_writeable.jpg "make_writeable")

finish_ctx:

PG事务的内存版本已经准备完毕，此时可以生成原始对象的操作日志，并更新对象上下文中OI与SS属性。





