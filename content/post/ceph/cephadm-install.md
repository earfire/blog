---
title: "通过cephadm工具安装部署ceph集群"
date: 2020-08-14T15:03:25+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## 前提

部署工具: cephadm

操作系统版本: CentOS 8.2

Ceph版本: Octopus


节点列表：

| Host         	| IP                |  Role             | 
| ------------- |:------------------|:------------------|
| ceph-mon1		| 192.168.71.101	| cephadm、mon、mgr |
| ceph-mon2		| 192.168.71.102	| mon、mgr          |
| ceph-osd1		| 192.168.71.103	| osd               |
| ceph-osd2		| 192.168.71.104	| osd               |


## 配置hosts解析、免密登录

安装上述节点列表，在各个节点上修改hostname

```
hostnamectl set-hostname ceph-mon1 (对应的主机名)
```

在ceph-mon1上配置hosts文件

```
cat >> /etc/hosts << EOF
192.168.71.101  ceph-mon1
192.168.71.102  ceph-mon2
192.168.71.103  ceph-osd1
192.168.71.104  ceph-osd2
EOF
```

复制hosts文件到其他节点

```
for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do scp /etc/hosts $i:/etc/; done
```


同时新建ssh公钥对，并配置免密登录到其他节点

```
ssh-keygen -t rsa -P ''

for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do ssh-copy-id $i; done
```



## 关闭SELINUX

关闭SELINUX

```
for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do ssh $i exec setenforce 0; done

//SELinux 配置永久生效
for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do ssh $i exec "sed -i '/^SELINUX=/c SELINUX=disabled' /etc/selinux/config"; done
```

## 安装依赖

安装python3和podman

```
for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do ssh $i exec yum install python3 podman -y; done

```

为podman配置国内源

```
cat > /etc/containers/registries.conf << EOF
unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
location = "anwk44qv.mirror.aliyuncs.com"
EOF
```

复制源配置文件到其他节点

```
for i in `tail -n 4 /etc/hosts | awk '{print $1}'`; do scp /etc/containers/registries.conf $i:/etc/containers/; done

```

## 安装cephadm

在ceph-mon1上执行

```
wget https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm

chmod +x cephadm && mv cephadm /usr/bin/cephadm

```

## 部署mon节点

在ceph-mon1上执行

```
mkdir -p /etc/ceph

cephadm bootstrap --mon-ip 192.168.71.101

This command will:
1. Create a monitor and manager daemon for the new cluster on the local host.
2. Generate a new SSH key for the Ceph cluster and adds it to the root user’s /root/.ssh/authorized_keys file.
3. Write a minimal configuration file needed to communicate with the new cluster to /etc/ceph/ceph.conf.
4. Write a copy of the client.admin administrative (privileged!) secret key to /etc/ceph/ceph.client.admin.keyring.
5. Write a copy of the public key to /etc/ceph/ceph.pub.
```

结束后执行

```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd1
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd2

```

添加节点到cluster

```
cephadm shell -- ceph orch apply mon --unmanaged


cephadm shell -- ceph orch host add ceph-mon2
cephadm shell -- ceph orch host add ceph-osd1
cephadm shell -- ceph orch host add ceph-osd2
```

用标签来指定mon节点

```
cephadm shell -- ceph orch host label add ceph-mon1 mon
cephadm shell -- ceph orch host label add ceph-mon2 mon

```

部署mon节点

```
cephadm shell -- ceph orch apply mon label:mon

//验证，mon为2 daemons，说明mon部署成功
[root@ceph-mon1 ~]# cephadm shell -- ceph -s
INFO:cephadm:Inferring fsid c013ddde-ffe7-11ea-bbe5-000c29699bfe
INFO:cephadm:Inferring config /var/lib/ceph/c013ddde-ffe7-11ea-bbe5-000c29699bfe/mon.ceph-mon1/config
INFO:cephadm:Using recent ceph image docker.io/ceph/ceph:v15
cluster:
id:     c013ddde-ffe7-11ea-bbe5-000c29699bfe
health: HEALTH_WARN
1 hosts fail cephadm check
Reduced data availability: 1 pg inactive
OSD count 0 < osd_pool_default_size 3

services:
mon: 2 daemons, quorum ceph-mon1,ceph-mon2 (age 2m)
mgr: ceph-mon1.gbtdqb(active, since 3m), standbys: ceph-osd1.ptrymz
osd: 0 osds: 0 up, 0 in

data:
pools:   1 pools, 1 pgs
objects: 0 objects, 0 B
usage:   0 B used, 0 B / 0 B avail
pgs:     100.000% pgs unknown
1 unknown

```


## 部署osd

列出节点上所有的可用设备

```
cephadm shell -- ceph orch device ls

[root@ceph-mon1 ~]# cephadm shell -- ceph orch device ls
INFO:cephadm:Inferring fsid c013ddde-ffe7-11ea-bbe5-000c29699bfe
INFO:cephadm:Inferring config /var/lib/ceph/c013ddde-ffe7-11ea-bbe5-000c29699bfe/mon.ceph-mon1/config
INFO:cephadm:Using recent ceph image docker.io/ceph/ceph:v15
HOST       PATH      TYPE   SIZE  DEVICE_ID  MODEL             VENDOR   ROTATIONAL  AVAIL  REJECT REASONS                                          
ceph-mon1  /dev/sda  hdd   30.0G             VMware Virtual S  VMware,  1           False  Insufficient space (<5GB) on vgs, LVM detected, locked  
ceph-mon2  /dev/sda  hdd   30.0G             VMware Virtual S  VMware,  1           False  LVM detected, Insufficient space (<5GB) on vgs, locked  
ceph-osd1  /dev/sdb  hdd   30.0G             VMware Virtual S  VMware,  1           True                                                           
ceph-osd1  /dev/sda  hdd   30.0G             VMware Virtual S  VMware,  1           False  LVM detected, locked, Insufficient space (<5GB) on vgs  
ceph-osd2  /dev/sdb  hdd   30.0G             VMware Virtual S  VMware,  1           True                                                           
ceph-osd2  /dev/sda  hdd   30.0G             VMware Virtual S  VMware,  1           False  Insufficient space (<5GB) on vgs, LVM detected, locked
```

部署osd节点

```
cephadm shell -- ceph orch daemon add osd ceph-osd1:/dev/sdb
cephadm shell -- ceph orch daemon add osd ceph-osd2:/dev/sdb

//验证，osd为2，说明osd部署成功
[root@ceph-mon1 ~]# cephadm shell -- ceph -s
INFO:cephadm:Inferring fsid c013ddde-ffe7-11ea-bbe5-000c29699bfe
INFO:cephadm:Inferring config /var/lib/ceph/c013ddde-ffe7-11ea-bbe5-000c29699bfe/mon.ceph-mon1/config
INFO:cephadm:Using recent ceph image docker.io/ceph/ceph:v15
cluster:
id:     c013ddde-ffe7-11ea-bbe5-000c29699bfe
health: HEALTH_WARN
clock skew detected on mon.ceph-mon2
Reduced data availability: 1 pg inactive
Degraded data redundancy: 1 pg undersized
OSD count 2 < osd_pool_default_size 3

services:
mon: 2 daemons, quorum ceph-mon1,ceph-mon2 (age 15m)
mgr: ceph-osd1.ptrymz(active, since 8m), standbys: ceph-mon1.gbtdqb
osd: 2 osds: 2 up (since 5s), 2 in (since 5s)

data:
pools:   1 pools, 1 pgs
objects: 0 objects, 0 B
usage:   2.0 GiB used, 58 GiB / 60 GiB avail
pgs:     100.000% pgs not active
1 undersized+peered

```



## 编译安装添加osd节点

| Host         	| IP                |  Role             | 
| ------------- |:------------------|:------------------|
| ceph-mon1		| 192.168.71.101	| cephadm、mon、mgr |
| ceph-mon2		| 192.168.71.102	| mon、mgr          |
| ceph-osd1		| 192.168.71.103	| osd               |
| ceph-osd2		| 192.168.71.104	| osd               |
| ceph-osd3(手动添加)	| 192.168.71.105	| osd               |


依照上述操作，配置hostname、hosts文件、免密登录、关闭SELINUX。这里不以podman形式运行osd，而是直接运行，不必安装python3和podman、为podman配置国内源。

### 编译

```
cd ceph
./install-deps.sh   //安装依赖包
./do_cmake.sh       //默认编译debug版本，如果编译release版本，传入‘-DCMAKE_BUILD_TYPE=RelWithDebInfo’给do_cmake.sh。

cd build
make                //编译
make install        //安装

//安装完成后运行ceph -v查看安装情况
//如果运行该命令时出现：ImportError: librados.so.2: cannot open shared object file: No such file or directory
//可用locate librados.so.2命令查看库的所在路径，将该路径下的librados.so拷贝到/usr/lib64/下，cp /root/ceph/build/lib/librados.so* /usr/lib64/
ceph -v

```


```
cephadm shell -- ceph auth get client.bootstrap-osd
```

