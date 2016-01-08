---
layout: post
title: "Ceph Async Messenger"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

ceph源码中有两种网络通信的实现方式，SimpleMessenger实现比较早，每一对通信的peer之间创建四个线程维护连接状态(每一端两个线程，分别负责读和写),
这样当集群规模上去后，会导致大量的线程被创建。随着linux中epoll的实现，高并发的网络io都是借助于epoll这样的系统调用，
比如libevent库。ceph源码中也基于epoll实现了AsyncMessenger，这有助于减少集群中网络通信所需要的线程数，
目前实现虽然还不太稳定，并不是默认的通信组件，但是未来一定会取代SimpleMessenger。

# Server

服务端需要监听端口，等待连接请求到来，然后接受请求建立连接进行通信。

## Initialization

以osd进程为例，在进程启动的过程中，会创建Messenger对象，用于管理网络连接，监听端口，接收请求，源码在文件src/ceph_osd.cc:


```cpp
int main(int argc, const char **argv) 
{
  ......

  // public用于客户端通信
  Messenger *ms_public = Messenger::create(g_ceph_context, g_conf->ms_type,
					   entity_name_t::OSD(whoami), "client",
					   getpid());

  // cluster用于集群内部通信
  Messenger *ms_cluster = Messenger::create(g_ceph_context, g_conf->ms_type,
					    entity_name_t::OSD(whoami), "cluster",
					    getpid());
  ......
}

// src/msg/Messenger.cc
Messenger *Messenger::create(CephContext *cct, const string &type,
			     entity_name_t name, string lname,
			     uint64_t nonce)
{
  ......

  // 在src/common/config_opts.h文件中，目前需要配置async相关选项才会生效
  // OPTION(enable_experimental_unrecoverable_data_corrupting_features, OPT_STR, "ms-type-async")
  // OPTION(ms_type, OPT_STR, "async")
  else if ((r == 1 || type == "async") &&
	   cct->check_experimental_feature_enabled("ms-type-async"))
    return new AsyncMessenger(cct, name, lname, nonce);

  ......
  return NULL;
}
```

类AsyncMessenger的构造函数需要注意，虽然在osd进程的启动过程中，会创建6个messenger，但是他们全部共享一个WorkerPool，
函数lookup\_or\_create\_singleton\_object<WorkerPool>保证只会创建一个pool，因为传入的名称WokerPool::name是一样的：

```cpp
AsyncMessenger::AsyncMessenger(CephContext *cct, entity_name_t name,
                               string mname, uint64_t _nonce)
  : SimplePolicyMessenger(cct, name,mname, _nonce),
    processor(this, cct, _nonce),
    lock("AsyncMessenger::lock"),
    nonce(_nonce), need_addr(true), did_bind(false),
    global_seq(0), deleted_lock("AsyncMessenger::deleted_lock"),
    cluster_protocol(0), stopped(true)
{
  ceph_spin_init(&global_seq_lock);
  cct->lookup_or_create_singleton_object<WorkerPool>(pool, WorkerPool::name); // 创建pool对象, 注意第二个参数是WorkerPool中的静态常量
  // 创建一个本地连接对象用于向自己发送消息
  local_connection = new AsyncConnection(cct, this, &pool->get_worker()->center);
  init_local_connection(); // 初始化本地对象
}

template<typename T>
void lookup_or_create_singleton_object(T*& p, const std::string &name) {
  ceph_spin_lock(&_associated_objs_lock);
  if (!_associated_objs.count(name)) { // name决定了一个进程只会有一个pool
    p = new T(this); // new一个对象，这里是WorkerPool
    _associated_objs[name] = reinterpret_cast<AssociatedSingletonObject*>(p); // 加入map
  } else {
    p = reinterpret_cast<T*>(_associated_objs[name]);
  }
  ceph_spin_unlock(&_associated_objs_lock);
}
```

另外需要注意，这个进程唯一的pool是在messenger的构造函数分配的，messenger的析构函数并不负责释放内存，因为多个messenger共享，
一个messenger销毁了并不代表其他messenger也一定会销毁，这个pool的指针存放在CephContext成员变量\_associated\_objs中，
因为daemon进程有一个全局唯一的CephContext，当CephContext析构的时候，会释放pool指针的内存。

一个osd进程只会有一个WorkerPool，那这个pool在初始化的时候干什么事情了？顾名思义，Worker的Pool，肯定是用来管理Worker的，
构造函数中恰恰就是新建了Worker类的对象，而Worker类继承于线程类，肯定就是单独干活的线程，源码在文件
src/msg/async/AsyncMessenger.[h|c]中:

```cpp
WorkerPool::WorkerPool(CephContext *c): cct(c), seq(0), started(false),
                                        barrier_lock("WorkerPool::WorkerPool::barrier_lock"),
                                        barrier_count(0)
{
  assert(cct->_conf->ms_async_op_threads > 0);

  for (int i = 0; i < cct->_conf->ms_async_op_threads; ++i) {
    Worker *w = new Worker(cct, this, i); // 新建Worker类对象
    workers.push_back(w); // 保存在vector容器中, 用于跟踪所有的worker
  }

  ......
}

class Worker : public Thread { // 继承线程类，说明Worker类单独包含线程
  static const uint64_t InitEventNumber = 5000; // 事件个数
  static const uint64_t EventMaxWaitUs = 30000000; // 事件最大的等待时间, 30秒
  CephContext *cct;
  WorkerPool *pool;
  bool done;
  int id;

 public:
  EventCenter center; // 事件中心
  Worker(CephContext *c, WorkerPool *p, int i)
    : cct(c), pool(p), done(false), id(i), center(c) {
    center.init(InitEventNumber); // 初始化事件驱动, 实际上就是初始化了epoll相关的结构
  }
  void *entry();
  void stop();
};
```

为了代码通用，这里单独抽象了一层出来，即EventCenter，用来管理各种事件的驱动，比如epoll, kqueue, select等。
源码在src/msg/async/Event.[h]c]:

```cpp
class EventCenter {
  ......

  FileEvent *file_events; // 所有io事件
  EventDriver *driver; // 具体的驱动
  map<utime_t, list<TimeEvent> > time_events; // 所有时间事件
  ......
};

// EventDriver接口
// epoll的驱动继承此接口，接口的实现就是对epoll三个系统调用epoll_create, epoll_ctl，epoll_wait的封装
class EventDriver {
 public:
  virtual ~EventDriver() {}       // we want a virtual destructor!!!
  virtual int init(int nevent) = 0;
  virtual int add_event(int fd, int cur_mask, int mask) = 0;
  virtual void del_event(int fd, int cur_mask, int del_mask) = 0;
  virtual int event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tp) = 0;
  virtual int resize_events(int newsize) = 0;
};

class EpollDriver : public EventDriver {
  int epfd; // epoll fd
  struct epoll_event *events; // 等待事件的结构体指针，可以查看epoll相关资料
  CephContext *cct;
  int size;
  ......
};
```

Worker构造函数中，调用了center的init函数，看看center.init干了些什么事情？

```cpp
int EventCenter::init(int n)
{
  ......
  driver = new EpollDriver(cct); // 新建一个驱动对象

  int r = driver->init(n); // 初始化具体的驱动

  int fds[2]; // pipe用来唤醒worker线程，后文会分析到
  if (pipe(fds) < 0) {
    lderr(cct) << __func__ << " can't create notify pipe" << dendl;
    return -1;
  }

  notify_receive_fd = fds[0];
  notify_send_fd = fds[1];

  ......

  create_file_event(notify_receive_fd, EVENT_READABLE, EventCallbackRef(new C_handle_notify())); // 监听pipe的可读事件
  return 0;
}

// 初始化epoll
int EpollDriver::init(int nevent)
{
  events = (struct epoll_event*)malloc(sizeof(struct epoll_event)*nevent); // nevent就是Worker类中的InitEventNumber
  memset(events, 0, sizeof(struct epoll_event)*nevent);
  epfd = epoll_create(1024); // 获取一个epoll fd
  size = nevent;
  return 0;
}
```

从osd进程，到AsyncMessenger类，接着到所有messenger共享的WorkerPool，然后初始化进程唯一pool的每个Worker，然后worker中借助于EventCenter统一管理所有事件，
并且初始化了具体的事件处理机制，如epoll，似乎所有工作已经就绪？
其实不然，首先，worker的线程并没有启动，其次，osd进程的messenger也并没有绑定到特定端口进行监听，所以osd启动的过程中，还得有其他步骤。

# Bind and Listen

在messenger创建以后，会设置策略以及限流的参数，接下来就会绑定地址，对网络层套接字的处理，比如socket/bind/listen/accept等，主要是通过类Processor来管理：

```cpp
// 继续ceph_osd.cc代码
int main(int argc, const char **argv) 
{
  ......
  // 设置协议
  ms_cluster->set_cluster_protocol(CEPH_OSD_PROTOCOL);
  ......

  // 设置策略以及限流
  ms_public->set_default_policy(Messenger::Policy::stateless_server(supported, 0));
  ms_public->set_policy_throttlers(entity_name_t::TYPE_CLIENT,
				   client_byte_throttler.get(),
				   client_msg_throttler.get());

  ......

  // 绑定地址
  r = ms_public->bind(g_conf->public_addr);
  if (r < 0)
    exit(1);
  r = ms_cluster->bind(g_conf->cluster_addr);
  if (r < 0)
    exit(1);

  ......
  ms_public->start(); // 启动线程

  ......
  err = osd->init(); // 这里很关键, 后文分析

  ......
  ms_public->wait(); // 等待线程结束
  ......
}

int AsyncMessenger::bind(const entity_addr_t &bind_addr)
{
  ......

  // bind to a socket
  set<int> avoid_ports;
  int r = processor.bind(bind_addr, avoid_ports); // 调用processor对象进行处理

  ......
}

// processor的处理就是对socket API的封装：socket, bind, listen
// 创建套接字，绑定到特定端口，进行监听
int Processor::bind(const entity_addr_t &bind_addr, const set<int>& avoid_ports)
{
  ......
  listen_sd = ::socket(family, SOCK_STREAM, 0);

  ......
  rc = ::bind(listen_sd, (struct sockaddr *) &listen_addr.ss_addr(), listen_addr.addr_size());

  ......
  rc = ::listen(listen_sd, 128);

  ......
  msgr->init_local_connection(); // 更新地址，但是因为还没有dispatch对象，不会处理连接

  return 0;
}

void init_local_connection() {
  Mutex::Locker l(lock);
  _init_local_connection();
}

void _init_local_connection() {
  assert(lock.is_locked());
  local_connection->peer_addr = my_inst.addr;
  local_connection->peer_type = my_inst.name.type();
  ms_deliver_handle_fast_connect(local_connection.get());
}

void ms_deliver_handle_fast_connect(Connection *con) {
  for (list<Dispatcher*>::iterator p = fast_dispatchers.begin(); // fast_dispatchers 目前为空
       p != fast_dispatchers.end();
       ++p)
    (*p)->ms_handle_fast_connect(con);
}
```

## Deal with Event

在绑定地址进行端口监听以后，就会等着连接到来，要处理连接请求，肯定得创建Worker线程来处理吧？

```cpp
// ceph_osd.cc 会继续调用messenger->start(), 参见前面代码
int AsyncMessenger::start()
{
  lock.Lock();
  ......
  pool->start(); // 启动所有线程

  lock.Unlock();
  return 0;
}

void WorkerPool::start()
{
  if (!started) {
    for (uint64_t i = 0; i < workers.size(); ++i) {
      workers[i]->create(); // 创建线程
    }
    started = true;
  }
}

// 线程入口函数
void *Worker::entry()
{
  ......

  center.set_owner(pthread_self());
  while (!done) { // 线程一直循环处理事件
    int r = center.process_events(EventMaxWaitUs); // 借助于事件中心处理事件, 注意最大的等待时间是30秒
  }

  return 0;
}

// 通过epoll_wait返回所有就绪的fd，然后一次调用其callback
int EventCenter::process_events(int timeout_microseconds)
{
  ......

  vector<FiredFileEvent> fired_events;
  next_time = shortest;
  numevents = driver->event_wait(fired_events, &tv); // 获取当前的io事件
  for (int j = 0; j < numevents; j++) {
    int rfired = 0;
    FileEvent *event;
    {
      Mutex::Locker l(file_lock);
      event = _get_file_event(fired_events[j].fd);
    }

    if (event->mask & fired_events[j].mask & EVENT_READABLE) {
      rfired = 1;
      event->read_cb->do_request(fired_events[j].fd); // 处理可读事件
    }

    if (event->mask & fired_events[j].mask & EVENT_WRITABLE) {
      if (!rfired || event->read_cb != event->write_cb)
        event->write_cb->do_request(fired_events[j].fd); // 处理可写事件
    }
  }
  ......
}

int EpollDriver::event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tvp)
{
  int retval, numevents = 0;

  retval = epoll_wait(epfd, events, size,
                      tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1); // epoll_wait系统调用，等待就绪事件或超时返回
    for (j = 0; j < numevents; j++) {
      int mask = 0;
      struct epoll_event *e = events + j;

      if (e->events & EPOLLIN) mask |= EVENT_READABLE;
      if (e->events & EPOLLOUT) mask |= EVENT_WRITABLE;
      if (e->events & EPOLLERR) mask |= EVENT_WRITABLE;
      if (e->events & EPOLLHUP) mask |= EVENT_WRITABLE;
	  // 记录下已经发生的事件
      fired_events[j].fd = e->data.fd;
      fired_events[j].mask = mask;
    }
	
  return numevents;
}
```

## Add Listen Fd

Worker线程循环不停的处理事件，其实就是调用epoll\_wait，返回就绪事件的fd，然后调用fd对应的回调read\_cb或write\_cb，很明显，epoll\_wait能够返回就绪的fd，
这个fd必然是之前添加进去的，什么时候添加的呢？还记得在第二步Bind的时候，Processor类中创建了listen\_fd，要想监听来自这个fd的请求，必然要将其添加到epoll进行管理。

但是从osd代码运行到这里，似乎都没有添加的动作？在osd调用messenger->start()后，紧接着就是：

> err = osd->init();

诀窍就在这里：

```cpp
int OSD::init()
{
  ......
  // i'm ready!
  client_messenger->add_dispatcher_head(this);
  cluster_messenger->add_dispatcher_head(this);
  ......
}

void add_dispatcher_head(Dispatcher *d) {
  bool first = dispatchers.empty(); // 刚开始当然为空, first为true
  dispatchers.push_front(d);
  if (d->ms_can_fast_dispatch_any())
    fast_dispatchers.push_front(d);
  if (first) 
    ready(); // 准备添加fd到epoll
}

void AsyncMessenger::ready()
{
  ldout(cct,10) << __func__ << " " << get_myaddr() << dendl;

  Mutex::Locker l(lock);
  Worker *w = pool->get_worker(); // 获取一个worker干活
  processor.start(w); // listen_sd在Processor中
}

int Processor::start(Worker *w)
{
  ldout(msgr->cct, 1) << __func__ << " " << dendl;

  // start thread
  if (listen_sd > 0) {
    worker = w;
	// 创建可读事件, 最终会调用epoll_ctl将listen_sd加进epoll进行管理
    w->center.create_file_event(listen_sd, EVENT_READABLE,
                                EventCallbackRef(new C_processor_accept(this))); // 注意事件的callback
  }

  return 0;
}
```

## Accept Connection

listen fd添加进去以后，初始化过程就算全部完成了。当新的连接请求到来，如前所述，worker线程会调用process\_event函数，回调就会被执行：

```cpp
// listen fd 的回调
class C_processor_accept : public EventCallback {
  Processor *pro;

 public:
  C_processor_accept(Processor *p): pro(p) {}
  void do_request(int id) {
    pro->accept(); // 回调
  }
};

void Processor::accept()
{
  while (errors < 4) {
    entity_addr_t addr;
    socklen_t slen = sizeof(addr.ss_addr());
    int sd = ::accept(listen_sd, (sockaddr*)&addr.ss_addr(), &slen); // 接受连接请求
    if (sd >= 0) {
      msgr->add_accept(sd); // 通过messenger处理接收套接字sd
      continue;
    } else {
	  ......
    }
  }
}

AsyncConnectionRef AsyncMessenger::add_accept(int sd)
{
  lock.Lock();
  Worker *w = pool->get_worker();
  AsyncConnectionRef conn = new AsyncConnection(cct, this, &w->center); // 创建连接
  w->center.dispatch_event_external(EventCallbackRef(new C_conn_accept(conn, sd))); // 分发事件, 外部新的连接，所以叫external
  accepting_conns.insert(conn); // 记录下即将生效的连接, 最终完成后会从此集合删除
  lock.Unlock();
  return conn;
}

void EventCenter::dispatch_event_external(EventCallbackRef e)
{
  external_lock.Lock();
  external_events.push_back(e); // 将事件的callback函数放入事件中心的队列中等待执行
  external_lock.Unlock();
  wakeup(); // 唤醒worker线程
}
```

不是很明白为什么需要放入队列，等待worker下一次的process\_event调用，是否可以直接执行完毕？

不管怎么样，放入队列后，需要执行队列中的callback，什么时候会执行呢？很明显是在worker线程中的process\_event函数，
但是worker线程可能睡眠在epoll\_wait(epoll管理的所有fd都没就绪，只能等待超时)，如果有新连接到来，需要立即接收连接请求，
所以要唤醒睡眠的worker线程，后面的wakeup函数就是达到此目的，这个函数向pipe的一端写入数据(pipe是在函数EventCenter::init()中创建的），
使得另一端可读，即notify\_receive\_fd就绪，epoll_wait会返回其可读事件，然后执行其回调(回调就是简单读pipe)，使得worker线程得以继续处理，
然后执行刚才放入队列中的回调。


```cpp
void EventCenter::wakeup()
{
  ldout(cct, 1) << __func__ << dendl;
  char buf[1];
  buf[0] = 'c';
  // wake up "event_wait"
  int n = write(notify_send_fd, buf, 1); // 唤醒worker线程
  // FIXME ?
  assert(n == 1);
}

int EventCenter::process_events(int timeout_microseconds)
{
  ......

  numevents = driver->event_wait(fired_events, &tv); // 本来worker线程可能睡眠在这里，会被wakeup唤醒

  // 这时候至少有一个fd就绪，即notify_receive_fd
  // 执行所有fd的callback, 对于notify_receive_fd，可以看其callback，就是简单读一下，什么也没干
  for (int j = 0; j < numevents; j++) {
	......
    event->read_cb->do_request(fired_events[j].fd);
	.....
  }

  ......

  // 紧接着处理刚才的队列, 这正是唤醒worker的目的
  {
    external_lock.Lock();
    while (!external_events.empty()) {
      EventCallbackRef e = external_events.front();
      external_events.pop_front();
      external_lock.Unlock();
      if (e)
        e->do_request(0); // 连接请求的callback
      external_lock.Lock();
    }
    external_lock.Unlock();
  }

  ......
}
```

## Add Accept Fd

从分析看，连接请求的callback会很快被执行。前面已经有了accept接收请求的fd，现在需要将那个fd加入epoll结构，管理起来，然后就可以进行通信，
callback最终就是做这些事情：

```cpp
// 队列中的回调类型
class C_conn_accept : public EventCallback {
  AsyncConnectionRef conn;
  int fd;

 public:
  C_conn_accept(AsyncConnectionRef c, int s): conn(c), fd(s) {}
  void do_request(int id) {
    conn->accept(fd);
  }
};

void AsyncConnection::accept(int incoming)
{
  ldout(async_msgr->cct, 10) << __func__ << " sd=" << incoming << dendl;
  assert(sd < 0);

  sd = incoming;
  state = STATE_ACCEPTING;
  center->create_file_event(sd, EVENT_READABLE, read_handler); // sd就是连接成功的fd，加进epoll管理
  process(); // 服务器端的状态机开始执行，会先向客户端发送BANNER消息
}
```

## Communication

注意服务端AsyncConnection状态机的初始状态是STATE\_ACCEPTING，服务器端的状态机会先向客户端发送BANNER消息。
以后收到消息，worker线程就会调用read\_handler处理，然后调用process，状态机不停的转换状态:

```cpp
// 注册的回调类
class C_handle_read : public EventCallback {
  AsyncConnectionRef conn;

 public:
  C_handle_read(AsyncConnectionRef c): conn(c) {}
  void do_request(int fd_or_id) {
    conn->process(); // 调用connection处理
  }
};

void AsyncConnection::process()
{
  int r = 0;
  int prev_state = state;
  Mutex::Locker l(lock);
  do {
    prev_state = state;

	// connection状态机
    switch (state) {
      case STATE_OPEN:
	  ......

	  default:
        {
          if (_process_connection() < 0)
            goto fail;
          break;
        }
	}
  }

  return 0;

fail:
  ......
}

// 单独处理连接信息
int AsyncConnection::_process_connection()
{
  int r = 0;

  switch(state) {
    case STATE_WAIT_SEND:
		......
  }
  ......
}
```

AsyncConnection就是负责通信的类，要理解这个状态机的原理，必须理解ceph的应用层通信协议，
可以参看[官方文档](http://docs.ceph.com/docs/master/dev/network-protocol/)的解释。

AsyncMessenger的框架就算介绍完成了，当有新的连接请求到来，就会重复执行以下这几步：

* accept connection

* add accept fd

* communication

由此可以看出，线程数不是随连接数线性增加的，只由最开始初始化的时候启动了多少个worker决定。

# Client

客户端的操作主要是发起connect操作，建立连接进行通信。所有的客户端都是基于librados库，然后通过RadosClient连接集群的:

```cpp
int librados::Rados::connect()
{
    return client->connect();
}

int librados::RadosClient::connect()
{
	......

	// 创建messenger
	messenger = Messenger::create(cct, cct->_conf->ms_type, entity_name_t::CLIENT(-1),
			"radosclient", nonce);

	......

	// 创建objecter
	// 发送消息的时候，比如librbd代码，都是通过objecter处理
	// objecter需要借助于messenger发送，所以需要将创建的messenger传给objecter类
	objecter = new (std::nothrow) Objecter(cct, messenger, &monclient,
				&finisher,
				cct->_conf->rados_mon_op_timeout,
				cct->_conf->rados_osd_op_timeout);


	// 同理，连接monitor也需要处理消息的收发
	monclient.set_messenger(messenger);

	objecter->init();
	messenger->add_dispatcher_tail(objecter);
	messenger->add_dispatcher_tail(this);

	messenger->start();

	......

	messenger->set_myname(entity_name_t::CLIENT(monclient.get_global_id())); // ID全局唯一，所以需要向monitor获取

	......
}
```

connect操作只是初始化了messenger对象，真正需要通信的时候，才会去建立连接，以objecter.cc中的op\_submit为例：

```cpp
ceph_tid_t Objecter::_op_submit(Op *op, RWLock::Context& lc)
{
	......
	int r = _get_session(op->target.osd, &s, lc);
	......
}

int Objecter::_get_session(int osd, OSDSession **session, RWLock::Context& lc)
{
    ......

    // session 不存在，会创建新的session，
    s->con = messenger->get_connection(osdmap->get_inst(osd));
    ......
}

ConnectionRef AsyncMessenger::get_connection(const entity_inst_t& dest)
{
	......
    conn = create_connect(dest.addr, dest.name.type());
	......
}

AsyncConnectionRef AsyncMessenger::create_connect(const entity_addr_t& addr, int type)
{
  // create connection
  Worker *w = pool->get_worker();
  AsyncConnectionRef conn = new AsyncConnection(cct, this, &w->center); // 创建connection
  conn->connect(addr, type); // 连接
  assert(!conns.count(addr));
  conns[addr] = conn;

  return conn;
}

void connect(const entity_addr_t& addr, int type)
{
    set_peer_type(type);
    set_peer_addr(addr);
    policy = msgr->get_policy(type);
    _connect();
}

void AsyncConnection::_connect()
{
  state = STATE_CONNECTING; // 这个初始化状态很关键，是客户端状态机的起始状态
  stopping.set(0);
  center->dispatch_event_external(read_handler); // 放入队列等待worker处理
}
```

这里和前面一样，worker会处理这个外部事件，read\_handler就会调用process函数，紧接着就过度到\_process\_connection:

```cpp
int AsyncConnection::_process_connection()
{
  int r = 0;

  switch(state) {

    case STATE_CONNECTING: // 初始状态
      {
		......

        sd = net.connect(get_peer_addr()); // 通过net类的功能，实际上就是调用connect系统调用，建立socket通信

		// 连接成功后，将socket fd加入epoll进行管理
        center->create_file_event(sd, EVENT_READABLE, read_handler);
        state = STATE_CONNECTING_WAIT_BANNER;
        break;
      }
  }
}
```

接下来就是客户端和服务端的通信，都是通过AsyncConnection的状态机完成。同理，客户端即使创建多个messenger，
他们仍然共享一个workerpool，线程数由这个pool初始化的时候决定，不会随着连接的增加而线性增加。

# Summary

* 进程中所有的AsyncMessenger共享一个workerpool管理所有worker

* Worker线程通过EventCenter负责具体的事件处理

* 应用层的网络通信由AsyncConnection的状态机处理
