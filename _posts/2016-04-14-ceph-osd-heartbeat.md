---
layout: post
title: "Ceph OSD Heartbeat"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

大规模分布式系统中，各种异常情况时有发生，如系统宕机，网络故障，磁盘损坏等等都有可能造成集群内部节点无法通信。
一个分布式系统要正常协调地运转，内部各节点进程间需要通过心跳机制来保证各节点处于正常工作状态，一旦发现故障，及时响应。

本文简单对ceph osd 进程间的心跳机制加以分析。

# HeartBeat Messenger

进程间心跳消息，需要通过ceph网络层传输，对于ceph网络层的处理，可以参考[这篇文章](http://blog.wjin.org/posts/ceph-async-messenger.html)。
在osd进程启动的过程中，创造了三个messenger用于心跳通信，参考文件ceph-osd.cc:

```cpp
int main(int argc, const char **argv) 
{
  ......

  Messenger *ms_hbclient = Messenger::create(g_ceph_context, g_conf->ms_type, // 发送ping心跳的messenger
					     entity_name_t::OSD(whoami), "hbclient",
					     getpid());
  Messenger *ms_hb_back_server = Messenger::create(g_ceph_context, g_conf->ms_type, // 接收来自back地址的ping心跳
						   entity_name_t::OSD(whoami), "hb_back_server",
						   getpid());
  Messenger *ms_hb_front_server = Messenger::create(g_ceph_context, g_conf->ms_type, // 接收来自front地址的ping心跳
						    entity_name_t::OSD(whoami), "hb_front_server",
						    getpid());
  ......
}
```
因为每个osd进程地位是完全对等的，一方面它需要主动发送心跳ping message到其他节点，另一方面，它也会收到其他节点发来的ping message。
所以他们的通信方式是: `ms_hbclient <-> ms_hb_back_server` 和 `ms_hbclient <-> ms_hb_front_server`。

在部署ceph的时候，一般会使用两个网卡，两个地址back和front, 将纵向和横向流量分开，所以osd进程使用两个messenger分别监听来自back和front的心跳消息。
同时要注意的是，osd启动的时候，会将自己的back和front地址告诉monitor，这些信息都存放在osdmap里面，
其他节点可以通过osdmap来找到监听地址，创建连接，然后进行心跳消息的通信。

虽然front地址是供客户端连接集群使用的，但是这里并不是和客户端进行心跳，osd集群内部检查front地址是否可用也是合理的，
可以看作osd将自己作为客户端，去检测连接集群是否正常，防止客户端连接不上集群。

# Send

osd使用单独的线程来发送心跳：

```cpp
void OSD::heartbeat_entry()
{
  Mutex::Locker l(heartbeat_lock);
  if (is_stopping())
    return;

  while (!heartbeat_stop) {
    heartbeat(); // 发送消息
	......
  }
}

void OSD::heartbeat()
{
  ......

  // 遍历所有peers，发送心跳，peers集合的选取需要遵循一定规则
  for (map<int,HeartbeatInfo>::iterator i = heartbeat_peers.begin();
       i != heartbeat_peers.end();
       ++i) {
    int peer = i->first;
    i->second.last_tx = now;
    if (i->second.first_tx == utime_t())
      i->second.first_tx = now;
    i->second.con_back->send_message(new MOSDPing(monc->get_fsid(), // 向back地址发送
					  service.get_osdmap()->get_epoch(),
					  MOSDPing::PING,
					  now));

    if (i->second.con_front)
      i->second.con_front->send_message(new MOSDPing(monc->get_fsid(), // 向front地址发送
					     service.get_osdmap()->get_epoch(),
						     MOSDPing::PING,
						     now));
  }
  ......
}
```

可以看出，心跳的发送流程是很简单的，也是很独立的。在设计分布式系统的时候，为了保证集群的内部状态正确，应尽量不要引入过多复杂的因素影响心跳的流程。
毕竟心跳快速正确的处理是确保集群运转正常的最基本条件。

# Receive

对心跳消息的处理，osd采用单独的dispatcher类:

```cpp
struct HeartbeatDispatcher : public Dispatcher {
    OSD *osd;
    HeartbeatDispatcher(OSD *o) : Dispatcher(cct), osd(o) {}
    bool ms_dispatch(Message *m) {
      return osd->heartbeat_dispatch(m); // 消息处理函数
    }
	......
} heartbeat_dispatcher;

int OSD::init()
{
  ......

  // 初始化的时候注册dispatcher，收到消息后才知道怎么处理
  hbclient_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_front_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_back_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);

  ......
}
```

当收到消息后，会通过messenger内部的dispatch线程调用事先加入的dispatcher:

```cpp
bool OSD::heartbeat_dispatch(Message *m)
{
  switch (m->get_type()) {
  case CEPH_MSG_PING:
    m->put();
    break;

  case MSG_OSD_PING:
    handle_osd_ping(static_cast<MOSDPing*>(m)); // 处理心跳
    break;

  case CEPH_MSG_OSD_MAP: // 这个消息在heartbeat messenger内部是不会产生的
    {
      ConnectionRef self = cluster_messenger->get_loopback_connection();
      self->send_message(m);
    }
    break;

  default:
    m->put();
  }

  return true;
}

void OSD::handle_osd_ping(MOSDPing *m)
{
  ......

  switch (m->op) {

  case MOSDPing::PING: // 处理心跳消息
    {

	  ......

      // 当进程内部状态不正确的时候，丢弃心跳消息，此时处理心跳已经变得没有意义
	  // 很多线程池会设置timeout时间，如果超时状态就会是unhealthy
      if (!cct->get_heartbeat_map()->is_healthy()) {
		break;
      }

      Message *r = new MOSDPing(monc->get_fsid(),
				curmap->get_epoch(),
				MOSDPing::PING_REPLY, // 注意是PING_REPLY
				m->stamp);
      m->get_connection()->send_message(r); // 发送回包
	  ......

    }
    break;

  case MOSDPing::PING_REPLY: // 处理心跳回包
    {
	  ......
	
	  // 更新时间戳，避免心跳超时
	  // osd有专门的tick线程进行周期性的检查，如果发现有心跳超时的，就会上报monitor
      map<int,HeartbeatInfo>::iterator i = heartbeat_peers.find(from);
      if (i != heartbeat_peers.end()) {
		if (m->get_connection() == i->second.con_back) {
			i->second.last_rx_back = m->stamp;
		if (i->second.con_front == NULL)
			i->second.last_rx_front = m->stamp;
		} else if (m->get_connection() == i->second.con_front) {
			i->second.last_rx_front = m->stamp;
		}
      }

    }
    break;
	......
}
```

# Check

对心跳是否超时的检查，一方面发送线程发送消息后会检查一下，另外还有专门的tick线程，也会检查心跳是否超时:

```cpp
void OSD::tick()
{
	......

    heartbeat_lock.Lock();
    heartbeat_check(); // 检查心跳是否超时
    heartbeat_lock.Unlock();

	......
}

void OSD::heartbeat_check()
{
  ......

  for (map<int,HeartbeatInfo>::iterator p = heartbeat_peers.begin();
       p != heartbeat_peers.end();
       ++p) {
	if (p->second.is_unhealthy(cutoff)) { // 检测超时
      if (p->second.last_rx_back == utime_t() ||
			p->second.last_rx_front == utime_t()) {
		failure_queue[p->first] = p->second.last_tx; // 插入队列，等待上报给monitor
      } else {
		failure_queue[p->first] = MIN(p->second.last_rx_back, p->second.last_rx_front);
      }
    }
  }
}
```

心跳超时上报的时候，也是在tick线程内完成:

```cpp
void OSD::do_mon_report()
{
  ......
  send_failures();
  ......
}

void OSD::send_failures()
{
  ......
  while (!failure_queue.empty()) {
    int osd = failure_queue.begin()->first;
    int failed_for = (int)(double)(now - failure_queue.begin()->second);
    entity_inst_t i = osdmap->get_inst(osd);
    monc->send_mon_message(new MOSDFailure(monc->get_fsid(), i, failed_for, osdmap->get_epoch())); // 向monitor发送消息，报告osd心跳超时
    failure_pending[osd] = i;
    failure_queue.erase(osd);
  }
  ......
}
```

当monitor收到消息后，会对消息进行处理，如果达到了阈值，就会通过paxos算法将osd标记为down，更新osdmap，并通知相关peers。

# Peer

心跳的收发都很简单，需要注意的是，一个osd怎么知道需要和哪些节点进行心跳？肯定不能是其他所有节点，这样集群内部心跳的开销就太大了。
所以，选取心跳的peer也得根据一些规则，主要实现是在下面这个函数:

```cpp
void OSD::maybe_update_heartbeat_peers()
{
  if (is_waiting_for_healthy()) { // 在osd启动的过程中，或者在osd收到更新osdmap的消息，osd状态可能变为waiting，此时需要更新peers集合
    utime_t now = ceph_clock_now(cct);
    if (last_heartbeat_resample == utime_t()) { // 第一次设置需要更新，这时候应该是osd刚启动
      last_heartbeat_resample = now;
      heartbeat_set_peers_need_update(); // 设置需要更新peers标志
    } else if (!heartbeat_peers_need_update()) { // 后续更新，应该是收到osdmap变更的消息
      utime_t dur = now - last_heartbeat_resample;
      if (dur > cct->_conf->osd_heartbeat_grace) { // 仅仅在超出grace时间后才更新，因为超过grace，osdmap的变更才可能导致pgmap变化
		heartbeat_set_peers_need_update(); // 设置需要更新peers标志
		last_heartbeat_resample = now;
		reset_heartbeat_peers();   // we want *new* peers!
      }
    }
  }

  Mutex::Locker l(heartbeat_lock);
  if (!heartbeat_peers_need_update())
    return; // 不需要更新直接返回
  heartbeat_need_update = false;

  heartbeat_epoch = osdmap->get_epoch();
  if (is_active()) { // 需要osd状态是active，不然更新没意义
    RWLock::RLocker l(pg_map_lock);
    for (ceph::unordered_map<spg_t, PG*>::iterator i = pg_map.begin(); // 遍历osd负责的所有pg
	 i != pg_map.end();
	 ++i) {
      PG *pg = i->second;
      pg->heartbeat_peer_lock.Lock();

      for (set<int>::iterator p = pg->heartbeat_peers.begin(); // 遍历pg对应的peers
	   p != pg->heartbeat_peers.end();
	   ++p)
		if (osdmap->is_up(*p)) // 如果为up，则加入心跳集合
			_add_heartbeat_peer(*p);

      for (set<int>::iterator p = pg->probe_targets.begin(); // 遍历probe目标集合
	   p != pg->probe_targets.end();
	   ++p)
		if (osdmap->is_up(*p)) // 如果为up，则加入心跳集合
			_add_heartbeat_peer(*p);

      pg->heartbeat_peer_lock.Unlock();
    }
  }

  // 后面流程就比较简单
  // 1) 加入'仅挨着'当前osd编号的下一个和上一个为up的节点
  // 2) 删除down的节点
  // 3) 对peers集合做调整
  ......
}
```

什么时候需要更新peers集合，也即这个函数什么时候会被调用？从实现看，影响peers集合主要是pgmap的变化，那什么时候pgmap可能改变呢？

* pg创建的时候，参考函数handle\_pg\_create

* osdmap变更的时候，osd承载的pg可能需要重新peering，导致osd状态可能会变为STATE\_WAITING\_FOR\_HEALTHY，参考函数handle\_osd\_map

* tick线程中周期性的检查，主要是因为osd启动过程中，会load\_pg，类似第二条

还有需要注意，设置peers更新标记，不仅仅是在这个函数内部，在pg peering状态机运作的过程中，会更新标记:

```cpp
void PG::update_heartbeat_peers()
{
  ......
	  
  bool need_update = false;
  heartbeat_peer_lock.Lock();
  if (new_peers == heartbeat_peers) {
  } else {
    heartbeat_peers.swap(new_peers);
    need_update = true; // 需要update
  }

  if (need_update)
    osd->need_heartbeat_peer_update(); // 更新
}
```

总结一下就是，osd启动或者异常退出，monitor会收到消息，然后进行paxos，将结果会反应在osdmap上，进而通知相关osd进程，
osd进程收到消息后，会处理map的变更，可能导致pg重新peering。monitor也会收到创建pool或修改pg\_num的消息，最终会导致创建pg，
osd收到消息创建pg，也会导致peering。osd启动的过程中，load\_pg也会导致peering，一旦有peering发生，osd进程的状态就是STATE\_WAITING\_FOR\_HEALTHY，
就可能导致更新peer集合。

# Optimization

发送心跳采用单独的线程，目前来看没有什么好优化的（社区好像有提议希望将心跳信息附带在op内部，不过还只是草案）。

对于收到消息后的分发，本人有一个优化的patch，见[pr8808](https://github.com/ceph/ceph/pull/8808)。
主要是在大规模集群的情况下，鉴于目前simple messenger导致线程数过多，messenger内部dispatch线程可能由于system schedule会被delay，
也有可能因为与很多心跳线程竞争dispatch queue lock而失败导致睡眠，从而导致处理心跳的回调变慢，进而超时。pr8808通过将心跳消息fast dispatch后，
减少了消息需要进入dispatch队列的竞争。

另外，在async messenger情况下，虽然连接线程数减少了，但是存在另外的问题，
因为进程中所有的async messenger共用workerpool，如果所有worker线程因为竞争锁而被block住，则系统无法进行消息的dispatch，心跳消息的处理也会block住，
参见[bug15758](http://tracker.ceph.com/issues/15758#change-70168)。


还有一个需要注意的是，心跳messenger内部是不会收到osdmap更新的消息的，见[pr8831](https://github.com/ceph/ceph/pull/8831)。

# Tuning

```text
osd_heartbeat_grace
osd_heartbeat_interval
```

大规模部署情况下，压测的时候（比如1000 vm 跑fio)，心跳可能会出问题，可能需要将grace时间调大，避免误报，
如果调大后，interval也应相应增大，避免发送频率太高，保证至少发送过3次心跳后没有回包才上报，比如目前grace为20，
interval为6，如果grace为30，则interval建议为9比较合适。

grace调大，也有副作用，如果某个osd异常退出，等待其他osd上报的时间必须为grace，在这段时间段内，这个osd负责的pg的io会hang住。
可以采用我之前的优化的patch，尽量不要将grace调的太大。

如果集群存在大规模的顺序读写，网络成为瓶颈的时候，可以通过下面这个参数调高心跳消息在内核网络层的优先级:

```text
osd_heartbeat_use_min_delay_socket
```

另外还需要注意，在压测的过程中，osd内部如果有线程池timeout，会导致心跳数据报的丢失，所以很多线程池的timeout时间应做调整，
线程池的timeout由以下map管理:

```cpp
class HeartbeatMap {
 public:
  // register/unregister
  heartbeat_handle_d *add_worker(std::string name); // 注册handler
  void remove_worker(heartbeat_handle_d *h);

 private:
  CephContext *m_cct;
  RWLock m_rwlock;
  time_t m_inject_unhealthy_until;
  std::list<heartbeat_handle_d*> m_workers; // 注册的所有timeout handler集合

  .....
  bool _check(heartbeat_handle_d *h, const char *who, time_t now); // 检查是否超时
};

struct heartbeat_handle_d {
  std::string name;
  atomic_t timeout, suicide_timeout;
  time_t grace, suicide_grace; // 超过grace时间，表示timeout；超过suicide_grace，进程会退出
  std::list<heartbeat_handle_d*>::iterator list_item;

  heartbeat_handle_d(const std::string& n)
    : name(n), grace(0), suicide_grace(0)
  { }
};
```
