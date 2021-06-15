---
title: "ceph数据恢复"
date: 2021-02-27T09:31:37+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## 概述

当PG完成Peering过程后，处于Active状态的PG就可以对外提供服务了。如果该PG的各个副本有不一致的情况，就需要进行恢复。Ceph的恢复过程有两种：Recovery和Backfill。

## Recovery

Peering完成之后，如果Primary检测到ActingBackfill中的任意一个副本(包括自身)还存在降级对象，那么可以通过日志来执行修复，这个过程称为Recovery。

### 资源预留

为了防止集群中大量PG同时执行Recovery从而严重影响正常业务，需要对Recovery进行约束。
例如：osd_max_backfills，单个OSD允许同时执行Recovery或者Backfill的PG个数。
osd_recovery_max_active，每个OSD允许并发进行Recovery/Backfill的对象数


### 过程

数据恢复的依据是在Peering过程产生的如下信息：
- 主副本上的缺失对象信息保存在pg_log.missing结构中。
- 各从副本上的缺失对象信息保存在peer_missing中的pg_missing_t结构中。
- 缺失对象的位置信息保存在missing_loc中。

Primary总是先完成自我修复，即先执行Pull操作，再执行Push操作修复其他副本上的对象。

OSD::do_recovery()执行实际的数据修复操作：

```
void OSD::do_recovery( //数据修复
  PG *pg, epoch_t queued, uint64_t reserved_pushes,
  ThreadPool::TPHandle &handle)
{
  uint64_t started = 0;
  
  float recovery_sleep = get_osd_recovery_sleep(); //做一次修复后的休眠时间，如果设置了该事件，每次线程开始先休眠相应的时间长度。

  ...

   {
    if (pg->pg_has_reset_since(queued)) { //检查PG的状态，如果该PG处于正在删除状态，或者既不处于peered状态，也不是主OSD，则直接退出
      goto out;
    }

    bool do_unfound = pg->start_recovery_ops(reserved_pushes, handle, &started); //修复

    if (do_unfound) {
      PG::RecoveryCtx rctx = create_context();
      rctx.handle = &handle;
      pg->find_unfound(queued, &rctx);
      dispatch_context(rctx, pg, pg->get_osdmap());
    }
  }

 out:
  service.release_reserved_pushes(reserved_pushes);
}

```

PrimaryLogPG用来处理PG相关的修复操作。函数start_recovery_ops调用recover_primary和recover_replicas来修复该PG上对象的主副本和从副本。修复完成后，如果仍需要Backfill过程，则抛出相关事件触发PG状态机，开始Backfill过程。

```

bool PrimaryLogPG::start_recovery_ops(
  uint64_t max,
  ThreadPool::TPHandle &handle,
  uint64_t *ops_started)
{
    uint64_t& started = *ops_started;
    started = 0;
    bool work_in_progress = false;
    bool recovery_started = false;
    recovery_queued = false;

    if (!state_test(PG_STATE_RECOVERING) &&
            !state_test(PG_STATE_BACKFILLING)) {
        /* TODO: I think this case is broken and will make do_recovery()
         * unhappy since we're returning false */
        dout(10) << "recovery raced and were queued twice, ignoring!" << dendl;
        return have_unfound();
    }

    const auto &missing = pg_log.get_missing(); //获取missing对象

    uint64_t num_unfound = get_num_unfound(); //为该PG上缺失的对象却没有找到该对象其他正确副本所在的OSD

    if (!missing.have_missing()) {
        info.last_complete = info.last_update;
    }

    if (!missing.have_missing() || // Primary does not have missing //主OSD没有缺失对象
            all_missing_unfound()) { // or all of the missing objects are unfound.
        // Recover the replicas.
        started = recover_replicas(max, handle, &recovery_started); //先恢复replicas
    }

    if (!started) {
        // We still have missing objects that we should grab from replicas.
        started += recover_primary(max, handle); //恢复主OSD
    }
    if (!started && num_unfound != get_num_unfound()) { //num_unfound有变化
        // second chance to recovery replicas
        started = recover_replicas(max, handle, &recovery_started);
    }

    if (started || recovery_started) //started:已经启动修复的对象数量
        work_in_progress = true;

    bool deferred_backfill = false;
    if (recovering.empty() && //空，没有正在进行Recovery操作的对象
            state_test(PG_STATE_BACKFILLING) && //状态
            !backfill_targets.empty() && started < max &&
            missing.num_missing() == 0 &&
            waiting_on_backfill.empty()) {
        if (get_osdmap()->test_flag(CEPH_OSDMAP_NOBACKFILL)) { //如果标志CEPH_OSDMAP_NOBACKFILL设置了
            deferred_backfill = true; //推迟backfill过程
        } else if (get_osdmap()->test_flag(CEPH_OSDMAP_NOREBALANCE) &&
                !is_degraded())  {
            deferred_backfill = true;
        } else if (!backfill_reserved) { //如果没有设置
            if (!backfill_reserving) {
                backfill_reserving = true;
                queue_peering_event(
                  PGPeeringEventRef(
                    std::make_shared<PGPeeringEvent>(
                      get_osdmap_epoch(),
                      get_osdmap_epoch(),
                      RequestBackfill()))); //抛出RequestBackfill事件给状态机，启动Backfill过程
            }
            deferred_backfill = true;
        } else {
            started += recover_backfill(max - started, handle, &work_in_progress); //开始Backfill过程
        }
    }

    osd->logger->inc(l_osd_rop, started);

    if (!recovering.empty() ||
            work_in_progress || recovery_ops_active > 0 || deferred_backfill)
        return !work_in_progress && have_unfound();

    int unfound = get_num_unfound();
    if (unfound) {
        dout(10) << " still have " << unfound << " unfound" << dendl;
        return true;
    }

    if (missing.num_missing() > 0) {
        // this shouldn't happen!
        osd->clog->error() << info.pgid << " Unexpected Error: recovery ending with "
            << missing.num_missing() << ": " << missing.get_items();
        return false;
    }

    if (needs_recovery()) {
        // this shouldn't happen!
        // We already checked num_missing() so we must have missing replicas
        osd->clog->error() << info.pgid
            << " Unexpected Error: recovery ending with missing replicas";
        return false;
    }

    if (state_test(PG_STATE_RECOVERING)) { //PG如果处于PG_STATE_RECOVERING状态
        state_clear(PG_STATE_RECOVERING);
        state_clear(PG_STATE_FORCED_RECOVERY);

        if (needs_backfill()) { //如果需要backfill过程,就向PG的状态机发送RequestBackfill事件
            dout(10) << "recovery done, queuing backfill" << dendl;
            queue_peering_event(
                    PGPeeringEventRef(
                        std::make_shared<PGPeeringEvent>(
                            get_osdmap_epoch(),
                            get_osdmap_epoch(),
                            RequestBackfill())));
        } else { //如果不需要，就抛出AllReplicasRecovered事件
            dout(10) << "recovery done, no backfill" << dendl;
            eio_errors_to_process = false;
            state_clear(PG_STATE_FORCED_BACKFILL);
            queue_peering_event(
                    PGPeeringEventRef(
                        std::make_shared<PGPeeringEvent>(
                            get_osdmap_epoch(),
                            get_osdmap_epoch(),
                            AllReplicasRecovered())));
        }
    } else { // backfilling //PG_STATE_BACKFILLING状态
        state_clear(PG_STATE_BACKFILLING);
        state_clear(PG_STATE_FORCED_BACKFILL);
        state_clear(PG_STATE_FORCED_RECOVERY);
        dout(10) << "recovery done, backfill done" << dendl;
        eio_errors_to_process = false;
        queue_peering_event(
                PGPeeringEventRef(
                    std::make_shared<PGPeeringEvent>(
                        get_osdmap_epoch(),
                        get_osdmap_epoch(),
                        Backfilled()))); //抛出Backfilled事件
    }

    return false;
}

```


## Backfill

### 资源预留

同样，Backfill也需先进行资源预留。资源预留成功之后，PG开始正式执行Backfill。

### 过程

数据结构BackfillInterval用来记录每个peer上的Backfill过程。

```
struct BackfillInterval {
    // info about a backfill interval on a peer
    eversion_t version; /// version at which the scan occurred
    map<hobject_t,eversion_t> objects; //本次扫描到的对象及其实时版本号
    hobject_t begin; //本次扫描的起点
    hobject_t end; //本次扫描的结束
};
```

函数recover_backfill作为Backfill过程的核心函数，控制整个Backfill修复进程。

```
uint64_t PrimaryLogPG::recover_backfill(uint64_t max,
    ThreadPool::TPHandle &handle, bool *work_started)
{
    //Primary通过backfill_info对Backfill的整体进度进行跟踪，peer_backfile_info则记录了每个Backfill副本的实时进度。
    // update our local interval to cope with recent changes
    backfill_info.begin = last_backfill_started;
    update_range(&backfill_info, handle);
    for (set<pg_shard_t>::iterator i = backfill_targets.begin();
            i != backfill_targets.end();
            ++i) {
        peer_backfill_info[*i].trim_to(
                std::max(peer_info[*i].last_backfill, last_backfill_started));
    }
    backfill_info.trim_to(last_backfill_started);

    while (ops < max) {
        if (backfill_info.begin <= earliest_peer_backfill() &&
                !backfill_info.extends_to_end() && backfill_info.empty()) {
            //需要继续扫描更多的对象
        }

        for (set<pg_shard_t>::iterator i = backfill_targets.begin();
                i != backfill_targets.end(); ++i) {
            pg_shard_t bt = *i;
            BackfillInterval& pbi = peer_backfill_info[bt];

            if (pbi.begin <= backfill_info.begin &&
                    !pbi.extends_to_end() && pbi.empty()) {
                epoch_t e = get_osdmap_epoch();
                MOSDPGScan *m = new MOSDPGScan( //获取该OSD目前拥有的对象列表
                        MOSDPGScan::OP_SCAN_GET_DIGEST, pg_whoami, e, last_peering_reset,
                        spg_t(info.pgid.pgid, bt.shard),
                        pbi.end, hobject_t());
                osd->send_message_osd_cluster(bt.osd, m, get_osdmap_epoch());
                waiting_on_backfill.insert(bt);
                sent_scan = true;
            }
        }

        if (check < backfill_info.begin) {//check对象
            to_remove.push_back();
        } else {
            need_ver_targs.push_back();
            keep_ver_targs.push_back();
            missing_targs.push_back();
            skip_targs.push_back();
        }
    }

    for (auto p : reqs) {
        osd->send_message_osd_cluster(p.first.osd, p.second, get_osdmap_epoch());
    }

    ...
}

``` 







