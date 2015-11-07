---
layout: post
title: "Ceph Librbd Flatten"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

在ceph中，创建一个image(块设备)后，可以对image生成快照snapshot，然后通过快照clone出新的image。
由于clone采取的是copy-on-write机制，会非常快。但是，当clone链很深的时候，可能会通过几次回溯才能找到parent image的object，
这样会比较慢，虽然ceph也提供copy-on-read机制来解决这个问题，但是要想完全避免这个问题，可以对image进行flatten操作，
这样会让父子image 分离，最新feature还支持deep-flatten，即对快照进行flatten。

# Code Analysis

## invoke flatten

先从librbd Image 类提供的API入手:

```cpp
  int Image::flatten()
  {
    ImageCtx *ictx = (ImageCtx *)ctx;
    tracepoint(librbd, flatten_enter, ictx, ictx->name.c_str(), ictx->id.c_str());
    librbd::NoOpProgressContext prog_ctx;
    int r = librbd::flatten(ictx, prog_ctx); // 调用internal.cc
    tracepoint(librbd, flatten_exit, r);
    return r;
  }
```

接着进入internal.cc文件:

```cpp
  int flatten(ImageCtx *ictx, ProgressContext &prog_ctx)
  {
    CephContext *cct = ictx->cct;
    ldout(cct, 20) << "flatten" << dendl;

    int r = ictx_check(ictx); // 检测image
    if (r < 0) {
      return r;
    }

    if (ictx->read_only) { // 只读打开的时候，是不能做flatten的
      return -EROFS;
    }

    {
      RWLock::RLocker parent_locker(ictx->parent_lock);
      if (ictx->parent_md.spec.pool_id == -1) { // 没有parent, 不存在flatten语义
		lderr(cct) << "image has no parent" << dendl;
		return -EINVAL;
      }
    }

    uint64_t request_id = ictx->async_request_seq.inc(); // 原子操作，获取此次异步请求的id，通过<client_id, request_id>就可以决定整个集群的通知事件
	// 第一个bind操作对应的是librbd.cc/async_flatten --> local_request
	// 第二个bind操作对应的是ImageWatcher的成员函数notify_flatten --> remote_request
    r = invoke_async_request(ictx, "flatten", false,
                             boost::bind(&async_flatten, ictx, _1,
                                         boost::ref(prog_ctx)),
                             boost::bind(&ImageWatcher::notify_flatten,
                                         ictx->image_watcher, request_id,
                                         boost::ref(prog_ctx)));

    if (r < 0 && r != -EINVAL) {
      return r;
    }

    notify_change(ictx->md_ctx, ictx->header_oid, ictx);
    ldout(cct, 20) << "flatten finished" << dendl;
    return 0;
  }
```

很多管理类操作，比如flatten, resize, snap\_create等都是通过函数invoke\_async\_request发起异步操作去执行的。
这个函数的逻辑比较复杂，这里通过ImageWatcher类来统一负责Image 的flatten, resize, snap\_create等操作，
同时也通过它实现了分布式锁(exclusive lock)，主要是通过ceph的[watch/notify机制](http://blog.wjin.org/posts/ceph-watchnotify-mechanism.html#sthash.fU8N80yn.dpuf)来实现的。

客户端打开一个image，会注册一个watch，由ImageWatcher类管理，同时客户端在必要的时候会持有image的owner\_lock，
注意区分这个锁和exclusive lock。ImageWatcher类记录了分布式锁（exclusive lock）的状态，m\_lock\_owner\_state 表示当前客户端是否持有exclusive lock，
m\_owner\_client\_id表示持有exclusive lock的客户端ID，owner单词容易引起误导。

如果客户端拥有exclusive lock，则flatten等操作可以被它自己执行，会调用local\_request。相反，如果锁被其他客户端占用，
则需要将操作转发给占有锁的客户端，即执行remote\_request操作。

辅助函数is\_lock\_supported，它的作用是判断是否支持获取exclusive lock：

```cpp
bool ImageWatcher::is_lock_supported() const {
  RWLock::RLocker l(m_image_ctx.snap_lock);
  return is_lock_supported(m_image_ctx.snap_lock);
}

bool ImageWatcher::is_lock_supported(const RWLock &) const {
  assert(m_image_ctx.owner_lock.is_locked());
  assert(m_image_ctx.snap_lock.is_locked());
  // 为真的条件是：1）支持exclusive lock 2) 非只读 3）不是快照
  return ((m_image_ctx.features & RBD_FEATURE_EXCLUSIVE_LOCK) != 0 &&
	  !m_image_ctx.read_only && m_image_ctx.snap_id == CEPH_NOSNAP);
}
```

接着看发起异步操作的函数：

```cpp
int invoke_async_request(ImageCtx *ictx, const std::string& request_type,
                         bool permit_snapshot,
                         const boost::function<int(Context*)>& local_request,
                         const boost::function<int()>& remote_request) {
  int r;
  do {
    C_SaferCond ctx;
    {
      RWLock::RLocker l(ictx->owner_lock); // 以读的方式获取owner_lock
      {
        RWLock::RLocker snap_locker(ictx->snap_lock); // 以读的方式获取snap_lock
        if (ictx->read_only ||
            (!permit_snapshot && ictx->snap_id != CEPH_NOSNAP)) { // permit_snapshot主要是针对快照的deep-flatten
          return -EROFS;
        }
      }

      while (ictx->image_watcher->is_lock_supported()) { // 支持exclusive lock
        r = prepare_image_update(ictx); // 尝试获取exclusive_lock
        if (r < 0) {
          return -EROFS;
        } else if (ictx->image_watcher->is_lock_owner()) {
          break; // 如果自己持有独占锁，会跳出循环，执行local_request
        }

        r = remote_request(); // 锁被其他客户端占用，则将请求转发给持有锁的客户端进行此操作
        if (r != -ETIMEDOUT && r != -ERESTART) { // 转发失败了会继续while循环
          return r; // 请求成功发送出去，返回
        }
        ldout(ictx->cct, 5) << request_type << " timed out notifying lock owner"
                            << dendl;
      }

	  // 跳出循环，说明自己拥有独占锁，自己执行
      r = local_request(&ctx);
      if (r < 0) {
        return r;
      }
    }

    r = ctx.wait(); // 等待local_request的callback执行，即signal信号
    if (r == -ERESTART) {
      ldout(ictx->cct, 5) << request_type << " interrupted: restarting"
                          << dendl;
    }
  } while (r == -ERESTART); // 不成功，继续循环
  return r;
}

int prepare_image_update(ImageCtx *ictx) {
  assert(ictx->owner_lock.is_locked() && !ictx->owner_lock.is_wlocked());
  if (ictx->image_watcher == NULL) { // 没有watcher, 这种情况发生在只读打开
    return -EROFS;
  } else if (!ictx->image_watcher->is_lock_supported() || // 第一个条件不支持exclusive lock
             ictx->image_watcher->is_lock_owner()) { // 第二个条件是支持的条件下，自己已经拥有锁
    return 0;
  }

  // 
  // need to upgrade to a write lock
  int r = 0;
  bool acquired_lock = false;
  ictx->owner_lock.put_read();
  {
    RWLock::WLocker l(ictx->owner_lock);
    if (!ictx->image_watcher->is_lock_owner()) {
      r = ictx->image_watcher->try_lock(); // 尝试获取exlcusive lock
      acquired_lock = ictx->image_watcher->is_lock_owner(); // 判断是否获取成功
    }
  }
  if (acquired_lock) {
    // finish any AIO that was previously waiting on acquiring the
    // exclusive lock
    ictx->flush_async_operations();
  }
  ictx->owner_lock.get_read();
  return r;
}
```

## proxy flatten

接下来就是remote\_request或local\_request的两条不同路径的调用。实际上，当执行remote\_request这条路径的时候，
会将请求通过watch/notify机制，发送给其他持有exclusive lock的客户端，当远端收到通知后，最终也会调用local\_request对应的函数：async\_flatten。
执行成功后通过watch/notify机制将消息发送给请求的客户端。

看看remote\_request是怎样将通知发送出去的，它对应的操作是：`ictx->image_watcher->notify_flatten(request_id, prog_ctx)`

```cpp
int ImageWatcher::notify_flatten(uint64_t request_id, ProgressContext &prog_ctx) {
  assert(m_image_ctx.owner_lock.is_locked()); // 客户端打开一个image，自己可以持有image的owner_lock，不要和exclusive lock混淆
  assert(!is_lock_owner()); // 不持有exclusive lock，这和以前描述一致，notify操作是要通知其他客户端去帮忙执行

  AsyncRequestId async_request_id(get_client_id(), request_id); // 全局唯一ID

  bufferlist bl;
  ::encode(NotifyMessage(FlattenPayload(async_request_id)), bl); // 封装flatten payload消息

  return notify_async_request(async_request_id, bl, prog_ctx); // 通知
}

int ImageWatcher::notify_async_request(const AsyncRequestId &async_request_id,
				       bufferlist &in,
				       ProgressContext& prog_ctx) {
  assert(m_image_ctx.owner_lock.is_locked());

  ldout(m_image_ctx.cct, 10) << this << " async request: " << async_request_id
                             << dendl;

  C_SaferCond ctx;

  {
    RWLock::WLocker l(m_async_request_lock);
    m_async_requests[async_request_id] = AsyncRequest(&ctx, &prog_ctx); // 记录通知请求
  }

  BOOST_SCOPE_EXIT( (&ctx)(async_request_id)(&m_task_finisher) // 退出函数时，清理掉记录以及timeout事件
                    (&m_async_requests)(&m_async_request_lock) ) {
    m_task_finisher->cancel(Task(TASK_CODE_ASYNC_REQUEST, async_request_id)); // 清理timeout事件

    RWLock::WLocker l(m_async_request_lock);
    m_async_requests.erase(async_request_id); // 清理请求记录
  } BOOST_SCOPE_EXIT_END

  schedule_async_request_timed_out(async_request_id); // 设置一个timeout的event事件
  int r = notify_lock_owner(in); // 通知持有exclusive lock的客户端帮忙执行
  if (r < 0) { // 出错了就直接返回
    return r;
  }

  // 1) 等待一段时间，如果超时了，上面注册的time事件就会唤醒这个wait，然后返回-ERESTART
  // 2) 如果在超时时间范围内，收到notify complete的通知，表示操作完成，那么唤醒的callback会在处理notify消息的时候提前调用，返回处理的结果
  //    处理消息最终会过度到这个函数：void ImageWatcher::handle_payload(const AsyncCompletePayload &payload, bufferlist *out)
  // 3) 无论什么情况，当离开函数作用域后，boost宏范围内的代码，在离开函数作用域时，自动清理掉time事件和请求记录
  return ctx.wait();
}
```

先看timeout这条路径，当timeout发生时，会调用async\_request\_timed\_out函数，这样会返回-ERESTART，invoke会重新发起调用：

```cpp
void ImageWatcher::schedule_async_request_timed_out(const AsyncRequestId &id) {
  Context *ctx = new FunctionContext(boost::bind(
    &ImageWatcher::async_request_timed_out, this, id)); // timeout发生时调用的函数是：async_request_timed_out

  Task task(TASK_CODE_ASYNC_REQUEST, id);
  m_task_finisher->cancel(task);  // 取消之前的同样的任务

  md_config_t *conf = m_image_ctx.cct->_conf;
  m_task_finisher->add_event_after(task, conf->rbd_request_timed_out_seconds, // 增加一个timeout的任务
                                   ctx);
}

void ImageWatcher::async_request_timed_out(const AsyncRequestId &id) {
  RWLock::RLocker l(m_async_request_lock);
  std::map<AsyncRequestId, AsyncRequest>::iterator it =
    m_async_requests.find(id);
  if (it != m_async_requests.end()) {
    ldout(m_image_ctx.cct, 10) << this << " request timed-out: " << id << dendl;
    it->second.first->complete(-ERESTART); // 调用callback, 实际上是唤醒notify_async_request函数中的ctx.wait()
  }
}
```

另外一条正常工作的路径，持有exclusive lock的客户端在处理完后，会发送async complete的消息，发出请求的客户端会接收这个消息，
然后调用ImageWatcher::handle_notify函数，可以参考另一篇文章watch/notify的分析，然后通过payload类型，调用:
> ImageWatcher::handle_payload(const AsyncCompletePayload &payload, bufferlist *out)

这个函数就会和上面timeout一样，唤醒wait操作，只是返回值不同：

```cpp
void ImageWatcher::handle_payload(const AsyncCompletePayload &payload,
                                  bufferlist *out) {
  RWLock::RLocker l(m_async_request_lock);
  std::map<AsyncRequestId, AsyncRequest>::iterator req_it =
    m_async_requests.find(payload.async_request_id);
  if (req_it != m_async_requests.end()) {
    ldout(m_image_ctx.cct, 10) << this << " request finished: "
                               << payload.async_request_id << "="
			       << payload.result << dendl;
    req_it->second.first->complete(payload.result); // 唤醒操作
  }
}
```

唤醒操作分析完后，回到前面的发送请求的函数notify\_async\_request，调用了notify\_lock\_owner 函数发送请求：

```cpp
int ImageWatcher::notify_lock_owner(bufferlist &bl) {
  assert(m_image_ctx.owner_lock.is_locked());

  // since we need to ack our own notifications, release the owner lock just in
  // case another notification occurs before this one and it requires the lock
  bufferlist response_bl;
  m_image_ctx.owner_lock.put_read();
  int r = m_image_ctx.md_ctx.notify2(m_image_ctx.header_oid, bl, NOTIFY_TIMEOUT, // 将请求消息通知给持有锁的客户端，并且会等待执行结果
				     &response_bl);
  m_image_ctx.owner_lock.get_read();
  if (r < 0 && r != -ETIMEDOUT) {
    lderr(m_image_ctx.cct) << this << " lock owner notification failed: "
			   << cpp_strerror(r) << dendl;
    return r;
  }

  // 对notify结果response_bl的处理
  typedef std::map<std::pair<uint64_t, uint64_t>, bufferlist> responses_t;
  responses_t responses;
  if (response_bl.length() > 0) {
    try {
      bufferlist::iterator iter = response_bl.begin();
      ::decode(responses, iter);
    } catch (const buffer::error &err) {
      return -EINVAL;
    }
  }

  bufferlist response;
  bool lock_owner_responded = false;
  for (responses_t::iterator i = responses.begin(); i != responses.end(); ++i) {
    if (i->second.length() > 0) {
      if (lock_owner_responded) { // 只会有一个消息长度不为0，这个消息是当前持有exclusive lock的客户端的信息
		return -EIO;
      }
      lock_owner_responded = true;
      response.claim(i->second);
    }
  }

  if (!lock_owner_responded) {
    return -ETIMEDOUT;
  }

  try {
    bufferlist::iterator iter = response.begin();

    ResponseMessage response_message;
    ::decode(response_message, iter); // 解析结果

    r = response_message.result;
  } catch (const buffer::error &err) {
    r = -EINVAL;
  }
  return r;
}
```

持有锁的客户端在收到flatten的消息后，最终会执行如下函数：

```cpp
void ImageWatcher::handle_payload(const FlattenPayload &payload, // flatten 的payload
				  bufferlist *out) {
  RWLock::RLocker l(m_image_ctx.owner_lock);
  if (m_lock_owner_state == LOCK_OWNER_STATE_LOCKED) {
    int r = 0;
    bool new_request = false;
    // 这种情况表示自己刚开始没有锁，把消息发送出去后，自己又拿到了exclusive lock，然后又收到了自己发出去的notify消息
	// 注意watch/notify相关代码会在不同的客户端都会执行，时刻注意执行这段代码时客户端的身份
    if (payload.async_request_id.client_id == get_client_id()) {
      r = -ERESTART;
    } else {
      RWLock::WLocker l(m_async_request_lock);
      if (m_async_pending.count(payload.async_request_id) == 0) {
		m_async_pending.insert(payload.async_request_id);
		new_request = true;
      }
    }

    if (new_request) { // 新的请求
      RemoteProgressContext *prog_ctx =
		new RemoteProgressContext(*this, payload.async_request_id);
      RemoteContext *ctx = new RemoteContext(*this, payload.async_request_id, // RemoteContext最终会发送notify complete的消息出来
					     prog_ctx);

      ldout(m_image_ctx.cct, 10) << this << " remote flatten request: "
				 << payload.async_request_id << dendl;
      r = librbd::async_flatten(&m_image_ctx, ctx, *prog_ctx); // 调用async_flatten处理
      if (r < 0) {
		delete ctx;
		RWLock::WLocker l(m_async_request_lock);
		m_async_pending.erase(payload.async_request_id);
      }
    }

    ::encode(ResponseMessage(r), *out);
  }
}
```

# do flatten

通过分析，无论local\_request还是remote\_request，都会调用下面这个函数进行实际的操作：

```cpp
  int async_flatten(ImageCtx *ictx, Context *ctx, ProgressContext &prog_ctx)
  {
    assert(ictx->owner_lock.is_locked());
    assert(!ictx->image_watcher->is_lock_supported() || // 不支持exclusive lock
	   ictx->image_watcher->is_lock_owner()); // 自己已经获得exclusive lock，只有持有exclusive lock的客户端才能干活

    CephContext *cct = ictx->cct;
    ldout(cct, 20) << "flatten" << dendl;

    int r;
    // ictx_check also updates parent data
    if ((r = ictx_check(ictx, true)) < 0) {
      lderr(cct) << "ictx_check failed" << dendl;
      return r;
    }

    uint64_t object_size;
    uint64_t overlap;
    uint64_t overlap_objects;
    ::SnapContext snapc;

    {
      RWLock::RLocker l(ictx->snap_lock);
      RWLock::RLocker l2(ictx->parent_lock);

      if (ictx->read_only || ictx->snap_id != CEPH_NOSNAP) {
        return -EROFS;
      }

      // can't flatten a non-clone
      if (ictx->parent_md.spec.pool_id == -1) {
		return -EINVAL;
      }
      if (ictx->snap_id != CEPH_NOSNAP || ictx->read_only) {
		return -EROFS;
      }

      snapc = ictx->snapc;
      assert(ictx->parent != NULL);
      r = ictx->get_parent_overlap(CEPH_NOSNAP, &overlap);
      assert(r == 0);
      assert(overlap <= ictx->size);

      object_size = ictx->get_object_size();
      overlap_objects = Striper::get_num_objects(ictx->layout, overlap);
    }

    AsyncFlattenRequest *req =
      new AsyncFlattenRequest(*ictx, ctx, object_size, overlap_objects, // 这个类就是真正干活的类，负责flatten的具体工作
			      snapc, prog_ctx);
    req->send(); // 执行请求
    return 0;
  }
```

接下来就是怎样执行这个请求的过程，其中涉及到AsyncRequest框架，很多maintenace操作（flatten, trim, resize等)继承此类，
还有个类AsyncObjectThrottle是限制操作并发数的，用来限流：

```cpp
void AsyncFlattenRequest::send() {
  assert(m_image_ctx.owner_lock.is_locked());
  CephContext *cct = m_image_ctx.cct;

  m_state = STATE_FLATTEN_OBJECTS;

  AsyncObjectThrottle::ContextFactory context_factory( // 定义一个factory函数对象，调用它的时候，实际上new了一个对象：AsyncFlattenObjectContext
    boost::lambda::bind(boost::lambda::new_ptr<AsyncFlattenObjectContext>(),
      boost::lambda::_1, &m_image_ctx, m_object_size, m_snapc,
      boost::lambda::_2));

  // 利用AsyncObjectThrottle对对象的操作进行限流
  // this 为本次请求的类型，这里是AsyncFlattenRequest派生类
  // context_factory为生产context的工厂对象，会生产特定于flatten的对象出来: AsyncFlattenObjectContext
  // create_callback_context() 这个参数也生成了一个函数对象，绑定到函数：AsyncRequest::complete，操作完成时的回调
  AsyncObjectThrottle *throttle = new AsyncObjectThrottle(
    this, m_image_ctx, context_factory, create_callback_context(), m_prog_ctx, 
    0, m_overlap_objects);

  throttle->start_ops(cct->_conf->rbd_concurrent_management_ops); // 开始操作, 参数为操作的对象的并发数
}
```

操作是以object为粒度的，所以可以并发执行，并发数可以配置。

总结一下大致流程：start\_ops一开始，并发执行了n个op操作，当一个op完成后，会回调到finish\_op函数，
然后这个函数继续启动下一个op执行(如果还有op没执行), m\_current\_ops初始化为零，start\_next\_op成功发送一个op后计数+1，
当发送的op执行完后，回调finish\_op，计数会-1，当计数再一次为零时就表示所有op都已经执行完毕（发送出去，执行，并且回调成功），这时候会继续回调上一层的callback。
相当于同时开了n条流水线在执行，某条流水线完工后，继续拿新的op执行，直到所有op（end object限定）都执行完毕。

```cpp
void AsyncObjectThrottle::start_ops(uint64_t max_concurrent) {
  assert(m_image_ctx.owner_lock.is_locked());
  bool complete;
  {
    Mutex::Locker l(m_lock);
    for (uint64_t i = 0; i < max_concurrent; ++i) { // 并发数
      start_next_op(); // 执行一个op
      if (m_ret < 0 && m_current_ops == 0) {
	break;
      }
    }
    complete = (m_current_ops == 0);
  }
  if (complete) {
    m_ctx->complete(m_ret);
    delete this;
  }
}

void AsyncObjectThrottle::start_next_op() {
  bool done = false;
  while (!done) {
    if (m_async_request->is_canceled() && m_ret == 0) {
      // allow in-flight ops to complete, but don't start new ops
      m_ret = -ERESTART;
      return;
    } else if (m_ret != 0 || m_object_no >= m_end_object_no) { // end object为结束条件
      return;
    }

    uint64_t ono = m_object_no++; // 当前执行的是哪个对象

	// 这里很关键，通过context工厂，生产一个对应于特定请求的context去执行
	// 刚才传入的工厂对象生产的是：AsyncFlattenObjectContext，继承于类C_AsyncObjectThrottle
	// 所以这里是基类指针，这个框架可以被trim之类的操作复用，只需要传入不同的工厂对象给AsyncObjectThrottle即可
    C_AsyncObjectThrottle *ctx = m_context_factory(*this, ono);

    int r = ctx->send(); // 真正执行context

    if (r < 0) {
      m_ret = r;
      delete ctx;
      return;
    } else if (r > 0) {
      // op completed immediately
      delete ctx;
    } else { // 成功发送了一个请求，等待请求完成后的回调
      ++m_current_ops; // 等待计数加1
      done = true;
    }
    m_prog_ctx.update_progress(ono, m_end_object_no);
  }
}
```

看看ctx的send函数干了什么，对一个imaget执行flatten，实际上就是对object的每个对象，通过一个写操作去触发copy-on-write，
只不过写操作的buffer内容为空：

```cpp
class AsyncFlattenObjectContext : public C_AsyncObjectThrottle {
public:
  AsyncFlattenObjectContext(AsyncObjectThrottle &throttle, ImageCtx *image_ctx,
                            uint64_t object_size, ::SnapContext snapc,
                            uint64_t object_no)
    : C_AsyncObjectThrottle(throttle, *image_ctx), m_object_size(object_size),
      m_snapc(snapc), m_object_no(object_no)
  {
  }

  virtual int send() {
    assert(m_image_ctx.owner_lock.is_locked());
    CephContext *cct = m_image_ctx.cct;

    if (m_image_ctx.image_watcher->is_lock_supported() &&
        !m_image_ctx.image_watcher->is_lock_owner()) {
      ldout(cct, 1) << "lost exclusive lock during flatten" << dendl;
      return -ERESTART;
    }

    bufferlist bl; // 不包含任何数据
    string oid = m_image_ctx.get_object_name(m_object_no);
    AioWrite *req = new AioWrite(&m_image_ctx, oid, m_object_no, 0, bl, m_snapc, // 写IO请求
                                 this); // this指针也很关键，这个是一个callback，当io完成后会回调
    if (!req->has_parent()) {
      // stop early if the parent went away - it just means
      // another flatten finished first or the image was resized
      delete req;
      return 1;
    }

    req->send(); // 发送IO请求
    return 0;
  }

private:
  uint64_t m_object_size;
  ::SnapContext m_snapc;
  uint64_t m_object_no;
};
```

IO请求发送完后，会回调callback，即this指针，注意继承关系，最后会过度下面这个函数：

```cpp
void C_AsyncObjectThrottle::finish(int r) {
  RWLock::RLocker l(m_image_ctx.owner_lock);
  m_finisher.finish_op(r); // m_finisher就是AsyncObjectThrottle
}

// 所以最终回调到了这个函数
void AsyncObjectThrottle::finish_op(int r) {
  assert(m_image_ctx.owner_lock.is_locked());
  bool complete;
  {
    Mutex::Locker l(m_lock);
    --m_current_ops; // 完成了一个，计数减一
    if (r < 0 && r != -ENOENT && m_ret == 0) {
      m_ret = r;
    }

    start_next_op(); // 继续下一个op
    complete = (m_current_ops == 0); // 最终所有回调都完成了
  }
  if (complete) {
    m_ctx->complete(m_ret); // 所有OP都完成了才会继续往上层回调，m_ctx就是AsyncRequest::complete
    delete this;
  }
}
```

当所有op完成后，就会回调AsyncRequest::complete，这里就会判断此次请求是否完成了，逻辑在should\_complete函数中控制，
当op执行完成后，需要更新header object等元数据，对于deep flatte情况，逻辑也需要在这里完成:

```cpp
  void complete(int r) {
    if (m_canceled && safely_cancel(r)) {
      m_on_finish->complete(-ERESTART);
      delete this;
    } else if (should_complete(r)) { // should_complete这里会更新一些元数据信息，简单的状态机
      m_on_finish->complete(filter_return_code(r)); // 全部完成，回调async_flatten中的callback
      delete this;
    }
  }
```
