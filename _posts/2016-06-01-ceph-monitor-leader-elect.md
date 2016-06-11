---
layout: post
title: "Ceph Monitor Leader Elect"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

monitor运行过程中，需要选举leader，然后所有更新的操作都是通过leader发出paxos propose完成，
如果非leader收到更新请求，会将请求转发到leader节点，让leader代为执行。

leader本身的选举，并不是paxos算法，ceph本身实现的比较简单高效，因为利用了节点在monmap中的rank值，人为的造成各个节点不平等，
rank值最小的获胜，简单快速的达到选举目的。

# When Start Elect

一个节点发起leader选举是通过函数Monitor::start\_election()完成，这个函数会在以下几种情况被调用:

1. 节点调用bootstrap函数引导启动，接着会probing，查询其他monitor信息(有可能需要同步数据)，完成后发起选举

2. 节点收到选举消息MMonElection，如果节点自己已经处于quorum或自己的编号更小，也会重新发起选举

3. 节点收到quorum enter/exit命令

## bootstrap

对于第一种情况，bootstrap被调用的地方非常频繁:

* monitor节点重新启动

* monitor节点由于信息不全，或者运行过程中很多事件超时，可能需要重启，即调用bootstrap然后probing，同步数据

* leader发出lease消息，等待其他节点回应，如果超时，会重新bootstrap，然后选举

monitor提供的paxos算法内部采用lease协议，保证副本数据在一定时间范围内可读，leader节点会不停的发送lease消息，延长各peon的时间。
如果peon节点down掉，leader节点不会收到lease\_ack消息，超时后就会重新选举。如果leader节点down掉，
所有的peon节点收不到来自leader的更新消息，也会重新选举。

```cpp
void Paxos::lease_ack_timeout()
{
  assert(mon->is_leader()); // leader调用
  assert(is_active());
  lease_ack_timeout_event = 0;
  mon->bootstrap(); // 重启
}

void Paxos::lease_timeout()
{
  assert(mon->is_peon()); // peon调用
  lease_timeout_event = 0;
  mon->bootstrap(); // 重启
}
```

## MMonElection

对于第二种情况，应该说这是第一种情况间接导致的，当某个节点发出选举消息后，其他节点收到消息会做相应的处理。
常见的case是其他节点已经形成一个quorum，并且有leader存在，此时收到选举消息后，发现是来自quorum外的节点，
表明有新节点加入，需要选举:


```cpp
void Elector::handle_propose(MMonElection *m)
{
  ......
  } else if (m->epoch < epoch) {
    // got an "old" propose,
    if (epoch % 2 == 0 &&    // in a non-election cycle
	  mon->quorum.count(from) == 0) {  // from someone outside the quorum
      mon->start_election(); // 本节点已经形成quorum，有节点重新启动
	}
  }

  if (mon->rank < from) {
	mon->start_election(); // 自己编号更小
  } else {
	  ......
  }
  
  m->put();
}
```

## quorum enter/exit

第三种情况，这个其实是一个命令，感觉主要用于运维测试。原理是设置一个bool值，然后调用选举函数，这样就会让此monitor加入或者退出quorum。
针对某个特定的monitor，可以通过如下命令验证:

> $ceph --admin-daemon=path-to-admin-socket quorum enter

> $ceph --admin-daemon=path-to-admin-socket quorum exit

Elector中的处理就是设置标志:

```cpp
void Elector::start_participating()
{
  if (!participating) {
    participating = true; // 参与选举
    call_election();
  }
}

void stop_participating() { participating = false; } // 不参与
```

# How Elect

整个选举的流程，完全是在Elector类中完成。此类中维护了一个election\_epoch，为偶数，表示已经加入quorum且处于稳定状态，为奇数，表示正在选举。
选举的时候，永远只选举rank值为最小的为leader。下表格中的quorum集合只列出了选举成功后的quroum，在选举过程中，会有一个outside quorum表示新加入集群的节点。

以下图演示选举变化过程，第一行中的0,1,2表示三个monitor的rank值:

```text
epoch  |  0  |  1  |  2  | quorum | leader | comment 
------ | --- | --- | --- | ------ | ------ | -------
epoch  |  0  |  0  |  0  |        |        | mon(0,1,2) startup
epoch  |  1  |  1  |  1  |        |        | electing
epoch  |  2  |  2  |  2  | 0,1,2  |   0    |
...
epoch  |  2  |  2  |  2  | 0,1,2  |   0    |
epoch  |  3  |  2  |     |        |        | mon2 down; leader lease_ack timeout -> electing
epoch  |  3  |  3  |     |        |        | electing
epoch  |  4  |  4  |     |  0,1   |   0    |
...
epoch  |  4  |  4  |  2  |  0,1   |   0    | mon2 up
epoch  |  4  |  4  |  3  |  0,1   |   0    | electing
epoch  |  4  |  4  |  4  | 0,1,2  |   0    |
...
epoch  |     |  4  |  5  |        |        | mon0 down; mon2 lease timeout -> electing
epoch  |     |  5  |  5  |        |        | mon1 lease timeout -> electing
epoch  |     |  6  |  6  |  1,2   |   1    | 
...
epoch  |     |  6  |  6  |  1,2   |   1    | 
epoch  |  4  |  6  |  6  |        |        | mon0 up
epoch  |  5  |  6  |  6  |        |        | electing
epoch  |  6  |  6  |  6  | 0,1,2  |   0    |
...
```

下面简单跟踪一下代码流程:

```cpp
void Monitor::start_election()
{
  wait_for_paxos_write();
  _reset();
  state = STATE_ELECTING;
  cancel_probe_timeout();

  elector.call_election(); // 开始选举
}

void call_election() {
  start();
}

void Elector::start()
{
  if (!participating) { // 默认值为true，可以通过命令quorum exit修改
    return;
  }

  acked_me.clear();
  classic_mons.clear();
  init();
  
  if (epoch % 2 == 0) 
    bump_epoch(epoch+1);  // 设置election_epoch为奇数，表示正在选举，这个值会存入store中

  start_stamp = ceph_clock_now(g_ceph_context);
  electing_me = true;
  acked_me[mon->rank] = CEPH_FEATURES_ALL;
  leader_acked = -1;

  for (unsigned i=0; i<mon->monmap->size(); ++i) {
    if ((int)i == mon->rank) continue;
    Message *m = new MMonElection(MMonElection::OP_PROPOSE, epoch, mon->monmap); // 发送消息给其他monitor
    mon->messenger->send_message(m, mon->monmap->get_inst(i));
  }
  
  reset_timer();
}
```

其他节点收到消息后，调用handle\_propose处理，如果接受，就会调用defer，发送回接受消息:

```cpp
void Elector::defer(int who)
{
  ......
  leader_acked = who;
  ack_stamp = ceph_clock_now(g_ceph_context);
  MMonElection *m = new MMonElection(MMonElection::OP_ACK, epoch, mon->monmap); // 发回确认消息
  m->sharing_bl = mon->get_supported_commands_bl();
  mon->messenger->send_message(m, mon->monmap->get_inst(who));
  
  // set a timer
  reset_timer(1.0);  // give the leader some extra time to declare victory
}
```

待收到所有ACK消息后(注意这里并不是收到大多数, leader必须是所有节点都确认)，就会宣告自己胜出:

```cpp
void Elector::handle_ack(MMonElection *m)
{
	......
    if (acked_me.size() == mon->monmap->size()) {
      victory(); // 选举成功
    }
	......
}

void Elector::victory()
{
  ......

  bump_epoch(epoch+1);     // 设置为偶数

  ......

  for (set<int>::iterator p = quorum.begin();
       p != quorum.end();
       ++p) {
    if (*p == mon->rank) continue;
    MMonElection *m = new MMonElection(MMonElection::OP_VICTORY, epoch, mon->monmap); // 告诉其他monitor
    m->quorum = quorum;
    m->quorum_features = features;
    m->sharing_bl = *cmds_bl;
    mon->messenger->send_message(m, mon->monmap->get_inst(*p));
  }
    
  // tell monitor
  mon->win_election(epoch, quorum, features, cmds, cmdsize, &copy_classic_mons); // 胜出，让monitor做相应的初始化
}
```

后面流程就是胜出的monitor一方执行win\_election，调用函数leader\_init初始化leader，失败的一方执行lose\_election，
调用函数peon\_init初始化peon，整个monitor集群差不多就可以开始稳定工作。

# Summary

总结来看，monitor选举的过程是非常简单迅速的，满足条件后向monmap中的所有节点发送消息MMonElection::PROPOSE消息，待收到所有确认消息后就会胜出。

逻辑处理主要是在函数handle\_propose中，举例如下，假设monitor A向monitor B发送PROPOSE消息，考虑两种情况:

> rank A < rank B

* epoch A < epoch B, B会发起选举(并且确认A), A收到B的选举消息，更新epoch，rank A较小，A不会确认B，A用新的epoch再次选举, B会再次收到A的PROPOSE消息，此时epoch相等

* epoch A = epoch B, B会确认A

* epoch A > epoch B, B会更新自己的epoch，确认A

> rank A > rank B

* epoch A < epoch B, B会发起选举，并且不确认A，A收到B的消息，更新epoch，由于rank A较大，A会确认B

* epoch A = epoch B, B会发起选举，并且不确认A, A会确认B

* epoch A > epoch B, B会更新自己的epoch，并且不确认A，然后发起选举，A会确认B
