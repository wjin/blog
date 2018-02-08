---
layout: post
title: "Ceph BlueStore FreelistManager"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

因为BlueStore采用裸设备，所以需要自己管理磁盘空间的分配和回收。如果以**block**表示磁盘的最小存储单位(Ceph中默认为4k)，一个block的状态可以为**使用**和**空闲**两种状态，实现中只需要记录一种状态的block，就可以推导出另一种状态的block。Ceph采用**记录空闲状态**的block，主要原因有二，一是因为在回收空间的时候，方便空闲空间的合并，二是因为已分配的空间在object的元数据Onode中会有记录。

管理空闲空间的类为FreelistManager，最开始有extent和bitmap两种实现，现在已经默认为bitmap实现，并将extent的实现废弃。空闲空间需要持久化到磁盘，并且在运行过程中通过事务更新，很自然的方式可以用k/v存储，将block按一定数量组成**段**，每个段对应一个k/v键值对，key为第一个block在磁盘物理地址空间的offset，value为段内每个block的状态，即由0/1组成的位图，0为空闲，1为使用，这样可以通过与1进行异或运算，将分配和回收空间两种操作统一起来。

# Data Structure

```cpp
class BitmapFreelistManager : public FreelistManager {
	std::string meta_prefix, bitmap_prefix; // rocksdb中key的前缀，meta为B，bitmap为b
	KeyValueDB *kvdb; // kvdb指针
	ceph::shared_ptr<KeyValueDB::MergeOperator> merge_op; // merge操作，实际上就是按位xor
	std::mutex lock;

	uint64_t size;            // 设备的大小
	uint64_t blocks;          // 设备总的block数

	uint64_t bytes_per_block; // block的大小，对应bdev_block_size
	uint64_t blocks_per_key;  // 每个key包含多少个block
	uint64_t bytes_per_key;   // 每个key对应的空间大小

	uint64_t block_mask;  // block掩码
	uint64_t key_mask;    // key的掩码

	bufferlist all_set_bl;

	// 遍历rocksdb key相关的成员
	KeyValueDB::Iterator enumerate_p;
	uint64_t enumerate_offset;
	bufferlist enumerate_bl;
	int enumerate_bl_pos; 
};
```

# Init

BlueStore在初始化osd的时候，会执行mkfs，初始化FreelistManager(create/init），后续如果重启进程，会执行mount操作，只会对FreelistManager执行init操作。

```cpp
int BlueStore::mkfs()
{
	......
	r = _open_fm(true);
	......
}

int BlueStore::_open_fm(bool create)
{
	......
	fm = FreelistManager::create(cct, freelist_type, db, PREFIX_ALLOC);

	if (create) { // 第一次初始化，需要固化meta参数
		fm->create(bdev->get_size(), min_alloc_size, t);
	}

	......
	int r = fm->init(bdev->get_size());
}

// create固化一些meta参数到kvdb中，init的时候，从kvdb读取这些参数
int BitmapFreelistManager::create(uint64_t new_size, uint64_t min_alloc_size,
		KeyValueDB::Transaction txn)
{
	txn->set(meta_prefix, "bytes_per_block", bl); // 4096
	txn->set(meta_prefix, "blocks_per_key", bl); // 128
	txn->set(meta_prefix, "blocks", bl);
	txn->set(meta_prefix, "size", bl);
}

// create/init均会调用下面这个函数，初始化block/key的掩码
void BitmapFreelistManager::_init_misc()
{
  bufferptr z(blocks_per_key >> 3); // 128 >> 3 = 16，即一个key的value(段)对应128个block，每个block用1个bit表示，需要16字节
  memset(z.c_str(), 0xff, z.length());
  all_set_bl.clear();
  all_set_bl.append(z);

  block_mask = ~(bytes_per_block - 1); // 0x FFFF FFFF FFFF F000
  bytes_per_key = bytes_per_block * blocks_per_key;
  key_mask = ~(bytes_per_key - 1); // 0xFFFF FFFF FFF8 0000
}
```

# Allocate & Release

最主要的接口就是用来分配和释放空间，前面已经提到，两种操作是完全一样的，都是异或操作:

```cpp
void BitmapFreelistManager::allocate(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	_xor(offset, length, txn);
}

void BitmapFreelistManager::release(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	_xor(offset, length, txn);
}

void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	// 注意offset和length都是以block边界对齐
	uint64_t first_key = offset & key_mask;
	uint64_t last_key = (offset + length - 1) & key_mask;

	if (first_key == last_key) { // 最简单的case，此次操作对应一个段
		bufferptr p(blocks_per_key >> 3); // 16字节大小的buffer
		p.zero(); // 置为全0
		unsigned s = (offset & ~key_mask) / bytes_per_block; // 段内开始block的编号
		unsigned e = ((offset + length - 1) & ~key_mask) / bytes_per_block; // 段内结束block的编号

		for (unsigned i = s; i <= e; ++i) { // 生成此次操作的掩码
			p[i >> 3] ^= 1ull << (i & 7); // i>>3定位block对应位的字节， 1ull<<(i&7)定位bit，然后异或将位设置位1
		}

		string k;
		make_offset_key(first_key, &k); // 将内存内容转换为16进制的字符
		bufferlist bl;
		bl.append(p);
		bl.hexdump(*_dout, false);
		txn->merge(bitmap_prefix, k, bl); // 和目前的value进行异或操作

	} else { // 对应多个段，分别处理第一个段，中间段，和最后一个段，首尾两个段和前面情况一样

		// 第一个段
		{
			// 类似上面情况
			......

			// 增加key，定位下一个段
			first_key += bytes_per_key;
		}

		// 中间段，此时掩码就是全1，所以用all_set_bl
		while (first_key < last_key) {
			string k;
			make_offset_key(first_key, &k);
			all_set_bl.hexdump(*_dout, false);

			txn->merge(bitmap_prefix, k, all_set_bl); // 和目前的value进行异或操作

			// 增加key，定位下一个段
			first_key += bytes_per_key;
		}

		// 最后一个段
		{
			// 和前面操作类似
		}
	}
}

// 上面merge操作对应的实现
struct XorMergeOperator : public KeyValueDB::MergeOperator {
  void merge(
    const char *ldata, size_t llen,
    const char *rdata, size_t rlen,
    std::string *new_value) override {
    assert(llen == rlen);
    *new_value = std::string(ldata, llen);
    for (size_t i = 0; i < rlen; ++i) {
      (*new_value)[i] ^= rdata[i]; // 按位异或
    }
  }
};
```

总结一下，xor函数看似复杂，全是位操作，仔细分析一下，分配和释放操作一样，都是将段的bit位和当前的值进行异或。一个段对应一组blocks，默认128个，在k/v中对应一组值。例如，当磁盘空间全部空闲的时候，k/v状态如下: (b00000000，0x00), (b00001000, 0x00), (b00002000, 0x00)......b为key的前缀，代表bitmap。

# Config

目前只有一个参数可供调整(部署之前，不能在运行中动态调整)，就是段的大小，这个参数如果调整的太小，对应的段的个数增加，k/v键值对数目会暴增，所以需要根据磁盘空间做决定，实际使用过程中不建议修改。

```cpp
bluestore_freelist_blocks_per_key // 默认为128
```

# Summary

* BlueStore用FreelistManager管理磁盘空闲空间，通过将磁盘空间分成大小相同的段，并将段的状态用位图记录在k/v存储中

* 分配和回收空间实现一样，对原来位图进行异或操作
