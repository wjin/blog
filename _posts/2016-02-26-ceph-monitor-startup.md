---
layout: post
title: "Ceph Monitor Startup"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}


# Introduction

monitor进程启动过程，主要会经历下面这个状态转换:

![img](/assets/img/post/ceph_mon_startup.png)

monior初始化完成后，会首先进入状态probing，询问其他monitor的信息，当收到应答消息后，会检查自己的数据决定是否需要同步数据。
因为monitor可能是新建的，加入集群的时候需要同步已有集群monitor的数据，如果monitor已经down掉很久，也需要同步数据。

数据同步好后(注意这里数据并不是完全一致，后续可能还需要paxos算法进行恢复，达到数据一致)，就可以加入quorum，发起选举，选举完成后，
如果胜出，就会变成leader状态，否则peon状态。成功与否取决于monitor在map中的rank值，较小的获胜。

# Main Thread

主线程的初始化工作，参见代码ceph\_mon.cc:

```cpp
int main(int argc, const char **argv) 
{
	......

	// 创建存储monitor数据的store，后端主要是用k/v形式存储
    MonitorDBStore store(g_conf->mon_data);
    int r = store.create_and_open(cerr);

	// 获取monitor map信息
    MonMap monmap;
    bufferlist mapbl;
    int err = obtain_monmap(*store, mapbl); // 获取map

	// 获取监听的地址, 第一次会mkfs，然后生成一个map
	// 以后直接从map里获取，所以如果第一次配置错误，修改配置文件，重启monitor进程是没有用的
    entity_addr_t ipaddr;
    if (monmap.contains(g_conf->name.get_id())) {
        ipaddr = monmap.get_addr(g_conf->name.get_id());
	} else {
		......
	}

	// 创建用于消息通信的messenger
	Messenger *msgr = Messenger::create(g_ceph_context, g_conf->ms_type,
	    entity_name_t::MON(rank), "mon", 0);

	// 绑定端口
	err = msgr->bind(ipaddr);

	// monitor实现的具体类
    mon = new Monitor(g_ceph_context, g_conf->name.get_id(), store,
		    msgr, &monmap);

	// 初始化
    err = mon->preinit();
    
	// messenger开始接收消息
	msgr->start();

	// 继续初始化
    mon->init();

	// 等待结束
    msgr->wait();

	// 关闭store
    store->close();

	......
}
```

需要注意的是，在部署的时候，会通过mkfs生成monitor map，存储在monitor的后端存储中，以后进程如果重启，是不需要读取配置的，避免出现未知错误。
使用过程中，经常有同事将配置写错，比如public 和 cluster 地址写反，修改配置后重启进程，这样是达不到修改效果的。
另外，由于monitor map特别重要，在monitor启动的过程中，如果需要sync数据，会先备份一份再修改，防止sync中途出错，再次重启后如果没有monitor map，monitor就没法启动。


# Monitor Class

从主线程中的流程看，初始化过程主要集中在类Monitor的preinit和init函数中，先看看Monitor类的构造函数然后分析这两个函数:

## Constructor

```cpp
Monitor::Monitor(CephContext* cct_, string nm, MonitorDBStore *s,
		 Messenger *m, MonMap *map) :
{
  ......

  // paxos算法实现
  paxos = new Paxos(this, "paxos");

  // 借助于paxos实现的不同服务
  paxos_service[PAXOS_MDSMAP] = new MDSMonitor(this, paxos, "mdsmap");
  paxos_service[PAXOS_MONMAP] = new MonmapMonitor(this, paxos, "monmap");
  paxos_service[PAXOS_OSDMAP] = new OSDMonitor(this, paxos, "osdmap");
  paxos_service[PAXOS_PGMAP] = new PGMonitor(this, paxos, "pgmap");
  paxos_service[PAXOS_LOG] = new LogMonitor(this, paxos, "logm");
  paxos_service[PAXOS_AUTH] = new AuthMonitor(this, paxos, "auth");

  health_monitor = new HealthMonitor(this); // 主要监控monitor的数据存储空间变化情况，查看磁盘是否满了
  config_key_service = new ConfigKeyService(this, paxos); // 存储一些用户自定义的k/v数据
  ......
}
```

## Preinit

接着是preinit函数:

```cpp
int Monitor::preinit()
{
  .......

  init_paxos();
  health_monitor->init();

  ......
}

void Monitor::init_paxos()
{
  paxos->init(); // 读取paxos算法的关键信息

  for (int i = 0; i < PAXOS_NUM; ++i) {
    paxos_service[i]->init(); // 只有LogMonitor重载了init函数，其他服务没什么需要初始化的
  }

  refresh_from_paxos(NULL); // 更新各服务信息
}

void Paxos::init()
{
  last_pn = get_store()->get(get_name(), "last_pn"); // 上次提议的编号
  accepted_pn = get_store()->get(get_name(), "accepted_pn"); // 已经接受的最大编号
  last_committed = get_store()->get(get_name(), "last_committed"); // 最后一次commit的版本
  first_committed = get_store()->get(get_name(), "first_committed"); // 第一次commit的版本
}
```

commit版本决定了是否需要向其他monitor sync数据。last\_pn和accepted\_pn主要用于paxos解决多个monitor数据一致性。

针对每个服务，PaxosService基类提供模板方法refresh和post\_refresh，进行服务的更新，对于每个派生类（paxos服务），具体需要更新的信息派生类自己实现抽象函数:

```cpp
void Monitor::refresh_from_paxos(bool *need_bootstrap)
{
  ......

  for (int i = 0; i < PAXOS_NUM; ++i) {
    paxos_service[i]->refresh(need_bootstrap); // 调用模板方法更新
  }
  for (int i = 0; i < PAXOS_NUM; ++i) {
    paxos_service[i]->post_refresh(); // 调用模板方法，更新后的处理
  }
}

void PaxosService::refresh(bool *need_bootstrap)
{
  // 版本比较关键，决定是否需要更新
  cached_first_committed = mon->store->get(get_service_name(), first_committed_name);
  cached_last_committed = mon->store->get(get_service_name(), last_committed_name);

  ......

  update_from_paxos(need_bootstrap); // 各服务实现自己的需求
}

void PaxosService::post_refresh()
{
  post_paxos_update(); // 各服务实现自己的需求

  if (mon->is_peon() && !waiting_for_finished_proposal.empty()) {
    finish_contexts(g_ceph_context, waiting_for_finished_proposal, -EAGAIN);
  }
}
```

preinit主要是将Paxos和PaxosService等服务进行了初始化，读取了上次记录在store中的数据，为此monitor和其他monitor互动做好准备。

## Init

接下来main thread初化messenger，准备消息的收发，然后调用init函数，和其他monitor进行互动:

```cpp

int Monitor::init()
{
  lock.Lock();

  timer.init(); // 初始化timer线程
  new_tick(); // 加入time事件

  // i'm ready!
  messenger->add_dispatcher_tail(this);

  bootstrap(); // 启动

  ......
  lock.Unlock();
  return 0;
}
```

bootstrap从名字上看，就可以知道是引导monitor正确启动的入口，在monitor进程运行的过程中，如果出现一些信息不对称或不全的情况，就会调用此函数重新启动，
因为重启过程中会sync数据:

```cpp
void Monitor::bootstrap()
{
  wait_for_paxos_write();

  sync_reset_requester();
  unregister_cluster_logger();
  cancel_probe_timeout();

  // 设置状态
  state = STATE_PROBING;

  _reset(); // 重置paxos及其服务

  // 只有一个monitor，没必要联系其他monitor进行leader选举
  if (monmap->size() == 1 && rank == 0) {
    win_standalone_election();
    return;
  }

  ......
  // 发送消息，收集信息
  for (unsigned i = 0; i < monmap->size(); i++) {
    if ((int)i != rank)
      messenger->send_message(new MMonProbe(monmap->fsid, MMonProbe::OP_PROBE, name, has_ever_joined),
			      monmap->get_inst(i));
  }

  ......
}
```

接下来的流程就是根据收到的消息进行状态转换:

```cpp
void Monitor::handle_probe_reply(MMonProbe *m)
{
    ......

	// 同步数据
    if (paxos->get_version() + g_conf->paxos_max_join_drift < m->paxos_last_version) {
      cancel_probe_timeout();
      sync_start(other, false);
      m->put();
      return;
    }

	// 满足条件，开始选举
    unsigned need = monmap->size() / 2 + 1;
    if (outside_quorum.size() >= need) {
      if (outside_quorum.count(name)) {
        start_election();
      }
    }

	......
}
```

如果commit版本有差异，就会同步数据，同步完成后会再一次bootstrap，然后probing，接着就会发起选举的消息，获胜变为leader，失败变为peon。
选举流程可以参考[ceph leader elect](http://blog.wjin.org/posts/ceph-monitor-leader-elect.html)，
选举完成后paxos算法需要做数据恢复，参考[ceph paxos](http://blog.wjin.org/posts/ceph-monitor-paxos.html)。

# Dispatch

待数据恢复完成后，说明数据已经完全一致，monitor就进入工作状态，准备收发消息了，从启动流程看，monitor不像osd进程那样，会启动很多的线程池来工作。
main thread发送probe相关消息后就开始wait，等待进程退出了。后期的主要工作就是根据收到的消息，进行相应的处理，根据ceph网络层面的实现，
后期所有的处理几乎都是在dispatch线程内完成的:

```cpp
  bool ms_dispatch(Message *m) {
    lock.Lock();
    _ms_dispatch(m);
    lock.Unlock();
    return true;
  }
```

在分发消息的时候，首先还是获取了monitor内部的锁，初始化的时候只创建了一个messenger，那么只会有一个dispatch线程(这里只考虑SimpleMessenger, 
AsyncMessenger会有多个worker线程进行分发)，怎么还需要这把锁？其实这里主要是防止和time线程竞争。

monitor单独有个time线程处理所有的超时，分布式系统，发送消息出去后，相应的消息未必会及时收到，可能网络出问题，也可能对端已经挂了，也可能被对端忽略了，
直接无视，所以设置timeout后怎么处理是非常关键的。在Elector和Paxos算法实现中的timeout处理均是通过Monitor内部的time线程完成。

后续Monitor, Elector以及Paxos的状态演进，要么是通过dispatch线程收到消息后的处理，要么是超时后time线程的处理，通过Monitor内部的锁进行互斥，
所以在Elector和Paxos及其服务的实现中，没有任何锁的机制。dispatch线程收到不同消息后，属于Monitor类自己的消息，自己处理，如果不行，
再转发给Elector, Paxos或PaxosService等进行处理。发生timeout事件的时候，time线程多数情况下需要做一下清理工作，然后可能重新bootstrap。

# Summary

* 获取monitor map，为后续monitor进程间通信做好准备

* 初始化paxos，加载一些paxos变量，主要是commit过的版本号

* bootstrap，然后进入probing阶段，此时可能根据commit的版本号进行数据同步

* leader选举

* 选举完成后，leader/peon分别初始化paxos算法

* paxos的leader通过collect阶段做数据恢复

* leader/peon变为active


