---
layout: post
title: "CephFS Remove File Error"
description: ""
category: ceph
tags: [ceph-ops]
---
{% include JB/setup %}

# 问题

在使用cephfs的过程中，用户报告删除文件的时候，出现错误: **No space left on device。** 通过简单搜索，发现社区有类似的记录，从邮件列表讨论看，一方面原因是参数的限制，另外一方面存在潜在的bug，特别是在硬链接的时候容易触发。

# 分析

cephfs分布式文件系统在删除文件的时候，和传统单机文件系统还是有区别的。对于单机文件系统，比如ext4，删除的时候，只需要将文件对应的inode等元数据释放即可，对于数据区不需要执行删除，后续直接覆盖写就好了。但是对于ceph这样的分布式文件系统，删除文件的时候不仅要操作文件系统的元数据，还必须把文件对应的内容删除掉，否则就会造成磁盘空间泄漏。因为一个文件可能对应后端rados集群的多个object，这些object名称全局唯一的，如果不删除object，那么object占用的磁盘空间就浪费了。

既然要删除内容本身，自然而然会遇到一个问题，当文件非常大的时候，文件会被映射到很多个rados object，删除大量object肯定会非常耗时，所以必须采用异步的方式，而且还必须控制好删除的粒度，避免影响集群性能。对于cephfs来说，删除文件就是一个**unlink**操作，将待删除文件移动到一个特殊的目录下面(ceph mds内部叫**stray**目录)，然后更新文件系统的元数据信息，告知用户删除操作成功，后续再慢慢清理文件系统的stray目录包含的内容。

stray是cephfs分布式文件系统的一个特殊目录，用户不可见，但是在实现的时候，和普通的目录并无区别，一样遵循文件系统内部设定的参数，比如参数**mds\_bal\_fragment\_size\_max**，这个参数限制目录能够创建的文件个数，默认是10万。前面提到删除的时候需要注意并发度，不要占用太多的集群带宽，但是当大量用户都执行批量删除的时候，stray目录增加的速度远大于删除的速度，stray目录很快就会被耗尽，此时就会发生用户报告的错误(这里stray目录实际上做了个分片，总共有10个stray，总计可以容纳一百万文件)。

另外一直种情况是硬连接的场景，stray目录的内容删除不掉，从邮件讨论看代码存在潜在bug，我们的使用场景应该没有硬连接。这里简单说一下ceph的实现，和单机文件系统一样，硬链接的时候增加inode的引用计数，多个dentry会对应同一个inode，为了提升性能，ceph实现的时候将inode嵌入到dentry，为了便于区分inode是否是硬链接，将其抽象成了类型linkage，分别用primary和remote来记录其类型。

单个目录文件个数太多，会影响性能，而且也有另外的bug，所以不建议在生产环境调整stray的空间。对于删除速度，有参数可以调整并发度，但是删除数据需要占用集群的带宽，如果调整太大，会影响集群的正常使用，鉴于后台清理工作不是那么急迫，建议保持默认的参数。

对于生产环境，可以增加一些stray相关的metrics，实时监控stray的大小，如有异常，提前通知用户。同时，可以修改ceph源码，将stray的分片大小改为可配置的参数，这样方便用户根据需要进行修改。

# 源码分析

涉及的相关重要数据结构，需要明白**目录**、**目录项**、**inode**等之间的关系:

```cpp
// 目录
class CDir : public MDSCacheObject {
	  typedef std::map<dentry_key_t, CDentry*> map_t;
	  map_t items; // 目录包含的目录项
	  ......
};

// 目录项
class CDentry : public MDSCacheObject, public LRUObject {
	struct linkage_t { // 抽象的inode结构体，主要为了实现硬连接，一般为primary，硬链接为remote
		CInode *inode;
		inodeno_t remote_ino;
		unsigned char remote_d_type;
	};

	protected:
	linkage_t linkage; // inode嵌入到目录项
	......
};

// inode
class CInode : public MDSCacheObject, public InodeStoreBase {
	protected:
		// parent dentries in cache
		CDentry         *parent; // 一般情况
		compact_set<CDentry*>    remote_parents; // 硬链接
	......
};

#define NUM_STRAY 10
class MDCache {
	public:
		CInode *strays[NUM_STRAY]; // strays目录的inode
	......
};
```

请求处理的大致流程:

```cpp
// Server组件用来维护session，处理客户端的请求，删除操作为unlink请求
void Server::handle_client_unlink(MDRequestRef& mdr)
{
	......
	CDentry::linkage_t *dnl = dn->get_linkage(client, mdr);
	straydn = prepare_stray_dentry(mdr, dnl->get_inode());
	......
}

CDentry* Server::prepare_stray_dentry(MDRequestRef& mdr, CInode *in)
{
	......
	CDir *straydir = mdcache->get_stray_dir(in); // 返回数组strays中的一个dir，这个数组长度为10
	straydn = mdcache->get_or_create_stray_dentry(in);
	......
}

CDentry *MDCache::get_or_create_stray_dentry(CInode *in)
{
	CDir *straydir = get_stray_dir(in);
	CDentry *straydn = straydir->lookup(straydname);

	if (!straydn) {
		straydn = straydir->add_null_dentry(straydname); // 增加一个dentry到stray目录
		straydn->mark_new();
	}

	......
}
```

加入stray目录后，还没有真正删除，删除是通过周期性的tick函数触发:

```cpp
void MDSDaemon::tick()
{
	if (mds_rank) {
		mds_rank->tick();
	}
	......
}

void MDSRankDispatcher::tick()
{
	if (is_active() || is_stopping()) {
		mdcache->trim();
	}
}

bool MDCache::trim(int max, int count)
{
	......
	trim_inode(NULL, subtree_in, NULL, expiremap);
	......
}

bool MDCache::trim_inode(CDentry *dn, CInode *in, CDir *con, map<mds_rank_t, MCacheExpire*>& expiremap)
{
	......
	maybe_eval_stray(in);
	......
}

void MDCache::maybe_eval_stray(CInode *in, bool delay) {
	......
	if (dn->get_projected_linkage()->is_primary() &&
			dn->get_dir()->get_inode()->is_stray()) {
		stray_manager.eval_stray(dn, delay);
	}
}
```

代码实现的时候，各种细节需要考虑，实际情况非常复杂，不过最终都会调用函数maybe\_eval\_stray，判断文件是否能够最终被删除，而删除操作通过StrayManager的purge函数完成，大致处理流程:

> eval\_stray -> \_\_eval\_stray -> enqueue -> consume -> purge 

purge完成后，回调\_purge\_stray\_purged，这边会一直迭代purge，直到条件不允许:

>  \_advance -> \_purge\_stray\_purged -> \_advance

# 选项

可调整的参数:

```shell
mds_bal_fragment_size_max // 单个目录容纳的文件个数

mds_max_purge_files // 删除时并发文件的上限
mds_max_purge_ops // 删除时操作个数的上限
mds_max_purge_ops_per_pg // 每个pg的操作上限
```

监控选项:

```shell
    "mds_cache": {
        "num_strays": {
            "type": 2,
            "description": "Stray dentries",
            "nick": "stry"
        },
        "num_strays_purging": {
            "type": 2,
            "description": "Stray dentries purging",
            "description": "Stray dentries",
            "nick": "stry"
        },
        "num_strays_purging": {
            "type": 2,
            "description": "Stray dentries purging",
            "nick": ""
        },
        "num_strays_delayed": {
            "type": 2,
            "description": "Stray dentries delayed",
            "nick": ""
        },
        "num_purge_ops": {
            "type": 2,
            "description": "Purge operations",
            "nick": ""
        },
        "strays_created": {
            "type": 10,
            "description": "Stray dentries created",
            "nick": ""
        },
        "strays_purged": {
            "type": 10,
            "description": "Stray dentries purged",
            "nick": "purg"
        },
        "strays_reintegrated": {
            "type": 10,
            "description": "Stray dentries reintegrated",
            "nick": ""
        },
        "strays_migrated": {
            "type": 10,
            "description": "Stray dentries migrated",
            "nick": ""
        },
        "num_recovering_processing": {
            "type": 2,
            "description": "Files currently being recovered",
            "nick": ""
        },
        "num_recovering_enqueued": {
            "type": 2,
            "description": "Files waiting for recovery",
            "nick": "recy"
        },
        "num_recovering_prioritized": {
            "type": 2,
            "description": "Files waiting for recovery with elevated priority",
            "nick": ""
        },
        "recovery_started": {
            "type": 10,
            "description": "File recoveries started",
            "nick": ""
        },
        "recovery_completed": {
            "type": 10,
            "description": "File recoveries completed",
            "nick": "recd"
        }
    }
```

# 参考

[1.https://www.mail-archive.com/ceph-users@lists.ceph.com/msg43007.html](https://www.mail-archive.com/ceph-users@lists.ceph.com/msg43007.html)

[2.http://lists.ceph.com/pipermail/ceph-users-ceph.com/2016-October/013397.html](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2016-October/013397.html)

