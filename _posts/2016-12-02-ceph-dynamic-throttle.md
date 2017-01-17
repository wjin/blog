---
layout: post
title: "Ceph Dynamic Throttle"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

在Ceph Jewel版本中，新增加了一种限流实现，叫BackoffThrottle，然后filestore存储引擎利用新的限流方式对后端journal以及apply op queue进行动态限流。
因为限流和性能关系密切，需要进行重点调优。BackoffThrottle的原理其实非常简单，就是动态地插入delay时间，阻塞调用线程，这个delay时间由一系列参数控制。
因为delay的时间动态地改变，可以看成是将限流均摊到每个调用线程的每次调用，而以前的限流是只要到达阈值就block住，很明显这会带来长尾效应，
而新引入的BackoffThrottle限流将时间均摊到每次操作后更平缓，避免了长尾效应，参见[pr](https://github.com/ceph/ceph/pull/7767)。

dealy值是怎么计算出来的呢？首先将[0,1]区间分成三份，[0, low_threshold), [low_threshhold, high_threshhold), [high_threshhold, 1]，三个区间的斜率分别为0, s0, s1。
另外假设限流的最大值为max，当前值为current，x的值为current/max，当x落入上述区间后，根据如下公式计算delay值:

```cpp
// 具体参考代码中的注释
delay = 0, x in [0, l)
delay = (x - l) * (e / (h - l)), x in [l, h)
delay = e + (x - h)((m - e)/(1 - h)), x in [h, 1)
```

![img](/assets/img/post/ceph_backoffthrottle.png)

如图所示，在第一个区间的时候，也就是压力不大的情况下，delay值为0，是不需要wait的。当压力增大，x落入第二个区间后，delay值开始起作用，并且逐步增大，
当压力过大的时候，会落入第三个区间，这时候delay值增加明显加快，wait值明显增大，尽量减慢io速度，减缓压力，故而得名dynamic throttle。

# Implementation

原理简单，源码实现也容易理解，以后自己编程过程中遇见类似情况，可以完全借鉴来用，简单看看实现，参考代码Throttle.[h|c]:

```cpp
class BackoffThrottle {
	unsigned next_cond = 0; // conds变量的索引
	vector<std::condition_variable> conds; // 这里相当于做了一个分片，避免所有线程都wait在一个条件变量上

	list<std::condition_variable*> waiters; // wait的fifo队列
	std::chrono::duration<double> _get_delay(uint64_t c) const; // 计算delay值的函数，参见上面的公式

	std::list<std::condition_variable*>::iterator _push_waiter() {
		unsigned next = next_cond++; // 获取此次wait的条件变量
		if (next_cond == conds.size())
			next_cond = 0;
		return waiters.insert(waiters.end(), &(conds[next])); // 插入链表
	}

	void _kick_waiters() {
		if (!waiters.empty())
			waiters.front()->notify_all(); // 唤醒睡眠在链表头部条件变量的线程
	}
};
```

核心实现函数就是get函数:

```cpp
std::chrono::duration<double> BackoffThrottle::get(uint64_t c)
{
	locker l(lock);
	auto delay = _get_delay(c); // 提前计算一下wait值

	// 不用wait，直接返回
	if (delay == std::chrono::duration<double>(0) &&
			waiters.empty() &&
			((max == 0) || (current == 0) || ((current + c) <= max))) {
		current += c;
		return std::chrono::duration<double>(0);
	}

	auto ticket = _push_waiter(); // 获取wait的条件变量并插入链表
	while (waiters.begin() != ticket) { // 等待自己变为链表头部
		(*ticket)->wait(l);
	}

	auto start = std::chrono::system_clock::now();
	delay = _get_delay(c); // 再次计算wait的值，此时自己已经是链表头部的条件变量
	while (true) {
		if (!((max == 0) || (current == 0) || (current + c) <= max)) { // 超过上限(current + c > max)，一直wait，等待唤醒
			(*ticket)->wait(l);
		} else if (delay > std::chrono::duration<double>(0)) { // wait一段时间
			(*ticket)->wait_for(l, delay);
		} else {
			break;
		}
		assert(ticket == waiters.begin());
		delay = _get_delay(c) - (std::chrono::system_clock::now() - start); // 重新计算wait值
	}

	waiters.pop_front(); // 清除条件变量
	_kick_waiters(); // 唤醒后面的wait

	current += c; // get成功，修改计数
	return std::chrono::system_clock::now() - start;
}
```

# Usage

使用的地方有两个，第一个是journal的限流JournalThrottle对BackoffThrottle进行了wrapper，另外一个就是filestore中op queue的ops和bytes。

# Tuning

对于性能调优，需要关注以下参数:

```cpp
filestore_queue_max_ops
filestore_queue_max_bytes

filestore_caller_concurrency

filestore_expected_throughput_bytes
filestore_expected_throughput_ops

filestore_queue_low_threshhold
filestore_queue_high_threshhold
filestore_queue_max_delay_multiple
filestore_queue_high_delay_multiple

journal_throttle_low_threshhold
journal_throttle_high_threshhold
journal_throttle_high_multiple
journal_throttle_max_multiple
```

