---
layout: post
title: "Ceph Scrub Mechanism"
description: ""
category: ceph 
tags: [ceph]
---
{% include JB/setup %}

# Introduction

通常情况下，ceph 采用三副本同步写的策略，维护数据的强一致性。同时，ceph也提供一种机制去检查各个副本之间的数据是否一致，
如果发现不一致就必须repair，这种机制就是scrub。

scrub分为两种，scrub 和 deep scrub。前者只比较object元数据信息，后者会真正读取对象文件内容进行比较，
会造成很大的IO流量。更为严重的是，在scrub的时候，会获取pg的锁，这样会hang住前台IO请求，目前社区正在对这一块进行改进。

ceph osd进程中，由周期性timer线程检查pg是否需要做scrub，另外，也可以通过命令行(ceph pg scrub pgid)触发scrub，
实现的时候主要是设置一个must\_scrub标志位完成，不难看出，scrub的粒度是以pg为单位进行的。

处理的流程主要是在timer线程内调度，在disk_tp线程池内对pg进行处理。

## timer 线程

![image](/assets/img/post/ceph_scrub_timer.png)

1. 以一定概率调度scrub处理函数 (考虑因素包括: pg是primary，scrub配置的时间段，系统当前负载等)

2. primary pg预留slot，并且发送消息让从pg也预留slot 

3. 主从pg均预留slot成功，将pg放入队列scrub\_wq

## disk\_tp线程池

![image](/assets/img/post/ceph_scrub_process.png)

1. 从scrub\_wq取出pg，调用PG::scrub进行处理

2. 随机睡眠一小段时间，进行参数判断，然后进入chunky\_scrub

3. chunky\_scrub采用简单的状态机处理 (获取从pg的scrub map，比较map是否一致）

源码中对状态机的注释比较清楚：

```text
 *           +------------------+
 *  _________v__________        |
 * |                    |       |
 * |      INACTIVE      |       |
 * |____________________|       |
 *           |                  |
 *           |   +----------+   |
 *  _________v___v______    |   |
 * |                    |   |   |
 * |      NEW_CHUNK     |   |   |
 * |____________________|   |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |     WAIT_PUSHES    |   |   |
 * |____________________|   |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |  WAIT_LAST_UPDATE  |   |   |
 * |____________________|   |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |      BUILD_MAP     |   |   |
 * |____________________|   |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |    WAIT_REPLICAS   |   |   |
 * |____________________|   |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |    COMPARE_MAPS    |   |   |
 * |____________________|   |   |
 *           |              |   |
 *           |              |   |
 *  _________v__________    |   |
 * |                    |   |   |
 * |WAIT_DIGEST_UPDATES |   |   |
 * |____________________|   |   |
 *           |   |          |   |
 *           |   +----------+   |
 *  _________v__________        |
 * |                    |       |
 * |       FINISH       |       |
 * |____________________|       |
 *           |                  |
 *           +------------------+
```

需要注意的是，正常情况下(不考虑recovery和pg状态发生变化), PG::scrub也会多次调用才能完成pg的scrub操作。

首先，primary pg发送消息给从pg获取scrub map，然后会到WAIT\_REPLICAS状态，此时由于等待从pg的消息，scrub会暂时执行完毕，
scrub状态记录在scrubber中。当收到从pg发来的map后，pg会重新被放入scrub\_wq队列，等待线程池重新从队列获取并执行，
此时scrubber的状态是WAIT\_REPLICAS，判断成功，继续比较scrub map。

其次，当scrub map 比较完成后，会对对象end进行判断，如果还有新的对象需要做scrub，会将pg重新加入scrub\_wq队列，
而chunky\_scrub会退出循环，完成本次执行，避免scrub执行太长时间，导致pg的其他IO hang住。在实现的时候，进入PG::scrub，
如果配置的sleep时间大于0，还会随机睡眠一下，也是为了缓解scrub占用pg锁的时间太过频繁。

# Code Analysis

## Scrub Schedule 

scrub是一个周期性事件，osd进程会定期调度scrub，如果满足条件就会将pg送入srcub\_wq队列，等待disk\_tp线程执行。
和很多周期性事件一样，它的检查是在OSD进程的timer线程内:

```cpp
// time事件callback入口，会周期性的调用
void OSD::tick()
{
	......
    if (is_active()) { // osd 状态必须是active
		if (!scrub_random_backoff()) { // 以一定概率调度
			sched_scrub(); // 调度scrub
    }

    check_replay_queue();
	}
	......
}

void OSD::sched_scrub()
{
  bool load_is_low = scrub_should_schedule(); // 负载低，在规定时间限制内，会返回true

  pair<utime_t, spg_t> pos;
  if (service.first_scrub_stamp(&pos)) { // 获取一个需要做scrub的pg
    do {
      utime_t t = pos.first;
      spg_t pgid = pos.second;

      utime_t diff = now - t;
      if ((double)diff < cct->_conf->osd_scrub_min_interval) { // 没有超过下限，不做scrub
		break;
      }

      if ((double)diff < cct->_conf->osd_scrub_max_interval && !load_is_low) { // 超过下限，但是还未到上限，load_is_low不满足也不做scrub
		break;
      }

	  // 满足scrub条件
      PG *pg = _lookup_lock_pg(pgid); // 获取pg并且lock pg
      if (pg) {
		if (pg->get_pgbackend()->scrub_supported() && pg->is_active() && // pg也必须是active状态
	    (load_is_low || (double)diff >= cct->_conf->osd_scrub_max_interval ||
	     pg->scrubber.must_scrub)) {
		  if (pg->sched_scrub()) { // 调度pg 的scrub
			pg->unlock(); // 成功将pg送入scrub_wq后，释放锁，当disk_tp线程从队列获取pg，开始做scrub的时候，会继续拿锁
			break; // break说明一次只成功调度一个pg做，下一次time继续调度
	      }
		}
	    pg->unlock();
      }
    } while (service.next_scrub_stamp(pos, &pos)); // 排在前面的pg可能状态不是active，所以这里继续循环寻找下一个pg
  }
  ......
}
```

\_lookup\_lock\_pg 函数获取一个需要做scrub的pg并上锁，而需要做scrub的pg以时间顺序保存在集合中:

```cpp
// OSD.h中定义的pg集合
set< pair<utime_t,spg_t> > last_scrub_pg;

// 从集合中取一个最久的pg
bool first_scrub_stamp(pair<utime_t, spg_t> *out) {
    Mutex::Locker l(sched_scrub_lock);
    if (last_scrub_pg.empty())
      return false;
    set< pair<utime_t, spg_t> >::iterator iter = last_scrub_pg.begin(); // 时间越小，排在越前面
    *out = *iter;
    return true;
}
```

timer线程中的调度机制只是将需要做scrub的pg放入scrub\_wq队列，等待disk\_tp线程执行，怎么判断一个pg需要scrub，
也就是什么时候将pg放入集合last\_scrub\_pg，这个策略是由pg自己决定的，符合计算机科学的策略与机制分离。
在pg初始化(PG::init)，在osd进程启动时加载pg(load\_pg)，以及pg分裂(add\_newly\_split\_pg)等情况下，如果pg是primary，
就会被放入集合。

继续跟踪OSD的sched\_scrub函数，最终会调用PG的sched\_scrub，这个函数会申请scrub slot，只有当所有副本均申请成功后，
才会排队进行scrub， 所以对于同一个pg，至少会两次tick周期调度到这里，才会真正进行queue\_scrub，当然如果向副本发送消息，
并且收到副本的结果在第二个if执行之前，理论上一次也行。

```cpp
bool PG::sched_scrub()
{
  // 注意条件，如果pg不是primary，直接返回了，说明只有primary pg才可以发起scrub
  if (!(is_primary() && is_active() && is_clean() && !is_scrubbing())) {
    return false;
  }

  ......

  bool ret = true;
  // 第一个if将pg本身加入peers，并且向副本发送scrub消息，让副本预留slot
  if (!scrubber.reserved) { // 第一次调用，reserve为false
    assert(scrubber.reserved_peers.empty()); // peers也为空集合
    if (osd->inc_scrubs_pending()) { // 为自己预留slot
      scrubber.reserved = true;
      scrubber.reserved_peers.insert(pg_whoami); // 将pg本身加入peers集合，一起计数
      scrub_reserve_replicas(); // 发送消息让副本也预留slot
    } else {
      ret = false;
    }
  }

  // 第二个if检查slot信息
  if (scrubber.reserved) {
    if (scrubber.reserve_failed) { // 还没收到副本的消息，或者副本预留成功，reserve_failed为false
      clear_scrub_reserved();
      scrub_unreserve_replicas();
      ret = false;
    } else if (scrubber.reserved_peers.size() == acting.size()) { // 判断是否所有副本都预留成功了
      if (time_for_deep) {
		state_set(PG_STATE_DEEP_SCRUB);
      }
      queue_scrub(); // 如果是，调度scrub
    } else {
      // 等待副本的消息
      dout(20) << "sched_scrub: reserved " << scrubber.reserved_peers << ", waiting for replicas" << dendl;
    }
  }

  ......
}
```

正常情况下，如果副本也预留slot成功，就会执行函数queue\_scrub，它的操作就是将pg放入scrub\_wq队列:

```cpp
bool PG::queue_scrub()
{
  assert(_lock.is_locked());
  if (is_scrubbing()) { // 已经在做scrub，返回
    return false;
  }
  scrubber.must_scrub = false;
  state_set(PG_STATE_SCRUBBING);
  if (scrubber.must_deep_scrub) {
    state_set(PG_STATE_DEEP_SCRUB);
    scrubber.must_deep_scrub = false;
  }
  if (scrubber.must_repair) {
    state_set(PG_STATE_REPAIR);
    scrubber.must_repair = false;
  }
  osd->queue_for_scrub(this); // 放入队列
  return true;
}

bool queue_for_scrub(PG *pg)
{
  return scrub_wq.queue(pg); // 放入队列
}
```

## Do Scrub

schedule成功调度后，pg进入队列scrub\_wq，它会被线程池disk\_tp处理，线程池的入口是Worker函数，它的工作就是从队列取出元素，然后调用其process函数进行处理:

```cpp
void _process(
  PG *pg,
  ThreadPool::TPHandle &handle)
{
  pg->scrub(handle); // 还是让pg执行scrub
  pg->put("ScrubWQ");
}

void PG::scrub(ThreadPool::TPHandle &handle)
{
  lock();

  ......
  chunky_scrub(handle);

  unlock();
}
```

chunky scrub是对classic scrub的改进，只针对某一范围内的object做scrub，而不是pg的所有object，
当一个范围内的scrub完成后，会重新进入队列，再次调度。而再次调度执行scrub这个时间段内，pg的锁是可以被前台进程获取，
其次，对一小部分对象做scrub，理论上也比对所有对象做scrub完成的要快(读取的数据更少)，因此chunky scrub对前台IO更友好，
现在默认都使用chunky scrub。实现的源码看看代码中的注释很容易明白，就是简单的状态机。

当scrub发现有不一致的pg的时候，会上报给monitor，这样就可以通过monitor获取到信息。

# Tuning

因为scrub影响前台IO，可能会造成slow request，在实际部署运营过程中，以下一些参数可能需要调整，特别是
osd\_scrub\_load\_threshold 和 osd\_scrub\_sleep，前者默认值0.5实在太小，几乎不会满足，后者为0也不会随机睡眠。
同时，deep_scrub可能也应该关闭，避免造成大的IO请求。

```text
osd_max_scrub
osd_scrub_min_interval & osd_scrub_max_interval
osd_scrub_begin_hour & osd_scrub_end_hour
osd_scrub_load_threshold
osd_scrub_sleep
osd_scrub_chunk_min & osd_scrub_chunk_max
```


