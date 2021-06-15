---
title: "ceph PG状态迁移"
date: 2021-02-03T20:36:02+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## 概述

PG有外部状态和内部状态。外部状态可为普通用户直接感知，通过ceph -s命令获取。PG内部通过状态机来驱动PG在不同外部状态之间的迁移。

## 状态

### 外部状态

| state		   | description 		   |   
| -----------------|:------------------------------|
| Activating	   | Peering已经完成，PG正在等待所有PG实例同步并固化Peering的结果(Info、Log等) |
| Active	   | PG可以正常处理来自客户端的读写请求 |
| Backfilling	   | PG正在执行Backfill。Backfill总是在Recovery完成之后进行的 |
| Backfill-toofull | 某个需要被Backfill的PG实例，其所在的OSD可用空间不足，Backfilll流程当前被挂起 |
| Backfill-wait	   | 等待Backfill资源预留 |
| Clean	           | PG当前不存在待修复的对象，Acting Set和Up Set内容一致，并且大小等于存储池副本数 |
| Creating	   | PG正在被创建 |
| Deep 	           | PG正在或者即将进行对象一致性扫描。Deep总是和Scrubbbing成对出现，表明将对PG中的对象执行深度扫描(同时扫描对象元数据和用户数据) |
| Degraded 	   | Peering完成后，PG检测到任意一个PG实例存在不一致(需要被同步/修复)的对象；或者当前Acting Set小于存储池副本数 |
| Down		   | Peering过程中，PG检测到某个不能被跳过的Interval中(例如该Interval期间，PG完成了Peering，并且成功切换至Active状态，从而有可能正常处理了来自客户端的读写请求)，当前剩余在线的OSD不足以完成数据修复 |
| Incomplete       | Peering过程中，由于1.无法逃出权威日志 2.通过choose_acting选出的Acting Set后续不足以完成数据修复(例如针对纠删码，存活的副本数小于k值)(与Down的区别在于这里选不出来的原因是由于某些剧本的日志不完整)等导致Peering无法正常完成 |
| Inconsistent     | PG通过Scrub检测到某个或者某些对象在PG实例间出现了不一致(主要是静默数据导致错误) |
| Peered           | Peering已经完成，但是PG当前Acting Set规模小于存储池规定的最小副本数min_size |
| Peering	   | PG正在进行Peering |
| Recovering       | Recovering资源预留成功，PG正在后台根据Peering的结果针对不一致的对象进行同步/修复 |
| Recovery-wait    | 等待Recovery资源预留 |
| Remapped	   | Peering完成，PG当前Acting Set与Up Set不一致 |
| Repair	   | PG在下一次执行Scrub的过程中，如果发现存在不一致的对象，并且能够修复，则自动进行修复 |
| Scrubbing	   | PG正在或者即将进行对象一致性扫描。Scrubbing仅扫描对象的元数据 |
| Stale 	   | Monitor检测到当前Primary所在的OSD宕掉；或者Primary超时未向Monitor上报PG相关的统计信息(例如出现临时性的网络拥塞) |
| Undersized	   | PG当前Acting Set小于存储池副本数 |

### 内部状态

内部状态由PG状态机来说明。

## PG创建

PG在主OSD的创建，当OSDMonitor收到存储池创建命令后，最终通过PGMonitor对该Pool的每一个PG对应的主OSD下发PG创建请求。函数OSD::handle_pg_create用于处理Monitor发送的创建PG的请求。

```
在OSD::handle_pg_create中调用enqueue_peering_evt将事件入队，由dequeue_peering_evt取出进行处理，主要

```


PG在从OSD的创建，Monitor不会的PG的从OSD发送创建PG请求，而是由该主OSD上的PG在Peering过程中完成的。如果该PGb不存在，就会创建该PG。该PG的状态机进入Stray状态。

PG的加载，当OSD重启，调用函数OSD::init()，该函数调用load_pgs函数加载已存在的PG。

PG的创建:

```
1.PG创建调用handle_inital()投递Initial事件，由RecoveryMachine/Initial收到Initialize事件后直接转换到Reset事件。


2.其次投递了ActMap事件.


3.状态Reset接收到ActMap事件，跳转到Started状态。
boost::statechart::result PG::RecoveryState::Reset::react(const ActMap&)
{
    ...
    return transit< Started >();
}


4.进入到Started，就进入默认的子状态Start。
struct Started : boost::statechart::state< Started, RecoveryMachine, Start >, NamedState {};
PG::RecoveryState::Start::Start(my_context ctx) 
  : my_base(ctx),
    NamedState(context< RecoveryMachine >().pg, "Start")
{
  context< RecoveryMachine >().log_enter(state_name);

  PG *pg = context< RecoveryMachine >().pg;
  if (pg->is_primary()) {
    ldout(pg->cct, 1) << "transitioning to Primary" << dendl;
    post_event(MakePrimary());
  } else { //is_stray
    ldout(pg->cct, 1) << "transitioning to Stray" << dendl;
    post_event(MakeStray());
  }
}
如果是主OSD，就调用post_event(MakePrimary())，抛出MakePrimary事件，进入主OSD的默认子状态Started/Primary/Peering。
如果是从OSD，就调用post_event(MakeStray())，抛出MakeStray事件，进入Started/Stary状态。

Primary为主OSD状态，Stray为从OSD状态。Stray是指该OSD上的PG副本不确定，但是可以响应主OSD的各种查询操作，它由两种可能：一种是最终转移到ReplicaActive处于活跃状态，成为PG的一个副本。
另一种，如果是数据迁移的源端，可能一直保持Stray状态，该OSD上的副本可能在数据迁移完成后，PG以及数据就都被删除了。
```

## Peering

如果状态机在Reset状态下收到ActMap事件，则意味着正式启动Peering。


### GetInfo

通过向状态机发送MakePrimary事件，针对Primary进入Started/Primary/Peering/GetInfo状态。主要是PG的主OSD通过发送消息获取所有从OSD的pg_info信息。

### GetLog

当所有Info收集完毕之后，Primary通过向状态机发送GotInfo事件，跳转至Started/Primary/Peering/GetLog状态。根据各个副本的pg_info选出权威日志，Primary日志同步，然后与本地日志进行合并，生成本地权威日志。在合并权威日志过程中，如果Primary发现本地有对象需要修复，则将其加入自身的missing列表。完成合并之后，Primary向状态机状态机发送GotLog事件。

### GetMissing

Primary发送GotLog事件后切换至Started/Primary/Peering/GetMissing状态，开始向能够通过Recovery恢复的副本发送Query消息，以获取它们的日志。通过与本地日志对比，Primary可以构建这些副本的Missing列表，作为后续执行Recovry的依据。生成完整的missing列表后，Primary将向状态机投递Activate事件。



### Active

Primary将向状态机投递Activate事件，进入tarted/Primary/Peering/Active/Activating状态。最后通过Active激活主OSD，并发送notify通知消息，激活相应的从OSD。



