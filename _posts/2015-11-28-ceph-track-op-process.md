---
layout: post
title: "Ceph Track Op Process"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

在ceph存储系统中，后端rados集群的osd进程，会对收到的消息进行解析，如果是op类型的操作，就会新建一个OpRequest的对象，
这个对象会贯穿整个操作的执行过程，直至完成销毁，ceph提供了一种机制来跟踪这个对象，记录一些事件，帮助我们进行性能分析与调优。

# Implementation

## Initialize OpTracker

在类OSD的实现中，定义了一个OpTracker类的对象op_tracker，这个成员具有管理TrackedOp类对象的功能：

```cpp
class OSD : public Dispatcher,
	    public md_config_obs_t {

  ......

  OpTracker op_tracker; // 管理TrackedOp对象的辅助类

  ......
};

// OSD构造函数
OSD::OSD(CephContext *cct_, ObjectStore *store_,
	 int id,
	 Messenger *internal_messenger,
	 Messenger *external_messenger,
	 Messenger *hb_clientm,
	 Messenger *hb_front_serverm,
	 Messenger *hb_back_serverm,
	 Messenger *osdc_messenger,
	 MonClient *mc,
	 const std::string &dev, const std::string &jdev) :
  Dispatcher(cct_),
  ......
  op_tracker(cct, cct->_conf->osd_enable_op_tracker,  // 第一个参数是CephContext，第二个是配置文件是否激活tracker，如果没激活，就不会跟踪op
                  cct->_conf->osd_num_op_tracker_shard), // 第三个参数是分片大小，内部实现的时候做了shard，避免所有op对象记录在同一个链表中，导致性能瓶颈
  ......
{

  monc->set_messenger(client_messenger);
  op_tracker.set_complaint_and_threshold(cct->_conf->osd_op_complaint_time, // 容忍处理op的最长时间，超过后，会打印警告日志
                                         cct->_conf->osd_op_log_threshold); // 警告日志个数上限
  op_tracker.set_history_size_and_duration(cct->_conf->osd_op_history_size, // 设置OpHistory的大小
                                           cct->_conf->osd_op_history_duration); // 设置OpHistory中op最长停留时间
}
```

OpTracker是用来管理TrackedOp对象的，它不仅管理正在进行中的op(inflight op)，也跟踪记录已经完成的op(通过OpHistory类对象)，
需要的时候，可以向osd进程发送命令dump过去已经完成的op。

很明显，OpRequest类需要继承TrackedOp类，以便被OpTracker类管理：

```cpp
struct OpRequest : public TrackedOp {
  friend class OpTracker;
  ......
};
```

## OpHistory

先看OpHistory类是怎么记录已经完成的op的，在文件src/common/TrackedOp.h和src/common/TrackedOp.cc文件中:

```cpp
// 头文件
class OpHistory {
  set<pair<utime_t, TrackedOpRef> > arrived; // 按到达时间排序
  set<pair<double, TrackedOpRef> > duration; // 按停留时间排序

  Mutex ops_history_lock; // 锁保护上面两个集合

  void cleanup(utime_t now); // 删除不满足条件的op
  bool shutdown;

  // 不能是无限制的全部记录op，需要按照如下两个条件清理
  uint32_t history_size; // op总个数
  uint32_t history_duration; // op最长停留时间

public:
  OpHistory() : ops_history_lock("OpHistory::Lock"), shutdown(false),
  history_size(0), history_duration(0) {}
  ~OpHistory() {
    assert(arrived.empty());
    assert(duration.empty());
  }
  void insert(utime_t now, TrackedOpRef op); // 插入新的op，会调用cleanup清理多余的
  void dump_ops(utime_t now, Formatter *f);
  void on_shutdown();
  void set_size_and_duration(uint32_t new_size, uint32_t new_duration) {
    history_size = new_size;
    history_duration = new_duration;
  }
};

// cpp文件
void OpHistory::insert(utime_t now, TrackedOpRef op)
{
  if (shutdown)
    return;

  Mutex::Locker history_lock(ops_history_lock);
  duration.insert(make_pair(op->get_duration(), op)); // 插入集合duration
  arrived.insert(make_pair(op->get_initiated(), op)); // 插入集合arrived
  cleanup(now); // 删除多余的
}

void OpHistory::cleanup(utime_t now)
{
  while (arrived.size() &&
	 (now - arrived.begin()->first >
	  (double)(history_duration))) { // 时间超时
    duration.erase(make_pair(
	arrived.begin()->second->get_duration(),
	arrived.begin()->second));
    arrived.erase(arrived.begin());
  }

  while (duration.size() > history_size) { // 个数超额
    arrived.erase(make_pair(
	duration.begin()->second->get_initiated(),
	duration.begin()->second));
    duration.erase(duration.begin());
  }
}
```

很简单的set集合管理TrackedOp对象，因为这些都是已经执行完成后的对象，插入和删除set集合的元素不需要做shard，不影响性能。

## OpTracker

接着看看真正的管理类OpTracker:

```cpp
class OpTracker {
  // 这个类用来给智能指针设置删除器对象
  class RemoveOnDelete {
    OpTracker *tracker;
  public:
    RemoveOnDelete(OpTracker *tracker) : tracker(tracker) {}
    void operator()(TrackedOp *op);
  };

  friend class RemoveOnDelete;
  friend class OpHistory;

  atomic64_t seq; // 递增的原子计数器，每个op唯一，用此值做shard

  // 对inflight op对象做了shard，避免竞争，hammer版本才新加的，性能瓶颈
  // 每一个分片的信息
  struct ShardedTrackingData {
    Mutex ops_in_flight_lock_sharded; // 保护xlist
    xlist<TrackedOp *> ops_in_flight_sharded; // 记录op的链表
    ShardedTrackingData(string lock_name):
        ops_in_flight_lock_sharded(lock_name.c_str()) {}
  };
  vector<ShardedTrackingData*> sharded_in_flight_list; // 分片的数组
  uint32_t num_optracker_shards; // 分片大小

  OpHistory history; // 借助类OpHistory管理已经执行完成的op

  float complaint_time; // op警告日志时间上限
  int log_threshold; // op警告日志条数上限

  void _mark_event(TrackedOp *op, const string &evt, utime_t now); // 记录事件

public:
  bool tracking_enabled; // 标志是否enable track
  CephContext *cct;
  ......
};
```

先看构造/析构函数，主要就是对分片进行初始化和销毁：

```cpp
  OpTracker(CephContext *cct_, bool tracking, uint32_t num_shards) : seq(0), 
                                     num_optracker_shards(num_shards),
				     complaint_time(0), log_threshold(0),
				     tracking_enabled(tracking), cct(cct_) {
    for (uint32_t i = 0; i < num_optracker_shards; i++) {
      char lock_name[32] = {0};
      snprintf(lock_name, sizeof(lock_name), "%s:%d", "OpTracker::ShardedLock", i);
      ShardedTrackingData* one_shard = new ShardedTrackingData(lock_name); // 初始化一个分片信息
      sharded_in_flight_list.push_back(one_shard); // 放在数组中
    }
  }
     
  ~OpTracker() {
    while (!sharded_in_flight_list.empty()) {
      assert((sharded_in_flight_list.back())->ops_in_flight_sharded.empty());
      delete sharded_in_flight_list.back(); // 销毁
      sharded_in_flight_list.pop_back();
    }    
  }
```

## Track OP

紧接着看看怎么跟踪op的，在osd收到消息后，会进行分发(两种分发途径），如果是op的请求，就会通过工厂方法create\_request创建OpRequest的对象:

```cpp
// 快速分发路径
void OSD::ms_fast_dispatch(Message *m)
{
  ......
  OpRequestRef op = op_tracker.create_request<OpRequest>(m); // 创建OpRequest
  ......
}

// 通常一般分发路径
void OSD::_dispatch(Message *m)
{
	......
	case MSG_OSD_REP_SCRUB:
		handle_rep_scrub(static_cast<MOSDRepScrub*>(m));
		break;

		// -- need OSDMap --

	default:
		{
			OpRequestRef op = op_tracker.create_request<OpRequest, Message*>(m); // 创建OpRequest
			op->mark_event("waiting_for_osdmap");
			// no map?  starting up?
			if (!osdmap) {
				dout(7) << "no OSDMap, not booted" << dendl;
				waiting_for_osdmap.push_back(op);
				break;
			}

			// need OSDMap
			dispatch_op(op);
		}
	......
}
```

注意到OpRequest继承自TrackedOp对象，在基类的构造函数中，会将OpRequest对象添加到OpTracker的链表中进行管理：

```cpp
TrackedOp(OpTracker *_tracker, const utime_t& initiated) :
    xitem(this),
    tracker(_tracker),
    initiated_at(initiated),
    lock("TrackedOp::lock"),
    seq(0),
    warn_interval_multiplier(1)
{
    tracker->register_inflight_op(&xitem); // 注册自己
    events.push_back(make_pair(initiated_at, "initiated")); // 注册初始化的事件
}

// 注册函数
void OpTracker::register_inflight_op(xlist<TrackedOp*>::item *i)
{
  if (!tracking_enabled) // 没有激活跟踪，什么也不干
    return;

  uint64_t current_seq = seq.inc(); // 原子计数器自增
  uint32_t shard_index = current_seq % num_optracker_shards; // 根据seq取模，获取分片索引
  ShardedTrackingData* sdata = sharded_in_flight_list[shard_index]; // 获取分片数据结构

  assert(NULL != sdata);
  {
    Mutex::Locker locker(sdata->ops_in_flight_lock_sharded);
    sdata->ops_in_flight_sharded.push_back(i); // 插入分片
    sdata->ops_in_flight_sharded.back()->seq = current_seq;
  }
}
```

在创建OpRequest对象的时候，会放入智能指针进行管理，但是需要注意智能指针的删除器，删除器中会自动从inflight op的链表中将自己删除，
然后加入OpHistory中继续记录，使得dump命令可以打印历史op：

```cpp
// 创建的时候，T为OpRequest, U为Message*
template <typename T, typename U>
typename T::Ref create_request(U params)
{
	// typename T::Ref 实际上定义为 typedef ceph::shared_ptr<OpRequest> Ref;
	// 就是shared_ptr类型
    typename T::Ref retval(new T(params, this), // 第一个参数是new OpRequest(Message*, OpTracker*)
			   RemoveOnDelete(this)); // 第二个参数给智能指针设置一个删除器对象，销毁时调用这个对象的operator()
    return retval;
}

// 智能指针销毁的函数
void OpTracker::RemoveOnDelete::operator()(TrackedOp *op) {
  op->mark_event("done"); // 标记op完成事件
  if (!tracker->tracking_enabled) { // 没有激活跟踪
    op->_unregistered(); // TrackedOp留下的hook，目前为空函数
    delete op; // 删除指针
    return; // 返回
  }

  tracker->unregister_inflight_op(op); // 激活跟踪，单独处理
}

void OpTracker::unregister_inflight_op(TrackedOp *i)
{
  // caller checks;
  assert(tracking_enabled);

  // 获取分片信息，将自己从分片链表中删除
  uint32_t shard_index = i->seq % num_optracker_shards;
  ShardedTrackingData* sdata = sharded_in_flight_list[shard_index];
  assert(NULL != sdata);
  {
    Mutex::Locker locker(sdata->ops_in_flight_lock_sharded);
    assert(i->xitem.get_list() == &sdata->ops_in_flight_sharded);
    i->xitem.remove_myself();
  }
  i->_unregistered(); // hook

  utime_t now = ceph_clock_now(cct);

  // 插入OpHistory进行管理，并没有析构指针
  // 这里是不需要删除的，继续将最开始动态分配的指针放入shared_ptr中，但是这次没有设置deleter对象，以后引用计数为0会自动调用delete
  history.insert(now, TrackedOpRef(i));
}
```

## Op Warning Info

当一个op处理的时间超过complaint\_time的时候，就会打印warning信息，这是因为在osd进程启动的时候，设置了time事件，会定期check:

```cpp
int OSD::init()
{
	......
		tick_timer.add_event_after(cct->_conf->osd_heartbeat_interval, new C_Tick(this)); // 添加time事件
	......
}

// time事件的callback
class C_Tick : public Context {
	OSD *osd;
	public:
	C_Tick(OSD *o) : osd(o) {}
	void finish(int r) {
		osd->tick(); // 调用OSD::tick
	}
};

void OSD::tick()
{
	check_ops_in_flight(); // 检查

	tick_timer.add_event_after(1.0, new C_Tick(this)); // 新加另外一个事件，所以会持续检查
}

void OSD::check_ops_in_flight()
{
	vector<string> warnings;
	if (op_tracker.check_ops_in_flight(warnings)) { // 有警告信息
		for (vector<string>::iterator i = warnings.begin();
				i != warnings.end();
				++i) {
			clog->warn() << *i; // 打印
		}
	}
	return;
}

bool OpTracker::check_ops_in_flight(std::vector<string> &warning_vector)
{
	// 检查每个分片，看看最老的op有没超过compliant_time
	......
}
```

## Op Event

除了能够自动检查一些耗时特别长的op以外，OpTracker类还提供了一种事件机制，可以在某个时间点标记某种事件的发生，以便以后分析：

```cpp
class TrackedOp {
protected:
  OpTracker *tracker; /// the tracker we are associated with

  list<pair<utime_t, string> > events; // <事件发生的时间，描述信息>
  mutable Mutex lock; // 保护链表

  void mark_event(const string &event); // 标记事件发生

  ......
};

void TrackedOp::mark_event(const string &event)
{
  if (!tracker->tracking_enabled) // 没有enable跟踪，直接返回
    return;

  utime_t now = ceph_clock_now(g_ceph_context);
  {
    Mutex::Locker l(lock);
    events.push_back(make_pair(now, event)); // 放入链表
  }
  tracker->mark_event(this, event); // 继续让OpTracker处理
  _event_marked(); // hook
}

void OpTracker::mark_event(TrackedOp *op, const string &dest, utime_t time)
{
  if (!tracking_enabled)
    return;
  return _mark_event(op, dest, time);
}

void OpTracker::_mark_event(TrackedOp *op, const string &evt,
			    utime_t time)
{
  // 打印事件发生信息，日志级别为5，需要调整日志级别
  dout(5);
  *_dout  <<  "seq: " << op->seq
	  << ", time: " << time << ", event: " << evt
	  << ", op: ";
  op->_dump_op_descriptor_unlocked(*_dout);
  *_dout << dendl;
}
```

如果我们需要分析处理op的主要耗时在什么地方，我们可以在进入某些关键函数之前，调用mark\_event标记事件开始，
然后在结束后再次调用mark\_event标记事件完成，当op完成后，在析构的时候，分析op的所有事件，就可以知道事件耗时多少。

# Summary

* ceph通过OpTracker类来跟踪op的执行过程，必要时打印一些警告信息

* 可以向osd发送dump命令，查看op的信息

* 通过标记事件，可以分析op的处理流程
