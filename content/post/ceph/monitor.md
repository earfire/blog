---
title: "ceph Monitor"
date: 2020-08-09T22:03:25+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## 简介

Monitor是基于Paxos算法构建的、具有分布式强一致性的集群，主要负责维护和传播集群表的权威副本。任何时刻、任意类型的客户端或OSD都可以通过和集群中任意一个Monitor进行交互，以索取或者请求更新集群表。

Ceph对Paxos做了简化，将集群表的更新操作进行了串行化处理，即任意时刻只允许由某个特定的Monitor统一发起集群表更新，并且一个时间段内所有的更新操作会被合并成一个单独的请求进行提交。


## 集群表

Ceph存在各种不同类型的、需要依赖于Monitor进行集中式管理的数据，例如集群表、集群级别的统计和告警等，因此产生了各种不同类型的Monitor，如AuthMonitor、HealthMonitor、LogMonitor、MDSMonitor、OSDMonitor、PGMonitor。所有类型的Monitor中，OSDMonitor是最主要的，它负责守护集群表。

集群表主要由两部分组成:一是集群拓扑结果和用于计算寻址的CRUSH规则，即CRUSH map(对其两者进行统一管理)，二是素有OSD的身份和状态信息。因为CRUSH map也是围绕OSD构建的，集群表也被称为OSDMap。OSDMap不仅记录了OSD相关的信息，还记录了存储池信息。


OSD采用点对点而不是广播方式传播OSDMap，主要是为了避免当集群存在大量OSD时引起广播风暴。


## 流程

### bootstrap

monitor进程的启动时调用Monitor::bootstrap()，进行初始化并向Peer端发送MMonProbe::OP_PROBE消息。

```
void Monitor::bootstrap()
{
    ...
    // reset
    state = STATE_PROBING;

    _reset();

    // sync store
    if (g_conf()->mon_compact_on_bootstrap) {
        dout(10) << "bootstrap -- triggering compaction" << dendl;
        store->compact();
        dout(10) << "bootstrap -- finished compaction" << dendl;
    }

    // singleton monitor?
    if (monmap->size() == 1 && rank == 0) {
        win_standalone_election();
        return;
    }

    reset_probe_timeout();

    // i'm outside the quorum
    if (monmap->contains(name))
        outside_quorum.insert(name);
    ```
    // probe monitors
    dout(10) << "probing other monitors" << dendl;
    for (unsigned i = 0; i < monmap->size(); i++) {
        if ((int)i != rank)
            send_mon_message(
                    new MMonProbe(monmap->fsid, MMonProbe::OP_PROBE, name, has_ever_joined,
                        ceph_release()),
                    i);
    }
    for (auto& av : extra_probe_peers) {
        if (av != messenger->get_myaddrs()) {
            messenger->send_to_mon(
                    new MMonProbe(monmap->fsid, MMonProbe::OP_PROBE, name, has_ever_joined,
                        ceph_release()),
                    av);
        }
    }
}
```

Monitor收到OP_PROBE的消息，经过dispatch逻辑之后进入Monitor::dispatch_op()，解析出来是MSG_MON_PROBE类型的消息，进而进入Monitor::handle_probe(), 如果是MMonProbe::OP_PROBE，则进入handle_probe_probe函数回复MMonProbe::OP_REPLY消息。收到REPLY消息的Monitor，会进入Monitor::handle_probe_reply进行处理。

```
void Monitor::handle_probe_reply(MonOpRequestRef op)
{
    // discover name and addrs during probing or electing states.
    if (!is_probing() && !is_electing()) {
        return;
    }

    //比对对方的monmap和自己monmap的epoch版本，如果自己的monmap版本低，则更新自己的map，然后重新进入bootstrap()阶段。
    ...
    //对比彼此的paxos的版本，是否进行sync_data
    
    //判断quorum，start_election()
}
```

### 选举Leader

在调用elector.call_election()，通过Elector::start()正式进行选举。

```
void Monitor::start_election()
{
    wait_for_paxos_write();
    _reset();
    state = STATE_ELECTING;

    logger->inc(l_mon_num_elections);
    logger->inc(l_mon_election_call);

    elector.call_election(); //Elector::start()
}
```

推选自己为leader，向Monmap中的所有其他节点发送PROPOSE消息，发起自荐

```
void Elector::start()
{
    if (!participating) {
        return;
    }

    acked_me.clear();
    init(); //get epoch
    ...
    // bcast to everyone else
    for (unsigned i=0; i<mon->monmap->size(); ++i) {
        if ((int)i == mon->rank) continue;
        MMonElection *m =
            new MMonElection(MMonElection::OP_PROPOSE, epoch, mon->monmap);
        m->mon_features = ceph::features::mon::get_supported();
        m->mon_release = ceph_release();
        mon->send_mon_message(m, i); 
    }

    reset_timer();
}
```

处理别人的自荐消息，根据情况决定是否支持，或者决定该推荐自己

```
void Elector::handle_propose(MonOpRequestRef op) 
{
    MMonElection *m = static_cast<MMonElection*>(op->get_req());
    int from = m->get_source().num(); //rank值，表示优先级用于解决冲突

    if ((required_features ^ m->get_connection()->get_features()) &
            required_features) {
        nak_old_peer(op);
        return;
    } else if (mon->monmap->min_mon_release > m->mon_release) {
        nak_old_peer(op);
        return;
    } else if (!m->mon_features.contains_all(required_mon_features)) {
        // all the features in 'required_mon_features' not in 'm->mon_features'
        mon_feature_t missing = required_mon_features.diff(m->mon_features);
        nak_old_peer(op);
    } else if (m->epoch > epoch) { //对方epoch比我大，用他的
        bump_epoch(m->epoch);
    } else if (m->epoch < epoch) {
        // got an "old" propose,
        if (epoch % 2 == 0 &&    // in a non-election cycle
                mon->quorum.count(from) == 0) {  // from someone outside the quorum
            // a mon just started up, call a new election so they can rejoin!
            // we may be active; make sure we reset things in the monitor appropriately.
            mon->start_election(); //对方epoch太小，不可能当选。自己来。
        } else {
            return;
        }
    }

    if (mon->rank < from) { //我的优先级高
        // i would win over them.
        if (leader_acked >= 0) {        // we already acked someone
            ceph_assert(leader_acked < from);  // and they still win, of course
        } else {
            // wait, i should win!
            if (!electing_me) {
                mon->start_election(); //自荐
            }
        }
    } else {
        // they would win over me
        if (leader_acked < 0 ||      // haven't acked anyone yet, or
                leader_acked > from ||   // they would win over who you did ack, or
                leader_acked == from) {  // this is the guy we're already deferring to
            defer(from); //他的优先级高，并且比已经响应过的其他节点的优先级高，回复ACK消息
        } else { //他的优先级比其他的低，忽略
            // ignore them!
            dout(5) << "no, we already acked " << leader_acked << dendl;
        }
    }
}

```

处理ACK消息

```
void Elector::handle_ack(MonOpRequestRef op)
{
    if (electing_me) { //自己正在自荐
        // thanks
        acked_me[from].cluster_features = m->get_connection()->get_features();
        acked_me[from].mon_features = m->mon_features;
        acked_me[from].mon_release = m->mon_release;
        acked_me[from].metadata = m->metadata;

        // is that _everyone_?
        if (acked_me.size() == mon->monmap->size()) {
            // if yes, shortcut to election finish
            victory(); //自荐成功
        }
    } else { //已经推荐别人了
        // ignore, i'm deferring already.
        ceph_assert(leader_acked >= 0);
    }
}

```

自荐成功

```
//自己当选Leader
void Elector::victory()
{
    // tell everyone! Leader通知大家自己当选，
    for (set<int>::iterator p = quorum.begin();
            p != quorum.end();
            ++p) {
        if (*p == mon->rank) continue;
        MMonElection *m = new MMonElection(MMonElection::OP_VICTORY, epoch,
                mon->monmap);
        m->quorum = quorum;
        m->quorum_features = cluster_features;
        m->mon_features = mon_features;
        m->sharing_bl = mon->get_local_commands_bl(mon_features);
        m->mon_release = min_mon_release;
        mon->send_mon_message(m, *p);
    }

    // tell monitor，设置state = STATE_LEADER; 调用paxos->leader_init();
    mon->win_election(epoch, quorum,
            cluster_features, mon_features, min_mon_release,
            metadata);
}

//收到别人当选Leader消息后的处理

void Elector::handle_victory(MonOpRequestRef op)
{
    MMonElection *m = static_cast<MMonElection*>(op->get_req());
    int from = m->get_source().num();
    leader_acked = -1; 
    bump_epoch(m->epoch);

    // they win，设置state = STATE_PEON，调用paxos->peon_init();
    mon->lose_election(epoch, m->quorum, from,
            m->quorum_features, m->mon_features, m->mon_release);

    // cancel my timer
    cancel_timer();

    // stash leader's commands
    vector<MonCommand> new_cmds;
    auto bi = m->sharing_bl.cbegin();
    MonCommand::decode_vector(new_cmds, bi);
    mon->set_leader_commands(new_cmds);
}


```


### Recovery阶段


