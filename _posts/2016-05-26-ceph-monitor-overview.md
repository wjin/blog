---
layout: post
title: "Ceph Monitor Overview"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

monitor在ceph集群中起着非常关键的作用，它维护着几张map(monmap, osdmap, pgmap等)， 通过paxos算法保证数据的一致性。

虽然一个monitor也可以工作，但是为了防止单点故障，monitor可以部署多个，一般情况部署三个在不同的故障域。在ceph的实现中，
monitor节点信息会存放在monmap这张表中，每个monitor在monmap中会有一个rank值，这个值非常关键，多个monitor会形成一个quorum（set\<int\>类型)，
其中存放的就是rank值，在选举leader的时候，rank值最小的获胜，所以monitor地位并不是平等的，这样做的目的可能是为了快速的选举出leader。

monitor维护的map，都是以PaxosService的服务提供，不同服务继承基类PaxosService实现自己的特性，这些服务通过paxos算法对数据进行更新，
只有leader才可以调用propose相关函数进行更新，如果peon节点收到更新的消息，则需要将消息转发给leader节点，
所以同一时刻paxos算法只会存在一个proposer，几乎没有竞争，决议会很快完成，更新也是非常迅速的。

monitor涉及的内容大致包括以下几个方面:

* startup
* data store
* data sync
* data check
* scrub
* leader elect
* timecheck
* lease
* paxos
* paxos service
* consistency

下面简单对每一方面进行介绍。

# Startup

monitor启动的流程可以参考[这篇文章](http://blog.wjin.org/posts/ceph-monitor-startup.html)，monitor经历的状态转换图如下:

![img](/assets/img/post/ceph_mon_stat.png)

# Data Store

monitor维护了很多map以及自身Elector和Paxos算法的数据，这些数据肯定是需要地方存储的，最开始的时候monitor采用文件存储，后来采用k/v存储，
主要是利用k/v的原子操作以及对key做有序排列，目前支持levelDB和rocksDB。主要实现是在文件MonitorDBStore.h中，将对key的操作封装成一个op，
然后考虑到同时对多个key操作的时候，需要确保事务性，所以使用的时候，都是以transaction的形式提交，一个transaction可能包含多个op。

```cpp
class MonitorDBStore
{
    boost::scoped_ptr<KeyValueDB> db; // 具体存储的backend，可以是levelDB或rocksDB
    ......

    struct Op { // 对key的操作
		uint8_t type;
		string prefix;
		string key, endkey;
		bufferlist bl;
		......
	};

	struct Transaction {
		list<Op> ops; // 多个op
		uint64_t bytes, keys;
		......
	};
	......
};
```

# Data Sync

除了基本的monitor的k/v元数据，更多的数据应该是通过paxos算法生成的各个map的不同版本的数据，这些数据需要保证一致性。
monitor可能发生故障，比如宕机或网络中断，在运行过程中也可能需要新加入monitor，新加入的monitor就需要同步已有monitor集群的数据。
monitor在启动的时候，会进入bootstrap阶段，然后probing其他monitor的信息，其他monitor收到消息后，会返回消息，内容主要包括以下几项:

* 当前的monmap
* 当前的quorum
* paxos的第一次提交first\_committed
* paxos的最后一次提交last\_committed

收到返回消息后，会做相应的处理，首先会判断monmap，如果收到的monmap版本更大(数据更新)，会更新自己的monmap。其次，如果paxos提交的序列号差异过大，那么需要sync对方的数据。
这里有两种情况，如果有重叠(my->last\_committed >= peer->first\_committed)，只会sync差异的序列号，如果没有，那么需要执行一次full sync，
即将对方的first\_committed到last\_committed的数据全部拷贝过来。当然，如果差异还在接受范围内，probing阶段可能跳过sync，后续由paxos恢复差异的版本数据。

注意这里只关心已经commit过的数据，paxos propose或accept过的数据，如果还没commit，这里不做处理，后面paxos算法初始化的时候会进行处理。

# Data Check

monitor有一个抽象基类QuorumService，用以派生一些针对quorum的服务，类图如下:

![img](/assets/img/post/ceph_mon_quorumservice.png)

这里感觉继承关系有些滥用，类ConfigKeyService提供用户一些接口，可以方便的在monitor存储一些自定义的key/value数据，
这需要通过leader向paxos算法发出propose完成，似乎和QuorumService关系不大。

HealthMonitor用来检查monitor状态，内部包含一个服务的map:

```cpp
class HealthMonitor : public QuorumService
{
  map<int,HealthService*> services; // 需要检查的服务
  ......
};
```

目前只实现了一个服务，即DataHealthService，这个用来检查monitor存储的数据，一方面检查磁盘空间使用情况，另一方面检查后端k/v存储的具体使用情况:

```cpp
class DataHealthService :
  public HealthService
{
  map<entity_inst_t,DataStats> stats; // 检查的项目
  ......
};

struct DataStats {
  ceph_data_stats_t fs_stats; // 文件系统使用情况
  LevelDBStoreStats store_stats; // k/v后端存储的使用情况，支持多个后端的情况下，名字不应该再用leveldb了
};
```

# Scrub

类似于osd进程需要通过scrub比较副本数据，及时发现并处理不一致的数据。各个monitor节点也需要保证数据一致（这里一致是指磁盘数据没有被损坏，不是paxos算法的一致性）。
因为monitor的数据会根据版本做trim，旧的数据意义不大，需求并不是那么强烈，所以并没有后台任务周期性地scrub(这里针对hammer版本代码而言，
最新master代码有周期性的time事件调度scrub执行)，而只是提供一种机制，即显示地发出scrub命令，由leader向各peon发送MMonScrub消息完成:

> ceph scrub

scrub的对象只是PaxosService的数据，不会保护monitor自身的一些元数据和paxos的数据，monitor自身的数据，在启动的时候应该就会做check，
paxos的数据因为很有可能每个节点的数据本身就不一样，比如正在处理proposal的时候，所以也不scrub，这也从侧面反应monitor的scrub不是那么重要。

# Leader Elect

ceph为了简化设计，monitor内部会选一个leader出来，负责发起paxos propose对数据进行更新。选举过程非常简单，
如前所述，选择编号最小的，参考[这篇文章](http://blog.wjin.org/posts/ceph-monitor-leader-elect.html)。

# Timecheck

分布式系统正常运转依赖于系统时间，ceph提供timecheck机制，用来检查每个monitor时间是否一致，如果误差过大(clock skew)，会发出警告消息。
检查由leader节点向peon节点发送消息MTimeCheck完成，leader会估算一个rtt(消息来回的时间)值，然后才是skew值:

```cpp
map<entity_inst_t, double> timecheck_skews; // clock skew
map<entity_inst_t, double> timecheck_latencies; // rtt
```

leader check的频率由以下参数控制，默认是300秒:

```text
mon_timecheck_interval
```

如果系统发生严重的时钟飘逸，有可能导致peon节点的lease失效，进而导致peon进入bootstrap，从而重新选举。可以通过将leader的时间调小(回滚)验证。

# Lease

monitor内部采用lease协议，保证副本数据在一定时间范围内可读写(写需要是leader节点)，同时也用来发现monitor的异常，然后重新选举。
leader节点会定期发送lease消息，延长各peon的时间。 如果peon节点down掉，leader节点不会收到lease\_ack消息，
超时后就会重新选举。如果leader节点down掉，所有的peon节点不会收到来自leader的lease更新消息，超时后也会重新选举。
这个协议实现在类Paxos内部，主要消息类型是MMonPaxos::OP\_LEASE，以下是几个关键参数:

```cpp
mon_lease_renew_interval // leader发送lease消息的间隔，默认为3秒
mon_lease // 每次延长lease的时间，默认为延长5秒，必须大于mon_lease_renew_interval
mon_lease_ack_timeout // 超时重新选举的时间，默认为10秒，必须大于mon_lease
```
注意最后一个参数，leader和peon共用的超时时间，名字取的不是很好。leader发出lease消息后，超过此值没有收到所有的回应消息(ack)，
就会重新进入bootstrap选举。peon在超过这个值的时候，如果还没有收到lease消息，也会进入bootstrap重新选举。
这里存在一段时间lease过期了，但是还没超时，这段时间内是不可读写的，无论是leader还是peon，这个时间点用变量lease_expire存放:

```cpp
// 判断lease是否在有效期内
bool Paxos::is_lease_valid()
{
  return ((mon->get_quorum().size() == 1)
      || (ceph_clock_now(g_ceph_context) < lease_expire));
}

void Paxos::extend_lease()
{
  assert(mon->is_leader());
  lease_expire = ceph_clock_now(g_ceph_context);
  lease_expire += g_conf->mon_lease; // leader发送消息的时候，延长时间

  ......
}

void Paxos::handle_lease(MMonPaxos *lease)
{
  ......

  // extend lease
  if (lease_expire < lease->lease_timestamp) {
    lease_expire = lease->lease_timestamp; // peon根据leader的消息，更新时间

    utime_t now = ceph_clock_now(g_ceph_context);
    if (lease_expire < now) { // 不可读写的时间段，只是打印警告消息
      utime_t diff = now - lease_expire;
      derr << "lease_expire from " << lease->get_source_inst() << " is " << diff 
		  << " seconds in the past; mons are probably laggy (or possibly clocks are too skewed)" << dendl;
    }
  }
  ......
}
```

# Paxos

paxos算法保证各monitor的数据一致，具体参见[这篇文章](http://blog.wjin.org/posts/ceph-monitor-paxos.html)。

# PaxosService

PaxosService比较简单，内部利用类Paxos的功能，提供一些模板方法，方便实现不同的服务，具体参见[这篇文章](http://blog.wjin.org/posts/ceph-monitor-paxosservice.html)。

# Consistency

### Monmap

monitor新加入的时候，会mkfs，将自己加入monmap，并且存在后端存储中。后续如果异常宕机或退出，再次启动后会读取原来的monmap，
然后通过probing机制发现其他的monitor，申请加入quorum，并且重新选举。如果在宕机过程中monmap有变化，probing阶段会share monmap并更新。
这里有个极端case需要注意，假设最开始有monitor 0,1,2:

```text
0,1,2
1,2 #0 down
1,2,3,4,5,6 #加入4个monitor: 3-6
3,4,5,6 #1,2 down
0,3,4,5,6 #0 up
```

这时候0是加入不进来的，probing会一直超时而完不成，因为0知道的monmap只知道1,2，probing只会向1,2发送消息，但是1,2已经down了。
而此时3,4,5,6因为大于一半，一直在正确工作，也不会重新bootstrap，导致他们发现不了0已经up。

当然这种case实际运维过程中应该不会遇到这么极端，但是需要明白monmap是非常重要的，它的一致性很关键，是一切后续流程的基石，所以在bootstrap阶段sync数据的时候，
都会备份一份monmap以防万一。

### Sync

probing阶段，monitor会sync数据，如果差距过大，即版本没重叠，就做全量sync，否则增量sync，这里也需要注意，如果差距不大，还没有超过需要sync的阈值，不会做数据sync，
这个阈值由参数paxos\_max\_join\_drift控制。这就意味着，probing完成后，进入electing阶段，新加入的这个monitor数据很可能是落后几个版本的，
这个数据的恢复需要paxos的recovering阶段来完成，从而达到数据的一致性。

### Paxos

leader选举完成后，leader节点会执行collect函数，即做数据recover，这会保证各monitor数据最终一致，commit的数据一定会一样，如有accept过但没commit的数据，会重新propose。
