---
layout: post
title: "Ceph Monitor Paxos"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

paxos算法主要用来解决分布式系统中的数据一致性，ceph monitor中实现了paxos算法，然后抽象出了PaxosService基类，基于此实现了不同的服务，
比如MonmapMonitor, OSDMonitor, PGMonitor等，分别对应monmap, osdmap, pgmap。

![img](/assets/img/post/ceph_mon_paxosservice.png)

paxos需要根据monitor状态来做转换，大致如下:

* monitor启动的时候，preinit会调用函数init\_paxos初始化paxos

* monitor进入bootstrap，准备重新选举的时候，会restart paxos

* monitor选举成功，成为leader的时候，会将paxos初始化leader

* monitor选举失败，成为peon的时候，会将paxos初始化为peon

* monitor运行过程中，leader上的PaxosService会提议一些值，进行paxos决议，即propose

* monitor发生故障后，重新启动，会对paxos做recover

搞清楚每一步大致做了什么以及类Monitor, Paxos以及PaxosService之间的关系，整个流程就会水落石出，前面四种情况都非常简单，
关键部分是做propose和recover，接下来分析下每个步骤。

# Init

monitor在启动的时候，会初始化paxos及其服务（Monitor::preinit()->Monitor::init_paxos()->Paxos::init()):

```cpp
void Monitor::init_paxos()
{
  paxos->init(); // 初始化paxos

  for (int i = 0; i < PAXOS_NUM; ++i) {
    paxos_service[i]->init(); // 初始化服务，只有LogMonitor实现了init函数，做了简单初始化
  }

  refresh_from_paxos(NULL); // 更新
}

void Paxos::init()
{
  // 加载paxos算法相关变量
  last_pn = get_store()->get(get_name(), "last_pn"); // 最后一次提议编号
  accepted_pn = get_store()->get(get_name(), "accepted_pn"); // 最后一次接受的提议编号
  last_committed = get_store()->get(get_name(), "last_committed"); // 最后一次commit的版本
  first_committed = get_store()->get(get_name(), "first_committed"); // 第一次commit的版本
}
```

Monitor的preinit只会调用一次，所以只会初始化一次paxos，即加载一些变量。但是，上面的函数refresh\_from\_paxos需要注意，
后面paxos运行过程中，会在refresh的时候反复调用，refresh发生在commit完成或者recover完成后。

# Restart

monitor进程会在多种情况下重新bootstrap，paxos也会相应的被重置，终止未决的提议以及清理一些timeout事件，然后进入recovering状态等待恢复:

```cpp
void Monitor::bootstrap()
{
  ......
  // reset
  state = STATE_PROBING;
  _reset();
  ......
}

void Monitor::_reset()
{
  ......
  paxos->restart(); // 重启paxos

  for (vector<PaxosService*>::iterator p = paxos_service.begin(); p != paxos_service.end(); ++p)
    (*p)->restart(); // 重启服务
  ......
}

void Paxos::restart()
{
  cancel_events(); // 取消所有timeout事件
  new_value.clear(); // 清理提议的值

  if (is_writing() || is_writing_previous()) {
    mon->lock.Unlock();
    mon->store->flush(); // 等待写完成
    mon->lock.Lock();
  }

  state = STATE_RECOVERING; // 重新回到recovering状态
  pending_proposal.reset(); // 重置待决议的事务
  ......
}
```

# Leader Init

Monitor选举完成后，就会告诉paxos，目前是leader还是peon，分别做相应的处理。

如果只有一个leader，paxos直接进入active状态，否则，发送消息给其他peon，进入recovering状态，等待其他monitor的响应:

```cpp
void Monitor::win_election(epoch_t epoch, set<int>& active, uint64_t features,
                           const MonCommand *cmdset, int cmdsize,
                           const set<int> *classic_monitors)
{
  // 更新状态
  state = STATE_LEADER;
  leader_since = ceph_clock_now(g_ceph_context);
  leader = rank;
  quorum = active;
  ......

  // 初始化leader的paxos
  paxos->leader_init();

  ......
  monmon()->election_finished(); // active服务
  for (vector<PaxosService*>::iterator p = paxos_service.begin();
       p != paxos_service.end(); ++p) {
    if (*p != monmon())
      (*p)->election_finished(); // active服务
  }

  ......

  finish_election(); // 完成

  // 启动leader的服务
  if (monmap->size() > 1 &&
      monmap->get_epoch() > 0) {
    timecheck_start(); // leader需要检查monitor时钟倾斜
    health_tick_start(); // 磁盘状态检查
    do_health_to_clog_interval();
  }
}

void Paxos::leader_init()
{
  // 清理工作
  cancel_events();
  new_value.clear();
  pending_proposal.reset();
  finish_contexts(g_ceph_context, pending_finishers, -EAGAIN);
  finish_contexts(g_ceph_context, committing_finishers, -EAGAIN);

  // 如果只有一个moitor，不需要collet其他monitor信息，直接进入active状态
  if (mon->get_quorum().size() == 1) {
    state = STATE_ACTIVE;
    return;
  }

  // 否则需要collect，进入recovering状态
  state = STATE_RECOVERING;
  lease_expire = utime_t();
  collect(0); // 收集信息
}
```
 
# Peon Init

peon节点直接进入recovering状态，等待leader的collect消息，协助leader recover流程，如果超时，会重新发起选举:

```cpp
void Monitor::lose_election(epoch_t epoch, set<int> &q, int l, uint64_t features) 
{
  // 更新状态
  state = STATE_PEON;
  leader_since = utime_t();
  leader = l;
  quorum = q;

  // 初始化peon的paxos
  paxos->peon_init();

  for (vector<PaxosService*>::iterator p = paxos_service.begin(); p != paxos_service.end(); ++p)
    (*p)->election_finished(); // active服务

  ......

  finish_election(); // 完成
}

void Paxos::peon_init()
{
  // 清理工作
  cancel_events();
  new_value.clear();

  // peon进入recovering状态，等待leader的collect消息
  state = STATE_RECOVERING;
  lease_expire = utime_t();

  reset_lease_timeout(); // 如果长时间没收到collect消息，会重新选举

  // 清理工作
  pending_proposal.reset();
  ......
}
```

election\_finished函数仅仅是PaxosService基类实现了，目的是调用\_active函数，然后各个服务实现自己的on\_active函数，
即在paxos进入active状态的时候各服务需要做哪些相应的处理。

```cpp
void PaxosService::election_finished()
{
  ......
  _active();
}

void PaxosService::_active()
{
  ......
  if (is_active())
    on_active();
}
```

# Propose

leader初始化后会进入collect阶段，用于做数据恢复。其实数据恢复，也是一个paxos propose过程，需要根据propose的时候存储了些什么值来做决策，
所以先看看propose怎么实现的。

如果需要对数据做修改，需要进行paxos算法表决(propose)，ceph这里为了简化数据恢复流程，一次只能决议一个值，
不难猜测，propose只能由leader节点提出，所以ceph更新操作还是挺快的，不会产生多个proposer竞争的活锁情况，那么什么时候需要修改数据？

源码中发现除了升级的特殊情况外，主要还包含以下三种情况需要做propose:

* ConfigKeyService服务在修改或删除key/value值的时候，这个服务将monitor当作一个存储k/v数据的黑盒子，
  参见ConfigKeyService::store\_put和store\_delete

* Paxos以及PaxosService对数据做trim的时候，trim的目的是为了节省存储空间，参见Paxos::trim和PaxosService::maybe\_trim

* PaxosService的各种服务，需要更新值的时候，参见PaxosService::propose\_pending

以上三种情况在决定做propose之前，会将操作封装成事务，存放在Paxos类的变量pending\_proposal中，
然后设置commit完成后需要调用的callback，接着就调用Paxos::trigger\_propose函数开始决议。

这里需要注意的是，事务操作pending\_proposal会被编码到bufferlist中，作为此次决议的值，会存放在paxos相关的k/v中，key为版本号，
value为bufferlist二进制数据。commit的时候需要将bufferlist中的二进制数据还原成transaction，然后执行其中的操作，
即让决议的值反应在各个服务中，更新相关map。

下面以一个简单例子来阐述流程，不考虑异常的情况，假设最开始leader和peon都处于稳定状态，且k/v中存储paxos的值如下:

```text
first_committed = 1
last_committed = 10
accepted_pn = 100
```

此时leader接收到决议的请求，整个算法运转流程如下:

> 1) leader的操作，参见函数begin

* leader节点首先将pending\_proposal编码到bufferlist new_value中
* 更新相关值在后端存储中
* 发送消息给peon，消息内容(v6=new\_value, last\_committed=10, pn=100)

leader状态由`active变为updating`，此时存储数据如下:

```text
first_committed = 1
last_committed = 10
accepted_pn = 100

# 此次提议增加的数据
v11=new_value; # 11是last_committed+1的值，这里key会有前缀，简单以v代替
pending_v=11
pending_pn=100
```

> 2) peon收到propose消息的处理，参见函数handle\_begin

* peon只处理pn >= accepted\_pn的消息，很明显，这里是相等，所以会处理。
* 更新相关值在后端存储中
* 发送消息告诉leader已经接受

peon状态由`active变为updating`，此时peon节点和leader节点存储的数据一样:

```text
first_committed = 1
last_committed =10 
accepted_pn = 100
v11=new_value
pending_v=11
pending_pn=100
```

> 3) leader收到peon发回来的accept消息, 参见函数handle\_accept

* 对accept\_pn和last\_committed做检查
* 检查通过后，放入accept集合
* 待收到quorum集合的`所有accept消息`以后，执行commit操作 (注意这里并不是paxos算法中大多数接受就ok)

commit操作由函数commit\_start开始:

* 更新后端存储中last\_committed值，即+1
* 将new\_value中的值解码成事务，然后向后端存储排队执行，注意这里采用异步写，此时leader状态从`updating变为writing`

采用异步写，在写的过程中，可以释放monitor lock，这样可以处理其他消息，time线程也可以获取锁处理事件。后端存储完成后，会回调commit\_finish:

* 将内存中last\_committed值+1
* 向peon发送commit消息
* 设置状态为`refresh`，刷新PaxosService服务

PaxosService的服务会检查是否需要更新，依据是refresh的时候会更新各服务的cached\_last\_committed，这个值有变化各服务就会相应的处理，
完成后leader就从`refresh回到active状态`，此时leader信息如下:

```text
first_committed = 1
last_committed =11 # 更新版本
accepted_pn = 100
v11=new_value
pending_v=11
pending_pn=100
# 还有根据最开始发起提议的时候，事务中记录的操作对后端存储的影响，即几个map的变化
```
需要注意的是，refresh完成后，在变回状态active之前，会开始lease协议，即发送lease消息给peon，这会帮助peon也变为active。

> 4) peon收到commit消息, 参见函数handle\_commit

* 更新内存中和后端存储中last\_committed值，即+1
* 将new\_value中的值解码成事务，然后调用后端存储接口执行请求，这里采用同步写，和leader节点不一样
* 刷新PaxosService服务

peon一直处在updating状态，最终信息如下:

```text
first_committed = 1
last_committed =11 # 更新版本
accepted_pn = 100
v11=new_value
pending_v=11
pending_pn=100
# 还有根据最开始发起提议的时候，事务中记录的操作对后端存储的影响，即几个map的变化
```

> 5) peon收到lease消息, 参见函数handle\_lease，`peon状态从updating变回active`

总结一下，一轮propose完成后，更新的数据如下:
	
* last\_committed会增加1
* paxos新commit过的k/v值
* 新提议的值内部的事务操作对服务的影响

leader经历的转态转换图:

![img](/assets/img/post/ceph_mon_paxos_1.png)

peon经历的转态转换图:

![img](/assets/img/post/ceph_mon_paxos_2.png)

发现pending\_v和pending_pn只在最开始设置了值后，就一直没有引用，不难猜测，这对值是用来在异常情况下做recover的。

# Recover

除了上面提到的状态外，paxos还有另外三种状态，recovering, updating\_previous和writing\_previous，这些状态都和异常情况下的恢复相关，
leader选举完成后，会进入collect阶段，此时paxos状态为recovering，会尝试恢复paxos数据(注意monitor probing阶段也会sync数据)。因为恢复的时候，
依赖于各自的paxos数据的版本(first\_committed和last\_committed)以及accepted\_pn编号，这个编号在所有monitor中全局唯一且单调递增的，先看编号是怎么产生;

```cpp
version_t Paxos::get_new_proposal_number(version_t gt)
{
  if (last_pn < gt) 
    last_pn = gt; // 保存旧值
  
  // 更新
  last_pn /= 100;
  last_pn++;
  last_pn *= 100;
  last_pn += (version_t)mon->rank;

  .......

  return last_pn;
}
```

在上次的值基础上增加100然后加上monitor的rank值。比如假设三个monitor，rank值分别为0,1,2，最开始pn为100，每次触发选举，假设monitor 0一直存在，
那么每次选举完成后，pn会增加100即，200，300。如果此时monitor 0宕机，那么monitor 1会获胜，pn为401，继续选举为501, 601，如果此时monitor 0恢复，
则pn为700等等。这个值只会在每次选举完成后，leader collect的时候更新一次，后期paxos决议的时候，不会更新。

接下来分两种情况，分别研究paxos的恢复逻辑。

### Peon Down

peon down的时间点不重要，重要的是什么时候leader被重新bootstrap开始选举，当leader没有收到peon的lease ack，
leader的事件lease\_ack\_timeout\_event(时间设定上看，好像accept不会timeout，需要确认)会在超时后执行，然后会进行重新选举，
因为leader还是编号最小的，仍然会选举成为leader（这里只考虑monitor个数大于monmap个数的一半，小于一半集群就不能工作了），
选举完成后会做一次collect操作，进行recover，这里分两方面，即down的时候集群需要恢复到正常工作状态，以后又重新up的时候也需要恢复正常。

> down

peon down隐含的条件是重新选举后leader节点不会发生变化，且其他peon的数据一定不会比leader的数据更新，即

* last\_committed(leader) >= last\_committed(peon)
* accepted\_pn(leader) > accepted\_pn(peon)

另外，timeout事件是在time线程内完成，time线程干活的时候会获取monitor lock，那么可以推断，leader的paxos流程可能被中断的情况包括以下几个点:

1. leader为active状态，未开始任何决议
2. leader为updating状态，即begin函数已经执行，等待accept中，此时leader有uncommitted数据，并且可能已经有部分accept消息
3. leader为writing状态，说明已经接收到所有accept消息，即commit\_start已经开始执行，事务已经排队等待执行
4. leader为writing状态，写操作已经执行完成，即事务已经生效，只是回调函数(commit\_finish)还没有被执行(回调函数没被执行是因为需要获取monitor lock的锁)

之所以会有3和4两种情况，是因为leader节点采用异步写的机制。leader不会被中断在refresh状态，因为一旦commit\_finish函数开始执行，
会将refresh状态执行完成，重新回到active状态，time线程才可能获取到锁执行。第1种情况不用特殊处理，第2种情况会存在uncommitted数据，
待重新选举完成后，leader会重新开始一个propose过程。第3和4种情况会等待已经在writing状态的数据commit完成后，才会重新选举:

```cpp
void Monitor::bootstrap()
{
  wait_for_paxos_write(); // 等待writing的数据完成
  ......
}

void Monitor::start_election()
{
  wait_for_paxos_write(); // 等待writing的数据完成
  ......
}
```

无论如何，一轮消息过后: collect -> handle\_collect -> handle\_last，数据就应该同步好。

> up 

peon重新up后，probing阶段会先sync数据，然后发起选举，这会导致其他节点也发起选举，leader仍然会获胜，且leader被中断的时机和上面情况类似，数据恢复也一样。

### Leader Down

leader可能core在paxos任意函数的任意时间点，这时候新的leader会从peon中选择一个编号最小的，peon的数据根据leader core的位置会有些变化，
还是分down的时候和又重新up的情况来分析:

> down

peon在lease超时后会重新选举，peon可能中断在active或updating状态，peon之间的状态并不是一样的，可能一些在active，一些在updating:

1. leader down在active状态，不需要特殊处理
2. leader down在updating状态，如果没有peon已经accept，不需要特殊处理，如果有peon已经accept，新的leader要么自己已经accept，要么会从其他peon学习到，会重新propose
3. leader down在writing状态，说明所有peon已经accept，新的leader会重新propose已经accept的值(此时down的leader可能已经写成功，也可能没有写成功)
4. leader down在refresh状态，down的leader已经写成功，如果有peon已经收到commit消息，新的commit会被新的leader在collect阶段学习到，如果没有peon收到commit消息，会重新propose

上面的3和4两种情况，意味着正在propose的数据一定会被重新propose（所有peon都没有收到commit消息的情况），
所以不用担心leader已经commit过数据了，而peon还没有commit数据的情况。

> up

leader重新up后，可能probing阶段就会做一次sync，此时数据可能会同步一部分，再一次被选举成leader，collect阶段会同步差异的几个版本数据，
同时，如果peon有uncommitted的数据，也会同步给leader，由新的leader重新propose。

唯一需要注意的是，leader down的时候存在的uncommitted的数据，由上面的情况可知，如果有peon已经接受，数据会被重新propose，
重新up后，根据pending\_v，由于版本较低，pending数据会被抛弃。如果leader已经commit过，peon也一定会commit，所以不会导致数据不一致。

另外，前面提到的updating\_previous状态，发生在新的leader学习到uncommitted值再次propose的情况，相应地，updating\_previous的下一阶段会进入writing\_previous，
所以leader节点最终的状态转换图如下:

![img](/assets/img/post/ceph_mon_paxos_3.png)

对于存在连续的宕机情况，只要存活的monitor的个数超过monmap一半，数据恢复不外乎就是上面这些情况的组合。
