---
layout: post
title: "Ceph BlueStore BlueFS"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

BlueStore存储引擎实现中，需要存储数据和元数据。由于kv存储系统自身的高效性以及对事务的支持，所以选择kv存储元数据是理所当然的(对象的omap属性算作数据，也是存放在kv中的)。Luminous目前默认采用RocksDB来存储元数据(RocksDB本身存在写放大以及compaction的问题，后续可能会针对Ceph的场景量身定制kv)，但是BlueStore采用裸设备，RocksDB不支持raw disk，幸运的是，RocksDB提供RocksEnv运行时环境来支持跨平台操作，那么能够想到的方案就是Ceph自己实现一个简单的文件系统，这个文件系统只提供RocksEnv需要的操作接口，这样就可以支持RocksDB的运行，而这个文件系统就是BlueFS。

作为文件系统本身，需要存放日志，保护文件系统数据的一致性。对于RocksDB，也可以对.log文件单独配置性能更好的磁盘。所以在BlueFS内部实现的时候，支持多种不同类型的设备(wal/db/slow)，实现非常灵活，大致原则是RocksDB的.log文件和BlueFS自身的日志文件优先使用wal，BlueFS中的普通文件(RocksDB的.sst文件)优先使用db，当当前设备空间不足的时候，自动降级到下一级的设备。

文件系统本身需要使用磁盘空间存放数据，但是BlueFS并不需要管理磁盘空闲空间，它将文件分配和释放空间的操作记录在日志文件中。每次重新加载的时候，扫描文件系统的日志，在内存中还原整个文件系统的元数据信息。运行过程中，磁盘空间使用情况大致如下(借用Ceph作者Sage的图):

![img](/assets/img/post/ceph_bluestore_bluefs.png)

# Data Structure

先看看在BlueFS中标识一个文件的inode:

```cpp
// 物理磁盘的位移和长度，代表块设备的一个存储区域
class AllocExtent {
	public:
		uint64_t offset; // BlockDevice的物理地址
		uint32_t length; // 长度
};

class bluefs_extent_t : public AllocExtent{
	public:
		uint8_t bdev; // 属于哪个block device
};

// 文件的inode
truct bluefs_fnode_t {
	uint64_t ino; // inode编号
	uint64_t size; // 文件大小
	utime_t mtime; // 修改时间
	uint8_t prefer_bdev; // 优先使用哪个block device
	mempool::bluefs::vector<bluefs_extent_t> extents; // 文件对应的磁盘空间
	uint64_t allocated; // 文件实际占用的空间大小，extents的length之和。应该是小于等于size
};
```

和一般文件系统类似，需要一个文件系统超级块，在mount文件系统的时候，需要读取超级块里的数据，才能识别文件系统:

```cpp
struct bluefs_super_t {
  uuid_d uuid; // 唯一的uuid
  uuid_d osd_uuid; // 对应的osd的uuid
  uint64_t version; // 版本
  uint32_t block_size; // 块大小

  bluefs_fnode_t log_fnode; // 记录文件系统日志的文件
};
```

接下来就是文件系统的操作，这些操作对文件系统进行修改，需要封装成事务并记录在日志中:

```cpp
struct bluefs_transaction_t {
	typedef enum {
		OP_NONE = 0,
		OP_INIT,        ///< initial (empty) file system marker

		// 给文件分配和释放空间
		OP_ALLOC_ADD,   ///< add extent to available block storage (extent)
		OP_ALLOC_RM,    ///< remove extent from availabe block storage (extent)

		// 创建和删除目录项
		OP_DIR_LINK,    ///< (re)set a dir entry (dirname, filename, ino)
		OP_DIR_UNLINK,  ///< remove a dir entry (dirname, filename)

		// 创建和删除目录
		OP_DIR_CREATE,  ///< create a dir (dirname)
		OP_DIR_REMOVE,  ///< remove a dir (dirname)

		// 文件更新
		OP_FILE_UPDATE, ///< set/update file metadata (file)
		OP_FILE_REMOVE, ///< remove file (ino)

		// bluefs日志文件的compaction操作
		OP_JUMP,        ///< jump the seq # and offset
		OP_JUMP_SEQ,    ///< jump the seq #
	} op_t;

	uuid_d uuid;          ///< fs uuid
	uint64_t seq;         ///< sequence number
	bufferlist op_bl;     ///< encoded transaction ops
};
```

最后看看文件系统本身的结构:

```cpp
class BlueFS {
	public:
		// 文件系统支持不同种类的块设备
		static constexpr unsigned MAX_BDEV = 3;
		static constexpr unsigned BDEV_WAL = 0;
		static constexpr unsigned BDEV_DB = 1;
		static constexpr unsigned BDEV_SLOW = 2;

		enum {
			WRITER_UNKNOWN,
			WRITER_WAL, // RocksDB的log文件
			WRITER_SST, // RocksDB的sst文件
		};

		// 文件
		struct File : public RefCountedObject {
			bluefs_fnode_t fnode; // 文件inode
			int refs; // 引用计数
			uint64_t dirty_seq; // dirty序列号
			bool locked;
			bool deleted;
			boost::intrusive::list_member_hook<> dirty_item;

			// 读写计数
			std::atomic_int num_readers, num_writers;
			std::atomic_int num_reading;
		};

		// 目录
		struct Dir : public RefCountedObject {
			mempool::bluefs::map<string,FileRef> file_map; // 目录包含的文件
		};

		// 文件系统的内存映像
		mempool::bluefs::map<string, DirRef> dir_map; // 所有的目录
		mempool::bluefs::unordered_map<uint64_t,FileRef> file_map; // 所有的文件

		map<uint64_t, dirty_file_list_t> dirty_files; // 脏文件，根据序列号排列

		// 文件系统超级块和日志
		......

		// 结构体FileWriter/FileReader/FileLock，用来对一个文件进行读写和加锁
		......

		vector<BlockDevice*> bdev; // BlueFS能够使用的所有BlockDevice，包括wal/db/slow
		vector<IOContext*> ioc; // bdev对应的IOContext
		vector<interval_set<uint64_t> > block_all;  // bdev对应的磁盘空间
		vector<Allocator*> alloc; // bdev对应的allocator
		......
};
```

# BlueFS Init

BlueFS的用户(RocksDB/RocksEnv)只会对文件系统进行常规的操作，比如创建/删除文件，打开文件进行读写等操作。但是在使用文件系统之前，文件系统必须格式化，这个是由BlueStore存储引擎统一管理的，部署osd的时候会完成BlueFS的初始化，流程如下:

```cpp
int OSD::mkfs(CephContext *cct, ObjectStore *store, const string &dev,
		uuid_d fsid, int whoami)
{
	......
	ret = store->mkfs();
	......
}

int BlueStore::mkfs()
{
	......
	r = _open_db(true);
	......
}

int BlueStore::_open_db(bool create)
{
	......
	bluefs = new BlueFS(cct);
	......

	// 依次添加 slow/db/wal等设备
	bluefs->add_block_device(...) // 添加设备，会新建一个BlockDevice及其对应的IOContext
	bluefs->add_block_extent(...) // 添加设备的存储空间，一般为SUPER_RESERVED到磁盘空间的上限，SUPER_RESERVED为8192，即从第三个4k开始
	......

	bluefs->mkfs(fsid); // 格式化文件系统，主要工作包括生成文件系统的超级块/log文件等
	bluefs->mount(); // mount文件系统
	......
}
```

整个流程比较简单，这里只需要明白，BlueFS是一个内存文件系统，mount的时候，通过扫码日志，在内存中还原出整个文件系统的状况，包括dir\_map和file\_map等。之所以这样做，是因为BlueFS仅仅为RocksDB服务，文件系统本身只包含少量的文件，内存空间和磁盘日志空间占用均不大。

```cpp
int BlueFS::mount()
{
	// 读取超级块
	int r = _open_super();
	......

	// 初始化allocator为磁盘所有的空间
	_init_alloc();
	......

	// 回放文件系统日志，日志项即为上面的事务OP，针对每个事务进行回放，文件系统的dir_map/file_map就会被更新
	r = _replay(false);

	for (auto& p : file_map) {
		for (auto& q : p.second->fnode.extents) {
			alloc[q.bdev]->init_rm_free(q.offset, q.length); // 将文件已经占用的内容从allocator中删除
		}
	}
	......
}
```

mount完成后，文件系统的所有数据，包括文件和目录，在内存中就初始化完成，后续就可以对文件系统进行读写等操作。打开文件进行写的操作是open\_for\_write，如果是新文件，更新dir\_map，注意打开文件的时候，文件的prefer\_bdev默认采用的BDEV\_DB设备，然后会根据目录名称(slow/wal目录有特殊后缀)进行适当调整，所以BlueFS的普通文件，优先使用db设备，而不会使用空间较少的wal设备。

文件系统提供的API比较少，其他实现也比较简单，同样是记录日志和更新内存中的dir\_map和file\_map等。当发生异常或重启进程的时候，回放文件系统的日志，将文件系统的内存状态还原到之前的状态。

另外一个值得注意的地方是BlueFS文件系统日志的compact操作，这个分为sync和async两种，大致流程是将文件系统的内存映像(文件和目录)重新生成事务，然后写入新的日志文件，然后将旧的日志文件删除，而不会对旧的日志文件做读写。

# Config

```cpp
// 普通文件
bluefs_alloc_size // 最小分配大小，默认为1MB
bluefs_max_prefetch // 预读时的最大字节数，默认为1MB，主要用在顺序读场景

// 日志文件
bluefs_min_log_runway // bluefs日志文件的可用空间小于此值时，新分配空间。默认为1MB
bluefs_max_log_runway // bluefs日志文件的单次分配大小，默认为4MB
bluefs_log_compact_min_ratio // 通过当前日志文件大小和预估的日志文件的大小的比率控制compact，默认为5
bluefs_log_compact_min_size // 通过日志文件大小控制compact，小于此值不做compact。默认为16MB
bluefs_compact_log_sync // 日志文件compact的方式，有sync和async两种，默认为false，即采用async方式

bluefs_min_flush_size // 因为写文件内容是写到内存中的，当文件内容超过此值就刷新到磁盘。默认为512kb

bluefs_buffered_io // bluefs调用BlockDevice的read/write时的参数，默认为false，即采用fd_direct
bluefs_sync_write // 是否采用synchronous写。默认为false，即采用aio_write。这时候在flush block device的时候，需要等待aio完成。参见函数_flush_bdev_safely

bluefs_allocator // bluefs分配磁盘空间的分配器，默认为stupid，即基于extent的方式。

bluefs_preextend_wal_files // 是否预先更新rocksdb wal文件的大小。默认为false

// 另外还有一些参数和BlueStore相关，以bluestore_bluefs_开头，这些参数主要控制将BlueStore的slow存储空间分配给BlueFS使用
```

# Metric

监控指标比较好理解，看看描述就能明白，重点应该关注db/wal的使用情况，因为通常情况下会使用更快的ssd，不要因为BlueFS的空间不够而使用BlueStore中的slow空间。

```cpp
    "bluefs": {
        "gift_bytes": 0,
        "reclaim_bytes": 0,
        "db_total_bytes": 240043163648,
        "db_used_bytes": 631242752,
        "wal_total_bytes": 0,
        "wal_used_bytes": 0,
        "slow_total_bytes": 0,
        "slow_used_bytes": 0,
        "num_files": 14,
        "log_bytes": 5361664,
        "log_compactions": 0,
        "logged_bytes": 0,
        "files_written_wal": 0,
        "files_written_sst": 0,
        "bytes_written_wal": 0,
        "bytes_written_sst": 0
    },
```

# Summary

* BlueFS同时支持多个设备(wal/db/slow)

* BlueFS是个简单的内存文件系统，只提供简单的操作用来支持RocksDB运行

* BlueFS有自己的日志，用来记录对文件系统的修改。异常情况下，回放日志可重建文件系统的完整的内存映像
