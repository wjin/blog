---
layout: post
title: "Ceph AsyncMessenger Stack"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

随着硬件设备的快速发展，存储系统的瓶颈逐渐转化为存储系统软件本身，因而出现了DPDK/SPDK这样的开发组件，
帮助存储系统开发者借助最新的硬件技术开发存储系统软件，提升性能。

对于ceph这样越来越受到开发者青睐的开源分布式存储系统，拥抱这样的新技术也是顺理成章，后端的单机存储引擎，最新的BlueStore已经支持SPDK。
对于网络层，ceph通过在AsyncMessenger这一层加一个抽象的NetworkStack，用以支持不同的协议栈（Posix/DPDK/RDMA)。

在早期的[这篇](http://blog.wjin.org/posts/ceph-async-messenger.html)文章中，详细分析过AsyncMessenger的工作流程，那时候参考的代码是hammer版本，
AsyncMessenger框架基本上没怎么改变，这篇文章分析怎么引入NetworkStack这一抽象层，以支持不同的协议栈，主要以PosixStack为例加以说明。
以master代码举例，commit值为5b97cce360fe1f6b15dfad0866d90c85262f8253。

# Initialize

还是先从构造函数入手:

```cpp
AsyncMessenger::AsyncMessenger(CephContext *cct, entity_name_t name,
		string mname, uint64_t _nonce)
	: SimplePolicyMessenger(cct, name,mname, _nonce),
	......
{
	ceph_spin_init(&global_seq_lock);
	StackSingleton *single;
	cct->lookup_or_create_singleton_object<StackSingleton>(single, "AsyncMessenger::NetworkStack"); // 一个进程对应一个唯一的stack，以前这里是WorkerPool
	stack = single->stack.get();
	stack->start(); // 启动线程
	......
}

struct StackSingleton {
	std::shared_ptr<NetworkStack> stack;
	StackSingleton(CephContext *c) {
		stack = NetworkStack::create(c, c->_conf->ms_async_transport_type); // 根据类型创建不同的stack
	}
	~StackSingleton() {
		stack->stop();
	}
};

NetworkStack::NetworkStack(CephContext *c, const string &t): type(t), started(false), cct(c)
{
	......
	for (unsigned i = 0; i < num_workers; ++i) {
		Worker *w = create_worker(cct, type, i); // 创建worker
		w->center.init(InitEventNumber, i); // 初始化事件处理器
		workers.push_back(w);
	}
	......
}
```

以前在构造函数中，是创建一个WorkerPool，用来管理所有worker，并且worker类继承自线程类，现在换成抽象的NetworkStack，并且将线程移入stack自己实现，
而worker类演变为仅仅对应一个EventCenter，根据各个stack实现自己的worker，继承关系如下:

![img](/assets/img/post/ceph_asyncmessenger_networkstack.png)
![img](/assets/img/post/ceph_asyncmessenger_worker.png)

接下来就是初始化干活的线程:

```cpp
void NetworkStack::start()
{
	......
	for (unsigned i = 0; i < num_workers; ++i) {
		if (workers[i]->is_init())
			continue;
		std::function<void ()> thread = add_thread(i); // 获取线程入口的匿名函数
		spawn_worker(i, std::move(thread)); // 启动线程
	}
	started = true;
	......
}

// PosixNetworkStack创建线程的实现
virtual void spawn_worker(unsigned i, std::function<void ()> &&func) override {
	threads.resize(i+1);
	threads[i] = std::thread(func); // 使用c++11的线程库创建线程
}

// 线程入口函数
std::function<void ()> NetworkStack::add_thread(unsigned i)
{
	Worker *w = workers[i];
	return [this, w]() {
		const uint64_t EventMaxWaitUs = 30000000;
		w->center.set_owner();
		w->initialize();
		w->init_done();
		while (!w->done) {
			int r = w->center.process_events(EventMaxWaitUs); // 处理事件
		}
	}
	w->reset();
	w->destroy();
};
```

这里在AsyncMessenger类构造函数中，直接就将stack的线程初始完毕，准备处理事件，而线程的入口函数和worker的事件处理器相关，这就涉及到不同worker的实现。
前面提到一个worker对应一个EventCenter，而在EventCenter里面，对应一个EventDriver，对于DPDK，因为有用户态的poll接口，这里实现了自己的DPDKDriver(仍然是继承EventDriver)，
并且在EventCenter里面增加了poll相关的结构体。

# Worker

Worker的抽象类实现比较简单，最主要就是两个抽象接口listen和connect，前者用来创建一个在给定地址监听的套接字，这个套接字可以处理接下来的监听请求，
后者用来创建一个已经连接成功的套接字，可以通过它读写数据。

```cpp
class Worker {
	unsigned id; // worker的ID
	EventCenter center; // 处理事件

	......
	virtual int listen(entity_addr_t &addr,
			const SocketOptions &opts, ServerSocket *) = 0;
	virtual int connect(const entity_addr_t &addr,
			const SocketOptions &opts, ConnectedSocket *socket) = 0;
	......
};
```

对于不同协议栈的实现，socket的实现肯定是不一样的，区别就在这里，所以抽象出来了两个通用的wrapper类ServerSocket和ConnectedSocket，用来隐藏细节:

```cpp
class ConnectedSocket {
	std::unique_ptr<ConnectedSocketImpl> _csi; // 依赖于具体协议栈的实现
	ssize_t read(char* buf, size_t len) { // 读数据
		return _csi->read(buf, len);
	}
	ssize_t send(bufferlist &bl, bool more) { // 写数据
		return _csi->send(bl, more);
	}
};

class ServerSocket {
	std::unique_ptr<ServerSocketImpl> _ssi; // 依赖于具体协议栈的实现

	int accept(ConnectedSocket *sock, const SocketOptions &opt, entity_addr_t *out, Worker *w) { // 接受连接请求，成功后第一个参数为已经成功连接的套接字
		return _ssi->accept(sock, opt, out, w);
	}
};
```

看看posix的实现，和以前一样：

```cpp
int PosixWorker::listen(entity_addr_t &sa, const SocketOptions &opt,
		ServerSocket *sock)
{
	int listen_sd = net.create_socket(sa.get_family(), true); // 创建套接字
	r = ::bind(listen_sd, sa.get_sockaddr(), sa.get_sockaddr_len()); // 绑定地址
	r = ::listen(listen_sd, 128); // 开始监听
	*sock = ServerSocket(
			std::unique_ptr<PosixServerSocketImpl>(
				new PosixServerSocketImpl(net, listen_sd))); // 创建serversocket，返回给用户，可以用此socket进行accept请求
	return 0;
}

int PosixWorker::connect(const entity_addr_t &addr, const SocketOptions &opts, ConnectedSocket *socket)
{
	int sd;
	sd = net.connect(addr); // 发起连接
	// 创建connectedsocket，返回给用户，可以用此socket进行send/read
	*socket = ConnectedSocket(std::unique_ptr<PosixConnectedSocketImpl>(new PosixConnectedSocketImpl(net, addr, sd, !opts.nonblock)));
	return 0;
}
```

# Socket

不同的通信模式实现不同的抽象socket接口，继承关系如下:

![img](/assets/img/post/ceph_asyncmessenger_socket.png)

以posix的实现为例说明:

```cpp
int PosixServerSocketImpl::accept(ConnectedSocket *sock, const SocketOptions &opt, entity_addr_t *out, Worker *w)
{
	int sd = ::accept(_fd, (sockaddr*)&ss, &slen); // 接受连接请求

	std::unique_ptr<PosixConnectedSocketImpl> csi(new PosixConnectedSocketImpl(handler, *out, sd, true)); // 创建连接成功的socket
	*sock = ConnectedSocket(std::move(csi)); // 返回给用户使用
	return 0;
}
```

ConnectedSocket的实现主要是read/send接口，对应于read和sendmsg等系统调用，这里不在介绍。

# Listen

到此，我们了解到不同stack创建了不同的worker，而worker通过listen和connect创建出可以监听的serversocket和可以收发数据的connectedsocket，不同的实现对应于不同的socket。
async messenger的其他模块就通过这两个抽象的套接字进行监听和数据的读写，实现网络通信的功能。

以进程怎么绑定到特定地址进行监听来举例说明，回顾以前的文章:

> AsyncMessenger.bind -> Processor.bind()

```cpp
int Processor::bind(const entity_addr_t &bind_addr,
		const set<int>& avoid_ports,
		entity_addr_t* bound_addr)
{
	......
	// 向worker的事件中心提交一个外部事件，当这个事件执行的时候，就是执行第二个参数的匿名函数对象，也就是调用worker的listen
	worker->center.submit_to(worker->center.get_id(), [this, &listen_addr, &opts, &r]() {
			r = worker->listen(listen_addr, opts, &listen_socket); // 事件被执行后，listen_socket就被更新，可以用它来进行accept
	}, false);
	......
}
```

和以前类似，监听的socket有了，但是并没有将fd加入事件中心进行管理，还是得等osd初始化的时候:

> OSD.init() -> AsyncMessenger.ready() -> Processor.start()

```cpp
void Processor::start()
{
	// 向worker的事件中心提交一个外部事件，当这个事件执行的时候，就是执行第二个参数的匿名函数对象，也就是将fd加入事件中心进行管理
	if (listen_socket) {
		worker->center.submit_to(worker->center.get_id(), [this]() {
			worker->center.create_file_event(listen_socket.fd(), EVENT_READABLE, listen_handler); }, false);
	}
}

// 上面加入的可读事件，回调是listen_handler，即执行如下函数
void Processor::accept()
{
	.......
	while (true) {
		int r = listen_socket.accept(&cli_socket, opts, &addr, w); // 执行监听socket的accept，成功后第一个参数的值为connectedsocket类型，可以用来收发数据
	}
	......
}
```

# Connect

再来看一个客户端连接的例子，回顾以前文章:

> AsyncMessenger.create_connect() -> AsyncConnection.connect() -> AsyncConnection._process_connection()

```cpp
ssize_t AsyncConnection::_process_connection()
{
	......
	case STATE_CONNECTING:
		{
			......
			r = worker->connect(get_peer_addr(), opts, &cs); // 调用worker的connect
			center->create_file_event(cs.fd(), EVENT_READABLE, read_handler); // 将连接成功的connectedsocket加入事件中心进行管理
			state = STATE_CONNECTING_RE;
			break;
		}
}
```

接下来就是通过connectedsocket进行数据的读取和发送，框架基本没变。

# Summary

总结一下，引入了一层NetworkStack以及对应不同stack的worker，其中worker最主要的用途就是listen和accept接口，用于创建一个抽象的ServerSocket和ConnectedSocket，
前者用来监听请求，后者用来读写数据。而AsyncMessenger其他地方的主要改动就是调用worker的这两个抽象接口，对于DPDK的轮询模式，
在worker对应的EventCenter内部增加了一些特殊处理，并且增加了DPDKDriver。
