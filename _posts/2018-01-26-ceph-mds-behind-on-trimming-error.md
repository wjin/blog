---
layout: post
title: "Ceph MDS Behind On Trimming Error"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

在CephFS集群运行过程中，如果一直**持续不停的写入大量文件**，会报告Warning信息:**mds Behind on trimming…**。从文档查看，这个错误是因为日志(MDLog)没来得急trim导致的。一直持续写入文件的时候，虽然data和metadata共用osd，但是优化后的osd负载并不高，trim操作不会因为后端集群负载而delay。告警信息本身影响不大，但是为什么不能及时trim值得深入研究。

# Trim Process

为了一探究竟，需要深入了解mds trim log的流程。首先明确CephFS mds进程管理日志的相关类。MDLog集中管理日志，跟踪所有的LogSegment，包含当前active的segment和expiring/expired的segment。当一个segment的使用量达到上限(比如超过segment所规定的事件个数)，就新起一个segment，segment数量会逐渐增加，需要定期做trim操作。LogSegment记录一连串的LogEvent事件(这里事件有很多种，对文件系统的更新操作都属于某种事件)。另外，MDLog中包含一个对象Journaler，用来向rados写日志/事件。

触发trim操作的入口函数是时间tick函数:

```cpp
void MDSDaemon::tick()
{
	tick_event = 0;

	reset_tick(); // 每隔mds_tick_interval秒，调用tick一次，默认值为5秒

	if (mds_rank) {
		mds_rank->tick();
	}
}

void MDSRankDispatcher::tick()
{
	......
	if (is_active() || is_stopping()) {
		......
		mdlog->trim();
	}
	......
}

void MDLog::trim(int m) // 函数默认参数-1
{
	// 获取配置参数
	unsigned max_segments = g_conf->mds_log_max_segments;
	int max_events = g_conf->mds_log_max_events; // 默认为-1
	if (m >= 0)
		max_events = m;

	if (mds->mdcache->is_readonly()) { // 只读，直接返回
		return;
	}

	// 调整max_events，通常情况下，仍然为-1
	if (max_events > 0 && max_events <= g_conf->mds_log_events_per_segment) {
		max_events = g_conf->mds_log_events_per_segment + 1;
	}

	submit_mutex.Lock();

	if (segments.empty()) { // segment为空，不需要trim，直接返回
		submit_mutex.Unlock();
		return;
	}

	// 设置trim的最长时间，每次2秒钟
	utime_t stop = ceph_clock_now(g_ceph_context);
	stop += 2.0;

	// 遍历segment，event或segment的条件满足其一即做trim
	map<uint64_t,LogSegment*>::iterator p = segments.begin();
	while (p != segments.end() &&
			((max_events >= 0 && num_events - expiring_events - expired_events > max_events) ||
			(segments.size() - expiring_segments.size() - expired_segments.size() > max_segments))) {

		if (stop < ceph_clock_now(g_ceph_context)) // 超出时间，退出循环
			break;

		int num_expiring_segments = (int)expiring_segments.size();
		if (num_expiring_segments >= g_conf->mds_log_max_expiring) // 超出并发trim的限制，退出循环
			break;

		// 计算op的优先级，后续执行try_to_expire的时候，可能会有执行rados op的操作，这个就是rados op的优先级
		// 如果expiring的segment比较多，优先级就越高
		int op_prio = CEPH_MSG_PRIO_LOW +
			(CEPH_MSG_PRIO_HIGH - CEPH_MSG_PRIO_LOW) *
			num_expiring_segments / g_conf->mds_log_max_expiring;

		LogSegment *ls = p->second;
		assert(ls);
		++p;

		// 如果LogSegment包含有pending事件，或者还不能做trim(日志还没落盘)，退出循环
		// 注意LogSegment的seq是log event中的序列号，event的序列号单调递增，全局唯一
		// 新建segment的时候，用当前event的序列号作为segment的序列号
		if (pending_events.count(ls->seq) || ls->end > safe_pos) {
			break;
		}

		// 已经在expiring或expired中，不做任何操作
		if (expiring_segments.count(ls)) {
		} else if (expired_segments.count(ls)) {
		} else { // 否则，将LogSegment加入expiring
			assert(expiring_segments.count(ls) == 0);
			expiring_segments.insert(ls);
			expiring_events += ls->num_events;
			submit_mutex.Unlock();

			uint64_t last_seq = ls->seq;
			try_expire(ls, op_prio); // 尝试expire

			submit_mutex.Lock();
			p = segments.lower_bound(last_seq + 1); // 更新循环迭代器
		}
	}

	// 将expired的segment删除，并更新journaler的expire位置，这个位置也很重要，mds发生异常的时候，此位置即为日志回放的起点
	_trim_expired_segments();
}
```
try\_expire调用try\_to\_expire判断一个LogSegment能否最终被trim，实现非常复杂。LogSegment中包含多个链表以及集合，比如dirty dir/inode/dentry以及和目录分片相关的信息。try\_to\_expire依次循环遍历这些链表和集合，如果不能做trim，就创建callback放入GatherBuild中，GatherBuild搜集所有callback，记录callback的上下文，即产生一个链式callback。

```cpp
void MDLog::try_expire(LogSegment *ls, int op_prio)
{
  MDSGatherBuilder gather_bld(g_ceph_context); // callback集中处理
  ls->try_to_expire(mds, gather_bld, op_prio); // 判断LogSegment是否能够执行expired

  if (gather_bld.has_subs()) { // 如果gather包含callback事件，暂时不能执行expired操作，等待下次继续调用
    gather_bld.set_finisher(new C_MaybeExpiredSegment(this, ls, op_prio));
    gather_bld.activate();
  } else { // 可以执行
    submit_mutex.Lock();
    expiring_segments.erase(ls);
    expiring_events -= ls->num_events;
    _expired(ls); // 将LogSegment放入expired集合
    submit_mutex.Unlock();
  }
  
  logger->set(l_mdl_segexg, expiring_segments.size());
  logger->set(l_mdl_evexg, expiring_events);
}
```

总结一下，整个流程大致是将需要做trim的segment先放入expring中的集合，然后执行try\_expire进行判断是否能够trim，并将其放入expired集合，最后调用\_trim\_expirted\_segments清理expired集合。

# Config

主要配置参数如下:

```cpp
mds_log_max_events // 默认值为-1，即没有上限
mds_log_events_per_segment // 每个segment包含事件数的上限，默认为1024。超过上限，新起一个segment
mds_log_segment_size // 每个segment的大小，默认值为0，采用object的大小，即4 mb
mds_log_max_segments // 最大的segment，默认值为30。另一层含义是: 当segment的个数超过此值的两倍，就会发生文章开头的告警信息
mds_log_max_expiring // 能同时做expiring的segment个数，默认值为20
```

# Tuning

如果要避免告警信息，可以从两个方面考虑:

1. 减少segment的个数

2. 增加trim的速度

对于第一点，可以调整segment的size和事件个数的上限。注意两者要同时调整，很多事件包含成员EMetaBlob，这个类包含成员还比较多，对象大小应该不小，默认的事件个数1024应该是根据默认对象大小4mb和事件平均大小估算的一个合理值。

对于第二点，首先trim函数每隔5秒钟调用一次，这个参数不应该调整，tick函数不只是做trim操作，还有其他逻辑。每次最多执行两秒中，这个写死在代码里，也不应该去改代码，因为涉及到submit\_mutex，不应该频繁的去抢占这个锁。

剩下就是对参数mds\_log\_max\_segments和mds\_log\_max\_expiring的调整，前者不应该调整的太大，太大的话虽然暂时避免了告警信息，但是一直持续写入，最终还是会告警，而且太大存在的LogSegment就太多，异常情况下mds回放日志就会耗费太长时间。最后剩下的参数mds\_log\_max\_expiring，可以根据自己集群的规模和硬件，在不影响客户端IO的情况下适当调大。


异常情况下的恢复流程如下，可以看见回放日志的起点是expire结束的时候:

```cpp
// 初始化回放日志的起点的流程:
MDLog::_recovery_thread() -> Journaler::recover() -> Journaler::_read_head() -> Journaler::_finish_read_head()

read_pos = requested_pos = received_pos = expire_pos = h.expire_pos; // journaler head的expire位置
```

同时将mds\_log\_max\_segment调整的比较大的时候，比如1000，观察到cephfs的元数据池使用量增加非常明显，大部分是日志的增加，文件系统本身的元数据只有几百MB:

```shell
GLOBAL:
SIZE      AVAIL     RAW USED     %RAW USED
1455T     1118T         336T         23.15
POOLS:
NAME                ID     USED      %USED     MAX AVAIL     OBJECTS
cephfs_metadata     1      5759M         0          301T     10128861
cephfs_data         2       112T     27.10          301T     57787299
```

鉴于此，生产环境不建议调整mds\_log\_max\_segments。从实际观察看，参数mds\_log\_max\_expiring很容易达到上限，导致trim不及时，容易发生告警信息，发现社区已经对此问题做了优化，参见[patch](https://github.com/ceph/ceph/pull/18783)，可以将此patch backport回来。另外如果不想修改代码，参数mds\_log\_max\_expiring调整多大不好判断，可以直接放任它不管，但是在监控告警层面，即从命令`Ceph Health`的结果中过滤告警信息，然后在metric系统中监控segment的相关指标，并在metric系统中对segment的阈值进行告警，这个阈值可以设置的比较大，只是以防代码有bug导致segment变的非常大，比如几百万(通常情况下，持续写入几百万的文件，segment的最大值观察到在1万左右)。

下面是一些可以监控的metric指标:

```cpp
 "mds_log": {
        # active/expiring/expired的event个数
        "ev": {
            "type": 2,
            "description": "Events",
            "nick": "evts"
        },
        "evexg": {
            "type": 2,
            "description": "Expiring events",
            "nick": ""
        },
        "evexd": {
            "type": 2,
            "description": "Current expired events",
            "nick": ""
        },

        # active/expiring/expired的segment个数
        "seg": {
            "type": 2,
            "description": "Segments",
            "nick": "segs"
        },
        "segexg": {
            "type": 2,
            "description": "Expiring segments",
            "nick": ""
        },
        "segexd": {
            "type": 2,
            "description": "Current expired segments",
            "nick": ""
        },

        # journal 过期/写/读的位置
        "expos": {
            "type": 2,
            "description": "Journaler xpire position",
            "nick": ""
        },
        "wrpos": {
            "type": 2,
            "description": "Journaler  write position",
            "nick": ""
        },
       "rdpos": {
            "type": 2,
            "description": "Journaler  read position",
            "nick": ""
        },
    }
```

# Summary

* trim速度受到参数控制，不应该为了避免告警信息而将参数mds\_log\_max\_segments调大。

* 根据硬件以及集群规模，适当调整参数mds\_log\_max\_expiring，加快trim的速度，或者backport社区的patch。

* 相对安全的做法是在命令ceph health中忽略告警信息，但在metric指标中增加segment的阈值告警。
