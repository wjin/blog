---
layout: post
title: "Ceph Librbd Introduction"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}


# Introduction

librbd 是ceph对外提供的块存储接口的抽象，可以供qemu虚拟机, fio测试程序的rbd engine等程序使用。
它提供c和c++两种接口，对于c++， 最主要的两个类就是`RBD` 和 `Image`。
RBD 主要负责创建，删除，克隆镜像等操作, 而Image 类负责镜像的读写等操作。

# HeadFiles

作为一个通用的库，对c/c++来说，得提供一个头文件，即所谓的API。
这个头文件是 **src/include/rbd/librbd.h** 和 **src/include/rbd/librbd.hpp**

```cpp

namespace librbd { // 库在librbd名字空间中

  using librados::IoCtx; // librados 库对外提供的接口

  class Image;
  typedef void *image_ctx_t;
  typedef void *completion_t;
  typedef void (*callback_t)(completion_t cb, void *arg); // 异步操作回调接口

  ...

class CEPH_RBD_API RBD
{
public:
  RBD();
  ~RBD();

  // This must be dynamically allocated with new, and
  // must be released with release().
  // Do not use delete.
  struct AioCompletion {
    void *pc;
    AioCompletion(void *cb_arg, callback_t complete_cb);
    bool is_complete();
    int wait_for_complete();
    ssize_t get_return_value();
    void release();
  };

  // 接下来一些API: open/create/clone/remove/rename 等

private:
  /* We don't allow assignment or copying */
  RBD(const RBD& rhs);
  const RBD& operator=(const RBD& rhs);
};

class CEPH_RBD_API Image
{
public:
  Image();
  ~Image();

  // 镜像的读写，flatten，trim等操作

private:
  friend class RBD;

  Image(const Image& rhs);
  const Image& operator=(const Image& rhs);

  image_ctx_t ctx; // viod*, 实际指向具体实现的类
};

}

```

ceph中几乎全是异步操作，异步操作在调用的时候会提供回调函数，librbd的异步接口是`librbd::RBD::AioCompletion`类实现的,
librbd的用户，如qemu里的驱动，在使用librbd时会先new一个`librbd::RBD::AioCompletion`对象出来，填入需要回调的callback和参数。
librbd::RBD::AioCompletion 的构造函数会创建一个`librbd::AioCompletion`(注意这里没有RBD作用域）, 赋值给`void *pc`。
librbd::RBD::AioCompletion里的其它函数foo就是对pc->foo的调用。这样只暴露了头文件，对实现细节进行了隐藏。

同理, Image类的实现，也只有一个成员`void *ctx`，初始化后，会真正指向ImageCtx, 隐藏细节。对image的操作foo最后都是对
ctx->foo的操作。

```cpp
RBD::AioCompletion::AioCompletion(void *cb_arg, callback_t complete_cb)
{
   librbd::AioCompletion *c = librbd::aio_create_completion(cb_arg,
                                       complete_cb);
   pc = (void *)c;
   c->rbd_comp = this;
}
```

# Implementation

库的实现代码在**src/librbd/**目录。对应头文件的cpp文件是**librbd.cc**。这个文件就是一些很简单的wrapper封装，
真正干活的是在**internal.cc文件**。

上文中的pc指针，对应到文件是**AioCompletion.h/AioCompletion.cc**。
而ctx指针，对应到**ImageCtx.h/ImageCtx.cc**文件。


`ctx怎么初始化的？` Image类的构造函数并没有提供此参数，并且也没有set_ctx之类的public成员函数。这里只有一个`friend class RBD`，
很明显，只有RBD类可以初始化ctx成员。

实际上，在使用librbd的过程中，是先通过RBD类，去create一个块设备，以后使用的时候，也是通过RBD类去open此前创建的设备。
在open的过程中，就会初始化ctx。


```cpp
  int RBD::open(IoCtx& io_ctx, Image& image, const char *name,
		const char *snap_name)
  {
    ImageCtx *ictx = new ImageCtx(name, "", snap_name, io_ctx, false);
    tracepoint(librbd, open_image_enter, ictx, ictx->name.c_str(), ictx->id.c_str(), ictx->snap_name.c_str(), ictx->read_only);

    int r = librbd::open_image(ictx);
    if (r < 0) {
      tracepoint(librbd, open_image_exit, r);
      return r;
    }

    image.ctx = (image_ctx_t) ictx; // 初始化ctx
    tracepoint(librbd, open_image_exit, 0);
    return 0;
  }


  // 析构过程
  Image::~Image()
  {
    if (ctx) {
      ImageCtx *ictx = (ImageCtx *)ctx;
      tracepoint(librbd, close_image_enter, ictx, ictx->name.c_str(), ictx->id.c_str());
      close_image(ictx);
      tracepoint(librbd, close_image_exit);
    }
  }

  void close_image(ImageCtx *ictx)
  {
	...

    delete ictx; // 释放ctx
  }
```

对于具体的IO，则是对应到文件**AioRequest.h/AioRequest.cc**，最终会通过`send`函数发送出去，留着以后读写流程详细分析。

对于一些比较耗时的管理操作，比如flatten, trim, resize等，hammer版本的代码新加了一些异步实现，对应的文件**Async...**之类的。

另外，对异步操作，也增加了提交请求的workqueue，然后由线程专门去提交, 避免了在特定条件下被block住。

# Python Demo

一个简单的python例子, 来源[ceph官方文档](http://docs.ceph.com/docs/v0.80.5/rbd/librbdpy/)。
主要就是四个对象：librados层Rados和IoCtx，librbd层就是RBD和Image。C++代码类似, 可以参考rbd命令的实现，文件是**src/rbd.cc**。

```python
	# 创建rados对象实例, 参数是集群的配置文件
	cluster = rados.Rados(conffile='my_ceph.conf')

	# 连接到rados集群
	cluster.connect()

	# 获取ioctx句柄用于操作某个pool
	ioctx = cluster.open_ioctx('mypool')

	# 创建 RBD 对象
	rbd_inst = rbd.RBD()

	# 创建一个块设备，注意和ioctx关联起来的，指明在哪个pool创建
	size = 4 * 1024**3 # 4 GiB
	rbd_inst.create(ioctx, 'myimage', size)

	# 创建Image的实例用于读写
	image = rbd.Image(ioctx, 'myimage')

	# 写数据
	data = 'foo' * 200
	image.write(data, 0)

	# 关闭
	image.close()
	ioctx.close()
	cluster.shutdown()
```
