---
layout: post
title: "Ceph BlueStore BlockDevice"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

Ceph新的存储引擎BlueStore在Luminous版本已经变成默认的存储引擎，这个存储引擎替换了以前的FileStore存储引擎，彻底抛弃了对文件系统的依赖，由Ceph OSD进程直接管理裸盘的存储空间，通过libaio的方式进行读写操作。实现的时候抽象出BlockDevice基类类型，统一管理各种类型的设备，如Kernel, NVME和NVRAM等，为裸盘的使用者(BlueFS/BlueStore)提供统一的操作接口。同时，为了紧跟存储技术的最新进展，将支持NVME的spdk集成进来，完全通过用户态程序操作NVME磁盘，提升iops的同时大大缩短io操作的延时。

# Source Code

BlockDevice类图继承关系如下:

### KernelDevice 

鉴于目前大多数部署还是使用的hdd和sata ssd，故以此为例作介绍，对应的派生类是KernelDevice，主要数据成员如下:

```cpp
class KernelDevice : public BlockDevice {
	int fd_direct, fd_buffered; // 分别存放以direct和buffered两种方式打开裸设备时的fd
	uint64_t size; // 设备总的大小
	uint64_t block_size; // 块的大小
	std::string path; // 路径
	bool aio, dio;

	// libaio相关的线程
	struct AioCompletionThread : public Thread {
		KernelDevice *bdev;
		explicit AioCompletionThread(KernelDevice *b) : bdev(b) {}
		void *entry() override {
			bdev->_aio_thread();
			return NULL;
		}
	} aio_thread;

	// aio操作相关的队列和回调函数
	aio_queue_t aio_queue;
	aio_callback_t aio_callback;
	void *aio_callback_priv;
	bool aio_stop;
};
```

### Init Device

BlueFS用户态文件系统会使用BlockDevice存放文件系统的数据，BlueStore也会使用BlockDevice存放object相关的数据，创建BlockDevice的时候，通过工厂函数create，根据不同的设备类型，创建不同的设备，并提供callback方法及其参数:

```cpp
BlockDevice *BlockDevice::create(CephContext* cct, const string& path,
		aio_callback_t cb, void *cbpriv) {

	// 通过设备的path，判断出设备的类型
	string type = "kernel";
	char buf[PATH_MAX + 1];
	int r = ::readlink(path.c_str(), buf, sizeof(buf) - 1);
	if (r >= 0) {
		buf[r] = '\0';
		char *bname = ::basename(buf);
		if (strncmp(bname, SPDK_PREFIX, sizeof(SPDK_PREFIX)-1) == 0)
			type = "ust-nvme";
	}

	if (type == "kernel") {
		return new KernelDevice(cct, cb, cbpriv); // kernel
	}
	......
	if (type == "ust-nvme") {
		return new NVMEDevice(cct, cb, cbpriv); // nvme
	}
	......
}
```

创建设备后，接下来就是打开设备并对设备的基础参数进行初始化:

```cpp
int KernelDevice::open(const string& p)
{
	// 分别以fd_direct/fd_buffered方式打开块设备
	fd_direct = ::open(path.c_str(), O_RDWR | O_DIRECT);
	fd_buffered = ::open(path.c_str(), O_RDWR);

	// 读取block size等参数
	block_size = cct->_conf->bdev_block_size;
	......

	// 如果是aio，初始化aio相关参数，并启动aio线程
	r = _aio_start();
	......
}
```

此时device已经ready，可以进行读写操作。需要说明的是，设备的空间怎么管理，是由设备的使用方，比如BlueFS和BlueStore完成，设备只提供IO操作接口。

### aio\_write

设备初始化完成后，就可以调用相应接口进行IO的读写操作。KernelDevice提供同步读写接口read/write和异步读写接口aio\_read/aio\_write。如果是异步的，调用aio\_write准备数据到buffer，后续还要调用aio\_submit将请求提交，io执行完成后会由线程aio thread执行回调函数。这里以最复杂的流程aio\_write为例介绍。

aio接口是通过libaio完成的，libaio怎么使用可以参考网上的[文章](http://blog.csdn.net/heyutao007/article/details/7065166)，Ceph将其封装在类IOContext中，每个device对应一个IOContext:

```cpp
struct IOContext {
	private:
		std::mutex lock;
		std::condition_variable cond;

	public:
		void *priv;

		std::list<aio_t> pending_aios;    // 待执行的aio
		std::list<aio_t> running_aios;    // 正在执行的aio

		// 计数
		std::atomic_int num_pending = {0};
		std::atomic_int num_running = {0};
};

struct aio_t {
	struct iocb iocb; // libaio相关的结构体
	......
	void pwritev(uint64_t _offset, uint64_t len) {
		offset = _offset;
		length = len;
		io_prep_pwritev(&iocb, fd, &iov[0], iov.size(), offset); // 准备数据
	}
	......
};

```

执行aio\_write，实际上是在Device对应的IOContext结构体的成员变量pending\_aios中追加了一个和libaio相关的aio\_t结构:

```cpp
int KernelDevice::aio_write(uint64_t off, bufferlist &bl, IOContext *ioc, bool buffered)
{
	......
	if (aio && dio && !buffered) {
		ioc->pending_aios.push_back(aio_t(ioc, fd_direct)); // 放入IOContext的pending队列，等待执行
		++ioc->num_pending;

		// 将待写入的数据准备在aio的buffer中
		aio_t& aio = ioc->pending_aios.back();
		bl.prepare_iov(&aio.iov);
		for (unsigned i=0; i<aio.iov.size(); ++i) {
			aio.bl.claim_append(bl);
			aio.pwritev(off, len); // 写buffer
		}
	}
}
```


### aio\_submit

使用方调用aio\_write准备数据后，紧接着会调用aio\_submit提交IO请求:

```cpp
void KernelDevice::aio_submit(IOContext *ioc)
{
	if (ioc->num_pending.load() == 0) {
		return;
	}

	// 获取pending的aio
	list<aio_t>::iterator e = ioc->running_aios.begin();
	ioc->running_aios.splice(e, ioc->pending_aios);
	......

	// 批量提交aio
	r = aio_queue.submit_batch(ioc->running_aios.begin(), e, 
			ioc->num_running.load(), priv, &retries);
}

int aio_queue_t::submit_batch(aio_iter begin, aio_iter end, 
		uint16_t aios_size, void *priv, 
		int *retries)
{
	......
	while (left > 0) {
		int r = io_submit(ctx, left, piocb + done); // 调用libaio相关的api提交io
	}
	......
}
```

### aio\_thread

提交aio请求后，device的使用方就完成了，需要单独的线程来检查io的完成情况，当真正完成的时候，执行回调函数通知调用方，此线程即为设备对应的aio thread线程，线程入口如下:

```cpp
void KernelDevice::_aio_thread()
{
	......
	while (!aio_stop) {
		int r = aio_queue.get_next_completed(cct->_conf->bdev_aio_poll_ms, // 调用libaio相关的aio，检查io是否完成
				aio, max);

		if (ioc->priv) {
			if (--ioc->num_running == 0) {
				aio_callback(aio_callback_priv, ioc->priv); // 执行回调
			}
		}
	}
	......
}

int aio_queue_t::get_next_completed(int timeout_ms, aio_t **paio, int max)
{
	......
	do {
		r = io_getevents(ctx, 1, max, event, &t); // 调用libaio相关的api，获取已经完成aio请求
	} while (r == -EINTR);
}
```

# Config

除了debug相关参数外，主要配置参数与libaio相关，目前看来，基本不需要做调整:

```shell
bdev_aio # 默认为true。不能修改，现在只支持aio方式操作磁盘
bdev_aio_poll_ms # libaio API io_getevents的超时时间，默认为250
bdev_aio_max_queue_depth # libaio API io_setup的最大队列深度, 默认为1024
bdev_aio_reap_max # libaio API io_getevents每次请求返回的最大条目数
bdev_block_size # 磁盘块大小，默认4096字节

# nvme相关参数
bdev_nvme_unbind_from_kernel
bdev_nvme_retry_count
```

# Summary

* 以BlockDevice为基类，抽象出了不同类型的Device，包括KernelDevice、NVMEDevice和PMEMDevice等。

* Device的使用者通过设备提供的create和open接口初始化设备。

* Device提供同步和异步的读写接口，以及flush等操作保证数据落盘。

* 异步写操作通过aio\_write和aio\_submit接口完成，Device的aio相关的线程会在io执行完成后，执行回调函数通知调用者。

