---
layout: post
title: "Ceph PerfCounters"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

ceph代码中对每个模块都加入了性能统计分析，在实际运营过程中，可以对这些关键指标进行监控，以便更精确的了解集群内部运行状态。
详细介绍可以参考[官方文档](http://docs.ceph.com/docs/master/dev/perf_counters/)，使用方式如下:

> ceph daemon osd.0 perf schema

> ceph daemon osd.0 perf dump

解释一下schema输出的意思，dump出来的结果，如果type为5，表示4+1，type为10，表示8+2，以此类推，引用官方文档如下:

```
bit	meaning
1	floating point value // 浮点数
2	unsigned 64-bit integer value // 整数
4	average (sum + count pair) // 设置4，有两个值，一般需要计算sum/avgcount以获取平均值，主要用来计算延时
8	counter (vs gauge) // 设置8，监控的时候dump出来的值可能需要减去上次dump的值，进而求得两次dump间隔内的差值
```

# Implementation

### PerfCountersCollection

首先，一个daemon进程有一个唯一的CephContext，里面包含一个PerfCountersCollection变量，跟踪daemon的所有PerfCounters:

```cpp
class CephContext {
	PerfCountersCollection *_perf_counters_collection; // daemon的唯一collection
	......
};

typedef std::set <PerfCounters*, SortPerfCountersByName> perf_counters_set_t;

class PerfCountersCollection
{
	public:
		void add(class PerfCounters *l); // 增加一个perfcounter
		void remove(class PerfCounters *l); // 删除一个perfcounter

	private:
		perf_counters_set_t m_loggers; // 集合，根据名字排序，daemon的所有模块的PerfCounters都记录在此
		std::map<std::string, PerfCounters::perf_counter_data_any_d *> by_path; // 所有perfcounter包含的字段的k/v
};
```

添加删除的API都很简单，这里不再详述。

### PerfCounters

每个模块都有一个独立的PerfCounters，包含多个关键的性能指标，即多个item，这些指标限定在给定模块的索引范围内[lower_bound, upper_bound], 以librbd为例:

```cpp
// src/librbd/internal.h
enum {
	l_librbd_first = 26000, // lower_bound

	l_librbd_rd,               // read ops
	l_librbd_rd_bytes,         // bytes read
	l_librbd_rd_latency,       // average latency
	......

	l_librbd_last, // upper_bound
};
```

由于有上下边界，并且每个item有一个索引，所以存放的时候只需要一个vector即可:

```cpp
class PerfCounters {
	struct perf_counter_data_any_d { // 每一个指标记录的值
		const char *name;
		const char *description;
		const char *nick;
		enum perfcounter_type_d type;
		atomic64_t u64; // 计数

		// 这里的两个count很有意思
		atomic64_t avgcount;
		atomic64_t avgcount2;
	};

	int m_lower_bound; // 索引下界
	int m_upper_bound; // 索引上界
	typedef std::vector<perf_counter_data_any_d> perf_counter_data_vec_t;
	perf_counter_data_vec_t m_data; // 所有的值
};
```

前面提到，当type中包含4的时候，需要设置两个值，即sum和avgcount，在读取两个值计算平均值的时候，需要保证读到的两个值是同一次修改的，否则就不准确，
最简单的做法是加锁，在修改的时候加锁，修改两个值，然后在读取的时候也加锁，读取两个值，但是这对性能影响太大，代码中使用了两个计数来巧妙的到达目的，从而避免了锁的使用:

```cpp
// 修改
void PerfCounters::inc(int idx, uint64_t amt)
{
	......
	perf_counter_data_any_d& data(m_data[idx - m_lower_bound - 1]);
	if (!(data.type & PERFCOUNTER_U64))
		return;
	if (data.type & PERFCOUNTER_LONGRUNAVG) {
		data.avgcount.inc(); // 增加第一个计数
		data.u64.add(amt); // 增加值，即sum
		data.avgcount2.inc(); // 增加第二个计数
	} else {
		data.u64.add(amt);
	}
}

// 读取
pir<uint64_t,uint64_t> read_avg() const {
	uint64_t sum, count;
	do {
		count = avgcount.read();
		sum = u64.read();
	} while (avgcount2.read() != count); // avgcount和avgcount2如果相等，那么读到的sum值就是对应的avgcount时设置的
	return make_pair(sum, count);
}
```

性能计数的修改是很频繁的操作，如果每次修改都需要加锁解锁，overhead还是比较大的，使用两个原子变量，仅仅在读的时候可能会多次读取，但是读的频率只发生在dump数据的时候，
影响不大，这样即满足了统计的需求，也降低了overhead。

### PerfCountersBuilder

这个类供模块使用，内部增加了对item的参数检查:

```cpp
class PerfCountersBuilder {
	PerfCounters* create_perf_counters(); // 将构造函数分配的指针返回给用户
	PerfCounters *m_perf_counters;
};

PerfCountersBuilder::PerfCountersBuilder(CephContext *cct, const std::string &name,
		                  int first, int last)
	  : m_perf_counters(new PerfCounters(cct, name, first, last)) // new一个PerfCounters
{
}

PerfCounters *PerfCountersBuilder::create_perf_counters()
{
	PerfCounters::perf_counter_data_vec_t::const_iterator d = m_perf_counters->m_data.begin();
	PerfCounters::perf_counter_data_vec_t::const_iterator d_end = m_perf_counters->m_data.end();

	// 检查item是否合法
	for (; d != d_end; ++d) {
		if (d->type == PERFCOUNTER_NONE) {
			assert(d->type != PERFCOUNTER_NONE);
		}
	}

	PerfCounters *ret = m_perf_counters;
	m_perf_counters = NULL; // 清除内部指针
	return ret; // 将动态分配的对象返回给用户
}
```

### Usage

看看模块使用的例子，以FileStore为例:

```cpp
FileStore::FileStore(CephContext* cct, const std::string &base,
		const std::string &jdev, osflagbits_t flags,
		const char *name, bool do_update) :
	......
{
	......
	PerfCountersBuilder plb(cct, internal_name, l_filestore_first, l_filestore_last); // 对象的构造函数中会动态创建PerfCounters

	// 增加item
	plb.add_u64(l_filestore_journal_queue_ops, "journal_queue_ops", "Operations in journal queue");
	......

	logger = plb.create_perf_counters(); // 获取动态分配的指针

	cct->get_perfcounters_collection()->add(logger); // 将PerfCounters加入collection
	......
}
```

# Summary

* 一个daemon包含一个唯一的PerfCountersCollection，用来记录所有模块的PerfCounters

* 每个需要性能计数的模块，实现一个PerfCounters，内部包含很多item(k/v)，并且分配一个范围，不同模块的这个范围实际上可以重复

* 模块使用PerfCountersBuilder来创建PerfCounters并检查item合法性
