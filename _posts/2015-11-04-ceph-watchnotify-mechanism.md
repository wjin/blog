---
layout: post
title: "Ceph Watch/Notify Mechanism"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

ceph中，有一个比较重要的watch/notify机制(粒度是object)，它用来在不同客户端之间进行通信，使得各客户端之间的状态保持一致，
并且新功能librbd exclusive lock也是基于它来实现的。

在块存储中，如果image的元数据发生变化(header object发生变化)，需要通知当前所有的客户端，这是怎么做到的呢？

客户端打开image的时候，如果不是只读，就会通过ImageWatcher类接口，注册一个watch在header object上，以后header object修改的事件发生，
自己就会收到通知。

librbd API 并没有watch这样的接口，用户使用块存储API并不需要关心这个细节，当使用librados的API时，如果需要，可以注册一些watch。
这里以打开一个image为例子来说明：

# Watch

## librbd level

客户端要读写某个image，首先得open：

```cpp
  int RBD::open(IoCtx& io_ctx, Image& image, const char *name,
		const char *snap_name)
  {
    ImageCtx *ictx = new ImageCtx(name, "", snap_name, io_ctx, false); // 最后一个false表示不是read_only，这样才会注册watch
    tracepoint(librbd, open_image_enter, ictx, ictx->name.c_str(), ictx->id.c_str(), ictx->snap_name.c_str(), ictx->read_only);

    int r = librbd::open_image(ictx); // 调用internal.cc
    if (r < 0) {
      tracepoint(librbd, open_image_exit, r);
      return r;
    }

    image.ctx = (image_ctx_t) ictx;
    tracepoint(librbd, open_image_exit, 0);
    return 0;
  }
```

仍然是调用internal.cc的代码：

```cpp
  int open_image(ImageCtx *ictx)
  {
    int r = ictx->init(); // 读取metadata初始化image信息
    if (r < 0)
      goto err_close;

    if (!ictx->read_only) { // 非只读
      r = ictx->register_watch(); // 将客户端自己注册为watcher
      if (r < 0) {
		lderr(ictx->cct) << "error registering a watch: " << cpp_strerror(r)
			 << dendl;
		goto err_close;
      }
    }

    {
      RWLock::RLocker owner_locker(ictx->owner_lock);
      r = ictx_refresh(ictx); // 刷新
    }
    if (r < 0)
      goto err_close;

    if ((r = _snap_set(ictx, ictx->snap_name.c_str())) < 0)
      goto err_close;

    return 0;

  err_close:
    close_image(ictx);
    return r;
  }
```

注册的过程，是通过ImageWatcher类调用librados API 完成：

```cpp
// 只会注册一次，watch一直存在
int ImageCtx::register_watch() {
    assert(image_watcher == NULL);
    image_watcher = new ImageWatcher(*this);
    return image_watcher->register_watch(); // 通过ImageWatcher管理类注册
}

int ImageWatcher::register_watch() {
  RWLock::WLocker l(m_watch_lock);
  assert(m_watch_state == WATCH_STATE_UNREGISTERED);
  int r = m_image_ctx.md_ctx.watch2(m_image_ctx.header_oid, // 注意是header_oid
				    &m_watch_handle, // handle会被初始化为linger op的地址，以后收到notify的时候用以区分是哪个op，进而区分是哪个watch（可能会注册多个watch)
				    &m_watch_ctx); // 以后收到notify的时候调用这个callback，对notify消息进行处理
  if (r < 0) {
    return r;
  }

  m_watch_state = WATCH_STATE_REGISTERED; // 修改状态
  return 0;
}

```

## librados level

注册操作，对后端rados集群来说，是一个OP操作，所以也会通过objecter封装，然后以消息的方式发送到后端集群。
其次，它还是一个写操作，因为要将watch的客户端信息存在磁盘上，这样OSD重启后仍然知道有哪些watch存在。

```cpp

int librados::IoCtx::watch2(const string& oid, uint64_t *cookie,
			    librados::WatchCtx2 *ctx2)
{
  object_t obj(oid);
  return io_ctx_impl->watch(obj, cookie, NULL, ctx2);
}

int librados::IoCtxImpl::watch(const object_t& oid,
			       uint64_t *handle,
			       librados::WatchCtx *ctx,
			       librados::WatchCtx2 *ctx2)
{
  ::ObjectOperation wr; // 定义一个写的操作
  version_t objver;
  C_SaferCond onfinish;

  Objecter::LingerOp *linger_op = objecter->linger_register(oid, oloc, 0); // 会new一个LingerOp对象，并记录在objecter的map/set中
  *handle = linger_op->get_cookie(); // cookie就是刚才new的LingerOp的地址转换为int64
  linger_op->watch_context = new WatchInfo(this,
					   oid, ctx, ctx2); // ctx为NULL，废弃的参数, ctx2为ImageWatcher类的成员: m_watch_ctx

  prepare_assert_ops(&wr);
  wr.watch(*handle, CEPH_OSD_WATCH_OP_WATCH); // 给wr增加一个watch操作，op的类型是：CEPH_OSD_WATCH_OP_WATCH
  bufferlist bl;
  objecter->linger_watch(linger_op, wr, // 请求objecter发送
			 snapc, ceph_clock_now(NULL), bl,
			 &onfinish, // 下面一行会wait在这个条件变量，等待操作完成
			 &objver);

  int r = onfinish.wait(); // 等待回调

  set_sync_op_version(objver);

  if (r < 0) {
    objecter->linger_cancel(linger_op);
  }

  return r;
}
```

## osdc level

接下来是通过objecter将操作发送出去，和其它通用的OP操作类似，只是在前面多了几步：`linger_watch -> _linger_submit -> _send_linger -> _op_submit`

```cpp
ceph_tid_t Objecter::linger_watch(LingerOp *info,
				  ObjectOperation& op,
				  const SnapContext& snapc, utime_t mtime,
				  bufferlist& inbl,
				  Context *oncommit,
				  version_t *objver)
{
  info->is_watch = true; // 这个true很关键，收到notify消息的时候，会带有cookie，即linger_op的地址，然后通过这个变量来判断本次消息对应的客户端的linger_op是watch还是notify
  info->snapc = snapc;
  info->mtime = mtime;
  info->target.flags |= CEPH_OSD_FLAG_WRITE;
  info->ops = op.ops;
  info->inbl = inbl;
  info->poutbl = NULL;
  info->pobjver = objver;
  info->on_reg_commit = oncommit; // 这个callback很关键，调用的时候会唤醒watch函数中的wait操作

  RWLock::WLocker wl(rwlock);
  _linger_submit(info); // 提交linger op
  logger->inc(l_osdc_linger_active);

  return info->linger_id;
}

void Objecter::_linger_submit(LingerOp *info)
{
  assert(rwlock.is_wlocked());
  RWLock::Context lc(rwlock, RWLock::Context::TakenForWrite);

  assert(info->linger_id);

  // Populate Op::target
  OSDSession *s = NULL;
  _calc_target(&info->target, &info->last_force_resend);

  // Create LingerOp<->OSDSession relation
  int r = _get_session(info->target.osd, &s, lc);
  assert(r == 0);
  s->lock.get_write();
  _session_linger_op_assign(s, info); // 关联session与linger op
  s->lock.unlock();
  put_session(s);

  _send_linger(info); // 发送
}

void Objecter::_send_linger(LingerOp *info)
{
  assert(rwlock.is_wlocked());

  RWLock::Context lc(rwlock, RWLock::Context::TakenForWrite);

  vector<OSDOp> opv;
  Context *oncommit = NULL;
  info->watch_lock.get_read(); // just to read registered status
  if (info->registered && info->is_watch) {
    ldout(cct, 15) << "send_linger " << info->linger_id << " reconnect" << dendl;
    opv.push_back(OSDOp());
    opv.back().op.op = CEPH_OSD_OP_WATCH;
    opv.back().op.watch.cookie = info->get_cookie();
    opv.back().op.watch.op = CEPH_OSD_WATCH_OP_RECONNECT;
    opv.back().op.watch.gen = ++info->register_gen;
    oncommit = new C_Linger_Reconnect(this, info);
  } else {
    ldout(cct, 15) << "send_linger " << info->linger_id << " register" << dendl;
    opv = info->ops;
    oncommit = new C_Linger_Commit(this, info); // 第一次会创建这个callback
  }
  info->watch_lock.put_read();
  Op *o = new Op(info->target.base_oid, info->target.base_oloc,
		 opv, info->target.flags | CEPH_OSD_FLAG_READ,
		 NULL, NULL,
		 info->pobjver);
  o->oncommit_sync = oncommit; // 将新建的callback赋值op的成员oncommit_sync，这个成员是watch/notify专用的，handle_osd_op_reply会回调此callback
  o->snapid = info->snap;
  o->snapc = info->snapc;
  o->mtime = info->mtime;

  o->target = info->target;
  o->tid = last_tid.inc();

  // do not resend this; we will send a new op to reregister
  o->should_resend = false;

  if (info->register_tid) {
    // repeat send.  cancel old registeration op, if any.
    info->session->lock.get_write();
    if (info->session->ops.count(info->register_tid)) {
      Op *o = info->session->ops[info->register_tid];
      _op_cancel_map_check(o);
      _cancel_linger_op(o);
    }
    info->session->lock.unlock();

    info->register_tid = _op_submit(o, lc); // 发送
  } else {
    // first send
    info->register_tid = _op_submit_with_budget(o, lc); // 发送
  }

  logger->inc(l_osdc_linger_send);
}
```

OP提交后，会通过消息发送到rados集群，集群会通过messge和op的类型分类，经过一系列操作后最终会将watch客户端的信息落盘，
然后会发送回处理完成的消息，这个消息和以前一样（`CEPH_MSG_OSD_OPREPLY`），会在ms\_dispatch函数处理，最后调用：`handle_osd_op_reply`

这个函数比较复杂，但最终会调用回调`op->oncommit_sync` :

```cpp
void Objecter::handle_osd_op_reply(MOSDOpReply *m)
{
	......
	if (op->oncommit_sync) {
      op->oncommit_sync->complete(rc); // 调用回调
      op->oncommit_sync = NULL;
      num_uncommitted.dec();
    }
	......
}
```

而oncommit\_sync由上面分析可知是C\_Linger\_Commit类型：

```cpp
  struct C_Linger_Commit : public Context {
    Objecter *objecter;
    LingerOp *info;
    C_Linger_Commit(Objecter *o, LingerOp *l) : objecter(o), info(l) {
      info->get();
    }
    ~C_Linger_Commit() {
      info->put();
    }
    void finish(int r) {
      objecter->_linger_commit(info, r); // 继续回调
    }
  };

void Objecter::_linger_commit(LingerOp *info, int r) 
{
  RWLock::WLocker wl(info->watch_lock);
  ldout(cct, 10) << "_linger_commit " << info->linger_id << dendl;
  if (info->on_reg_commit) {
    info->on_reg_commit->complete(r); // 继续回调, 最终唤醒watch函数中的wait操作
    info->on_reg_commit = NULL;
  }

  // only tell the user the first time we do this
  info->registered = true;
  info->pobjver = NULL;
}
```

这样watch操作就算完成了，接下来就会收到notify消息，当然客户端也可以主动发送notify消息，先看怎么发送，然后看怎么处理接收消息。

# Notify

## send

当某个客户端的事件需要通知给其他客户端的时候，就会调用notify相关函数。这些通知事件，基本上都是由ImageWatcher类发起，
比如notify\_header\_update, notify\_released\_lock等。最终也会通过objecter创建一个OP发送出去，和watch操作不同的地方是，
这个OP是读类型，因为没有信息需要落盘执行写操作。

```cpp
int librados::IoCtx::notify2(const string& oid, bufferlist& bl,
			     uint64_t timeout_ms, bufferlist *preplybl)
{
  object_t obj(oid);
  return io_ctx_impl->notify(obj, bl, timeout_ms, preplybl, NULL, NULL);
}

int librados::IoCtxImpl::notify(const object_t& oid, bufferlist& bl,
				uint64_t timeout_ms,
				bufferlist *preply_bl,
				char **preply_buf, size_t *preply_buf_len)
{
  bufferlist inbl;

  struct C_NotifyFinish : public Context {
    Cond cond;
    Mutex lock;
    bool done;
    int result;
    bufferlist reply_bl;

    C_NotifyFinish()
      : lock("IoCtxImpl::notify::C_NotifyFinish::lock"),
	done(false),
	result(0) { }

    void finish(int r) {}
    void complete(int r) {
      lock.Lock();
      done = true;
      result = r;
      cond.Signal();
      lock.Unlock();
    }
    void wait() {
      lock.Lock();
      while (!done)
	cond.Wait(lock);
      lock.Unlock();
    }
  } notify_private;

  Objecter::LingerOp *linger_op = objecter->linger_register(oid, oloc, 0); // new一个linger op
  linger_op->on_notify_finish = &notify_private; // 这个callback也很关键
  linger_op->notify_result_bl = &notify_private.reply_bl;

  uint32_t prot_ver = 1;
  uint32_t timeout = notify_timeout;
  if (timeout_ms)
    timeout = timeout_ms / 1000;
  ::encode(prot_ver, inbl);
  ::encode(timeout, inbl);
  ::encode(bl, inbl);

  // Construct RADOS op
  ::ObjectOperation rd;
  prepare_assert_ops(&rd);
  rd.notify(linger_op->get_cookie(), inbl);

  // Issue RADOS op
  C_SaferCond onack; // 这个callback当操作成功并收到OP回包后会回调
  version_t objver;
  objecter->linger_notify(linger_op, // 发送出去
			  rd, snap_seq, inbl, NULL,
			  &onack, &objver);

  int r_issue = onack.wait(); // 等待OP成功执行

  if (r_issue == 0) {
    notify_private.wait(); // 等待回来的notify消息，不是OP执行成功的消息
  } else {
    ldout(client->cct, 10) << __func__ << " failed to initiate notify, r = "
			   << r_issue << dendl;
  }

  ......
  // 这个和watch操作不一样，watch是注册一个linger_op，一直存在，以后用来接收notify通知，通过cookie区别
  // 而notify在成功执行完成后，这个linger_op就没用了，需要取消
  objecter->linger_cancel(linger_op);
  ......

}
```

注意这里有两个wait操作，第一个类似于watch操作的步骤：`linger_notify -> _linger_submit -> _send_linger -> _op_submit`
等待op处理完成的消息`CEPH_MSG_OSD_OPREPLY`，会在ms\_dispatch函数处理，最后调用：`handle_osd_op_reply`，唤醒第一个wait。

第二个wait会在收到另外一个消息`CEPH_MSG_WATCH_NOTIFY`时，由ms\_dispatch调用函数`handle_watch_notify`处理，这个函数最终会唤醒第二个wait。
这是因为后端rados集群收到notify消息后，会将消息发送给所有的watch，客户端自己本身也是watch，所以也会收到，不同的客户端会处理不同的分支。

## receive

```cpp
void Objecter::handle_watch_notify(MWatchNotify *m)
{
  RWLock::RLocker l(rwlock);
  if (!initialized.read()) {
    return;
  }

  LingerOp *info = reinterpret_cast<LingerOp*>(m->cookie); // 如前面介绍一样，cookie就是op的地址
  if (linger_ops_set.count(info) == 0) {
    ldout(cct, 7) << __func__ << " cookie " << m->cookie << " dne" << dendl;
    return;
  }
  RWLock::WLocker wl(info->watch_lock);
  if (m->opcode == CEPH_WATCH_EVENT_DISCONNECT) {
    if (!info->last_error) {
      info->last_error = -ENOTCONN;
      if (info->watch_context) {
	finisher->queue(new C_DoWatchError(this, info, -ENOTCONN));
	_linger_callback_queue();
      }
    }
  } else if (!info->is_watch) { // 作为notify自己的时候，这个是false，发现是自己发出去的notify有了回应，所以调用callback回去唤醒以前的wait操作
    // notify completion; we can do this inline since we know the only user
    // (librados) is safe to call in fast-dispatch context
    assert(info->on_notify_finish);
    info->notify_result_bl->claim(m->get_data());
    info->on_notify_finish->complete(m->return_code); // 唤醒自己的wait操作 
  } else { // 其他watch客户端，也会收到这个消息，cookie是自己注册时的OP，注册watch时这个值为true
    finisher->queue(new C_DoWatchNotify(this, info, m)); // 这里就会执行注册时的ImageWatcher::m_watch_ctx
    _linger_callback_queue();
  }
}
```

看看watch时设置的callback是怎么被调用的：

```cpp
struct C_DoWatchNotify : public Context {
  Objecter *objecter;
  Objecter::LingerOp *info;
  MWatchNotify *msg;
  C_DoWatchNotify(Objecter *o, Objecter::LingerOp *i, MWatchNotify *m)
    : objecter(o), info(i), msg(m) {
    info->get();
    info->_queued_async();
    msg->get();
  }
  void finish(int r) {
    objecter->_do_watch_notify(info, msg); // 回调
  }
};

void Objecter::_do_watch_notify(LingerOp *info, MWatchNotify *m)
{
  rwlock.get_read();
  assert(initialized.read());

  if (info->canceled) {
    rwlock.put_read();
    goto out;
  }

  rwlock.put_read();

  switch (m->opcode) {
  case CEPH_WATCH_EVENT_NOTIFY:
	// 注册的时候有这么一句：linger_op->watch_context = new WatchInfo(this, oid, ctx, ctx2);
    info->watch_context->handle_notify(m->notify_id, m->cookie,
				       m->notifier_gid, m->bl);
    break;
  }

 out:
  info->finished_async();
  info->put();
  m->put();
  _linger_callback_finish();
}

// 继续WatchInfo
struct WatchInfo : public Objecter::WatchContext {
  librados::IoCtxImpl *ioctx;
  object_t oid;
  librados::WatchCtx *ctx;
  librados::WatchCtx2 *ctx2;

  WatchInfo(librados::IoCtxImpl *io, object_t o,
	    librados::WatchCtx *c, librados::WatchCtx2 *c2)
    : ioctx(io), oid(o), ctx(c), ctx2(c2) {
    ioctx->get();
  }
  ~WatchInfo() {
    ioctx->put();
  }

  void handle_notify(uint64_t notify_id,
		     uint64_t cookie,
		     uint64_t notifier_id,
		     bufferlist& bl) {

    if (ctx2)
      ctx2->handle_notify(notify_id, cookie, notifier_id, bl); // 回调ctx2, 就是ImageWatcher::m_watch_ctx
  }
};

void ImageWatcher::WatchCtx::handle_notify(uint64_t notify_id,
        	                           uint64_t handle,
                                           uint64_t notifier_id,
	                                   bufferlist& bl) {
  image_watcher.handle_notify(notify_id, handle, bl); // 继续回调
}

// 最终对notify消息进行解析, 不出所料，仍然在ImageWatcher类中
void ImageWatcher::handle_notify(uint64_t notify_id, uint64_t handle,
				 bufferlist &bl) {
  NotifyMessage notify_message;
  if (bl.length() == 0) {
    // legacy notification for header updates
    notify_message = NotifyMessage(HeaderUpdatePayload());
  } else {
    try {
      bufferlist::iterator iter = bl.begin();
      ::decode(notify_message, iter);
    } catch (const buffer::error &err) {
      lderr(m_image_ctx.cct) << this << " error decoding image notification: "
			     << err.what() << dendl;
      return;
    }
  }

  apply_visitor(HandlePayloadVisitor(this, notify_id, handle), // 对notify消息进行处理
		notify_message.payload);
}
```

对不同消息类型(payload)，调用不同的重载函数处理，然后发送回确认消息：

```cpp
    struct HandlePayloadVisitor : public boost::static_visitor<void> { // 继承boost::static_visitor，以后用apply_visitor解析
      ImageWatcher *image_watcher;
      uint64_t notify_id;
      uint64_t handle;

      HandlePayloadVisitor(ImageWatcher *image_watcher_, uint64_t notify_id_,
			   uint64_t handle_)
		: image_watcher(image_watcher_), notify_id(notify_id_), handle(handle_)
      {
      }

      inline void operator()(const WatchNotify::HeaderUpdatePayload &payload) const {
		bufferlist out;
		image_watcher->handle_payload(payload, &out);
		image_watcher->acknowledge_notify(notify_id, handle, out);
      }

      template <typename Payload>
      inline void operator()(const Payload &payload) const { // apply_visitor会调用这里
		bufferlist out;
		image_watcher->handle_payload(payload, &out); // 根据payload类型，调用不同的重载函数处理
		image_watcher->acknowledge_notify(notify_id, handle, out); // 确认notify消息，会将确认消息发送给其他客户端
      }
    };
```

# Summary

1. 一个客户端，当不是以只读打开image的时候，会注册watch接收消息

2. 客户端可能主动发送notify消息到其他客户端，也同时接受来自其他客户端的消息

3. 客户端自己也会收到自己发出去的notify消息，通过cookie确定linger op(这里可能会存在两个或以上的linger op，一个是自己注册watch的时候创建的，一个是notify消息创建的)，通过linger op的is\_watch可以决定是不是自己发出消息对应的回应消息
