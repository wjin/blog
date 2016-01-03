---
layout: post
title: "Ceph Dynamic Log/Option Mechanism"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

像ceph这样的大型系统，增加日志功能以及参数配置，并且能够在运行时动态的修改日志级别和参数值，对于分析问题，性能调优都是非常有帮助的。

# Implementation

## Log Class

在ceph中，无论是后端osd daemon进程，还是客户端进程，或者其他命令行工具，启动的时候都会创建一个CephContext这样的对象
(global\_init -> global\_pre\_init -> common\_init)，这个对象的初始化过程中，
就会初始化log相关的信息，包括启动打印log的线程。

```cpp
CephContext::CephContext(uint32_t module_type_)
  : nref(1),
    _conf(new md_config_t()), // 初始化配置对象
    _log(NULL),
    _module_type(module_type_),
    _crypto_inited(false),
    _service_thread(NULL),
    _log_obs(NULL),
    _admin_socket(NULL),
    _perf_counters_collection(NULL),
    _perf_counters_conf_obs(NULL),
    _heartbeat_map(NULL),
    _crypto_none(NULL),
    _crypto_aes(NULL),
    _lockdep_obs(NULL)
{
  ......
  _log = new ceph::log::Log(&_conf->subsys); // 创建日志管理类对象
  _log->start(); // 启动线程
  ......
}
```

log子系统主要的实现在目录src/log，有一个很重要的类Log，并且它继承线程类，说明Log类自带线程处理日志，
因为打印日志，会影响系统的性能，特别是c++的流，对性能影响更明显。ceph这里采取了一些优化：

* ceph 对每个子系统的日志都预先定义了日志级别，并且可动态修改

* 每条log都带有日志级别，低于预先定义的级别才会被打印

* log 信息只需要提交到Log类的线程即可，这边单独的线程接管后台打印日志的任务，对提交log的线程影响较小

Log类的实现比较简单，维护两个队列，m\_new用于提交新日志，flush的时候获取m\_new的entry用来刷新，m\_recent用来存放最近的日志，
比如用户通过admin socket发送dump log的命令时，就会将m\_recent的日志dump到文件：

```cpp

class Log : private Thread // 继承线程类
{
  Log **m_indirect_this;

  SubsystemMap *m_subs; // 每个子系统的日志级别的map
  
  pthread_mutex_t m_queue_mutex; // 这个锁专门用来提交日志
  pthread_mutex_t m_flush_mutex; // 这个锁用来打印提交的日志
  pthread_cond_t m_cond_loggers;
  pthread_cond_t m_cond_flusher;

  pthread_t m_queue_mutex_holder;
  pthread_t m_flush_mutex_holder;

  EntryQueue m_new;    // 提交日志的队列
  EntryQueue m_recent; // 存放最近的日志

  ......

  void *entry(); // 线程入口

  void _flush(EntryQueue *q, EntryQueue *requeue, bool crash);

  void _log_message(const char *s, bool crash);

public:
  Log(SubsystemMap *s);
  virtual ~Log();

  void set_flush_on_exit();

  void set_max_new(int n);
  void set_max_recent(int n);
  void set_log_file(std::string fn);
  void reopen_log_file();

  void flush(); // 刷新

  void dump_recent(); // 打印最近的日志，一般用来响应admin socket的请求

  Entry *create_entry(int level, int subsys); // 创建日志
  void submit_entry(Entry *e); // 提交日志

  void start(); // 启动刷新日志的线程
  void stop(); // 终止线程

  /// true if the log lock is held by our thread
  bool is_inside_log_lock();

  /// induce a segv on the next log event
  void inject_segv();
};
```

## Log Level & Const Option 

这里需要特别注意Log的构造函数，需要提供一张系统的map，注意到这个map是在CephContext的构造函数初始化列表中new出来的（在new Log类之前），
这个map是来干什么的？

```cpp
// 描述单个子系统的日志信息
struct Subsystem {
  int log_level; // 日志级别
  int gather_level; // gather级别，这个一般用的少
  std::string name; // 子系统名称
  
  Subsystem() : log_level(0), gather_level(0) {}     
};

class SubsystemMap {
	// 每个子系统会占据vector一项
	std::vector<Subsystem> m_subsys; // 日志信息
	unsigned m_max_name_len;

	......

	// 添加一个子系统信息
	void SubsystemMap::add(unsigned subsys, std::string name, int log, int gather)
	{
		if (subsys >= m_subsys.size())
			m_subsys.resize(subsys + 1);
		m_subsys[subsys].name = name;
		m_subsys[subsys].log_level = log;
		m_subsys[subsys].gather_level = gather;
		if (name.length() > m_max_name_len)
			m_max_name_len = name.length();
	}

	// 是否需要打印日志
	// sub 就是vector的下标，用来寻址当前是哪个子系统的日志信息
	// level为当前这条日志信息的级别
	// 比如对某个子系统设置的级别是5，如果当前这个子系统的这条日志的级别是3，那么就会被打印
	bool should_gather(unsigned sub, int level) {
		assert(sub < m_subsys.size());
		return level <= m_subsys[sub].gather_level ||
			level <= m_subsys[sub].log_level;
	}
};
```

m\_subsys 怎么是个vector?按理说该是个map\<string, int\>，string为子系统名称，int为日志级别，这样可以根据名字查找一个子系统的日志级别。
这里的关键还是在日志级别的配置文件中，c++这边直接include了配置文件，所以在配置文件中的位置，决定了这里的下标，所以用了vector而不是map。
同时，在include的时候，也定义了一些选项的常量，并在构造函数中进行初始化。

具体的配置文件是src/common/Config\_opts.h, 对它的处理在src/common/Config.h和src/common/Config.cc文件：

```cpp
// src/common/Config_opts.h
// 配置文件里，包含三个部分：OPTION, DEFAULT_SUBSYS, SUBSYS
// include的时候，会根据不同情况选择不同部分，未被选择的项就会为空

......

OPTION(xio_portal_threads, OPT_INT, 2) // (选项名称，选项类型，选项值）

DEFAULT_SUBSYS(0, 5)
SUBSYS(lockdep, 0, 1)
SUBSYS(context, 0, 1) // (子系统名称，log级别，gather级别)
......
```

看看怎么include这个文件进来，src/common/Config.h的例子：

```cpp
struct md_config_t {

	......

	// 这里需要取得option的值，并定义常量
	// 定义特殊类型的OPTION
	#define OPTION_OPT_INT(name) const int name;
	#define OPTION_OPT_LONGLONG(name) const long long name;
	#define OPTION_OPT_STR(name) const std::string name;
	#define OPTION_OPT_DOUBLE(name) const double name;
	#define OPTION_OPT_FLOAT(name) const float name;
	#define OPTION_OPT_BOOL(name) const bool name;
	#define OPTION_OPT_ADDR(name) const entity_addr_t name;
	#define OPTION_OPT_U32(name) const uint32_t name;
	#define OPTION_OPT_U64(name) const uint64_t name;
	#define OPTION_OPT_UUID(name) const uuid_d name;

	// 定义OPTION
	#define OPTION(name, ty, init) OPTION_##ty(name)

	// 定义其他不需要的两项 SUBSYS/DEFAULT_SUBSYS 为空
	#define SUBSYS(name, log, gather)
	#define DEFAULT_SUBSYS(log, gather)

	// 把config文件include进来, 实际上获取了所有的OPTION
	#include "common/config_opts.h"

	// 取消以前所有的宏定义
	#undef OPTION_OPT_INT
	#undef OPTION_OPT_LONGLONG
	#undef OPTION_OPT_STR
	#undef OPTION_OPT_DOUBLE
	#undef OPTION_OPT_FLOAT
	#undef OPTION_OPT_BOOL
	#undef OPTION_OPT_ADDR
	#undef OPTION_OPT_U32
	#undef OPTION_OPT_U64
	#undef OPTION_OPT_UUID
	#undef OPTION
	#undef SUBSYS
	#undef DEFAULT_SUBSYS

	......
};

// 定义一个枚举，实际上就是生成了每个子系统的下标，这里借助了枚举类型的自增
// 所以在文件中的位置决定了子系统在vector中的下标
enum config_subsys_id {
	ceph_subsys_,   // default

	// 因为这里不需要OPTION字段，先定义为空 
	#define OPTION(a,b,c)

	// 生成枚举item的宏
	#define SUBSYS(name, log, gather) \
		ceph_subsys_##name,
	
	// DEFAULT_SUBSYS这里也不需要，定义为空 
	#define DEFAULT_SUBSYS(log, gather)

	// 把config文件include进来, 实际上获取了所有的SUBSYS
	#include "common/config_opts.h"

	// 取消以前所有的宏定义
	#undef SUBSYS
	#undef OPTION
	#undef DEFAULT_SUBSYS

	ceph_subsys_max
};
```

前面是头文件，在看看构造函数的实现src/common/Config.cc：

```cpp
md_config_t::md_config_t()
  : cluster("ceph"),

// 头文件定义了name 常量，这里对常量进行初始化
#define OPTION_OPT_INT(name, def_val) name(def_val),
#define OPTION_OPT_LONGLONG(name, def_val) name((1LL) * def_val),
#define OPTION_OPT_STR(name, def_val) name(def_val),
#define OPTION_OPT_DOUBLE(name, def_val) name(def_val),
#define OPTION_OPT_FLOAT(name, def_val) name(def_val),
#define OPTION_OPT_BOOL(name, def_val) name(def_val),
#define OPTION_OPT_ADDR(name, def_val) name(def_val),
#define OPTION_OPT_U32(name, def_val) name(def_val),
#define OPTION_OPT_U64(name, def_val) name(((uint64_t)1) * def_val),
#define OPTION_OPT_UUID(name, def_val) name(def_val),

#define OPTION(name, type, def_val) OPTION_##type(name, def_val)

// 以下两项不需要
#define SUBSYS(name, log, gather)
#define DEFAULT_SUBSYS(log, gather)

// include文件进来
#include "common/config_opts.h"

// 取消所有的宏
#undef OPTION_OPT_INT
#undef OPTION_OPT_LONGLONG
#undef OPTION_OPT_STR
#undef OPTION_OPT_DOUBLE
#undef OPTION_OPT_FLOAT
#undef OPTION_OPT_BOOL
#undef OPTION_OPT_ADDR
#undef OPTION_OPT_U32
#undef OPTION_OPT_U64
#undef OPTION_OPT_UUID
#undef OPTION
#undef SUBSYS
#undef DEFAULT_SUBSYS

  lock("md_config_t", true, false)
{
  init_subsys();
}

void md_config_t::init_subsys()
{
// 将子系统加入vector，第一个参数就是头文件中定义的宏，就是下标
#define SUBSYS(name, log, gather) \
  subsys.add(ceph_subsys_##name, STRINGIFY(name), log, gather);
#define DEFAULT_SUBSYS(log, gather) \
  subsys.add(ceph_subsys_, "none", log, gather);

// 过滤选项
#define OPTION(a, b, c)

// include 文件
#include "common/config_opts.h"

// 取消所有的宏
#undef OPTION
#undef SUBSYS
#undef DEFAULT_SUBSYS
}
```

## Log Thread

Log创建的时候，记录了整个系统（进程）的日志级别即systemmap，接下来就看看这个线程怎么干活的：

```cpp
// 线程入口函数
void *Log::entry()
{
  pthread_mutex_lock(&m_queue_mutex);
  m_queue_mutex_holder = pthread_self();
  while (!m_stop) {
    if (!m_new.empty()) { // 有新日志
      m_queue_mutex_holder = 0;
      pthread_mutex_unlock(&m_queue_mutex);
      flush(); // 刷新日志
      pthread_mutex_lock(&m_queue_mutex);
      m_queue_mutex_holder = pthread_self();
      continue;
    }

    pthread_cond_wait(&m_cond_flusher, &m_queue_mutex);
  }
  m_queue_mutex_holder = 0;
  pthread_mutex_unlock(&m_queue_mutex);
  flush();
  return NULL;
}

void Log::flush()
{
  pthread_mutex_lock(&m_flush_mutex);
  m_flush_mutex_holder = pthread_self();
  pthread_mutex_lock(&m_queue_mutex);
  m_queue_mutex_holder = pthread_self();
  EntryQueue t; // 临时队列
  t.swap(m_new); // O(1)的交换，这样m_new又可以接收新日志的提交了
  pthread_cond_broadcast(&m_cond_loggers);
  m_queue_mutex_holder = 0;
  pthread_mutex_unlock(&m_queue_mutex); // 提前释放锁，以便其他线程继续提交
  _flush(&t, &m_recent, false); // 真正打印临时队列的信息, 并且会记录到m_recent

  // trim
  while (m_recent.m_len > m_max_recent) { // m_recent有大小限制，超出部分删除
    delete m_recent.dequeue();
  }

  m_flush_mutex_holder = 0;
  pthread_mutex_unlock(&m_flush_mutex);
}

void Log::_flush(EntryQueue *t, EntryQueue *requeue, bool crash)
{
  Entry *e;
  char buf[80];
  while ((e = t->dequeue()) != NULL) {
    unsigned sub = e->m_subsys; // vector下标，判断本条日志属于哪个子系统

    bool should_log = crash || m_subs->get_log_level(sub) >= e->m_prio; // 日志级别小于等于子系统设置的级别才会被打印

    bool do_fd = m_fd >= 0 && should_log;
    bool do_syslog = m_syslog_crash >= e->m_prio && should_log;
    bool do_stderr = m_stderr_crash >= e->m_prio && should_log;

    if (do_fd || do_syslog || do_stderr) {
	  ......

      if (do_fd) {
		int r = safe_write(m_fd, buf, buflen);
		......
      }

      if (do_syslog) {
		syslog(LOG_USER, "%s%s", buf, s.c_str());
      }

      if (do_stderr) {
		cerr << buf << s << std::endl;
      }
    }

    requeue->enqueue(e); // 将日志存放到m_recent队列
  }
}

```

## Submit Log Entry

线程不停的从m\_new队列里获取需要刷新的entry，并且交换到临时队列里去打印，然后立即释放锁继续让队列m\_new接收新的请求，
虽然所有线程都可以提交日志，但是这把锁占用时间还是很短的，尽可能避免对性能的影响。

然后看看其他线程是怎么提交打印日志请求的呢？以librbd里的一个简单例子，打印image 信息：

```cpp
// 每个子系统在文件开头都会有这几个宏
#define dout_subsys ceph_subsys_rbd
#undef dout_prefix
#define dout_prefix *_dout << "librbd: "

......

// 和通常的cout没什么区别，只是多了两个参数，一个是CephContext，另一个就是日志级别
// cct这个参数只是客户端打印日志需要，后端daemon进程这些不需要这个参数，可以直接用另外一个宏dout(20)
// 因为daemon进程有个唯一的全局CephContext，客户端这边可能会打开多个CephContext，所以需要加上

ldout(ictx->cct, 20) << "info " << ictx << dendl; // dendl 必须配合ldout/dout之类的使用

```

宏的主要定义是在文件src/common/Dout.h：

```cpp
#define dout_impl(cct, sub, v)						\
  do {									\
  if (cct->_conf->subsys.should_gather(sub, v)) {			\
    if (0) {								\
      char __array[((v >= -1) && (v <= 200)) ? 0 : -1] __attribute__((unused)); \
    }									\
    ceph::log::Entry *_dout_e = cct->_log->create_entry(v, sub);	\
    ostream _dout_os(&_dout_e->m_streambuf);				\
    CephContext *_dout_cct = cct;					\
    std::ostream* _dout = &_dout_os;

#define ldout(cct, v)  dout_impl(cct, dout_subsys, v) dout_prefix

#define dendl std::flush;				\
  _ASSERT_H->_log->submit_entry(_dout_e);		\
    }						\
  } while (0)

// 这个宏在src/include/Assert.h文件里
#define _ASSERT_H _dout_cct
```

将`ldout(ictx->cct, 20) << "info " << ictx << dendl`展开：

```cpp
do {
	if (cct->_conf->subsys.should_gather(sub, v)) { // 日志级别不满足的话，什么也不做，对性能几乎没影响

		ceph::log::Entry *_dout_e = cct->_log->create_entry(v, sub); // 创建entry

		 // 定义一个输出流对象_dout_os，并用entry里的buffer来初始化，以后输出到流的内容，就输出到entry的buffer了
		ostream _dout_os(&_dout_e->m_streambuf);

		CephContext *_dout_cct = cct;
		std::ostream* _dout = &_dout_os; // 流对象地址赋值给_dout变量，

		// 接下来是 dout_prefix宏的展开
		*_dout << "librbd: "

		// 每次打印的自己的信息
		<< "info " << ictx <<

		// 接下来是 dendl的展开
		std::flush;
		_dout_cct->_log->submit_entry(_dout_e); // 提交entry
	}
} while (0)
```

上面的creat\_entry和submit\_entry就很简单了，都是Log类的简单函数：

```cpp
Entry *Log::create_entry(int level, int subsys)
{
  if (true) {
    return new Entry(ceph_clock_now(NULL), // new一个entry对象
		   pthread_self(),
		   level, subsys);
  } else {
    // kludge for perf testing
    Entry *e = m_recent.dequeue();
    e->m_stamp = ceph_clock_now(NULL);
    e->m_thread = pthread_self();
    e->m_prio = level;
    e->m_subsys = subsys;
    return e;
  }
}

void Log::submit_entry(Entry *e)
{
  pthread_mutex_lock(&m_queue_mutex); // 前面提到的这把锁是用来提交entry的

  // wait for flush to catch up
  while (m_new.m_len > m_max_new)
    pthread_cond_wait(&m_cond_loggers, &m_queue_mutex);

  m_new.enqueue(e); // 进队列
  pthread_cond_signal(&m_cond_flusher);
  pthread_mutex_unlock(&m_queue_mutex);
}
```

另外需要注意一下，entry的实现里有个小技巧，对长度为很小的固定日志，做了一个小优化：

```cpp
struct Entry {
  utime_t m_stamp;
  pthread_t m_thread;
  short m_prio, m_subsys;
  Entry *m_next;

  char m_static_buf[CEPH_LOG_ENTRY_PREALLOC]; // 宏定义为80
  PrebufferedStreambuf m_streambuf;

  Entry()
    : m_thread(0), m_prio(0), m_subsys(0),
      m_next(NULL),
      m_streambuf(m_static_buf, sizeof(m_static_buf)) // 初始化m_streambuf为一个80字节的静态buffer，以后不够用了才会动态扩展为string
  {}

  ......
};
```

## Dynamic Change Config

### Single Daemon

可以修改配置文件，然后重启进程，使得配置生效，虽然ceph本身能够容忍这样的操作，但是ceph也提供了另外一种机制来动态修改参数(包括日志级别和参数配置)。
这是基于unix domain socket实现的，在ceph中叫AdminSocket，这个类的实现先不管，就是监听unix domain socket，然后接受到命令请求的时候解析执行，
比如如下的命令查看osd 0 的配置：

> ceph daemon path\_to\_admin\_socket/osd.0.asok config show

或

> ceph daemon osd.0 config show

这是怎么实现的呢？和Log类一样，AdminSocket也是在CephContext中初始化的：

```cpp
CephContext::CephContext(uint32_t module_type_)
  : nref(1),
    _conf(new md_config_t()),
    _log(NULL),
    _module_type(module_type_),
    _service_thread(NULL),
    _log_obs(NULL),
    _admin_socket(NULL),
    _perf_counters_collection(NULL),
    _perf_counters_conf_obs(NULL),
    _heartbeat_map(NULL),
    _crypto_none(NULL),
    _crypto_aes(NULL)
{
  _log = new ceph::log::Log(&_conf->subsys);
  _log->start();

  _log_obs = new LogObs(_log);
  _conf->add_observer(_log_obs);

  _admin_socket = new AdminSocket(this); // new AdminSocket 对象

  _admin_hook = new CephContextHook(this); // 执行命令的hook
  _admin_socket->register_command("perfcounters_dump", "perfcounters_dump", _admin_hook, ""); // 注册能够识别的命令
  ......
}
```

初始化后就会立即注册自己能够识别的命令及其对应的操作(AdminSocketHook)，然后启动AdminSocket类的线程(AdminSocket也继承自Thread类，和Log类一样)，
开始接收新请求的到来，一旦有请求到来，就会调用AdminSocket::do_accept()，如果有命令匹配(以前注册过命令)，就会调用注册时的hook, 即AdminSocketHook::call()：

```cpp
class CephContextHook : public AdminSocketHook {
  CephContext *m_cct;

public:
  CephContextHook(CephContext *cct) : m_cct(cct) {}

  bool call(std::string command, cmdmap_t& cmdmap, std::string format, // 执行命令
	    bufferlist& out) {
    m_cct->do_command(command, cmdmap, format, &out); // 调用CephContext的do_command
    return true;
  }
};

void CephContext::do_command(std::string command, cmdmap_t& cmdmap,
			     std::string format, bufferlist *out)
{
  ......
  if (command == "perfcounters_dump" || command == "1" ||
      command == "perf dump") {
    _perf_counters_collection->dump_formatted(f, false);
  }
  ......
  else {
    if (command == "config show") {
      _conf->show_config(f); // 打印所有config
    }
    else if (command == "config set") { // 修改参数的命令
	  ......
	  int r = _conf->set_val(var.c_str(), valstr.c_str()); // 调用md_config_t::set_val()改变配置
	  ......
	}
	......
  }
}
```

后面逻辑就比较简单，日志级别可以动态修改，以前定义的类中的常量也可以被修改(没错，就是这样):

```cpp
// 对于选项，set_val会调用下面这个函数
int md_config_t::set_val_impl(const char *val, const config_option *opt)
{
  assert(lock.is_locked());
  int ret = set_val_raw(val, opt); // 改变值
  if (ret)
    return ret;
  changed.insert(opt->name); // 记录下改变的选项，需要通知观察者，这里用了设计模式中的观察者模式
  return 0;
}

int md_config_t::set_val_raw(const char *val, const config_option *opt)
{
  assert(lock.is_locked());
  switch (opt->type) {
    case OPT_INT: {
      std::string err;
      int f = strict_sistrtoll(val, &err); // 转换为整数
      if (!err.empty())
		return -EINVAL;
      *(int*)opt->conf_ptr(this) = f; // 根据常量在对象中的地址，设置地址的值，修改了常量
      return 0;
    }
	......
  }
}

void *config_option::conf_ptr(md_config_t *conf) const
{
  void *v = (void*)(((char*)conf) + md_conf_off); // 获取偏移地址
  return v;
}

// 定义数组，记录每个option的一个三元组（名字，类型，在md_config_t中的地址偏移)
struct config_option config_optionsp[] = {
#define OPTION(name, type, def_val) \
       { STRINGIFY(name), type, offsetof(struct md_config_t, name) }, // 第三个参数记录地址偏移

#define SUBSYS(name, log, gather)
#define DEFAULT_SUBSYS(log, gather)

#include "common/config_opts.h"

#undef OPTION
#undef SUBSYS
#undef DEFAULT_SUBSYS
};

#ifndef offsetof
#define offsetof(STRUCTURE,FIELD) ((int)((char*)&((STRUCTURE*)0)->FIELD)) // 将0转换为结构体指针然后获取偏移，在linux kernel中的链表的实现就见过这么用
#endif
```

对于日志级别，直接修改了以前的systemmap，即vector中的值，在打印日志的时候，会获取到新的值进行判断，对于常量选项，某个子系统如果对某些选项感兴趣，
应注册观察者，当值发生了变化，就会收到通知。

```cpp
void md_config_t::_apply_changes(std::ostream *oss)
{

  ......
  // 通知以前注册的观察者，即daemon进程(osd,mon,mds等)
  for (rev_obs_map_t::const_iterator r = robs.begin(); r != robs.end(); ++r) {
    md_config_obs_t *obs = r->first;
    obs->handle_conf_change(this, r->second);
  }

  changed.clear();
}
```

### Multi Daemon

注意到前面这些方式，都是向某个daemon进程的admin socket发送命令或者修改配置重启某个daemon进程，只针对单个进程，
如果我想修改所有的osd进程的某项配置，可以修改配置，然后分发到所有机器，然后重启所有机器的osd进程；当然也可以用ansible之类的工具，
远程执行脚本，脚本自动找出机器所有daemon进程的admin socket并发送命令。

ceph还提供了另外一种方，可以同时修改所有进程的配置，比如可以通过如下命令对所有osd进程进行修改：

> ceph tell osd.* injectargs '--key value'

这是怎么做到的？
这条命令的执行和以前admin socket的机制截然不同，这条命令不会通过admin socket去直接向某个osd进程发送命令，而是通过librados API接口，
向rados集群的所有osd进程发送命令请求的消息，当osd进程收到消息后，解析出是命令请求的操作，会将操作送入command队列(类似读写操作的op队列)，
然后由command线程池的线程获取队列元素执行，最终线程会调用下面这个函数：

```cpp
void _process(Command *c)
{
	osd->osd_lock.Lock();
	if (osd->is_stopping()) {
		osd->osd_lock.Unlock();
		delete c;
		return;
	}
	osd->do_command(c->con.get(), c->tid, c->cmd, c->indata); // 执行命令请求
	osd->osd_lock.Unlock();
	delete c;
}

void OSD::do_command(Connection *con, ceph_tid_t tid, vector<string>& cmd, bufferlist& data)
{
  ......
  else if (prefix == "injectargs") { // injectargs 的命令请求
    vector<string> argsvec;
    cmd_getval(cct, cmdmap, "injected_args", argsvec); // 解析参数到argsvec

    if (argsvec.empty()) {
      r = -EINVAL;
      ss << "ignoring empty injectargs";
      goto out;
    }
    string args = argsvec.front();
    for (vector<string>::iterator a = ++argsvec.begin(); a != argsvec.end(); ++a)
      args += " " + *a;
    osd_lock.Unlock();
    cct->_conf->injectargs(args, &ss); // 修改参数，最终还是通过md_config_t类完成
    osd_lock.Lock();
  }
  ......
}

int md_config_t::injectargs(const std::string& s, std::ostream *oss)
{
  ......
  ret = parse_injectargs(nargs, oss); // 设置参数值

  ......
  _apply_changes(oss); // 和以前一样，回调通知自己，告诉参数有变

  return ret;
}
```

另外需要注意下，命令ceph tell ...，最开始是c++实现的，现在改为python实现的，在src/ceph.in文件，最终安装后默认会放到/usr/bin/ceph，
然后通过python库以及提供的集群配置文件(默认/etc/ceph/ceph.conf)，就会调用c++ librados接口，向集群发送消息。
所以其实我们并不需要登录到rados集群的机器，在管理机就可以执行，只要网络联通，安装了ceph以及设置好了配置文件，这对我们进行性能分析和参数调优非常方便。


# Summary

* ceph对每个子系统定义了日志级别，启动的时候从配置文件构造了一张包含日志级别的map，并可动态修改

* 采用单独线程处理所有提交的日志

* 日志级别不满足的时候，几乎无任何额外开销（do{}while(0))

* 快速提交日志，锁很短的时间

* 日志流的优化，预先定义静态buffer

* 动态修改系统参数配置，通过观察者模式通知相关者

* 在管理机动态调整集群所有osd参数，方便进行性能调优
