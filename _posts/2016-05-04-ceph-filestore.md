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

但是，在阅读代码的过程中，一些细节还是需要注意，比如怎么保证PG内部OP请求的执行顺序，各个限流组件怎么协调工作等等。
本文顺着代码流程，分析写流程中一些值得关注的细节，然后总结下throttle和tuning参数。

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
      
      osr->queue_journal(o->op); // journal层面的sequence记录在OpSequencer中的journal queue中

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

这里的回调就是C\_JournaledAhead，然后会执行下面这个函数，主要干两件事情：1）将op放入filestore队列排队 2）将ondisk回调放入finisher：

```cpp
void FileStore::_journaled_ahead(OpSequencer *osr, Op *o, Context *ondisk)
{
  queue_op(osr, o); // 将op在filestore层面排队，准备写入文件系统

  list<Context*> to_queue;
  osr->dequeue_journal(&to_queue);

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
op\_tp中可能多个线程获取同一个OpSequencer准备执行:

```cpp
void FileStore::queue_op(OpSequencer *osr, Op *o)
{
  // queue_op按提交时候的顺序调用，必然导致属于同一个OpSequence的OP按照提交顺序
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
