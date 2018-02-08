---
layout: post
title: "Ceph BlueStore Allocator"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

Allocator用来从空闲空间分配block，BlockDevice的空闲空间由FreelistManager管理，FreelistManager提供allocate/release的操作接口，即分配和回收磁盘的block，为什么还会有Allocator？直接用FreelistManager的接口不就完事了吗？

首先，这个问题本身就不太准确，BlueStore中可能存在多个BlockDevice，wal/db对应的BlockDevice，使用者只是BlueFS，BlueFS不用FreelistManager管理块设备空间的使用情况，而是将其持久化记录在文件系统的日志文件中(本身也不可能，FreelistManager依赖于RocksDB存放段的位图信息，而RocksDB依赖于BlueFS，如果BlueFS依赖于FreelistManager，循环依赖)。

其次，如果没有单独的wal/db设备，BlueFS只能借用BlueStore的slow存储空间，这部份空间也只能特殊管理，不能用FreelistManager，理由和上面一样。

最后，BlueStore自己的slow空间(存放data)，当写object文件的时候，先通过Allocator分配磁盘存储空间，仅仅在内存中将空间标记为已分配，并封装在写操作的事务中，待后续完成写操作的时候，才会更新FreelistManger的空闲空间，并将对象的磁盘空间信息记录在对象的metadata中。不同的写case(new/cow/overwrite)，封装的事务也不同，目的是保证数据的一致性，所以磁盘空间的管理和写操作是密切相关的，不能简单的调用FreelistManager的接口完事。

综上，Allocator只负责在内存中将空闲空间标记为已分配，最终磁盘空间使用情况的持久化操作，由Allocator的使用者负责，BlueFS将其记录在文件系统的日志中，BlueStore通过FreelistManager将其存储在k/v中，并在对象metadata中记录对象的磁盘空间信息。

目前系统中有StupidAllocator(基于extent)和BitMapAllocator两种实现，最开始用的Stupid，后来换成BitMap，但是最近因为性能问题又将默认的改回Stupid :(

![img](/assets/img/post/ceph_bluestore_allocator.png)

# Data Structure

Allocator实现的时候，主要数据结构用到了区间树，高效的管理(offset, length):

```cpp
class StupidAllocator : public Allocator {
	int64_t num_free;     // 总的空闲大小
	int64_t num_reserved; // 预留空间

	// 初始化的时候，free数组的长度为10，即有十颗区间树
	// 根据每个区间的长度，分别插入不同的区间树
	std::vector<btree_interval_set<uint64_t,allocator>> free;

	uint64_t last_alloc;
};
```

# Init

创建Allocator后，调用者紧接着会向Allocator中加入或删除空闲空间:

```cpp
// 增加空闲空间
void StupidAllocator::init_add_free(uint64_t offset, uint64_t length)
{
	std::lock_guard<std::mutex> l(lock);
	_insert_free(offset, length); // 向free中插入数据
	num_free += length; // 更新可用空间
}

// 删除空闲空间
void StupidAllocator::init_rm_free(uint64_t offset, uint64_t length)
{
	std::lock_guard<std::mutex> l(lock);
	btree_interval_set<uint64_t,allocator> rm;

	rm.insert(offset, length);
	for (unsigned i = 0; i < free.size() && !rm.empty(); ++i) {
		btree_interval_set<uint64_t,allocator> overlap;
		overlap.intersection_of(rm, free[i]); // 求交集
		if (!overlap.empty()) { // 删除
			free[i].subtract(overlap);
			rm.subtract(overlap);
		}
	}

	num_free -= length; // 更新可用空间
}
```

核心实现函数就是向free的区间树插入区间:

```cpp
// 根据区间的长度，选取将要存放的区间树，长度越大，bin值越大
unsigned StupidAllocator::_choose_bin(uint64_t orig_len)
{
	uint64_t len = orig_len / cct->_conf->bdev_block_size;

	// cbits = (sizeof(v) * 8) - __builtin_clzll(v);
	// 结果是最高位1的下标，len越大，值越大
	int bin = std::min(int)cbits(len), (int)free.size() - 1);
	return bin;
}

void StupidAllocator::_insert_free(uint64_t off, uint64_t len)
{
	unsigned bin = _choose_bin(len); // 计算区间树的id

	while (true) {
		free[bin].insert(off, len, &off, &len);
		unsigned newbin = _choose_bin(len);
		if (newbin == bin)
			break;

		// 插入数据后，可能合并区间，导致区间长度增大，可能要调整bin，此时需要将旧的删除，然后插入新的bin
		free[bin].erase(off, len);
		bin = newbin;
	}
}
```

Allocate & Release用来分配和回收空闲空间，最终都是调用区间树的insert和erase API，不再赘述。

# Usage

Allocator本身比较简单，更多的应该关注怎么使用它，特别是怎么将分配信息最终持久化。

### BlueFS

BlueFS分配空间的函数:

```cpp
int BlueFS::_allocate(uint8_t id, uint64_t len, mempool::bluefs::vector<bluefs_extent_t> *ev)
{
	if (alloc[id]) {
		r = alloc[id]->reserve(left); // 预留空间
	}

	if (r < 0) {
		if (id != BDEV_SLOW) { // 空间不足，递归降级到下一级的设备
			return _allocate(id + 1, len, ev);
		}
		......
	}

	AllocExtentVector extents;
	extents.reserve(4);
	int64_t alloc_len = alloc[id]->allocate(left, min_alloc_size, hint, &extents); // 真正分配空间的操作
	......
}
```

这个函数在多种情况下会调用，包括为普通文件分配空间的\_preallocate以及和日志文件相关的compact/flush等操作。以普通文件为例:

```cpp
int BlueFS::_preallocate(FileRef f, uint64_t off, uint64_t len)
{
  if (f->deleted) {
    return 0;
  }

  uint64_t allocated = f->fnode.get_allocated();
  if (off + len > allocated) {
    uint64_t want = off + len - allocated;
    int r = _allocate(f->fnode.prefer_bdev, want, &f->fnode.extents); // 分配空间
    if (r < 0)
      return r;
    f->fnode.recalc_allocated();
    log_t.op_file_update(f->fnode); // 记录日志，日志就是文件的inode信息，inode中包含extents，即物理磁盘空间
  }
  return 0;
}
```

在BlueFS的超级快(SuperBlock)中，记录了日志文件的inode，异常情况下通过重新mount文件系统，读取超级块，定位到日志文件，然后读取日志进行回放，重建所有文件的内存映像(file\_map)，遍历file\_map，即可初始化Allocator的空间。

```cpp
int BlueFS::mount()
{
	......
	r = _replay(false); // 重建file_map
	......

	for (auto& p : file_map) {
		for (auto& q : p.second->fnode.extents) {
			alloc[q.bdev]->init_rm_free(q.offset, q.length); // 将已有文件占用的磁盘块从allocator中删除
		}
	}
}
```

### BlueStore

BlueStore的情况，涉及到读写object文件的IO流程，留着分析BlueStore时介绍。

# Config

配置参数比较简单，目前默认用stupid，不需要调整任何参数。

```cpp
bluefs_allocator // 默认为stupid
bluestore_allocator // 默认为stupid

// bitmap allocator 相关的参数
bluestore_bitmapallocator_blocks_per_zone
bluestore_bitmapallocator_span_size
```

# Summary

* Allocator用来分配磁盘空间，只在内存做标记，实现包含Stupid和BitMap两种，Stupid即为基于extent的方式

* Allocator的使用者在创建Allocator的时候需要初始化可用空间，并且在分配空间后，需要使用者在合适时机将磁盘空间使用信息持久化

* Allocator的使用者包含BlueFS和BlueStore，BlueFS通过文件系统的日志文件固化磁盘空间使用情况，BlueStore通过FreelistManager将磁盘空间信息固化到k/v中

* Allocator默认的stupid实现比较简单，理解Allocator在整个BlueStore存储引擎中的作用以及使用方式比其实现更有意义
