---
layout: post
title: "Cephfs Kernel Client Throughput Limitation"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# 问题描述

测试cephfs内核客户端的吞吐性能，direct写时单个客户端性能有上限，只能接近150 mb/s:

```text
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# dd if=/dev/zero of=test bs=1M count=1024 oflag=direct
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 7.63581 s, 141 MB/s
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#
```

查看网卡流量，并没有打满:

```text
root@n8-147-034:~# ifstat
eth0
KB/s in  KB/s out
704.64  144464.6
330.61  144193.2
550.67  143631.0
604.46  143381.1
```

查看集群负载也很低，osd磁盘很空闲，验证多台机器同时并发测试，总吞吐可以上去，怀疑单个客户端的上限有瓶颈。

# 源码分析

集群没有打满，网络也不是瓶颈，那么只能从内核客户端cephfs的写IO入手，寻找问题的根源。
cephfs内核客户端写IO的代码在文件fs/ceph/file.c:

```c
const struct file_operations ceph_file_fops = {
	.open = ceph_open,
	.release = ceph_release,
	.llseek = ceph_llseek,
	.read_iter = ceph_read_iter,
	.write_iter = ceph_write_iter, // 写文件的hook函数
	.mmap = ceph_mmap,
	.fsync = ceph_fsync,
	.lock = ceph_lock,
	.flock = ceph_flock,
	.splice_write = iter_file_splice_write,
	.unlocked_ioctl = ceph_ioctl,
	.compat_ioctl   = ceph_ioctl,
	.fallocate  = ceph_fallocate,
};

static ssize_t ceph_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    ......

	if (iocb->ki_flags & IOCB_DIRECT)
		written = ceph_direct_read_write(iocb, &data, snapc, // direct情况
				&prealloc_cf);
	......
}

static ssize_t
ceph_direct_read_write(struct kiocb *iocb, struct iov_iter *iter,
		struct ceph_snap_context *snapc,
		struct ceph_cap_flush **pcf)
{
	......

	while (iov_iter_count(iter) > 0) {
		......
		req = ceph_osdc_new_request(&fsc->client->osdc, &ci->i_layout, // 创建请求
				vino, pos, &size, 0,
				/*include a 'startsync' command*/
				write ? 2 : 1,
				write ? CEPH_OSD_OP_WRITE :
				CEPH_OSD_OP_READ,
				flags, snapc,
				ci->i_truncate_seq,
				ci->i_truncate_size,
				false);

		ret = ceph_osdc_start_request(req->r_osdc, req, false); // 开始请求

		if (!ret)
			ret = ceph_osdc_wait_request(&fsc->client->osdc, req); // 等待结束
		......
	}

	......
}
```

从代码实现看，主要流程是new\_request, start\_request, wait\_request三个步骤。
直觉告诉我这里的wait会被block住，跟踪一下这里的wait实现:

```c
// net/ceph/osd_client.c
int ceph_osdc_wait_request(struct ceph_osd_client *osdc,
		struct ceph_osd_request *req)
{
	return wait_request_timeout(req, 0); // 第二个参数是0

}

static int wait_request_timeout(struct ceph_osd_request *req,
		unsigned long timeout)
{
	......
	// 这个函数返回，要么是第一个参数的条件满足，要么是超时
	left = wait_for_completion_killable_timeout(&req->r_completion,
			ceph_timeout_jiffies(timeout));
	......
}
```

先看超时的时间，传入的是0，最终结果是LONG\_MAX，差不多就是一直wait:

```c
// include/linux/ceph/libceph.h
static inline unsigned long ceph_timeout_jiffies(unsigned long timeout)
{
	// 这里是gnu对c语言条件运算符的扩展，其实就是: timeout ? timeout : MAX_SCHEDULE_TIMEOUT;
	return timeout ?: MAX_SCHEDULE_TIMEOUT;
}

// include/linux/sched.h
// long的最大值
#define MAX_SCHEDULE_TIMEOUT    LONG_MAX
```

接下来看条件的满足:

```c
// kernel/sched/completion.c
long __sched
wait_for_completion_killable_timeout(struct completion *x,
		unsigned long timeout)
{
	return wait_for_common(x, timeout, TASK_KILLABLE);
}

static long __sched
wait_for_common(struct completion *x, long timeout, int state)
{
	// 注意第二个参数schedule_timeout
	return __wait_for_common(x, schedule_timeout, timeout, state);
}

static inline long __sched
__wait_for_common(struct completion *x,
		long (*action)(long), long timeout, int state)
{
	might_sleep();

	spin_lock_irq(&x->wait.lock);
	timeout = do_wait_for_common(x, action, timeout, state);
	spin_unlock_irq(&x->wait.lock);
	return timeout;
}

static inline long __sched
do_wait_for_common(struct completion *x,
		long (*action)(long), long timeout, int state)
{
	if (!x->done) {
		DECLARE_WAITQUEUE(wait, current);

		__add_wait_queue_tail_exclusive(&x->wait, &wait);
		do { // 循环，直到x->done完成，这个是条件的判断。而x就是发送io请求时，传递的结构体，完成后会更新这个结构体内容
			if (signal_pending_state(state, current)) {
				timeout = -ERESTARTSYS;
				break;
			}
			__set_current_state(state);
			spin_unlock_irq(&x->wait.lock);
			timeout = action(timeout); // action就是上面传入的schedule_timeout
			spin_lock_irq(&x->wait.lock);
		} while (!x->done && timeout);
		__remove_wait_queue(&x->wait, &wait);
		if (!x->done)
			return timeout;
	}
	x->done--;
	return timeout ?: 1;
}
```

从kernel的注释看，函数schedule\_timeout就是sleep直到timeout:
```
/**
 * schedule_timeout - sleep until timeout
 * @timeout: timeout value in jiffies
 *
 * Make the current task sleep until @timeout jiffies have
 * elapsed.
signed long __sched schedule_timeout(signed long timeout)
{
```

从源码分析看，已经比较明确，一次请求下发后，只有等请求完成了才会进行下一次请求，IO并不是并发的下发给后端的集群。

接下来的问题是，每次请求的size如何决定？
这个和文件的layout属性和当前写的位置相关，如果从文件offset 0开始写的话，以及采用默认属性，最大就是ceph object size的大小，即4MB。
ceph的layout解释可以参考[官方文档](http://docs.ceph.com/docs/master/architecture/#data-striping)。

```c
// net/ceph/osd_client.c
struct ceph_osd_request *ceph_osdc_new_request(struct ceph_osd_client *osdc,
		struct ceph_file_layout *layout,
		struct ceph_vino vino,
		u64 off, u64 *plen,
		unsigned int which, int num_ops,
		int opcode, int flags,
		struct ceph_snap_context *snapc,
		u32 truncate_seq,
		u64 truncate_size,
		bool use_mempool)
{
	......
	/* calculate max write size */
	r = calc_layout(layout, off, plen, &objnum, &objoff, &objlen);
	......
}

/*
 * calculate the mapping of a file extent onto an object, and fill out the
 * request accordingly.  shorten extent as necessary if it crosses an
 * object boundary.
 *
 * fill osd op in request message.
 */
static int calc_layout(struct ceph_file_layout *layout, u64 off, u64 *plen,
		u64 *objnum, u64 *objoff, u64 *objlen)
{
	......
	// 根据layout计算
	r = ceph_calc_file_object_mapping(layout, off, orig_len, objnum,
			objoff, objlen);
	return 0;
}
```

# 实验证明

### 调整文件属性

为了更明显的观察延时，我们将文件的属性调整一下，即从4m到64m:

```shell
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# touch foo
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# getfattr -n ceph.file.layout foo
# file: foo
ceph.file.layout="stripe_unit=4194304 stripe_count=1 object_size=4194304 pool=cephfs_data"

root@n8-147-034:/mnt/cephfs/jinwei_test/dd# setfattr -n ceph.file.layout.object_size -v 67108864 foo
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# setfattr -n ceph.file.layout.stripe_unit -v 67108864 foo
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# getfattr -n ceph.file.layout foo
# file: foo
ceph.file.layout="stripe_unit=67108864 stripe_count=1 object_size=67108864 pool=cephfs_data"
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#
```

### 获取文件inode

```shell
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#  stat -c %i foo | xargs printf '%x\n'
10000000388
```

### 文件对应的对象

```shell
# 文件没有内容，这时候是没有对象的
root@n10-075-086:~# rados -p cephfs_data ls | grep 10000000388

# 写入两个对象
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# dd if=/dev/zero of=foo bs=64M count=2 oflag=direct
2+0 records in
2+0 records out
134217728 bytes (134 MB) copied, 0.958122 s, 140 MB/s
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#

# 再次查看
root@n10-075-086:~# rados -p cephfs_data ls | grep 10000000388
10000000388.00000000
10000000388.00000001
root@n10-075-086:~#
```

查看两个对象对应的osd信息，分别对应osd 121和130:

```text
root@n10-075-086:~# ceph osd map cephfs_data 10000000388.00000000
osdmap e2231 pool 'cephfs_data' (2) object '10000000388.00000000' -> pg 2.201b2309 (2.309) -> up ([121,95,41], p121) acting ([121,95,41], p121)
root@n10-075-086:~# ceph osd map cephfs_data 10000000388.00000001
osdmap e2231 pool 'cephfs_data' (2) object '10000000388.00000001' -> pg 2.5052a9b5 (2.9b5) -> up ([130,77,32], p130) acting ([130,77,32], p130)
root@n10-075-086:~#
```

再次执行刚才的dd命令，并在两个primary osd(121, 130)上观察op的情况，并同时用ftrace，观察kernel客户端写的过程。

### osd机器OP请求

通过以下命令查看osd的op信息，ID为上面的121和130:

> ceph daemon osd.ID dump\_historic\_ops

```json
		{
            "description": "osd_op(client.717850.1:5196 2.201b2309 10000000388.00000000 [] snapc 1=[] RETRY=1 ondisk+retry+write+ordersnap+known_if_redirected e2236)",
            "initiated_at": "2017-10-13 16:04:19.049346",
            "age": 47.697939,
            "duration": 0.426153, // 总耗时大约426毫秒
            "type_data": [
                "commit sent; apply or cleanup",
                {
                    "client": "client.717850",
                    "tid": 5196
                },
                [
                    {
                        "time": "2017-10-13 16:04:19.049346",
                        "event": "initiated"
                    },
                    {
                        "time": "2017-10-13 16:04:19.121712", // 网络层读消息，大约70毫秒，一个object 64mb，万兆网卡差不多需要这个时间
                        "event": "queued_for_pg"
                    },
                    {
                        "time": "2017-10-13 16:04:19.121801",
                        "event": "reached_pg"
                    },
                    {
                        "time": "2017-10-13 16:04:19.121867",
                        "event": "started"
                    },
                    {
                        "time": "2017-10-13 16:04:19.125718",
                        "event": "waiting for subops from 41,95"
                    },
                    {
                        "time": "2017-10-13 16:04:19.164887",
                        "event": "commit_queued_for_journal_write"
                    },
                    {
                        "time": "2017-10-13 16:04:19.164955",
                        "event": "write_thread_in_journal_buffer"
                    },
                    {
                        "time": "2017-10-13 16:04:19.356237",
                        "event": "journaled_completion_queued"
                    },
                    {
                        "time": "2017-10-13 16:04:19.356259",
                        "event": "op_commit"
                    },
                    {
                        "time": "2017-10-13 16:04:19.394599",
                        "event": "op_applied"
                    },
                    {
                        "time": "2017-10-13 16:04:19.465524",
                        "event": "sub_op_commit_rec from 41"
                    },
                    {
                        "time": "2017-10-13 16:04:19.475450",
                        "event": "sub_op_commit_rec from 95"
                    },
                    {
                        "time": "2017-10-13 16:04:19.475473",
                        "event": "commit_sent"
                    },
                    {
                        "time": "2017-10-13 16:04:19.475499",  // op结束时间

                        "event": "done"
                    }
                ]
            ]
        }
```

上面是osd 121的信息，操作的对象是10000000388.00000000，op持续了426.153ms，主要耗费时间在网络读数据的延时和副本操作的延时。op开始时间**16:04:19.049346**，结束时间**16:04:19.475499**。

```json
		{
            "description": "osd_op(client.717850.1:5197 2.5052a9b5 10000000388.00000001 [] snapc 1=[] ondisk+write+ordersnap+known_if_redirected e2236)",
            "initiated_at": "2017-10-13 16:04:19.491627",
            "age": 73.131756,
            "duration": 0.439539,
            "type_data": [
                "commit sent; apply or cleanup",
                {
                    "client": "client.717850",
                    "tid": 5197
                },
                [
                    {
                        "time": "2017-10-13 16:04:19.491627", // op开始时间，注意是在上一个op完成之后
                        "event": "initiated"
                    },
                    {
                        "time": "2017-10-13 16:04:19.558547",
                        "event": "queued_for_pg"
                    },
                    {
                        "time": "2017-10-13 16:04:19.558630",
                        "event": "reached_pg"
                    },
                    {
                        "time": "2017-10-13 16:04:19.558777",
                        "event": "started"
                    },
                    {
                        "time": "2017-10-13 16:04:19.562634",
                        "event": "waiting for subops from 32,77"
                    },
                    {
                        "time": "2017-10-13 16:04:19.580423",
                        "event": "commit_queued_for_journal_write"
                    },
                    {
                        "time": "2017-10-13 16:04:19.580483",
                        "event": "write_thread_in_journal_buffer"
                    },
                    {
                        "time": "2017-10-13 16:04:19.783500",
                        "event": "journaled_completion_queued"
                    },
                    {
                        "time": "2017-10-13 16:04:19.783521",
                        "event": "op_commit"
                    },
                    {
                        "time": "2017-10-13 16:04:19.820925",
                        "event": "op_applied"
                    },
                    {
                        "time": "2017-10-13 16:04:19.917342",
                        "event": "sub_op_commit_rec from 77"
                    },
                    {
                        "time": "2017-10-13 16:04:19.931116",
                        "event": "sub_op_commit_rec from 32"
                    },
                    {
                        "time": "2017-10-13 16:04:19.931140",
                        "event": "commit_sent"
                    },
                    {
                        "time": "2017-10-13 16:04:19.931166", // op结束时间
                        "event": "done"
                    }
                ]
            ]
        }
```

这是osd 130的信息，操作的对象是10000000388.00000001，op持续了439.539ms。op开始时间**16:04:19.491627**，结束时间**16:04:19.931166**。

可以很清楚的看见，先写第一个对象，再写第二个对象，对象之间是没有并发写的，这区别于块存储，块存储的实现，至少librbd的实现，如果一次io对应多个object，多个请求是同时发出的，而不会等第一个对象完成了才下发第二个对象的IO，参见如下代码:

```c++
template <typename I>
void AbstractAioImageWrite<I>::send_object_requests(
    const ObjectExtents &object_extents, const ::SnapContext &snapc,
    AioObjectRequests *aio_object_requests) {
  I &image_ctx = this->m_image_ctx;
  CephContext *cct = image_ctx.cct;

  AioCompletion *aio_comp = this->m_aio_comp;
  for (ObjectExtents::const_iterator p = object_extents.begin();
       p != object_extents.end(); ++p) {

    C_AioRequest *req_comp = new C_AioRequest(aio_comp);
    AioObjectRequestHandle *request = create_object_request(*p, snapc,
                                                            req_comp);
    if (request != NULL) {
      if (aio_object_requests != NULL) {
        aio_object_requests->push_back(request);
      } else {
        request->send(); // 发送请求
      }
    }
  }
```

### 写文件的客户端ftrace信息

enable ftrace步骤:

```shell
# 修改tracer类型
root@n8-147-034:/sys/kernel/debug/tracing# echo function > current_tracer
root@n8-147-034:/sys/kernel/debug/tracing# cat current_tracer
function

# 添加trace的函数，这几个函数就是前面分析代码的函数
root@n8-147-034:/sys/kernel/debug/tracing# echo ceph_osdc_new_request >> set_ftrace_filter
root@n8-147-034:/sys/kernel/debug/tracing# echo ceph_osdc_start_request >> set_ftrace_filter
root@n8-147-034:/sys/kernel/debug/tracing# echo ceph_osdc_wait_request >> set_ftrace_filter

# 查看是否正确
root@n8-147-034:/sys/kernel/debug/tracing# cat set_ftrace_filter
ceph_osdc_new_request [libceph]
ceph_osdc_wait_request [libceph]
ceph_osdc_start_request [libceph]
root@n8-147-034:/sys/kernel/debug/tracing#
```

观察日志:

```shell
root@n8-147-034:/sys/kernel/debug/tracing# cat trace
# tracer: function
#
# entries-in-buffer/entries-written: 6/6   #P:48
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
<...>-118883 [005] .... 3978299.661306: ceph_osdc_new_request <-ceph_direct_read_write
<...>-118883 [005] .... 3978299.662699: ceph_osdc_start_request <-ceph_direct_read_write
<...>-118883 [005] .... 3978299.662806: ceph_osdc_wait_request <-ceph_direct_read_write

# 暂停了500ms

<...>-118883 [005] .... 3978300.175224: ceph_osdc_new_request <-ceph_direct_read_write
<...>-118883 [005] .... 3978300.176661: ceph_osdc_start_request <-ceph_direct_read_write
<...>-118883 [005] .... 3978300.176749: ceph_osdc_wait_request <-ceph_direct_read_write
root@n8-147-034:/sys/kernel/debug/tracing#
```

这里用了差不多500ms才开始下一个请求，而上面从osd端的分析看，第一个IO用了426ms才完成，osd完成IO后通知kernel客户端有网络延时，然后加上kernel调度的延时，差不多能够匹配。

# 结论

通过源码分析，然后分别从集群osd端和kernel客户端进行验证，direct的情况，cephfs性能确实有限制。但是，用户也不用过于担心性能跟不上，因为通常情况下，不会是direct写，kernel客户端有page cache，写会非常快，

```text
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# dd if=/dev/zero of=foo bs=64M count=1000
1000+0 records in
1000+0 records out
67108864000 bytes (67 GB) copied, 53.5032 s, 1.3 GB/s
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#
```

更贴近真实的使用场景，用户先写数据，最后调用一次sync操作:

```text
root@n8-147-034:/mnt/cephfs/jinwei_test/dd# dd if=/dev/zero of=foo bs=64M count=1000 conv=fdatasync
1000+0 records in
1000+0 records out
67108864000 bytes (67 GB) copied, 68.61 s, 978 MB/s
root@n8-147-034:/mnt/cephfs/jinwei_test/dd#
```

