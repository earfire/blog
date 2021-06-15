---
title: "ceph源码分析ceph_osd.cc"
date: 2020-08-04T15:13:44+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---

## main


## global_init

![Local Picture](/images/ceph/ceph_osd_cc/global_init.png "global_init")

## osd specific args

解析osd参数赋值给定义的相关变量


## Preforker

Preforker是对系统的fork调用的封装，用于创建守护进程。子进程被创建后，继续下面的执行，而父进程退出。

## common_init_finish

进一步初始化，cct->init_crypto()和cct->start_service_thread()，在start_service_thread()里会调用_admin_socket->init()，继而创建AdminSocket对应的处理线程。

## osd args

执行osd参数命令，get_journal_fsid、get_device_fsid、dump_pg_log...

## whoami

获取id值，即启动参数-i，例如-i osd.2，whoami=2。

## the store

获取store_type值，首先从data_path/type文件(若文件存在)读取，否则if (mkfs), 从g_conf读取，否则最后根据data_path/current、data_path/block进行store_type推断。

## ObjectStore

创建ObjectStore实例，ObjectStore *create(g_ceph_context, store_type, data_path, journal_path, flags); 会根据不同的type，如if (type == "bluestore") {return new BlueStore(cct, data);}

## osd args

执行osd参数命令，mkkey、mkfs、mkjournal、check_allows_journal、check_needs_journal、flushjournal、dump_journal、convertfilestore。

## peek_meta

通过OSD::peek_meta(store->read_meta)，读取osd元数据，获得magic、cluster_fsid、osd_fsid、whoami、require_osd_release。预先读取的这些引导数据，可用于身份验证。其引导数据不能直接由OSD自身的ObjectStore接管。为此，当创建OSD时，总是预留少量存储空间(默认为100MB)，并使用操作系统自带的本地文件系统(例如XFS)格式化，用于保存和访问这部分引导数据。这些引导数据的作用如下：

| 引导数据类型  | 作用                  |
| ------------- |:----------------------|
| magic         | 校验数据，用于验证该OSD能否被当前的Ceph版本所识别和接管，也用于验证这些引导数据是否已经损坏 |
| whoami        | OSD在集群中的数字标识，这是一个从0开始编号的整数，全集群唯一，可以直接作为OSD的身份标识 |
| cluster_fsid  | OSD归属集群的UUID |
| osd_fsid      | OSD自身的UUID     |
| require_osd_release | 正常启动所要求的OSD版本号     |


## Messengers

从g_conf获取ms_type、ms_public_type、ms_cluster_type，Messenger::create根据type(simpe、async、xio)创建各个Messenger，ms_public、ms_cluster、ms_hb_back_client、ms_hb_front_client、ms_hb_back_server、ms_hb_front_server、ms_objecter，然后进行set_cluster_protocol、set_policy等操作。

OSD中驻留的Messenger及其功能如下：

| Messenger     | 功能                  |
| ------------- |:----------------------|
| public        | 主要用于客户端与OSD之间进行通信 |
| cluster       | 主要用于OSD之间进行通信 |
| heartbeat     | 主要用于OSD之间的通信链路检测 。由于心跳检测报文优先级最高，在设计上，为了防止其他类型的报文产生干扰，为heartbeat分配了独立的Messenger，即将heartbeat和其他任意类型的通信服务进行了隔离。此外，为了保证能够侦测到任意网络平面的故障，OSD之间同时通过公共网络和集群网络进行heartbeat |
| objecter      | 主要用于实现Cache-Tier相关的、跨存储池之间的报文转发 |


## global_init_preload_erasure_code

加载erasure_code静态库文件

## OSD

实例osd对象，通过osd->pre_init()判断osd目录是否已经mount，如果已挂载则会退出。 然后调用osd->init()、osd->final_init()完成osd的初始化操作。

osd->init的流程图如下：

![Local Picture](/images/ceph/ceph_osd_cc/osd_init.png "osd_init")


## ms_start

网络层启动，监听等待。


