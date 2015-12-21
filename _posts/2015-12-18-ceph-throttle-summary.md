---
layout: post
title: "Ceph Throttle Summary"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

ceph的io栈比较长，就像流水线生产一样，io操作会经过很多队列，由特定线程或线程池取出进行操作，最终将对象数据存放在磁盘上。
中间很多关键步骤的参数，比如队列深度，线程池个数等都可以通过设置参数控制，ceph也提供一种机制，对每个组件进行限流，
防止所有操作拥塞在流水线的一个地方加剧竞争。

ceph源码中，有四种基本的限流实现，下面一一分析。

# SimpleThrottle

这是最简单的一种，设置一个max值以及当前计数，需要资源时，计数+1，如果操作达到上限max，就阻塞等待，使用完后返回资源，计数-1。
具体实现源码在文件src/common/Throttle.h 和 src/common/Throttle.c，参见类SimpleThrottle:

```cpp
// 类头文件
class SimpleThrottle {
public:
  SimpleThrottle(uint64_t max, bool ignore_enoent);
  ~SimpleThrottle();
  void start_op(); // 使用计数+1，超过最大限制会阻塞
  void end_op(int r); // 使用计数-1
  bool pending_error() const;
  int wait_for_ret();
private:
  mutable Mutex m_lock;
  Cond m_cond;
  uint64_t m_max; // 并发最大限制数
  uint64_t m_current; // 当前的并发数
  int m_ret;
  bool m_ignore_enoent;
};

// 简单的callback类，自动加减计数
class C_SimpleThrottle : public Context {
public:
  C_SimpleThrottle(SimpleThrottle *throttle) : m_throttle(throttle) {
    m_throttle->start_op();
  }
  virtual void finish(int r) {
    m_throttle->end_op(r);
  }
private:
  SimpleThrottle *m_throttle;
};

// 成员函数实现
// 获取资源
void SimpleThrottle::start_op()
{
  Mutex::Locker l(m_lock);
  while (m_max == m_current) // 阻塞等待
    m_cond.Wait(m_lock);
  ++m_current; // 增加计数
}

// 释放资源
void SimpleThrottle::end_op(int r)
{
  Mutex::Locker l(m_lock);
  --m_current; // 减少计数
  if (r < 0 && !m_ret && !(r == -ENOENT && m_ignore_enoent))
    m_ret = r;
  m_cond.Signal();
}
```

这个简单的限流主要用在librbd中，比如对镜像回滚、导出、导入等操作。

# Throttle

不同于SimpleThrottle，Throttle这个类实现的限流一次可以请求多个资源，而不是每次只能请求一个资源，
并且在资源不足的情况下，按照fifo顺序对请求进行排序，具体实现源码也是在文件src/common/Throttle.h 和 src/common/Throttle.c，
参见类Throttle:

```cpp
class Throttle {
  ......
  ceph::atomic_t count, max; // 当前并发数和最大值
  Mutex lock;
  list<Cond*> cond; // FIFO 等待队列
  
public:
  Throttle(CephContext *cct, std::string n, int64_t m = 0, bool _use_perf = true);
  ~Throttle();

private:
  void _reset_max(int64_t m);
  bool _should_wait(int64_t c) { // 是否超过并发的限制
    int64_t m = max.read();
    int64_t cur = count.read();
    return
      m &&
      ((c <= m && cur + c > m) || // normally stay under max
       (c >= m && cur > m));     // except for large c
  }

  // 核心实现函数，如果需要阻塞，会新建一个条件变量，然后放入链表中，等待自己的条件变量被唤醒
  // 如果多次调用，会形成一个阻塞链表，按照FIFO顺序被唤醒
  bool _wait(int64_t c);

public:

  bool wait(int64_t m = 0); // 将并发参数max设置为m，必要时会等待之前已经在等待的操作完成，阻塞

  ......

  // 将并发参数max设置为m，必要时会等待之前已经在等待的操作完成
  // 等待可以进行c个并发，阻塞
  // 接着将当前并发数增加为c
  bool get(int64_t c = 1, int64_t m = 0); // 获取资源，增加计数 ,阻塞

  int64_t put(int64_t c = 1); // 释放资源，减少计数，非阻塞
};
```

核心函数就这一个内部函数\_wait，以及另外两个常见的API get/put：

```cpp
bool Throttle::_wait(int64_t c)
{
  utime_t start;
  bool waited = false;
  if (_should_wait(c) || !cond.empty()) { // 需要等待
    Cond *cv = new Cond; // 新建自己的条件变量
    cond.push_back(cv); // 插入fifo队列
    do {
      waited = true;
      cv->Wait(lock); // 睡眠
    } while (_should_wait(c) || cv != cond.front()); // 唤醒后，需要继续检查并发数是否够用，所以用do while循环

    delete cv;
    cond.pop_front();

    // 唤醒后面新加入的waiters
	// 执行到这里，说明计数已经满足，可以执行，继续唤醒后面的waiters，而不是等待自己释放了计数才去唤醒
	// 这是因为前面一次可能释放了很多计数，后面几个新加的waiters都可以得到满足
    if (!cond.empty())
      cond.front()->SignalOne();
  }
  return waited;
}

bool Throttle::get(int64_t c, int64_t m)
{
  if (0 == max.read()) {
    return false;
  }

  bool waited = false;
  {
    Mutex::Locker l(lock);
    if (m) { // 重置并发数max
      assert(m > 0);
      _reset_max(m);
    }
    waited = _wait(c); // 获取资源
    count.add(c); // 增加计数
  }

  return waited;
}

int64_t Throttle::put(int64_t c)
{
  if (0 == max.read()) {
    return 0;
  }

  Mutex::Locker l(lock);
  if (c) {
    if (!cond.empty()) // 唤醒下一个
      cond.front()->SignalOne();

    count.sub(c); // 减少计数
  }
  return count.read();
}
```

这个throttle实现比较通用，除了后面讲的特定场所的throttle，其他都是使用类Throttle来达到限流目的。

# WBThrottle

WB代表write back，这个限流是专为FileStore设计的，主要是防止FileStore写太快，后端存储设备速度跟不上。对于ssd块设备，可以disable这个功能。
WBThrottle类自带线程，限流从三个粒度（io个数，字节数，对象个数）一起控制:

```cpp
class WBThrottle : Thread, public md_config_obs_t { // 继承线程类
  ghobject_t clearing;

  // <soft, hard>
  // 当三个粒度的其中一个超过soft值，就开始回刷fd
  // 当三个粒度的其中一个超过hard值，throttle就开始起作用，_do_op会阻塞
  pair<uint64_t, uint64_t> size_limits; // 未刷新字节数
  pair<uint64_t, uint64_t> io_limits; // 未刷新io个数
  pair<uint64_t, uint64_t> fd_limits; // 未刷新fd个数，也即对象个数

  uint64_t cur_ios; // 当前未刷新io个数
  uint64_t cur_size; // 当前未刷新字节数

  // 跟踪对象的一些信息
  class PendingWB {
  public:
    bool nocache;
    uint64_t size;
    uint64_t ios;
    PendingWB() : nocache(true), size(0), ios(0) {}
    void add(bool _nocache, uint64_t _size, uint64_t _ios) {
      if (!_nocache)
		nocache = false; // only nocache if all writes are nocache
      size += _size;
      ios += _ios;
    }
  };

  // lru实现
  list<ghobject_t> lru;
  ceph::unordered_map<ghobject_t, list<ghobject_t>::iterator> rev_lru;
  void remove_object(const ghobject_t &oid) {
    assert(lock.is_locked());
    ceph::unordered_map<ghobject_t, list<ghobject_t>::iterator>::iterator iter =
      rev_lru.find(oid);
    if (iter == rev_lru.end())
      return;

    lru.erase(iter->second);
    rev_lru.erase(iter);
  }
  ghobject_t pop_object() {
    assert(!lru.empty());
    ghobject_t oid(lru.front());
    lru.pop_front();
    rev_lru.erase(oid);
    return oid;
  }
  void insert_object(const ghobject_t &oid) {
    assert(rev_lru.find(oid) == rev_lru.end());
    lru.push_back(oid);
    rev_lru.insert(make_pair(oid, --lru.end()));
  }

  ceph::unordered_map<ghobject_t, pair<PendingWB, FDRef> > pending_wbs; // 等待刷新的对象集合

  // 如果还没达到soft值，就睡眠
  // 达到soft值，就从lru中取出一个，然后刷新此对象上的操作
  bool get_next_should_flush(
    boost::tuple<ghobject_t, FDRef, PendingWB> *next ///< [out] next to flush
    ); ///< @return false if we are shutting down

public:
  enum FS {
    BTRFS,
    XFS
  };

private:
  FS fs;

  void set_from_conf();
public:
  WBThrottle(CephContext *cct);
  ~WBThrottle();

  void start(); // 创建线程
  void stop(); // 销毁线程

  /// Set fs as XFS or BTRFS
  void set_fs(FS new_fs) {
    Mutex::Locker l(lock);
    fs = new_fs;
    set_from_conf();
  }

  // 将对象的操作请求，插入到 pending_wbs 以及 lru中，等待刷新
  void queue_wb(
    FDRef fd,              ///< [in] FDRef to oid
    const ghobject_t &oid, ///< [in] object
    uint64_t offset,       ///< [in] offset written
    uint64_t len,          ///< [in] length written
    bool nocache           ///< [in] try to clear out of cache after write
    );

  // 如果三个指标中的一个超过了hard值，就睡眠
  void throttle();

  /// md_config_obs_t
  const char** get_tracked_conf_keys() const;
  void handle_conf_change(const md_config_t *conf,
			  const std::set<std::string> &changed);

  // 线程入口，调用 get_next_should_flush 获取一个flush的FD
  // 然后调用 fdatasync 或 fsync 刷新此对象上的数据
  void *entry();
};
```

这个类唯一使用的地方就是FileStore，FileStore包含成员WBThrottle，在mount的时候，会调用其start()函数启动线程，umount的时候调用其stop()函数结束线程。
限流的地方发生在函数FileStore::\_do\_op，这里会调用WBThrottle::throttle()进行限流，然后经过一系列调用，
会到FileStore::\_write()函数，此函数会调用文件系统接口执行写操作到page cache，然后将写操作通过WBThrottle::queue\_wb()加入pending\_wbs等待线程刷新,
即执行fsync/fdatasync系统调用。

```cpp
// 三种粒度任一一项超过hard值，就会阻塞
void WBThrottle::throttle()
{
  Mutex::Locker l(lock);
  while (!stopping && !(
	   cur_ios < io_limits.second &&
	   pending_wbs.size() < fd_limits.second &&
	   cur_size < size_limits.second)) {
    cond.Wait(lock);
  }
}

void FileStore::_do_op(OpSequencer *osr, ThreadPool::TPHandle &handle)
{
  wbthrottle.throttle(); // 限流

  ....
  int r = _do_transactions(o->tls, o->op, &handle); // 执行写操作
  ......
}

int FileStore::_do_transactions(
  list<Transaction*> &tls,
  uint64_t op_seq,
  ThreadPool::TPHandle *handle)
{
  ......
  for (list<Transaction*>::iterator p = tls.begin();
       p != tls.end();
       ++p, trans_num++) {
    r = _do_transaction(**p, op_seq, trans_num, handle); // 依次执行每个事务
    ......
  }
  
  return r;
}

unsigned FileStore::_do_transaction(
  Transaction& t, uint64_t op_seq, int trans_num,
  ThreadPool::TPHandle *handle)
{
  Transaction::iterator i = t.begin();
  SequencerPosition spos(op_seq, trans_num, 0);

  while (i.have_op()) {
	......
    switch (op->op) {
    case Transaction::OP_NOP:
      break;
      
    case Transaction::OP_WRITE:
      {
		  ......
          r = _write(cid, oid, off, len, bl, fadvise_flags); // 写操作
      }
      break;
	......
	}
  }
}

int FileStore::_write(coll_t cid, const ghobject_t& oid,
                     uint64_t offset, size_t len,
                     const bufferlist& bl, uint32_t fadvise_flags)
{
  ......
  r = bl.write_fd(**fd); // 写文件内容

  // flush?
  if (!replaying &&
      g_conf->filestore_wbthrottle_enable)
    wbthrottle.queue_wb(fd, oid, offset, len, // 记录需要刷新的信息
			  fadvise_flags & CEPH_OSD_OP_FLAG_FADVISE_DONTNEED);

  lfn_close(fd);
}

void WBThrottle::queue_wb(
  FDRef fd, const ghobject_t &hoid, uint64_t offset, uint64_t len,
  bool nocache)
{
  Mutex::Locker l(lock);
  ceph::unordered_map<ghobject_t, pair<PendingWB, FDRef> >::iterator wbiter =
    pending_wbs.find(hoid);
  if (wbiter == pending_wbs.end()) { // 还没记录过
    wbiter = pending_wbs.insert( // 新建一个item
      make_pair(hoid,
		make_pair(
		  PendingWB(),
		  fd))).first;
    logger->inc(l_wbthrottle_inodes_dirtied);
  } else { // 已经记录了
    remove_object(hoid); // 从lru中删除旧的对象, 后面会加入新的对象到lru
  }

  cur_ios++; // 更新io操作数
  cur_size += len; // 更新字节数

  wbiter->second.first.add(nocache, len, 1);
  insert_object(hoid); // 将对象插入到lru

  cond.Signal(); // 唤醒wb自带的线程执行刷新，以及在_do_op中阻塞的线程执行写操作
}
```

WBThrottle自带线程的入口函数为entry：

```cpp
void *WBThrottle::entry()
{
  Mutex::Locker l(lock);
  boost::tuple<ghobject_t, FDRef, PendingWB> wb;
  while (get_next_should_flush(&wb)) { // 获取一个新的item
    lock.Unlock();

	// 执行sync操作
#ifdef HAVE_FDATASYNC
    ::fdatasync(**wb.get<1>());
#else
    ::fsync(**wb.get<1>());
#endif

    lock.Lock();
    cur_ios -= wb.get<2>().ios; // 更新io操作计数
    cur_size -= wb.get<2>().size; // 更新size计数
    cond.Signal(); // 唤醒之前在_do_op中的wait操作
    wb = boost::tuple<ghobject_t, FDRef, PendingWB>(); // 重置wb
  }
  return 0;
}

bool WBThrottle::get_next_should_flush(
  boost::tuple<ghobject_t, FDRef, PendingWB> *next)
{
  assert(lock.is_locked());
  assert(next);
  while (!stopping &&
         cur_ios < io_limits.first &&
         pending_wbs.size() < fd_limits.first &&
         cur_size < size_limits.first)
         cond.Wait(lock); // 三个条件都小于soft值，睡眠

  if (stopping)
    return false;
  assert(!pending_wbs.empty());
  ghobject_t obj(pop_object()); // 从lru中获取需要刷新的对象，并从lru中删除
  
  // 获取本次刷新的item
  ceph::unordered_map<ghobject_t, pair<PendingWB, FDRef> >::iterator i =
    pending_wbs.find(obj);
  *next = boost::make_tuple(obj, i->second.second, i->second.first);
  pending_wbs.erase(i); // 删除item
  return true;
}
```

WBThrottle类中的condition变量cond，\_do\_op中的操作可能会因为throttle中三个值的任意一个值超过hard上限而wait在这个条件变量，
throttle内部的刷新线程，也可能因为throttle中的三个值全部没有达到soft下限而wait在这个条件变量。是否考虑用两个条件变量分别等待？
避免一些不必要的唤醒。

另外需要注意，FileStore中也自带一个sync线程，会定时刷新osd存放数据的整个目录current，而WBThrottle每次只是针对一个对象的回刷。

# AsyncObjectThrottle

最后一种throttle是在librbd/AsyncObjectThrottle.h 文件中，主要是用来限制一些对image的管理操作，比如flatten/snapshot/resize等。当管理操作执行的时候，
限制能够同时并发处理多少个object，原理可以参考分析flatten时的[文章](http://blog.wjin.org/posts/ceph-librbd-flatten.html)。

# Summary

* WBThrottle 针对于FileStore回刷机制，并且自带线程

* AsyncObjectThrottle用在管理image时，限制同时操作的对象个数

* 大部分情况下都会用Throttle来达到限流目的，按fifo队列排序，比较公平
