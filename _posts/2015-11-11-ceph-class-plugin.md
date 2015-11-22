---
layout: post
title: "Ceph Class Plugin"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

ceph提供了librados API来访问后端rados存储集群，但是这些API也仅仅是对对象的简单操作（增删改查等），
增加新的API会导致librados变得越来越复杂，而且也根本不可能应对特定场景下的特殊需求。
所以ceph提供了可动态加载插件(ceph中叫class，不是c++中类的概念，实际上就是动态链接库)的方式，不同应用可以根据自己的需求定制特定的插件，
rados后端存储集群会根据用户请求，让OSD进程主动加载插件(dlopen)，执行用户的特殊需求。

# Implementation

## manage plugin 

插件就是动态链接库，它的管理，对于c/c++，肯定都是利用dlopen相关函数，为了统一管理加载的插件，OSD进程借助于类src/osd/ClassHandler来管理，
其中有两个类中类（ClassData/ClassMethod)，前者描述一个插件，后者描述插件包含的方法(函数)。

ClassHandler中有一个map来记录osd加载的所有插件，即 <插件名，描述插件的句柄(ClassData)>, 
ClassData中有一个map记录此插件提供的所有方法，即 <方法名，描述方法的句柄(ClassMethod)>, 实现的头文件比较简单:

```cpp
class ClassHandler
{
public:
  CephContext *cct;

  struct ClassData;

  struct ClassMethod {
    struct ClassHandler::ClassData *cls;
    string name; // 名称
    int flags;
    cls_method_call_t func; // c函数
    cls_method_cxx_call_t cxx_func; // c++版本函数

    int exec(cls_method_context_t ctx, bufferlist& indata, bufferlist& outdata); // 发起函数调用，实际上就是调用前面的func或cxx_func 
    void unregister();

    int get_flags() {
      Mutex::Locker l(cls->handler->mutex);
      return flags;
    }

    ClassMethod() : cls(0), flags(0), func(0), cxx_func(0) {}
  };

  struct ClassData {
    enum Status { // 依赖插件的状态
      CLASS_UNKNOWN,
      CLASS_MISSING,         // missing
      CLASS_MISSING_DEPS,    // missing dependencies
      CLASS_INITIALIZING,    // calling init() right now
      CLASS_OPEN,            // initialized, usable
    } status;

    string name; // 插件名称
    ClassHandler *handler;
    void *handle;

    map<string, ClassMethod> methods_map; // <插件的函数名，描述插件函数的句柄>, 插件注册的方法都在这个map里

    set<ClassData *> dependencies;         // 插件依赖的其他插件(so)
    set<ClassData *> missing_dependencies;

    ClassData() : status(CLASS_UNKNOWN), 
		  handler(NULL),
		  handle(NULL) {}
    ~ClassData() { }

    ClassMethod *register_method(const char *mname, int flags, cls_method_call_t func); // 注册一个插件的方法
    ClassMethod *register_cxx_method(const char *mname, int flags, cls_method_cxx_call_t func); // cxx表示c++版本的函数
    void unregister_method(ClassMethod *method); // 清除方法
  };

private:
  Mutex mutex;
  map<string, ClassData> classes; // <插件的名字，描述插件的句柄>, 所有的插件都会记录在这个map里

  ClassData *_get_class(const string& cname);
  int _load_class(ClassData *cls); // 加载so

public:
  ClassHandler(CephContext *cct_) : cct(cct_), mutex("ClassHandler") {}
  
  int open_all_classes();

  int open_class(const string& cname, ClassData **pcls); // 调用_load_class, 然后dlopen加载so
  
  ClassData *register_class(const char *cname); // 注册插件
  void unregister_class(ClassData *cls); // 清除插件

  void shutdown();
};
```

register函数只是注册一个插件，占用map一项，实际并没有去dlopen so，unregister就更简单了，什么都没干：

```cpp
ClassHandler::ClassData *ClassHandler::register_class(const char *cname)
{
  assert(mutex.is_locked());

  ClassData *cls = _get_class(cname);
  dout(10) << "register_class " << cname << " status " << cls->status << dendl;

  if (cls->status != ClassData::CLASS_INITIALIZING) {
    dout(0) << "class " << cname << " isn't loaded; is the class registering under the wrong name?" << dendl;
    return NULL;
  }
  return cls;
}

ClassHandler::ClassData *ClassHandler::_get_class(const string& cname)
{
  ClassData *cls;
  map<string, ClassData>::iterator iter = classes.find(cname);

  if (iter != classes.end()) {
    cls = &iter->second;
  } else {
    cls = &classes[cname]; // 记录注册的插件
    dout(10) << "_get_class adding new class name " << cname << " " << cls << dendl;
    cls->name = cname;
    cls->handler = this;
  }
  return cls;
}

void ClassHandler::unregister_class(ClassHandler::ClassData *cls)
{
  /* FIXME: do we really need this one? */
}
```

在以后需要调用插件的时候，可以通过函数ClassHandler::open\_class去dlopen插件so及其所依赖的其他插件，并且调用插件的入口函数\_\_cls\_init对插件进行初始化工作：

```cpp
int ClassHandler::open_class(const string& cname, ClassData **pcls)
{
  Mutex::Locker lock(mutex);
  ClassData *cls = _get_class(cname);
  if (cls->status != ClassData::CLASS_OPEN) {
    int r = _load_class(cls);
    if (r)
      return r;
  }
  *pcls = cls;
  return 0;
}

int ClassHandler::_load_class(ClassData *cls)
{
  // already open
  if (cls->status == ClassData::CLASS_OPEN)
    return 0;

  if (cls->status == ClassData::CLASS_UNKNOWN ||
      cls->status == ClassData::CLASS_MISSING) {

	......

    cls->handle = dlopen(fname, RTLD_NOW); // dlopen 插件

	......
  }

  // 递归调用_load_class解决依赖库
  ......

  // initialize
  void (*cls_init)() = (void (*)())dlsym(cls->handle, "__cls_init"); // 插件的入口，很关键，就是分析symbol，找到__cls_init，每个插件都有这个函数
  if (cls_init) {
    cls->status = ClassData::CLASS_INITIALIZING;
    cls_init(); // 初始化插件，调用插件入口函数__cls_init
  }

  ......
}
```

在osd进程启动的过程中，会new一个ClassHandler这样的对象，为以后插件的动态加载做好准备：

```cpp
int OSD::init()
{
  ......

  class_handler = new ClassHandler(cct);
  cls_initialize(class_handler); // 这里很关键

  ......
}
```

osd自己new了一个ClassHandler，宣称自己有了管理插件的能力，但是这个能力得暴露出来吧？
其实这是通过cls\_initialize函数实现的，源码在src/objclass/Class\_api.cc:

```cpp
static ClassHandler *ch; // 静态全局变量

void cls_initialize(ClassHandler *h)
{
  ch = h; // 全局变量记录ClassHandler 对象的地址，以后要用就找它了
}
```

## write plugin

到此，似乎明白了osd进程通过ClassHandler拥有了加载插件的功能，但是插件怎么写？什么时候使用插件？

ceph封装了一些简单的接口，在src/objclass/目录下。一部分是插件版本管理以及初始化的函数接口，
另外一部分就是访问对象的一些API的包装，编写插件时可以方便的调用:

### 

```cpp

// 插件的版本控制
#define CLS_VER(maj,min) \
int __cls_ver__## maj ## _ ##min = 0; \
int __cls_ver_maj = maj; \
int __cls_ver_min = min;

// 插件的名称
#define CLS_NAME(name) \
int __cls_name__## name = 0; \
const char *__cls_name = #name;

// 方法特征，读/写
#define CLS_METHOD_RD		0x1
#define CLS_METHOD_WR		0x2
#define CLS_METHOD_PUBLIC	0x4

void __cls_init(); // 入口函数，写插件时必须实现的函数，并且在此函数中注册以后需要使用的方法

typedef void *cls_handle_t;
typedef void *cls_method_handle_t;
typedef void *cls_method_context_t;
typedef int (*cls_method_call_t)(cls_method_context_t ctx,
				 char *indata, int datalen,
				 char **outdata, int *outdatalen);
typedef struct {
	const char *name;
	const char *ver;
} cls_deps_t;

```

编写插件需要的基本函数，都是通过静态全局变量ch调用了osd的ClassHandler：

```cpp
// 注册插件自己，名字全局唯一
int cls_register(const char *name, cls_handle_t *handle)
{
  ClassHandler::ClassData *cls = ch->register_class(name); // 占用ClassHandler的map的一项
  *handle = (cls_handle_t)cls;
  return (cls != NULL);
}

int cls_unregister(cls_handle_t handle)
{
  ClassHandler::ClassData *cls = (ClassHandler::ClassData *)handle;
  ch->unregister_class(cls);
  return 1;
}

// 注册插件的方法, c函数
int cls_register_method(cls_handle_t hclass, const char *method,
                        int flags,
                        cls_method_call_t class_call, cls_method_handle_t *handle)
{
  if (!(flags & (CLS_METHOD_RD | CLS_METHOD_WR)))
    return -EINVAL;
  ClassHandler::ClassData *cls = (ClassHandler::ClassData *)hclass;
  cls_method_handle_t hmethod =(cls_method_handle_t)cls->register_method(method, flags, class_call); // 占用ClassData的map一项
  if (handle)
    *handle = hmethod;
  return (hmethod != NULL);
}

// 注册插件的方法，c++函数
int cls_register_cxx_method(cls_handle_t hclass, const char *method,
                            int flags,
			    cls_method_cxx_call_t class_call, cls_method_handle_t *handle)
{
  ClassHandler::ClassData *cls = (ClassHandler::ClassData *)hclass;
  cls_method_handle_t hmethod = (cls_method_handle_t)cls->register_cxx_method(method, flags, class_call);
  if (handle)
    *handle = hmethod;
  return (hmethod != NULL);
}

int cls_unregister_method(cls_method_handle_t handle)
{
  ClassHandler::ClassMethod *method = (ClassHandler::ClassMethod *)handle;
  method->unregister();
  return 1;
}
```

## plugin demo

主要的函数明白后，编写插件就是很easy的一件事情，看一个hello world的例子：

```cpp
CLS_VER(1,0) // 插件的版本
CLS_NAME(hello) // 插件的名称

cls_handle_t h_class; // 插件的句柄
cls_method_handle_t h_say_hello; // 插件方法的句柄
/* 插件的其他方法 */

static int say_hello(cls_method_context_t hctx, bufferlist *in, bufferlist *out)
{
  // 此函数的逻辑，这里可以非常复杂的处理自己的逻辑，这也是插件的威力
  // src/objclass目录下面封装了很多访问对象及其属性的API，可供使用
  // 这不同于客户端，这是在存储集群进程osd直接访问
  out->append("Hello, world!");
  return 0;
}

// 初始化，所有插件必须提供，open_class的时候初始化
void __cls_init()
{
  cls_register("hello", &h_class); // 注册插件

  cls_register_cxx_method(h_class, "say_hello", // 注册插件的方法
			  CLS_METHOD_RD,
			  say_hello, &h_say_hello);
  /* 注册插件的其他方法 */
}
```

## access plugin

写插件也很容易，我们可以添加自己的复杂逻辑，但是怎么访问呢？

一个简单的例子，这是src/cls/lock/Cls\_lock\_client.cc里的代码，这个是lock插件的客户端，顺便提一下，
当我们写插件的时候，一般会写插件的服务端代码，供osd加载调用，另外还会对应的写一个客户端代码，供我们的客户端使用。
客户端调用本地代码的时候，实际上还是会将此请求封装成一个op操作，然后发送到后端集群，后端集群通过解析，
发现是请求的特定插件的方法，就会加载插件并且调用指定的方法：

```cpp
// lock一个对象的操作
int lock(IoCtx *ioctx,
		const string& oid,
		const string& name, ClsLockType type,
		const string& cookie, const string& tag,
		const string& description, const utime_t& duration,
		uint8_t flags)
{
	ObjectWriteOperation op; // 对rados层来说，这是一个写操作

	lock(&op, name, type, cookie, tag, description, duration, flags); // 调用下面这个函数初始化插件自定义的操作

	return ioctx->operate(oid, &op); // 通过objecter.cc，将这个写操作发送出去
}

void lock(ObjectWriteOperation *rados_op,
		const string& name, ClsLockType type,
		const string& cookie, const string& tag,
		const string& description,
		const utime_t& duration, uint8_t flags)
{
	cls_lock_lock_op op; // 注意这个op，是插件自己的op，发送到后端集群后，会自己解析

	// 初始化插件自己定义的op
	op.name = name;
	op.type = type;
	op.cookie = cookie;
	op.tag = tag;
	op.description = description;
	op.duration = duration;
	op.flags = flags;

	bufferlist in;
	::encode(op, in); // 将插件的op封装到bufferlist，作为输入参数

	rados_op->exec("lock", "lock", in); // 第一个参数是插件名，第二个参数是插件对应的方法，第三个参数是方法的输入参数
}
```

插件自己的操作参数初始化完后，接着就初始化本次写操作的op内容，以便将插件的请求发送到后端集群，这个和以前类似，只是插件的op的类型比较特殊：

```cpp
void librados::ObjectOperation::exec(const char *cls, const char *method, bufferlist& inbl)
{
	::ObjectOperation *o = (::ObjectOperation *)impl;
    o->call(cls, method, inbl); // 调用objecter.cc的ObjectOperation
}

void call(const char *cname, const char *method, bufferlist &indata)
{
    add_call(CEPH_OSD_OP_CALL, cname, method, indata, NULL, NULL, NULL); // CEPH_OSD_OP_CALL，后端收到op消息后，通过这个值判断是插件的请求
}

void add_call(int op, const char *cname, const char *method, bufferlist &indata, bufferlist *outbl, Context *ctx, int *prval)
{
    OSDOp& osd_op = add_op(op); // 增加op

	// 初始化op各成员
    unsigned p = ops.size() - 1;
    out_handler[p] = ctx;
    out_bl[p] = outbl;
    out_rval[p] = prval;

    osd_op.op.op = op;
    osd_op.op.cls.class_len = strlen(cname);
    osd_op.op.cls.method_len = strlen(method);
    osd_op.op.cls.indata_len = indata.length();

	// 插件的所有输入参数，都是此次写操作的indata，包括插件名，方法名，输入参数
    osd_op.indata.append(cname, osd_op.op.cls.class_len);
    osd_op.indata.append(method, osd_op.op.cls.method_len);
    osd_op.indata.append(indata);
}
```

通过前面的ioctx->operate调用，此次操作就算发送出去了，看看后端集群对此次操作的响应：

```cpp
int ReplicatedPG::do_osd_ops(OpContext *ctx, vector<OSDOp>& ops)
{
	......

	switch (op.op) {

		......

		case CEPH_OSD_OP_CALL: // op 类型
			{
				// 解析输入参数
				string cname, mname;
				bufferlist indata;
				try {
					bp.copy(op.cls.class_len, cname);
					bp.copy(op.cls.method_len, mname);
					bp.copy(op.cls.indata_len, indata);
				} catch (buffer::error& e) {
					dout(10) << "call unable to decode class + method + indata" << dendl;
					dout(30) << "in dump: ";
					osd_op.indata.hexdump(*_dout);
					*_dout << dendl;
					result = -EINVAL;
					tracepoint(osd, do_osd_op_pre_call, soid.oid.name.c_str(), soid.snap.val, "???", "???");
					break;
				}

				ClassHandler::ClassData *cls;
				result = osd->class_handler->open_class(cname, &cls); // dlopen加载插件并初始化
				assert(result == 0);   // init_op_flags() already verified this works.

				ClassHandler::ClassMethod *method = cls->get_method(mname.c_str()); // 初始化的时候，注册了方法句柄
				if (!method) {
					dout(10) << "call method " << cname << "." << mname << " does not exist" << dendl;
					result = -EOPNOTSUPP;
					break;
				}

				int flags = method->get_flags();
				if (flags & CLS_METHOD_WR)
					ctx->user_modify = true;

				bufferlist outdata;
				dout(10) << "call method " << cname << "." << mname << dendl;
				int prev_rd = ctx->num_read;
				int prev_wr = ctx->num_write;
				result = method->exec((cls_method_context_t)&ctx, indata, outdata); // 执行方法
				......
			}
			break;
		......
	}

	......
}
```

这里为什么通过句柄来描述方法，为什么不直接调用？原因是这里有c和c++版本的函数，c版本的函数怎么能处理bufferlist输入参数呢？
所以需要对输入参数和结果参数转化一下：

```cpp
int ClassHandler::ClassMethod::exec(cls_method_context_t ctx, bufferlist& indata, bufferlist& outdata)
{
  int ret;
  if (cxx_func) {
    // C++ call version
    ret = cxx_func(ctx, &indata, &outdata); // c++直接调用
  } else {
    // C version
    char *out = NULL;
    int olen = 0;
    ret = func(ctx, indata.c_str(), indata.length(), &out, &olen); // 参数转化后调用

    if (out) { // 处理结果参数，转化回bufferlist
      // assume *out was allocated via cls_alloc (which calls malloc!)
      buffer::ptr bp = buffer::claim_malloc(olen, out); 
      outdata.push_back(bp);
    }
  }
  return ret;
}
```

这样插件的请求就算成功执行了，实际上就是一个remote call :(

# Summary

1. ceph的可动态加载插件丰富了librados的API，可以根据客户端的特定需求编写特定的插件

2. osd进程通过ClassHandler类管理插件，启动的时候new一个对象，并且将对象地址赋值给全局变量ch

3. 插件通过一些wrapper函数（对ch的调用），注册自己到osd的ClassHandler的map里

4. osd在需要的时候，会自动加载插件并初始化插件(\_\_cls\_init)，然后执行用户请求

5. 插件的执行，其实就是客户端这边发送一个op请求（仍然是通过librados的基本API完成)，osd收到请求后，解析出op类型进行处理，一般就是对注册的插件方法进行调用
