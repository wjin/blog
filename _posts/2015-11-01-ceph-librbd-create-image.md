---
layout: post
title: "Ceph Librbd Create Image"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview
# Code Analysis
## librbd level

首先从librbd.cc开始，简单wrapper，实际上调用了internal.cc的create:

```cpp
  // io_ctx 参数用来连接librados，name就是image的名字，size为大小
  // order为对象尺寸，比如，默认值为22, 即4mb (1 << 22)
  int RBD::create(IoCtx& io_ctx, const char *name, uint64_t size, int *order)
  {
    tracepoint(librbd, create_enter, io_ctx.get_pool_name().c_str(), io_ctx.get_id(), name, size, *order);
    int r = librbd::create(io_ctx, name, size, order); // 调用internal.cc的实现
    tracepoint(librbd, create_exit, r, *order);
    return r;
  }
```

接下来重点是internal.cc的代码, 首先处理了image一些基本参数信息：包括format, order, stripe等的设置,
stripe的解释，可以参看官方解释[data striping](http://docs.ceph.com/docs/v0.80.5/architecture/#data-striping):

```cpp
  int create(librados::IoCtx& io_ctx, const char *imgname, uint64_t size,
	     int *order)
  {
    CephContext *cct = (CephContext *)io_ctx.cct(); // cct是管理ceph集群配置的一个句柄，到处都会用到
    bool old_format = cct->_conf->rbd_default_format == 1; // 一般现在用format2，所以不是old
    uint64_t features = old_format ? 0 : cct->_conf->rbd_default_features; // 新的format会支持新加的feature
    return create(io_ctx, imgname, size, old_format, features, order, 0, 0);
  }

  // 真正实现的函数
  int create(IoCtx& io_ctx, const char *imgname, uint64_t size,
	     bool old_format, uint64_t features, int *order,
	     uint64_t stripe_unit, uint64_t stripe_count)
  {
    if (!order)
      return -EINVAL;

    CephContext *cct = (CephContext *)io_ctx.cct();

    if (features & ~RBD_FEATURES_ALL) {
      lderr(cct) << "librbd does not support requested features." << dendl;
      return -ENOSYS;
    }

    // make sure it doesn't already exist, in either format
    int r = detect_format(io_ctx, imgname, NULL, NULL); // 检测image的格式
    if (r != -ENOENT) {
      if (r) {
	lderr(cct) << "Could not tell if " << imgname << " already exists" << dendl;
	return r;
      }
      lderr(cct) << "rbd image " << imgname << " already exists" << dendl;
      return -EEXIST;
    }

    if (!*order)
      *order = cct->_conf->rbd_default_order;
    if (!*order)
      *order = RBD_DEFAULT_OBJ_ORDER;

    if (*order && (*order > 64 || *order < 12)) {
      lderr(cct) << "order must be in the range [12, 64]" << dendl;
      return -EDOM;
    }

    Rados rados(io_ctx); // 用于连接rados, 即ceph集群
    // 获取客户端id, 这个会由RadosClient类对象通过monitor分配得到
	// Rados类包含RadosClient, connect的时候会被初始化
    uint64_t bid = rados.get_instance_id();

    // if striping is enabled, use possibly custom defaults
    if (!old_format && (features & RBD_FEATURE_STRIPINGV2) &&
	!stripe_unit && !stripe_count) {
      stripe_unit = cct->_conf->rbd_default_stripe_unit;
      stripe_count = cct->_conf->rbd_default_stripe_count;
    }

    // normalize for default striping
    if (stripe_unit == (1ull << *order) && stripe_count == 1) {
      stripe_unit = 0;
      stripe_count = 0;
    }
    if ((stripe_unit || stripe_count) &&
	(features & RBD_FEATURE_STRIPINGV2) == 0) {
      lderr(cct) << "STRIPINGV2 and format 2 or later required for non-default striping" << dendl;
      return -EINVAL;
    }
    if ((stripe_unit && !stripe_count) ||
	(!stripe_unit && stripe_count))
      return -EINVAL;

    if (old_format) { // 旧的格式
      if (stripe_unit && stripe_unit != (1ull << *order))
	return -EINVAL;
      if (stripe_count && stripe_count != 1)
	return -EINVAL;

      return create_v1(io_ctx, imgname, bid, size, *order);
    } else {
	  // format2 会走这条路径
      return create_v2(io_ctx, imgname, bid, size, *order, features,
		       stripe_unit, stripe_count);
    }
  }
```


具体到image的创建，实际上只会记录一些image的基本信息。
比如创建元数据对象`rbd_id.foo`和`rbd_header.foo`, 对于image的真正数据对象`rbd_data*`,
根本不会创建，这是因为ceph选择`thin-provisioning`这种方式，可以做到妙极创建块设备，后端也可以超额分配容量。


创建过程，很多步骤都是请求插件cls_client(`src/cls/rbd`)完成, 插件会向rados发送命令执行:

```cpp
  int create_v2(IoCtx& io_ctx, const char *imgname, uint64_t bid, uint64_t size,
		int order, uint64_t features, uint64_t stripe_unit,
		uint64_t stripe_count)
  {
    ostringstream bid_ss;
    uint32_t extra;
    string id, id_obj, header_oid;
    int remove_r;
    ostringstream oss;
    CephContext *cct = (CephContext *)io_ctx.cct();

    ceph_file_layout layout;

    id_obj = id_obj_name(imgname);

	// 首先向librados发送请求，创建rbd_id.foo 这样的元数据对象
    int r = io_ctx.create(id_obj, true);
    if (r < 0) {
      lderr(cct) << "error creating rbd id object: " << cpp_strerror(r)
		 << dendl;
      return r;
    }

    extra = rand() % 0xFFFFFFFF;
    bid_ss << std::hex << bid << std::hex << extra;
    id = bid_ss.str();
    r = cls_client::set_id(&io_ctx, id_obj, id); // 通过插件设置image的id
    if (r < 0) {
      lderr(cct) << "error setting image id: " << cpp_strerror(r) << dendl;
      goto err_remove_id;
    }

    ldout(cct, 2) << "adding rbd image to directory..." << dendl;
    r = cls_client::dir_add_image(&io_ctx, RBD_DIRECTORY, imgname, id); // 将image的<name,id>加到rbd目录，所有创建的image都在这里，方便ls命令列出pool所有的image
    if (r < 0) {
      lderr(cct) << "error adding image to directory: " << cpp_strerror(r)
		 << dendl;
      goto err_remove_id;
    }

    oss << RBD_DATA_PREFIX << id;
    header_oid = header_name(id);
    r = cls_client::create_image(&io_ctx, header_oid, size, order, // 创建image, 实际上会创建一个header对象：rbd_header.foo
				 features, oss.str());
    if (r < 0) {
      lderr(cct) << "error writing header: " << cpp_strerror(r) << dendl;
      goto err_remove_from_dir;
    }

    if ((stripe_unit || stripe_count) &&
	(stripe_count != 1 || stripe_unit != (1ull << order))) {
      r = cls_client::set_stripe_unit_count(&io_ctx, header_oid, // 设置元数据信息
					    stripe_unit, stripe_count);
      if (r < 0) {
	lderr(cct) << "error setting striping parameters: "
		   << cpp_strerror(r) << dendl;
	goto err_remove_header;
      }
    }

    if ((features & RBD_FEATURE_OBJECT_MAP) != 0) { // object map 相关信息
      if ((features & RBD_FEATURE_EXCLUSIVE_LOCK) == 0) {
        lderr(cct) << "cannot use object map without exclusive lock" << dendl;
        goto err_remove_header;
      }

      memset(&layout, 0, sizeof(layout));
      layout.fl_object_size = 1ull << order;
      if (stripe_unit == 0 || stripe_count == 0) {
        layout.fl_stripe_unit = layout.fl_object_size;
        layout.fl_stripe_count = 1;
      } else {
        layout.fl_stripe_unit = stripe_unit;
        layout.fl_stripe_count = stripe_count;
      }

      librados::ObjectWriteOperation op;
      cls_client::object_map_resize(&op, Striper::get_num_objects(layout, size),
                                    OBJECT_NONEXISTENT);
      r = io_ctx.operate(ObjectMap::object_map_name(id, CEPH_NOSNAP), &op);
      if (r < 0) {
        goto err_remove_header;
      }
    }

    ldout(cct, 2) << "done." << dendl;
    return 0; // 完成创建

  // 中途错误处理
  err_remove_header:
    remove_r = io_ctx.remove(header_oid);
    if (remove_r < 0) {
      lderr(cct) << "error cleaning up image header after creation failed: "
		 << dendl;
    }
  err_remove_from_dir:
    remove_r = cls_client::dir_remove_image(&io_ctx, RBD_DIRECTORY,
					    imgname, id);
    if (remove_r < 0) {
      lderr(cct) << "error cleaning up image from rbd_directory object "
		 << "after creation failed: " << cpp_strerror(remove_r)
		 << dendl;
    }
  err_remove_id:
    remove_r = io_ctx.remove(id_obj);
    if (remove_r < 0) {
      lderr(cct) << "error cleaning up id object after creation failed: "
		 << cpp_strerror(remove_r) << dendl;
    }

    return r;
  }
```

## librados level

无论是通过cls插件，还是直接调用librados的API，最终都是将操作封装成消息，发送到后端的OSD集群。

简单看一下创建对象的实现，其它操作类似。

在`create_v2`函数中，首先是通过librados层的类IoCtx的create函数，创建一个具体的对像: `rbd_id.foo`

```cpp
int librados::IoCtx::create(const std::string& oid, bool exclusive)
{
  object_t obj(oid);
  return io_ctx_impl->create(obj, exclusive); // IoCtx 通过IoCtxImpl 实现
}

int librados::IoCtxImpl::create(const object_t& oid, bool exclusive)
{
  ::ObjectOperation op; // 这个是全局的ObjectOperation, 实际上是引用的src/osdc/Objecter.cc里面定义的类
  prepare_assert_ops(&op); // 一些对object version的处理
  op.create(exclusive); // 将op的真正内容进行初始化
  return operate(oid, &op, NULL); // 借助于osdc/objecter.cc里的函数，真正将op通过消息发送到OSD
}
```

接下来看看OP是怎么通过消息发送出去到OSD端的：

```cpp
int librados::IoCtxImpl::operate(const object_t& oid, ::ObjectOperation *o,
				 time_t *pmtime, int flags)
{
  utime_t ut;
  if (pmtime) {
    ut = utime_t(*pmtime, 0);
  } else {
    ut = ceph_clock_now(client->cct);
  }

  /* can't write to a snapshot */
  if (snap_seq != CEPH_NOSNAP)
    return -EROFS;

  if (!o->size())
    return 0;

  Mutex mylock("IoCtxImpl::operate::mylock");
  Cond cond;
  bool done;
  int r;
  version_t ver;

  Context *oncommit = new C_SafeCond(&mylock, &cond, &done, &r);

  int op = o->ops[0].op.op;
  ldout(client->cct, 10) << ceph_osd_op_name(op) << " oid=" << oid << " nspace=" << oloc.nspace << dendl;

  // 利用前面准备的ObjectOperation对象，初始化Objecter::Op对象，这里会new一个Objecter::Op出来
  Objecter::Op *objecter_op = objecter->prepare_mutate_op(oid, oloc,
	                                                  *o, snapc, ut, flags,
	                                                  NULL, oncommit, &ver);
  // 将op提交
  objecter->op_submit(objecter_op);

  mylock.Lock();
  while (!done)
    cond.Wait(mylock);
  mylock.Unlock();
  ldout(client->cct, 10) << "Objecter returned from "
	<< ceph_osd_op_name(op) << " r=" << r << dendl;

  set_sync_op_version(ver);

  return r;
}
```

发送之前，先简单的做一个限流操作：

```cpp
ceph_tid_t Objecter::op_submit(Op *op, int *ctx_budget)
{
  RWLock::RLocker rl(rwlock);
  RWLock::Context lc(rwlock, RWLock::Context::TakenForRead);
  return _op_submit_with_budget(op, lc, ctx_budget);
}

// 简单的throttle处理，以免发送太快
ceph_tid_t Objecter::_op_submit_with_budget(Op *op, RWLock::Context& lc, int *ctx_budget)
{
  assert(initialized.read());

  assert(op->ops.size() == op->out_bl.size());
  assert(op->ops.size() == op->out_rval.size());
  assert(op->ops.size() == op->out_handler.size());

  // throttle.  before we look at any state, because
  // take_op_budget() may drop our lock while it blocks.
  if (!op->ctx_budgeted || (ctx_budget && (*ctx_budget == -1))) {
    int op_budget = _take_op_budget(op);
    // take and pass out the budget for the first OP
    // in the context session
    if (ctx_budget && (*ctx_budget == -1)) {
      *ctx_budget = op_budget;
    }
  }

  C_CancelOp *cb = NULL;
  if (osd_timeout > 0) {
    cb = new C_CancelOp(this);
    op->ontimeout = cb;
  }

  ceph_tid_t tid = _op_submit(op, lc); // 真正处理发送请求的地方

  if (cb) {
    cb->set_tid(tid);
    Mutex::Locker l(timer_lock);
    timer.add_event_after(osd_timeout, op->ontimeout);
  }

  return tid;
}
```


在提交之前，会计算是否需要更新map，如果需要，会向monitor发送获取osdmap的消息, op会继续留在session中, 等待时机成熟时再发送。
这里有两个条件会重新发送session中滞留的op:

* `schedule_tick`定时扫描发送, 它设置了一个定时器，到时间后会被执行，执行完后重新设置新的定时器，所以会一直周期性地扫描

* `handle_osd_map`调用`scan_request`再进行发送出去, `handle_osd_map`会在收到消息`CEPH_MSG_OSD_MAP`后，由dispatch函数通过消息分类机制调用

```cpp
ceph_tid_t Objecter::_op_submit(Op *op, RWLock::Context& lc)
{
  assert(rwlock.is_locked());

  ldout(cct, 10) << __func__ << " op " << op << dendl;

  // pick target
  assert(op->session == NULL);
  OSDSession *s = NULL;

  // 计算是否需要更新map
  bool const check_for_latest_map = _calc_target(&op->target, &op->last_force_resend) == RECALC_OP_TARGET_POOL_DNE;

  // Try to get a session, including a retry if we need to take write lock
  int r = _get_session(op->target.osd, &s, lc); // 获取session， session中包含connection与OSD的连接信息
  if (r == -EAGAIN) {
    assert(s == NULL);
    lc.promote();
    r = _get_session(op->target.osd, &s, lc);
  }
  assert(r == 0);
  assert(s);  // may be homeless

  // We may need to take wlock if we will need to _set_op_map_check later.
  if (check_for_latest_map && !lc.is_wlocked()) {
    lc.promote();
  }

  _send_op_account(op); // 统计op信息

  // send?
  ldout(cct, 10) << "_op_submit oid " << op->target.base_oid
           << " " << op->target.base_oloc << " " << op->target.target_oloc
	   << " " << op->ops << " tid " << op->tid
           << " osd." << (!s->is_homeless() ? s->osd : -1)
           << dendl;

  assert(op->target.flags & (CEPH_OSD_FLAG_READ|CEPH_OSD_FLAG_WRITE));

  bool need_send = false;

  // 判断是否需要发送
  if ((op->target.flags & CEPH_OSD_FLAG_WRITE) &&
      osdmap->test_flag(CEPH_OSDMAP_PAUSEWR)) {
    ldout(cct, 10) << " paused modify " << op << " tid " << last_tid.read() << dendl;
    op->target.paused = true;
    _maybe_request_map();
  } else if ((op->target.flags & CEPH_OSD_FLAG_READ) &&
	     osdmap->test_flag(CEPH_OSDMAP_PAUSERD)) {
    ldout(cct, 10) << " paused read " << op << " tid " << last_tid.read() << dendl;
    op->target.paused = true;
    _maybe_request_map();
  } else if ((op->target.flags & CEPH_OSD_FLAG_WRITE) && _osdmap_full_flag()) {
    ldout(cct, 0) << " FULL, paused modify " << op << " tid " << last_tid.read() << dendl;
    op->target.paused = true;
    _maybe_request_map();
  } else if (!s->is_homeless()) {
    need_send = true;
  } else {
    _maybe_request_map();
  }

  MOSDOp *m = NULL;
  if (need_send) {
    m = _prepare_osd_op(op); // 如果需要发送，则创建一条包含op的消息
  }

  s->lock.get_write();
  if (op->tid == 0)
    op->tid = last_tid.inc(); // 获取唯一的tid
  _session_op_assign(s, op); // 这里会将op的信息记录到session里面，firefly版本是直接记录在objecter的map中，显然和特定的session关联起来更合理

  if (need_send) {
    _send_op(op, m); // 将消息发送出去
  }

  // Last chance to touch Op here, after giving up session lock it can be
  // freed at any time by response handler.
  ceph_tid_t tid = op->tid;
  if (check_for_latest_map) {
    _send_op_map_check(op); // 需要更新map，这里会向monitor发送获取osdmap的消息, op会留在session中, 以后会由schedule_tick或handle_osd_map调用scan_request再进行发送
  }
  op = NULL;

  s->lock.unlock();
  put_session(s);

  ldout(cct, 5) << num_unacked.read() << " unacked, " << num_uncommitted.read() << " uncommitted" << dendl;

  return tid;
}
```

记录op到session的map里，等收到回包的时候好处理：

```cpp
void Objecter::_session_op_assign(OSDSession *to, Op *op)
{
  assert(to->lock.is_locked());
  assert(op->session == NULL);
  assert(op->tid);

  get_session(to);
  op->session = to;
  to->ops[op->tid] = op; // 记录op信息，当OSD处理完后，会发送消息回来，回来的消息在handle_osd_op_reply函数中处理，这个tid就是找到对应op的关键

  if (to->is_homeless()) {
    num_homeless_ops.inc();
  }

  ldout(cct, 15) << __func__ << " " << to->osd << " " << op->tid << dendl;
}
```
