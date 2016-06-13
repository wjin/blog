---
layout: post
title: "Ceph Monitor PaxosService"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

PaxosService是一个虚基类，内部利用Paxos类的功能，包装了一些接口，即提供一些模板方法，用来构建基于paxos的服务。
目前所有服务如下图所示:

![img](/assets/img/post/ceph_mon_paxosservice.png)

如果考虑需要实现自己的一个能够利用paxos的服务，应该从何入手？大致应该考虑如下几个方面:

* Init
* Restart
* Process
* Update
* Active

# Init

monitor进程启动的时候，会初始化paxos及其服务，如果服务需要特殊初始化，应该重载基类PaxosService::init接口:

```cpp
virtual void init() {}
```

调用流程:  Monitor::preinit() -> Monitor::init_paxos() -> FooService::init()

# Restart

monitor进程在很多情况下会重新进入bootstrap流程，这个过程会restart服务，应该重载基类PaxosService::on_restart接口:

```cpp
virtual void on_restart() { }
```

调用流程: Monitor::bootstrap() -> Monitor::\_reset() -> PaxosService::restart() -> FooService::on\_restart()

# Process

服务需要正常工作，一是对收到的命令进行响应(当然命令也封装在消息中)，二是对收到的消息进行响应，如果需要进行paxos round，则发起决议，待完成后更新处理结果，
这些流程基类PaxosService都提供了模板方法，服务只需要实现特定接口即可:

```cpp
bool PaxosService::dispatch(MonOpRequestRef op)
{

  PaxosServiceMessage *m = static_cast<PaxosServiceMessage*>(op->get_req()); // 消息类型

  ......

  // 只读消息，处理后直接返回
  if (preprocess_query(op)) 
    return true;

  // 如果不是只读，那么要做更新，需要paxos round，必须leader节点处理
  if (!mon->is_leader()) {
    mon->forward_request_leader(op); // 非leader，转发消息
    return true;
  }
  
  // 如果目前不可更新，等待重试
  if (!is_writeable()) {
    wait_for_writeable(op, new C_RetryMessage(this, op));
    return true;
  }

  // 更新
  if (prepare_update(op)) { // 准备更新，

    double delay = 0.0;
    if (should_propose(delay)) {

      if (delay == 0.0) {
		propose_pending(); // 发起决议
      } else {
		mon->timer.add_event_after(delay, proposal_timer); // 等待一段时间后再决议
	  }
    }
  }
  ......

  return true;
}
```

不难看出，基类大部份功能都实现好了，服务需要实现的主要接口为preprocess\_query和prepare\_update。前者处理一些只读命令或消息，
后者处理需要修改操作的命令或消息。

需要决议什么内容？派生类需要实现接口encode\_pending:

```cpp
void PaxosService::propose_pending()
{
  ......

  MonitorDBStore::TransactionRef t = paxos->get_pending_transaction(); // 获取paxos的transaction

  ......

  encode_pending(t); // 将决议的内容放入transaction中
  have_pending = false;

  proposing = true;
  paxos->queue_pending_finisher(new C_Committed(this)); // 完成后的回调
  paxos->trigger_propose(); // 发起决议
}
```

# Update

决议完成后，需要更新决议的内容，需要实现接口update\_from_paxos:

调用流程如下: Paxos::do\_refresh() -> Monitor::refresh\_from\_paxos() -> PaxosService::refresh() -> FooService::update\_from\_paxos()

# Active

更新完成后，需要执行最开始的回调，然后重新回到active状态，服务需要重载PaxosService::on\_active接口:

# Summary

![img](/assets/img/post/ceph_mon_sequence.png)
