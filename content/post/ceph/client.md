---
title: "ceph客户端"
date: 2020-04-16T18:21:02+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## 简介

客户端是系统对外提供的功能接口，上层应用通过它来访问Ceph存储系统，包括Librados和Osdc两个模块，通过它们可直接访问RADOS对象存储系统。

相关的数据结构如下，

| type     		 | position               |   
| ---------------|:-----------------------|
| RadosClient    | librados/RadosClient.h |
| IoCtxImpl      | librados/IoCtxImpl.h   |
| Objecter       | osdc/Objecter.h        |
| ObjectOperation| osdc/Objecter.h        |
| op_target_t    | osdc/Objecter.h        |
| Op             | osdc/Objecter.h        |
| Striper        | osdc/Striper.h         |


## Librados

librados是RADOS对象存储系统的访问的接口库，它提供了pool的创建、删除、对象的创建、删除、读写等基本操作接口。

在最上层是类RadosClient，它是Librados的核心管理类，处理整个RADOS系统层面以及pool层面的管理。类IoctxImpl实现单个pool层的对象读写等操作。

### RadosClient

```
class librados::RadosClient : public Dispatcher
{
    std::unique_ptr<CephContext,
        std::function<void(CephContext*)> > cct_deleter;

    public:
    using Dispatcher::cct;
    const ConfigProxy& conf;
    private:
    enum {
        DISCONNECTED,
        CONNECTING,
        CONNECTED,
    } state; //和Monitor的网络连接状态

    MonClient monclient; //Monitor客户端
    MgrClient mgrclient; //Manager客户端
    Messenger *messenger; //消息网络接口

    uint64_t instance_id; //rados客户端实例的id

    bool _dispatch(Message *m);
    bool ms_dispatch(Message *m) override;

    bool ms_get_authorizer(int dest_type, AuthAuthorizer **authorizer) override;
    void ms_handle_connect(Connection *con) override;
    bool ms_handle_reset(Connection *con) override;
    void ms_handle_remote_reset(Connection *con) override;
    bool ms_handle_refused(Connection *con) override;

    Objecter *objecter; //对象指针

    Mutex lock;
    Cond cond;
    SafeTimer timer; //定时器
    int refcnt; //引用计数

public:
    Finisher finisher; //用于执行回调函数的finisher类

    int connect(); //初始化函数，从配置文件检测是否有初始的Monitor的地址信息，创建网络通信模块messenger，并设置Policy，创建objecter，初始化monclient、mgrclient、Timer、Finisher。
    int pool_create(string& name, int16_t crush_rule=-1); //pool的同步创建，其实现过程为调用Objecter::create_pool函数，构造PoolOp操作，通过Monitor的客户端monc发送请求给Monitor创建一个pool，并同步等待请求的返回。
    int pool_create_async(string& name, PoolAsyncCompletionImpl *c, nt16_t crush_rule=-1); //pool的异步创建，与同步方式的区别在于注册了回调函数，当创建成功后，执行回调函数通知完成。
    int pool_delete(const char *name); //pool的同步删除
    int pool_delete_async(const char *name, PoolAsyncCompletionImpl *c); //pool的异步删除
    int64_t lookup_pool(const char *name); //用于查找pool
    int pool_list(std::list<std::pair<int64_t, string> >& ls); //用于列出所有的pool
    int get_pool_stats(std::list<string>& ls, map<string,::pool_stat_t> *result, bool *per_pool); //用于获取pool的统计信息
    int get_fs_stats(ceph_statfs& result); //用于获取系统的统计信息

    //命令处理。
    //函数mon_command处理Monitor相关的命令，它调用函数monclient.start_mon_command把命令发送给Monitor处理。
    //函数osd_command处理OSD相关的命令，它调用函数objecter->osd_command把命令发送给OSD处理。
    //函数pg_command处理PG相关的命令，它调用函数objecter->pg_command把命令发送给该PG的主OSD处理。
    int mon_command(const vector<string>& cmd, const bufferlist &inbl,
            bufferlist *outbl, string *outs);
    void mon_command_async(const vector<string>& cmd, const bufferlist &inbl,
            bufferlist *outbl, string *outs, Context *on_finish);
    int mon_command(int rank,
            const vector<string>& cmd, const bufferlist &inbl,
            bufferlist *outbl, string *outs);
    int mon_command(string name,
            const vector<string>& cmd, const bufferlist &inbl,
            bufferlist *outbl, string *outs);
    int mgr_command(const vector<string>& cmd, const bufferlist &inbl,
            bufferlist *outbl, string *outs);
    int osd_command(int osd, vector<string>& cmd, const bufferlist& inbl,
            bufferlist *poutbl, string *prs);
    int pg_command(pg_t pgid, vector<string>& cmd, const bufferlist& inbl,
            bufferlist *poutbl, string *prs);
    //创建一个pool相关的上下文信息的IoCtxImlp对象
    int create_ioctx(const char *name, IoCtxImpl **io);
    int create_ioctx(int64_t, IoCtxImpl **io);

};
```

### IoCtxImpl

```
struct librados::IoCtxImpl {
    std::atomic<uint64_t> ref_cnt = { 0 };
    RadosClient *client;
    int64_t poolid;
    snapid_t snap_seq; //一般也为快照的id(snap id)，当打开一个image时，如果打开的是一个卷的快照，那么snap_seq的值就是该snap对应的快照序号。否则snap_seq就为CEPH_NOSNAP(-2)，来表示操作的不是卷的快照，而是卷本身
    ::SnapContext snapc;
    uint64_t assert_ver;
    version_t last_objver;
    uint32_t notify_timeout;
    object_locator_t oloc;

    Mutex aio_write_list_lock;
    ceph_tid_t aio_write_seq;
    Cond aio_write_cond;
    xlist<AioCompletionImpl*> aio_write_list;
    map<ceph_tid_t, std::list<AioCompletionImpl*> > aio_write_waiters;

    Objecter *objecter;
    ...
};

类IoCtxImpl是pool相关的上下文信息，一个pool对应一个IoCtxImpl对象，可以在该pool里创建、删除对象，完成对象数据读写等各种操作，包括同步和异步的实现。其处理过程比较简单，过程类似:
1.把请求封装成ObjectOperation类。
2.然后再添加pool的地址信息，封装成Obejcter::Op对象。
3.调用函数objeter->op_submit发送给相应的OSD，如果是同步操作，就等待操作完成。如果是异步操作，就不用等待，直接返回，当操作完成后，调用相应的回调函数通知。
```



## OSDC

OSDC模块实现了请求的封装和通过网络模块发送请求的逻辑，其核心类Objecter完成对象的地址计算、消息的发送等工作。

### ObjectOperation

类ObjectOperation用于操作相关的参数统一封装在该类里，该类可以一次封装多个对象的操作：

```
struct ObjectOperation {
    vector<OSDOp> ops; //多个操作 
    int flags; //操作的标志
    int priority; //优先级

    vector<bufferlist*> out_bl; //每个操作对应的输出缓存区队列
    vector<Context*> out_handler; //每个操作对应的回调函数队列
    vector<int*> out_rval; //每个操作对应的操作结果队列
    ...
};

类OSDOp封装对象的一个操作。结构体ceph_osd_op封装一个操作的操作码和相关的输入输出参数：
struct OSDOp {
    ceph_osd_op op; //具体操作数据的封装
    sobject_t soid; //src oid，并不是op操作的对象，而是源操作对象

    bufferlist indata, outdata; //操作的输入输出data
    errorcode32_t rval; //操作返回值
    ...
};
```

### op_target

op_target封装了对象所在的PG，以及PG对象的OSD列表等地址信息。

```
struct op_target_t {
    int flags = 0;

    epoch_t epoch = 0;  ///< latest epoch we calculated the mapping

    object_t base_oid; //读取的对象
    object_locator_t base_oloc; //对象的pool信息
    object_t target_oid; //最终读取的目标对象
    object_locator_t target_oloc; //最终目标对象的pool信息.   //这里由于Cache Tier的存在，导致产生最终读取的目标和pool的不同

    ///< true if we are directed at base_pgid, not base_oid
    bool precalc_pgid = false;

    ///< true if we have ever mapped to a valid pool
    bool pool_ever_existed = false;

    ///< explcit pg target, if any
    pg_t base_pgid;

    pg_t pgid; ///< last (raw) pg we mapped to
    spg_t actual_pgid; ///< last (actual) spg_t we mapped to
    unsigned pg_num = 0; ///< last pg_num we mapped to
    unsigned pg_num_mask = 0; ///< last pg_num_mask we mapped to
    unsigned pg_num_pending = 0; ///< last pg_num we mapped to
    vector<int> up; ///< set of up osds for last pg we mapped to
    vector<int> acting; ///< set of acting osds for last pg we mapped to
    int up_primary = -1; ///< last up_primary we mapped to
    int acting_primary = -1;  ///< last acting_primary we mapped to
    int size = -1; ///< the size of the pool when were were last mapped
    int min_size = -1; ///< the min size of the pool when were were last mapped
    bool sort_bitwise = false; ///< whether the hobject_t sort order is bitwise
    bool recovery_deletes = false; ///< whether the deletes are performed during recovery instead of peering

    bool used_replica = false;
    bool paused = false;

    int osd = -1;      ///< the final target osd, or -1
    ...
};
```

### Op

Op封装了完成一个操作的相关的上下文信息，包括target地址信息、链接信息等。

```
struct Op : public RefCountedObject {
    OSDSession *session;
    int incarnation;

    op_target_t target; //target地址信息

    ConnectionRef con;  // for rx buffer only
    uint64_t features;  // explicitly specified op features

    vector<OSDOp> ops; //对应多个操作的封装

    snapid_t snapid; //快照的id
    SnapContext snapc; //pool层级的快照信息
    ceph::real_time mtime;

    bufferlist *outbl; //输出的bufferlist
    vector<bufferlist*> out_bl; //每个操作对应的bufferlist
    vector<Context*> out_handler; //每个操作对应的回调函数
    vector<int*> out_rval; //每个操作对应的输出结果

    int priority;
    Context *onfinish;
    uint64_t ontimeout;
    ...
};
```

## 客户端操作

### 写操作消息封装

IoCtxImpl的write函数完成具体写操作。

```
int librados::IoCtxImpl::write(const object_t& oid, bufferlist& bl,
                   size_t len, uint64_t off) 
{
  if (len > UINT_MAX/2)
    return -E2BIG;
  ::ObjectOperation op; //创建ObjectOperation对象，封装写操作。
  prepare_assert_ops(&op);
  bufferlist mybl;
  mybl.substr_of(bl, 0, len);
  op.write(off, mybl);
  return operate(oid, &op, NULL); //完成处理
}


int librados::IoCtxImpl::operate(const object_t& oid, ::ObjectOperation *o,
                 ceph::real_time *pmtime, int flags)
{
  ...

  Context *oncommit = new C_SafeCond(&mylock, &cond, &done, &r);

  int op = o->ops[0].op.op;
  Objecter::Op *objecter_op = objecter->prepare_mutate_op(oid, oloc, //把ObjectOperation封装成Op类，添加object_locator_t相关的pool信息
                              *o, snapc, ut, flags,
                              oncommit, &ver);
  objecter->op_submit(objecter_op); //把消息发出去

  ...

}
```


### 发送数据op_submit

Objecter::op_submit会调用_op_submit_with_budget，函数_op_submit_with_budget用来处理Throttle相关的流量限制，然后调用_op_submit函数。

函数_op_submit完成了关键的地址寻址和发送工作。

```
void Objecter::_op_submit(Op *op, shunique_lock& sul, ceph_tid_t *ptid)
{
  ...

  bool check_for_latest_map = _calc_target(&op->target, nullptr) //计算对象的目标OSD
    == RECALC_OP_TARGET_POOL_DNE;

  // Try to get a session, including a retry if we need to take write lock
  int r = _get_session(op->target.osd, &s, sul); //获取目标OSD的链接

  ...

  if (need_send) {
    _send_op(op); //发送
  }
}
```

### 对象寻址_calc_target

_calc_target完成对象到osd的寻址过程。

```
int Objecter::_calc_target(op_target_t *t, Connection *con, bool any_change)
{
  t->epoch = osdmap->get_epoch();
  const pg_pool_t *pi = osdmap->get_pg_pool(t->base_oloc.pool); //获取pg_pool_t对象
  
  //检查如果强制重发，force_resend设置为true
  bool force_resend = false;
  if (osdmap->get_epoch() == pi->last_force_op_resend) {
    if (t->last_force_resend < pi->last_force_op_resend) {
      t->last_force_resend = pi->last_force_op_resend;
      force_resend = true;
    } else if (t->last_force_resend == 0) {
      force_resend = true;
    }
  }

  // apply tiering
  // 检查cache tier，如果是读操作，并且有读缓存，就设置t->target_oloc.pool为该pool的read_tier值。
  // 如果是写操作，并且有写缓存，就设置t->target_oloc.pool为该pool的write_tier值。
  t->target_oid = t->base_oid;
  t->target_oloc = t->base_oloc;
  if ((t->flags & CEPH_OSD_FLAG_IGNORE_OVERLAY) == 0) {
    if (is_read && pi->has_read_tier())
      t->target_oloc.pool = pi->read_tier;
    if (is_write && pi->has_write_tier())
      t->target_oloc.pool = pi->write_tier;
    pi = osdmap->get_pg_pool(t->target_oloc.pool);
    if (!pi) {
      t->osd = -1;
      return RECALC_OP_TARGET_POOL_DNE;
    }
  }

  int ret = osdmap->object_locator_to_pg(t->target_oid, t->target_oloc, //获取目标对象所在的PG
                       pgid);

  vector<int> up, acting;
  osdmap->pg_to_up_acting_osds(pgid, &up, &up_primary, //通过CRUSH算法，获取该PG对应的OSD列表
                   &acting, &acting_primary);

  if (acting_primary == -1) {
      t->osd = -1;
  } else {
      int osd;
      bool read = is_read && !is_write;
      if (read && (t->flags & CEPH_OSD_FLAG_BALANCE_READS)) { //读操作，如果设置了CEPH_OSD_FLAG_BALANCE_READS标志，随机选择一个副本读取
        int p = rand() % acting.size();
        if (p)
          t->used_replica = true;
        osd = acting[p];
        ldout(cct, 10) << " chose random osd." << osd << " of " << acting
                       << dendl;
      } else if (read && (t->flags & CEPH_OSD_FLAG_LOCALIZE_READS) && //读操作，如果设置了CEPH_OSD_FLAG_LOCALIZE_READS标志，尽可能从本地副本读取
                 acting.size() > 1) {

      } else { //写操作，target的OSD就设置为主OSD
        osd = acting_primary;
      }
      t->osd = osd;
  }

  ...
}
```
