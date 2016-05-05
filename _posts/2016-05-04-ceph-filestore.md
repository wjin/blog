---
layout: post
title: "Ceph FileStore"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

ceph后端存储引擎有多种实现(filestore, kstore, memstore, bluestore), bluestore将来会成为默认的后端存储，
但是需要一些时间，现在大部分部署都是使用filestore。filestore的代码还是比较好理解的，
执行流程可以参考网上的[这篇文章](http://blog.csdn.net/ywy463726588/article/details/42679869)。

在阅读代码的过程中，一些细节还是需要注意，比如不同PG的OP操作可以并行执行，同一PG内部OP请求必须串行执行，
各个限流组件怎么协调工作，journal回放的时候需要注意非幂等操作。

本文顺着代码流程，分析写流程中一些值得关注的细节，然后总结下throttle， 非幂等操作和tuning参数。

# Write

## OSD::osd\_op\_tp

OSD::osd\_op\_tp线程池在执行PG写操作的时候，是通过函数queue\_transactions提交请求的:

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
  ......

  // 这里的OpSequencer非常关键，同一个PG会使用同样的sequencer，保证PG操作串行化
  OpSequencer *osr;
  if (!posr)
    posr = &default_osr;

  ......

  if (journal && journal->is_writeable() && !m_filestore_journal_trailing) {
    Op *o = build_op(tls, onreadable, onreadable_sync, osd_op);

    op_queue_reserve_throttle(o, handle); // filestore层对整个op的限流，释放的时候是在FileStore::_finish_op

    journal->throttle(); // 对journal的限流

    // 在journal层面为op生成唯一sequence, 因为journal是单线程写，所以写一定是串行的
    uint64_t op_num = submit_manager.op_submit_start();
    o->op = op_num;

    if (m_filestore_journal_parallel) {
		......
    } else if (m_filestore_journal_writeahead) { // ext4, xfs都需要wal
      
      osr->queue_journal(o->op); // journal层面的sequence记录在OpSequencer中的journal queue中, JournalingObjectStore::finisher线程中会deque_journal

      _op_journal_transactions(o->tls, o->op, // 提交OP，注意这里的callback以及sequence
			       new C_JournaledAhead(this, osr, o, ondisk),
			       osd_op);
    } else {
      assert(0);
    }
    submit_manager.op_submit_finish(op_num);
    return 0;
  }

  ......
}

void JournalingObjectStore::_op_journal_transactions(
  list<ObjectStore::Transaction*>& tls, uint64_t op,
  Context *onjournal, TrackedOpRef osd_op)
{
  if (journal && journal->is_writeable()) {
	......
    journal->submit_entry(op, tbl, data_align, onjournal, osd_op); // 放入journal队列，等待write线程执行journal写请求
  } else if (onjournal) {
    apply_manager.add_waiter(op, onjournal);
  }
}

void FileJournal::submit_entry(uint64_t seq, bufferlist& e, int alignment,
			       Context *oncommit, TrackedOpRef osd_op)
{
  ......

  // 获取journal限流资源
  throttle_ops.take(1);
  throttle_bytes.take(e.length());

  {
	// 注意锁的顺序
    Mutex::Locker l1(writeq_lock);  // ** lock **
    Mutex::Locker l2(completions_lock);  // ** lock **

	// write线程执行完成后，会处理这里的completion
    completions.push_back(
      completion_item(
	seq, oncommit, ceph_clock_now(g_ceph_context), osd_op));

    if (writeq.empty())
      writeq_cond.Signal();

    writeq.push_back(write_item(seq, e, alignment, osd_op)); // 放入队列，等待write线程执行
  }
}
```

执行到这里，请求已经提交到journal的队列里面，OSD::osd\_op\_tp工作就结束了。

## FileJournal::write\_thread

写journal线程通过Filejournal::write\_thread完成，流程比较简单，执行完成后，就会调用:

```cpp
void FileJournal::queue_completions_thru(uint64_t seq)
{
  ......

  // journal的一次写可以同时写入多个op请求日志，所以这里是循环处理所有已经完成的op
  // 将回调全部放入finisher线程的队列
  while (!completions_empty()) {
    completion_item next = completion_peek_front();
    if (next.seq > seq) // sequence判断是否已经写入完成
      break;
    completion_pop_front();

	......
    if (next.finish) // 放入finisher队列，等待回调
      finisher->queue(next.finish); // finisher线程实际上是JournalingObjectStore中的finisher
	......
  }
  finisher_cond.Signal();
}
```

## JournalingObjectStore::finisher

这里的回调就是C\_JournaledAhead，然后会执行下面这个函数，主要干两件事情：1）将op放入filestore队列排队 2）将ondisk回调放入FileStore::ondisk\_finisher：

```cpp
void FileStore::_journaled_ahead(OpSequencer *osr, Op *o, Context *ondisk)
{
  queue_op(osr, o); // 将op在filestore层面排队，准备写入文件系统

  list<Context*> to_queue;
  osr->dequeue_journal(&to_queue); // journal已经写成功，出队列

  // journal写好了，数据就真正落盘了，所以执行ondisk回调
  // 注意此时数据还未写入文件系统，所以不可读
  if (ondisk) {
    ondisk_finisher.queue(ondisk); // 放入ondisk_finisher的队列，等待回调
  }

  if (!to_queue.empty()) {
    ondisk_finisher.queue(to_queue);
  }
}
```

journal是单线程顺序执行的，且每条op请求都有唯一的sequence，使得queue\_op一定是按提交时候的顺序调用。
但是同一个PG可能连续提交了很多次op请求，这些请求会放入PG对应的OpSequencer中进行排队，然后同时将OpSequencer放入
op\_wq队列等待FileStore::op\_tp执行，所以如果PG连续提交请求，OpSequencer会在op\_wq中同时出现多次，
op\_tp中可能多个线程同时获取同一个OpSequencer准备执行写文件系统的操作:

```cpp
void FileStore::queue_op(OpSequencer *osr, Op *o)
{
  // queue_op按提交时候的顺序调用，必然导致属于同一个OpSequencer的OP按照提交顺序
  // 在OpSequencer内部排队, 保证了PG内部op的先后顺序
  osr->queue(o);
  op_wq.queue(osr);
}
```

## FileStore::op\_tp

PG对应的OpSequencer排队以后，说明PG有OP需要执行，这时候线程池就会对其处理，入口函数:

```cpp
void FileStore::_do_op(OpSequencer *osr, ThreadPool::TPHandle &handle)
{
  wbthrottle.throttle(); // filestore层面writeback的限流

  ......

  // op_tp线程池的多个线程可以并发对同一个OpSequencer执行请求
  // 锁保证同一个OpSequencer中(也即PG中）只能有一个OP在执行
  osr->apply_lock.Lock();

  Op *o = osr->peek_queue(); // 获取一个op

  apply_manager.op_apply_start(o->op);
  int r = _do_transactions(o->tls, o->op, &handle); // 执行写请求到文件系统
  apply_manager.op_apply_finish(o->op);
}
```

写执行完成后，线程还会执行一个finish函数:

```cpp
void FileStore::_finish_op(OpSequencer *osr)
{
  list<Context*> to_queue;
  Op *o = osr->dequeue(&to_queue); // 将op从OpSequencer出队列
  osr->apply_lock.Unlock();  // 释放锁，这时候其他线程就可以继续对此QpSequencer执行apply操作

  op_queue_release_throttle(o); // 释放filestore的throttle，见queue_transactions

  if (o->onreadable_sync) {
    o->onreadable_sync->complete(0);
  }
  if (o->onreadable) {
    op_finisher.queue(o->onreadable);
  }
  if (!to_queue.empty()) {
    op_finisher.queue(to_queue); // 放入op_finisher队列，等待执行apply回调，标志数据可读
  }
  delete o;
}
```

OpSequencer中apply\_lock保证PG内部OP的串行化，并不是保证内部队列q和jq的互斥，q和jq的互斥是另外一把锁qlock在保证。
所以在apply的过程中，OSD::osd\_op\_tp可以继续向jq中提交请求，更重要的是，JournalingObjectStore::finisher线程可以继续将
写journal完成的op在q中排队。

如前所述，同一个OpSequencer可能进入FileStore::op\_wq多次，然后被多个FileStore:op\_tp中的线程获取执行，然后q和jq共用一把锁，是否会影响性能？
其实也还好，虽然FileStore::osd\_tp是线程池，会有多个线程，但是这些线程在开始处理apply
的时候，会先获取apply\_lock，然后在执行完成的时候，从q出队列op的时候获取qlock，所以不会同时出现多个FileStore::osd\_tp的线程去
抢qlock这个锁，可以认为同一时刻q只增加了两个线程去抢qlock，即JournalingObjectStore::finisher 和 其中一个FileStore::osd\_tp线程。

# Throttle

filestore实现中，提供了三个限流的地方: 1) journal 2) filestore apply 3)filestore writeback

## journal

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
	......
    journal->throttle();
	......
}

int FileJournal::prepare_multi_write(bufferlist& bl, uint64_t& orig_ops, uint64_t& orig_bytes)
{
	......
	put_throttle(1, peek_write().bl.length());
	......

}
```

## filestore apply

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
	......
	op_queue_reserve_throttle(o, handle);
	......
}

void FileStore::_finish_op(OpSequencer *osr)
{
	......
    op_queue_release_throttle(o);
	......
}
```

## filestore writeback

wbthrottle，参见另外一篇[文章](http://blog.wjin.org/posts/ceph-throttle-summary.html)。

# Non-idempotent OP

在osd异常崩溃的情况下，journal中的数据不一定全部都存放在了FileStore的data disk中，因为apply到了FileStore中，并不代表数据就在disk中了，
此时很有可能数据在page cache中，需要sync线程调用fdatasync之类的系统调用才能保证数据落盘。

所以为了保证异常情况下数据的一致性，需要对journla的日志做回放，从什么地方开始回放，FileStore中会将已经apply到文件系统并进行
过fdatasync的序列号记录在文件commit\_op\_seq中，回放的时候就从此文件记录的序列号开始。

然而，回放的时候，部分op可能已经在disk中生效，但是commit\_op\_seq并没有体现，此时如果仍然回放，对于有些操作，反复执行多次会出问题，也即非幂等操作。

一个例子：

* clone一个object，这个操作已经提交到日志
* 将操作apply到FileStore也已经完成
* 源object后续做了更新，也apply到了FileStore

假设如上操作都已经体现在了disk中，但是sync线程并未来得及更新commit\_op\_seq，此时系统崩溃。
再次启动后，osd启动回放日志，第二次执行clone操作，将拷贝到新版本的数据，而不是期望版本的数据。
FileStore需要保证回放处理这种情况的正确性。

具体做法是在对象文件的属性中记录下最后操作的一个三元组（序列号，事务编号，OP编号），因为journal提交的时候有一个唯一的序列号，通过这个序列号，
就可以找到提交时候的事务，然后根据事务编号和OP编号最终定位出最后操作的OP。

```cpp
struct SequencerPosition {
  uint64_t seq;  ///< seq
  uint32_t trans; ///< transaction in that seq (0-based)
  uint32_t op;    ///< op in that transaction (0-based)
  ......
};
```

看clone例子，操作前，先检查下，如果可以继续执行，就执行操作，操作完成后，设置一个guard。

```cpp
int FileStore::_clone(const coll_t& cid, const ghobject_t& oldoid, const ghobject_t& newoid,
		      const SequencerPosition& spos)
{
  ......
  if (_check_replay_guard(cid, newoid, spos) < 0)
    return 0;

  ......

  // clone is non-idempotent; record our work.
  _set_replay_guard(**n, spos, &newoid);

  ......
}
```

# Tuning

```text
# journal 
journal_queue_max_bytes
journal_queue_max_ops

# filestore apply 
filestore_queue_max_bytes
filestore_queue_max_ops

# filestore writeback
filestore_wbthrottle_enable
filestore_wbthrottle_xfs_bytes_start_flusher
filestore_wbthrottle_xfs_bytes_hard_limit
filestore_wbthrottle_xfs_ios_start_flusher
filestore_wbthrottle_xfs_ios_hard_limit
filestore_wbthrottle_xfs_inodes_start_flusher
```

filestore writeback打开以后，一方面需要注意对应限流参数的调整，ext4和xfs是共用一套参数，
另一方面，如果io压力持续过大，可能导致FileStore::op\_tp被throttle住而超时，也有可能导致
FileStore::op\_wq限流起作用。

如果关闭writeback，可能导致FileStore:sync\_thread超时，需要调整参数filestore\_commit\_timeout，
ssd情况下可以关闭wbthrottle。


其他影响性能的参数：

```text
filestore_op_threads
filestore_fd_cache_size

journal_max_write_bytes
journal_max_write_entries

filestore_max_sync_interval
```
