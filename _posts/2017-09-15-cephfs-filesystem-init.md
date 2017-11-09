---
layout: post
title: "Cephfs Filesystem Init"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

文件存储是ceph分布式存储系统提供的三种接口之一，兼容posix协议，挂载上后就可以当做ext4或xfs本地文件系统使用。
部署的时候，需要部署mds进程服务，这个是cephfs特有的，mds本身不存储数据，数据仍然存储在OSD，
但是访问文件系统的文件时，必须通过mds进程授权，获取文件inode等信息，才知道向哪个osd进程发送io请求。

# Daemon Start

mds daemon进程实现在文件src/ceph\_mds.cc，比较简单:

```cpp
int main(int argc, const char **argv) 
{
	......
	// 创建用于通信的messenger
	Messenger *msgr = Messenger::create(g_ceph_context, g_conf->ms_type,
			entity_name_t::MDS(-1), "mds",
			nonce);

	// 开始接收消息
	msgr->start();

	// 创建mds daemon对象
	mds = new MDSDaemon(g_conf->name.get_id().c_str(), msgr, &mc);

	// 初始化daemon
	r = mds->init();

	// 等待结束
	msgr->wait();
}

class MDSDaemon : public Dispatcher, public md_config_obs_t {
	Beacon  beacon; // beacon专门用来向mon发送和接收beacon消息，可以理解为mds向mon的心跳消息
	Messenger    *messenger; // 网络通信的组件
	MonClient    *monc; // 连接mon的组件
	MDSMap       *mdsmap; // mdsmap信息
	Objecter     *objecter; // 连接osd的组件
	MDSRankDispatcher *mds_rank; // 文件系统真正的核心实现类
}
```
主要类是MDSDaemon，看看这个类初始化干了什么事情:

```cpp
int MDSDaemon::init()
{
	// 初始化messenger, objecter, monc，以及订阅需要的map
	monc->sub_want("mdsmap", 0, 0);

	// 初始化beacon对象
	beacon.init(mdsmap);
}

void Beacon::init(MDSMap const *mdsmap)
{
	_send(); // 周期性地向mon发送beacon消息
}
```

这里将Beacon单独出一个类，而不是在MDSDaemon内部实现，主要是怕daemon忙碌的时候，由于mds\_lock的原因，耽误了beacon消息的收发。
daemon启动以后，就等待mdsmap消息和beacon消息，然后做出状态切换，比如谁是Active，谁是Standby，以及daemon负责哪些文件系统等。
而mdsmap的改变，和集群其他map一样，毫无疑问，是要通过paxos决议完成的，所以接下来先看看文件系统涉及的map。

# FSMap & MDSMap & MDSMonitor

### FSMap

FSMap用来记录整个集群内所有的文件系统，因为单个集群支持创建多个文件系统(虽然目前这个feature还不稳定)。

```cpp
class Filesystem
{
	public:
		fs_cluster_id_t fscid; // 文件系统的编号
		MDSMap mds_map; // 对应的mds信息
		......
};

class FSMap {
	......
	std::map<fs_cluster_id_t, std::shared_ptr<Filesystem> > filesystems; // 所有的文件系统
	std::map<mds_gid_t, fs_cluster_id_t> mds_roles; // 记录mds与文件系统的对应关系，如果是多活模式，多个mds会对应同一个文件系统

	std::map<mds_gid_t, MDSMap::mds_info_t> standby_daemons; // standby的mds
};
```

### MDSMap

一个文件系统，对应一个mdsmap，用来记录负责文件系统的mds进程的相关信息，主要信息如下:

```cpp
class MDSMap {
	std::set<int64_t> data_pools; // 数据池对应的pool id，一个文件系统是可以设置多个data pool的
	int64_t metadata_pool; // 元数据池对应的pool id

	mds_rank_t max_mds; // 文件系统包含的active mds个数

	// 与rank相关的信息
	std::set<mds_rank_t> in;
	std::set<mds_rank_t> failed, stopped, damaged;
	std::map<mds_rank_t, mds_gid_t> up;
	
	// mds的具体信息
	std::map<mds_gid_t, mds_info_t> mds_info;
};
```

内部类DaemonState枚举定义了mds所有的状态，状态怎么转化的可以参考具体的文档。mds\_info\_t包含mds的具体信息，比如id, name, state等。

### MDSMonitor

这个类是paxos的一个服务，可以参考以前monitor的相关文章，更新fsmap的时候(包括mds状态)，需要通过paxos完成:

```cpp
class MDSMonitor : public PaxosService {
	public:
		FSMap fsmap; // 当前的map
		FSMap pending_fsmap; // 待决议的map

		struct beacon_info_t {
			utime_t stamp;
			uint64_t seq;
		};
		map<mds_gid_t, beacon_info_t> last_beacon; // 记录daemon的beacon信息
};
```

# Create FileSystem

当集群部署好以后，没有文件系统，mds也处于standby状态，用户需要先创建文件系统才能使用:

> ceph fs new fs\_name metadata\_pool data\_pool

很显然，当执行这条命令后，fsmap会被更新，paxos执行流程可以参考[这里](http://blog.wjin.org/posts/ceph-monitor-paxosservice.html)，大致是这样:

> MDSMonitor::prepare\_update() -> MDSMonitor::prepare\_command() -> MDSMonitor::management\_command() -> MDSMonitor::create\_new\_fs()

首先是更新了MDSMonitor::pending\_fsmap成员信息，然后等待paxos进行propose，待paxos完成决议后，执行MDSMonitor::update\_from\_paxos(),
更新当前的fsmap为刚才决议的内容，并向订阅者推送最新fsmap/mdsmap，注意这时候虽然有文件系统，但是还没分配mds为其服务，mds收到新map的时候做不了太多事情。

# Allocate Rank

文件系统创建后，还没有mds进程为其服务，怎么分配mds是通过周期性tick函数调度:

```cpp
void MDSMonitor::tick()
{
	......
	bool do_propose = false;
	for (auto i : pending_fsmap.filesystems) {
		do_propose |= maybe_expand_cluster(i.second); // 如果需要更新，会做paxos决议
	}

	......

	if (do_propose) {
		propose_pending(); // 开始决议
	}
}

bool MDSMonitor::maybe_expand_cluster(std::shared_ptr<Filesystem> fs)
{
	bool do_propose = false;

	while (fs->mds_map.get_num_in_mds() < size_t(fs->mds_map.get_max_mds()) &&
			!fs->mds_map.is_degraded()) { // 当前负责文件系统的mds个数小于最大值
		......
		mds_gid_t newgid = pending_fsmap.find_replacement_for({fs->fscid, mds},
				name, g_conf->mon_force_standby_active);
		......
		pending_fsmap.promote(newgid, fs, mds); // 增加一个mds，并且会根据不同情况设置mds的初始状态，没有文件系统的时候其状态为STATE_CREATING
		do_propose = true;
	}

	return do_propose;
}
```

和上面一样，paxos完成后，执行MDSMonitor::update\_from\_paxos()，再次更新fsmap的内容，并向订阅者推送最新fsmap/mdsmap。

# FileSystem Init

mds进程收到最新map后，发现有文件系统派发给自己，就需要初始化，准备好为文件系统服务:

```cpp
void MDSDaemon::handle_mds_map(MMDSMap *m)
{
	......

	if (mds_rank == NULL) { // 新建一个rank，准备为文件系统服务
		mds_rank = new MDSRankDispatcher(whoami, mds_lock, clog,
				timer, beacon, mdsmap, messenger, monc, objecter,
				new C_VoidFn(this, &MDSDaemon::respawn),
				new C_VoidFn(this, &MDSDaemon::suicide));
		mds_rank->init();
	}

	......

	mds_rank->handle_mds_map(m, oldmap); // rank处理mdsmap，会对文件系统进行初始化
}

// MDSRankDispatcher继承自MDSRank，后者就是文件系统的核心，每个组建都单独独立出一个类完成
class MDSRank {
	const mds_rank_t whoami; // rank id

	MDSMap *&mdsmap; // 当前的mdsmap

	Objecter     *objecter; // 和osd通信的组件

	// 各个子系统的实现
	Server       *server;
	MDCache      *mdcache;
	Locker       *locker;
	MDLog        *mdlog;
	MDBalancer   *balancer;
	ScrubStack   *scrubstack;
	DamageTable  damage_table;

	InoTable     *inotable;

	SnapServer   *snapserver;
	SnapClient   *snapclient;

	......
};
```

rank初始化完成后，就会对最新的mdsmap进行处理:

```cpp
void MDSRankDispatcher::handle_mds_map(
		MMDSMap *m,
		MDSMap *oldmap)
{
	......
	// 很多case
	else if (is_creating()) { // 第一次的时候，需要新建文件系统
		boot_create(); // 初始化文件系统
	}
	......
}

void MDSRank::boot_create()
{
	......
	// 创建journal	
	mdlog->create(fin.new_sub());

	// 新建存放jorunal日志的segment 
	mdlog->prepare_new_segment();

	if (whoami == mdsmap->get_root()) {
		mdcache->create_empty_hierarchy(fin.get()); // 创建文件系统的根目录，分配ionde和dentry等
	}

	......
}

void MDCache::create_empty_hierarchy(MDSGather *gather)
{
	// 创建根目录的inode，默认编号为1
	CInode *root = create_root_inode();

	// 创建根目录
	CDir *rootdir = root->get_or_open_dirfrag(this, frag_t());

	......
}
```

文件系统就这样初始化完成了，后续就等待客户端进行mount操作，然后进行读写。

# Summary

本文总结了一下cephfs的初始化流程，从mds daemon启动开始，到文件系统最终被创建出来可以被用户使用，主要涉及paxos算法维护fsmap以及mdsmap的变更。但是，这还只是冰山一角，文件系统的核心实现(MDSRank中的各个组件)以及各种高大上的feature等等，还得慢慢挖掘，路漫漫其修远兮...
