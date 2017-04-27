---
layout: post
title: "Ceph Manager Overview"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

从Kraken版本开始，ceph新增加了一个daemon进程mgr，主要目的是将monitor的一些非paxos相关的服务(比如pg相关统计信息)单独移出来，
减轻monitor的负担，并且提供python插件接口用来获取统计信息，供其他监控系统使用。mgr采用master-standby模式，
部署的时候可以部署多个，避免单点故障，但只会存在一个mgr为active状态并提供服务。

# Daemon Start

mgr daemon进程实现在文件src/ceph_mgr.cc，比较简单:

```cpp
int main(int argc, const char **argv)
{
	......
	MgrStandby mgr;

	......
	int rc = mgr.init(); // 初始化

	......
	return mgr.main(args); // 等待退出
}
```

看看MgrStandby做了些什么初始化工作，代码在src/mgr/MgrStandby.[h\|cc]:

```cpp
int MgrStandby::init()
{
	// 初始化messenger, monitor client, objecter等组件
	......

	// 订阅mgrmap
	monc->sub_want("mgrmap", 0, 0);

	// 向monitor发送beacon消息，告诉自己已经启动
	send_beacon(); // 这个函数会周期性地定时调用，目的是告诉monitor自己还活着
	return 0;
}

int MgrStandby::main(vector<const char *> args)
{
	......
	client_messenger->wait(); // 和其他daemon一样，等待结束
	return 0;
}
```

执行到这里，似乎初始化就完成了:( mgr进程开始等待MMgrMap消息的到来，前面提到mgr是master-standby模式，
多个mgr进程，谁是master，谁是standby角色，这个得由mgrmap决定，最终由monitor通过paxos算法维护mgrmap的一致性，所以先看看mgrmap。

# MgrMap & MgrMonitor

mgrmap的信息比较简单，代码在src/mon/MgrMap.h:

```cpp
class StandbyInfo
{
	public:
		uint64_t gid; // 全局id, 这个id是由monitor client向monitor获取的
		std::string name;
};

class MgrMap
{
	public:
		epoch_t epoch = 0;

		uint64_t active_gid = 0; // active mgr进程的id
		entity_addr_t active_addr; // active mgr进程的地址
		std::string active_name; // active mgr进程的名字

		bool available = false; // 是否已经初始化完成，可以使用

		std::map<uint64_t, StandbyInfo> standbys; // standby mgr进程的信息汇总
};
```

mgrmap也是通过paxos算法更新，和其他map一样，存在一个paxos service服务，即MgrMonitor，继承自基类PaxosService，参考代码src/mon/MgrMonitor.[h\|cc]:

```cpp
class MgrMonitor : public PaxosService
{
	MgrMap map; // 当前的mgrmap
	MgrMap pending_map; // 待更新的mgrmap

	std::map<uint64_t, utime_t> last_beacon; // 记录每个mgr进程发送beacon消息的时间戳
	......
};
```

paxos运行流程可以参考[这篇文章](http://blog.wjin.org/posts/ceph-monitor-paxosservice.html)。对于MgrMonitor，目前只对两种消息进行响应，
即MSG\_MGR\_BEACON和MSG\_MON\_COMMAND:

* 收到beacon消息后，如果自己已经是active或者standby的，就不更新mgrmap，否则就需要更新mgrmap，
将mgr加进来，即执行prepare\_update阶段并返回true，当paxos完成后，执行函数update\_from\_paxos，然后调用check\_sub，
向已经订阅过mgrmap的客户端发送最新的map。

* 收到command消息后，执行命令，目前支持的命令仅仅是mgr fail，即将mgr进程强制移除mgrmap。

同时需要注意，MgrMonitor有个tick函数，内部周期性地检查active和standby mgr进程，如果beacon超时，就会将对应的mgr从map中删除，
超时时间由参数mon\_mgr\_beacon\_grace控制。

# Handle Mgrmap

前面知道mgr启动的时候向monitor发送了beacon消息，会导致mgrmap更新，并且自己订阅此类信息，所以会收到消息MMgrMap，处理函数如下:

```cpp
void MgrStandby::handle_mgr_map(MMgrMap* mmap)
{
	auto map = mmap->get_map();
	const bool active_in_map = map.active_gid == monc->get_global_id();

	if (active_in_map) {
		if (!active_mgr) {
			active_mgr.reset(new Mgr(monc, client_messenger, objecter)); // 如果自己在mgrmap中是active的，创建实例mgr，准备干活
			active_mgr->background_init(); // 执行初始化
		}
	} else {
		if (active_mgr != nullptr) { // 否则，销毁实例
			active_mgr->shutdown();
			active_mgr.reset();
		}
	}
}
```

可以发现，所有mgr daemon只会有一个进程创建mgr实例进行工作，即其他都处于standby状态，并未创建mgr实例对象。

# Mgr Instance

前面说了这么多，mgr到底是来干什么的还并没有涉及，这个就得从类Mgr入手了，代码在src/mgr/Mgr.[h\|cc]:

```cpp
class Mgr {
	......
	PyModules py_modules; // python插件相关的处理类
	DaemonStateIndex daemon_state; // 集群daemon进程metadata相关统计信息
	ClusterState cluster_state; // 整个集群状态
	DaemonServer server; // mgr是个service，需要绑定地址并进行监听，其他daemon，比如osd, mds等就可以连接mgr进行通信
	......
};
```

从此类的实现看，主要包含以下几方面:
 
* python插件相关的处理

* daemon进程的metadata统计信息，包括PerfCounters等

* 集群状态相关信息

* 与daemon进程通信的机制

下面简单分析下这几个模块。

### PyModules

mgr可以动态加载python插件，主要涉及代码PyModules.[h\|cc], MgrPyModule.[h\|cc], PyState.[h\|cc]:

```cpp
class PyModules
{
	std::map<std::string, std::unique_ptr<MgrPyModule>> modules; // 所有的插件，比如目前默认有两个插件rest和fsstatus，参考目录src/pybind/mgr
	std::map<std::string, std::unique_ptr<ServeThread>> serve_threads; // 每个插件的执行线程
	DaemonStateIndex &daemon_state; // daemon状态的引用，方便插件获取信息
	ClusterState &cluster_state; // cluster状态的引用，方便插件获取信息
};
```

看看插件的初始化流程:

```cpp
// mgr实例初始化的过程中，加载插件
void Mgr::init()
{
	......
	py_modules.init();
	py_modules.start();
	......
}

int PyModules::init()
{
	global_handle = this; // 这个全局变量定义在PyState.cc文件中，此文件实现了ceph_state模块，即封装了一些c++ wrapper函数，供python模块使用

	......
	auto py_logger = Py_InitModule("ceph_logger", log_methods); // 加载模块ceph_logger供插件调用，主要用来打印日志
	Py_InitModule("ceph_state", CephStateMethods); // 加载模块ceph_state供插件调用，主要用来获取集群状态daemonstate和clusterstate等相关信息

	......
	boost::tokenizer<> tok(g_conf->mgr_modules);
	for(const auto& module_name : tok) {
		auto mod = std::unique_ptr<MgrPyModule>(new MgrPyModule(module_name)); // 创建python模块实例
		int r = mod->load(); // 实例化python模块并且加载模块提供的command
	}
	......
}
```
需要注意的是，python插件一方面可以调用mgr提供的接口，参见模块ceph\_logger和ceph\_state中的函数，这是插件编写者主动调用，
另一方面，插件可能也需要在某些情况下被通知到，MgrPyModule::notify函数完成此功能，即mgr c++端主动调用python函数，插件编写者可以实现notify函数接口，
以便在状态信息发生变化时做出响应，具体怎么写插件，可以参考[官方文档](http://docs.ceph.com/docs/master/mgr/plugins/)。

### DaemonServer

DaemonServer的作用是让mgr在某个地址上监听，这样osd和mds等daemon进程可以连接此服务，同时也处理一些命令请求(mon内部的PGMonitor将来可能会被删除掉，移到mgr内部来)。

```cpp
class DaemonServer : public Dispatcher
{
	......
	Messenger *msgr; // 这个messenger用来绑定地址，让mgr成为一个service

	// 其他组件
	MonClient *monc;
	DaemonStateIndex &daemon_state;
	ClusterState &cluster_state;
	PyModules &py_modules;
	......
};
```

主要就是这个messenger需要注意，区别于其他地方的client\_messenger。在MgrStandy内部有个client\_messenger，并且将其传给了mgr，monc，objecter等，
这个是mgr作为客户端主动连接其他服务的时候使用的。而DaemonServer内部这个messenger，是作为服务端，即被动连接使用的，这个messenger的地址，
会更新在mgrmap中，其他daemon进程获取到mgrmap后就知道如何去连接mgr服务:

```cpp
void MgrStandby::send_beacon()
{
	bool available = active_mgr != nullptr && active_mgr->is_initialized();
	auto addr = available ? active_mgr->get_server_addr() : entity_addr_t(); // DaemonServer内部的messenger地址
	MMgrBeacon *m = new MMgrBeacon(monc->get_fsid(),
			monc->get_global_id(),
			g_conf->name.get_id(),
			addr,
			available);

	monc->send_mon_message(m);
	......
}
```

有了server，肯定还有client，相关代码在src/mgr/MgrClient.[h\|cc]，这个类似于MonClient用来连接mon，MgrClient用来连接mgr。

以osd daemon为例，在启动过程中，会发送消息MMgrOpen，mgr收到后，会回复MMgrConfigure消息，主要是返回一个period时间，后续osd就根据设定的period，
定期将状态信息上报给mgr，即消息MMgrReport和MPGStats:

```cpp
int OSD::init()
{
	......
	mgrc.init();
	client_messenger->add_dispatcher_head(&mgrc); // 将mgr client加入messenger的dispatch列表，这样mgr client可以处理mgrmap等消息

	......
	monc->sub_want("mgrmap", 0, 0); // 订阅mgrmap
	......
}
```

osd收到mgrmap消息后，会分发给MgrClient处理:

```cpp
bool MgrClient::handle_mgr_map(MMgrMap *m)
{
	map = m->get_map();
	m->put();

	if (!session || session->con->get_peer_addr() != map.get_active_addr()) {
		reconnect(); // 建立连接
	}

	return true;
}

void MgrClient::reconnect()
{
	......
	if (g_conf && !g_conf->name.is_client()) {
		auto open = new MMgrOpen();
		open->daemon_name = g_conf->name.get_id();
		session->con->send_message(open); // 发送MMgrOpen消息给mgr进程
	}
	......
}
```

如果向mgr进程发送命令请求，流程也类似，都是通过MgrClient完成。

### ClusterState

ClusterState主要包括fsmap, pgmap，以及健康状态信息，结构如下:

```cpp
class ClusterState
{
	FSMap fsmap; // fsmap信息
	PGMap pg_map; // pgmap信息
	bufferlist health_json; // health信息
	bufferlist mon_status_json; // monitor状态信息
};
```

Mgr实例在初始化过程中订阅过mgrdigest和fsmap:

```cpp
void Mgr::init()
{
	......
	monc->sub_want("mgrdigest", 0, 0);
	monc->sub_want("fsmap", 0, 0);
}
````

health\_json和mon\_status\_json由MgrMonitor定期将mon进程内部记录的状态信息通过消息MMgrDigest发送给订阅的mgr进程，
fsmap由MDSMonitor在完成paxos算法更新fsmap后，通过消息MFSMap推送给订阅者mgr，这个只会推送一次，不同于digest。
对于pgmap的更新，则是osd进程通过MgrClient定期将自己的pg统计信息通过消息MPGStats发送给mgr完成更新。

### DaemonState

这个主要就是osd进程的PerfCounters信息，由osd进程通过MgrClient定期发送消息MMgrReport完成更新。
内部实现的时候对上报信息稍微进行了加工，方便通过域名，ID或服务类型进行查找。

# Summary

* mgr采用master-standby模式，mon进程通过paxos算法维护mgrmap的一致性，决定谁是master或active mgr

* mgr进程启动的时候，通过订阅mgrmap，并且周期性地发送beacon消息给mon，mon根据消息更新mgrmap

* mgr收到mgrmap的更新，如果自己被选为active，则会实例化类Mgr进行干活

* Mgr类功能主要包括: 
	* python模块相关处理对象PyModules
	
	* 将自己作为服务的DaemonServer
	
	* 集群统计信息ClusterState与daemon统计信息DaemonState

注意区分MgrStandby中的client messenger和DaemonServer中的server messenger，同时也注意区分各个类中dispatch消息的类型。
最后说明一下，mgr只是将一些信息进行了存储，不会影响集群的正确运转，存储的信息可以通过python插件，比如默认的rest插件，
提供rest接口方便其他监控系统获取信息。

另外，鉴于mgr新添加不久，这块代码还不够稳定，未来可能会有一些较大变动，比如bug修复以及将PGMonitor功能移到mgr内部，
也可能添加更多的接口供python插件使用等等，暂时分析到此，如有必要再更新，记录下代码commit ID: 3c257ef131bbd825ac9c1373a21f92d4dee8b047。
