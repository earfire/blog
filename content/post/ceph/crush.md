---
title: "ceph CRUSH算法"
date: 2020-03-14T20:42:11+08:00
draft: false
tags: ["ceph"]
categories: ["ceph"]
---


## 简介

CRUSH(Controlled Replication Under Scalable Hashing),是一种基于哈希的数据分布算法。与另一种基于集中式的元数据查询的存储方式(文件的分布信息需要先通过访问集中元数据服务器获得)不同，它以数据唯一标识符、当前存储集群的拓扑结构以及数据分布策略作为CRUSH的输入，经过计算获得数据分布位置，直接与OSDs进行通信，从而避免集中式查询操作，实现去中心化和高度并发。

数据分布算法：
数据分布和负载均衡、灵活应对集群伸缩、支持大规模集群。
在分布式存储系统中，由两种基本实现方法，一种是基于集中式的元数据查询的方式，如HDFS的实现：文件的分布信息是通过访问集中元数据服务器获得；另一种是基于分布式算法以计算获得。例如一致性哈希算法(DHT)。ceph的数据分布算法CRUSH属于后者。


## stable_mod

对象寻址过程分为两步，一是对象到PG的映射; 二是PG到OSD列表映射，后一步通过CRUSH算法来实现。

对象到PG的映射：任何程序通过客户端访问集群时，首先由客户端生成一个字符串形式的对象名，然后基于对象名和命名空间计算得出一个32位哈希值。针对此哈希值，对该存储池的PG总数量pg_num取模(掩码计算)，得到该对象所在的PG的id号。

```
//pg_num对应的最高位为n，则其掩码为2^n-1，当pg_num为12，n=4。需要将某个对象映射至PG时，直接执行hash&(2^n-1)即可。
//但当pg_num不是2的整数次幂，如果直接hash映射后会产生空穴，编号12-15的PG并不存在，所以有了stable_mod：

if ((hash & (2^n - 1)) < pg_num)
    return (hash & (2^n-1));
else
    return (hash & (2^(n-1)-1));
```


## 抽签算法

## CRUSH算法

针对指定输入x(要计算PG的pg_id)，CRUSH将输出一个包含n个不同目标存储对象(例如磁盘)的集合(OSD列表)。CRUSH的计算过程使用x、cluster map、placement rule作为哈希函数输入。因此如果cluster map不发生变化(一般placement rule不会轻易变化)，那么结果就是确定的。

cluster map集群的层级化描述，形如"数据中心->机架->主机->磁盘"这样的层级拓扑。用树来表示，每个叶子节点都是真实的最小物理存储设备(例如磁盘)，称为devices；所有中间节点统称为bucket，每个bucket可以是一些devices的集合，也可以是低一级的buckets集合；根节点称为root，是整个集群的入口。

placement rule数据分布策略，它决定一个PG的对象副本如何选择(从定义的cluster map的拓扑结构中)的规则，以此完成数据映射。

每条palcement rule可以包含多个操作，这些操作共有3种类型：

```
take(root) //从cluster map选择指定编号的bucket，并以此作为后续步骤的输入。例如系统默认的placement rule总是以cluster map中的root节点作为输入
select(replicas, type) //replicas为副本策略(多副本和纠删码), type为想要设置的容灾域类型，选择时会选择位于不同容灾域(例如不同机架)下的节点设备。
emit(void) //输出最终选择结果给上级调用者并返回
```

## 数据结构

CRUSH算法相关的数据结构如下：

| type     		| position          |   
| ------------- |:------------------|
| crush_map		| crush/crush.h		|
| crush_bucket	| crush/crush.h		|
| crush_rule	| crush/crush.h		|

```
//crush_map定义了crush_bucket的层级结构和一系列的crush_rule规则。
struct crush_map {
    struct crush_bucket **buckets;
    struct crush_rule **rules; //pg映射策略
    ...
};

//cursh_bucket保存了Bucket的相关信息。
struct crush_bucket {
    __s32 id;        //bucket id
    __u16 type;      //bucket类型，
    __u8 alg;        //选择算法
    __u8 hash;       //hash函数
    __u32 weight;    //权重，按容量，或按性能
    __u32 size;      //bucket下的item数量
    __s32 *items;    //array of children: < 0 are buckets, >= 0 items
};

//crush_rule规则
struct crush_rule_step {
    __u32 op; //step操作步的操作码
    __s32 arg1; //如果是take，参数就是选择的bucket的id号，如果是select，就是选择的数量
    __s32 arg2; //如果是select，是选择的类型
};

struct crush_rule_mask {
    __u8 ruleset; //ruleset的编号
    __u8 type; //类型
    __u8 min_size; //最小size
    __u8 max_size; //最大size
};

struct crush_rule {
    __u32 len; //steps的数组的长度
    struct crush_rule_mask mask; //ruleset相关的配置参数
};

```

## 算法实现

```
/**
 * crush_do_rule - calculate a mapping with the given input and rule
 * @map: the crush_map 包含device、type、buckets的拓扑结构和rules等
 * @ruleno: the rule id 当前pool所使用的rule规则ruleset. ruleset的号
 * @x: hash input 一般为pg的id
 * @result: pointer to result vector 输出osd列表，用于存放选中的osd
 * @result_max: maximum result size 输出osd列表的数量，需要选择的osd个数
 * @weight: weight vector (for map leaves) 所有osd的权重,通过它来判断osd是否out
 * @weight_max: size of weight vector 所有osd的数量
 * @cwin: Pointer to at least map->working_size bytes of memory or NULL.
 */
int crush_do_rule(const struct crush_map *map,
        int ruleno, int x, int *result, int result_max,
        const __u32 *weight, int weight_max,
        void *cwin, const struct crush_choose_arg *choose_args)
{
    //函数crush_do_rule完成了CRUSH算法的选择过程。
    //由rule->len，根据规则，一步步进行选择处理
    //调用crush_choose_firstn, crush_choose_indep。
}

crush_choose_firstn和crush_choose_indep都会调用crush_bucket_choose函数来进行bucket的选择。

static int crush_bucket_choose(const struct crush_bucket *in,
        struct crush_work_bucket *work,
        int x, int r,
        const struct crush_choose_arg *arg,
        int position)
{
    //根据in->alg算法类型，CRUSH_BUCKET_UNIFORM、CRUSH_BUCKET_LIST、CRUSH_BUCKET_TREE、CRUSH_BUCKET_STRAW、CRUSH_BUCKET_STRAW2，调用对应的选择算法函数。
    switch (in->alg) {
    case CRUSH_BUCKET_UNIFORM:
        return bucket_uniform_choose(
            (const struct crush_bucket_uniform *)in,
            work, x, r);
    case CRUSH_BUCKET_LIST:
        return bucket_list_choose((const struct crush_bucket_list *)in,
                      x, r);
    case CRUSH_BUCKET_TREE:
        return bucket_tree_choose((const struct crush_bucket_tree *)in,
                      x, r);
    case CRUSH_BUCKET_STRAW:
        return bucket_straw_choose(
            (const struct crush_bucket_straw *)in,
            x, r);
    case CRUSH_BUCKET_STRAW2:
        return bucket_straw2_choose(
            (const struct crush_bucket_straw2 *)in,
            x, r, arg, position);
    default:
        return in->items[0];
    }
}

static int bucket_straw_choose(const struct crush_bucket_straw *bucket,
                   int x, int r)
{
    __u32 i;
    int high = 0; 
    __u64 high_draw = 0; 
    __u64 draw;

    for (i = 0; i < bucket->h.size; i++) {
        draw = crush_hash32_3(bucket->h.hash, x, bucket->h.items[i], r); //计算hash值
        draw &= 0xffff; //获取低16位，并乘以权重相关的修正值
        draw *= bucket->straws[i];
        if (i == 0 || draw > high_draw) { //选取draw最大的item为选中的item
            high = i;
            high_draw = draw;
        }
    }
    return bucket->h.items[high];
}

```

