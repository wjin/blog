---
layout: post
title: "Ceph Librbd Overview"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Overview

在早期firefly和hammer版本的时候，大致看过librbd这块代码，那时候feature还比较少，代码大概也就1w行左右，还比较简单，到jewel版本的时候，新增加了很多feature，
不管是代码量还是复杂度，都上去了，最新的maintainer用了一年多时间，贡献量已经刷到top5 :(

不过虽然代码量增多了，但是作者的设计功底还是值得称赞的，代码本身的可读性非常高。本文大致描述一下librbd各个类或文件的作用，代码参考jewel版本10.2.3。

# Class

* **AioCompletion**

处理上层IO回调。用户的一次异步IO请求(主要包括read/write/discard/flush等)是否完成，由此类控制，内部有一个成员AsyncOperation async\_op，
用于控制单个IO操作的的start/finish。一次IO可能对应多个object的请求，内部计数pending\_count控制是否此次IO已经完成，回调过程:

> complete\_request -> complete -> complete\_cb

* **AioImageRequestWQ**

异步提交IO请求的队列，aio接口变为彻底的non-block模式。线程池有且仅有一个线程处理队列的请求，由于有一个唯一的IO队列请求入口，所以block IO变得非常容易，此类提供block IO的功能。
线程池从队列获取请求的时候，会调用AioImageRequestWQ::process函数进行处理，最终调用AioImageRequest::send。

* **AioImageRequest**

IO请求的处理，可能一次image的请求会被拆分为多个对象的请求，即AioObjectRequest，在发送对象请求前会先设置AioCompletion::pending\_count为请求总数，
当所有对象的请求都完成后，回调AioCompletion::complete\_request内部的判断才会为真，标志用户的这次io请求全部完成，进而回调用户最开始设置的回调函数。
拆分成对象的请求逻辑在函数send\_request中，主要借助Striper::file\_to\_extents函数，将逻辑线性地址转换为对象的extent。

* **AioObjectRequest**

单个对象的IO处理逻辑，向rados发送请求。主要是读写操作的逻辑，当image是clone操作产生的时候，读写时需要注意对象在当前image不存在的时候，需要去parent image读取，即copyup操作。
无论是读或写，内部逻辑都是一个小的状态机，参见should\_complete函数。

* **CopyupRequest**

处理从parent image读取object内容的请求。将这种操作也归类为一种IO操作，内部有一个成员AsyncOperation m\_async\_op，控制IO的开始与结束。
需要注意COR和COW两种操作的区别，还有当读到内容后需要更新object map等。

* **ImageCtx**

image的大杂烩，包括成员(ImageWatcher/ImageState/ExclusiveLock/ObjectMap/Operation/Journal)等变量，并且还有很多把锁，以及两个最主要的队列aio\_work\_queue和op\_work\_queue。
另外，还有用于跟踪IO的链表async\_ops和用于跟踪Operation的链表async\_requests。

* **ImageWatcher**

因为存在多个客户端同时打开一个image，为了保护数据的一致性，新增加了exclusive\_lock特性，只有持有exclusive\_lock的客户端才能进行image数据以及元数据的修改，
并且在修改以后，需要通知其他客户端关于image的变化，通知消息的发送及响应就由这个类来负责处理。在open image的时候，会添加watch，用来接收发送消息。
内部实现在osd这端借助于watch/notify机制。

* **WatchNotifyTypes**

主要封装了ImageWatcher类需要处理的消息类型，没有什么逻辑处理，比较简单。

* **ImageState**

有exclusive lock feature后，image打开关闭的流程变得复杂，这个类用于处理image的状态(open/close/refresh/snap)，在open的时候，会初始化ImageWatcher 变量，向rados注册接收通知的句柄。
具体的实现内部是一个小状态机，针对不同状态执行不同的action，具体请求实现在目录image。需要注意这个文件里有个类ImageUpdateWatchers，这个和前面的ImageWatcher没有关系，
前面是针对多个客户端的处理，不同客户端即不同进程之间监视，这个类是本客户端自己的监视，如果发现image状态有了改变，需要通知注册的watch，进行相应的处理，不会跨越osd的watch/notify。

* **Operations**

处理对image的维护操作(flatten/resize/rename/snap等)。内部实现的时候，首先会获取exclusive lock，如果成功获取，则执行操作，否则将操作封装成message，
由ImageWatcher类发出，由远端持有exclusive lock的客户端代为执行。

* **AsyncOperation**

区别于Operations的维护操作，这个类是IO相关类，统一控制IO的开始与结束，start\_op的时候会将自己插入链表ImageCtx::async\_ops，finish\_op的时候从链表删除。

* **AsyncRequest**

统一控制Operations的开始与结束(Operations类处理的操作，以及ObjectMap的部分操作，最后都会封装成一个request，即这个类的派生类)，start\_request的时候会将自己插入链表ImageCtx::async\_requests，finish\_requests的时候从链表删除。

* **ExclusiveLock**

分布式锁的实现，内部一个状态机处理各种状态，执行对应的action，即发送对应的请求，比如acquire/release等，每个请求的实现在目录exclusive\_lock。
同时，这个类提供block request的功能，可以禁止对image的维护操作。

* **ObjectMap**

处理object map的删除，更新等。内部实现就是维护一个image的所有对象的位图，可以快速判断对象文件是否存在，而不用向rados后端集群发送消息等待消息结果返回的时候才判断。
具体每个请求的实现在目录object\_map。

另外还有一些是与journal feature和data group相关的文件，因暂时用不上，留作后面分析。

# Thread

* ImageWatcher类中有一个TaskFinisher成员，其中包含两个线程，Finisher和SafeTimer。这两个线程用来接收和发送notify message，以及消息超时后的处理。
这些消息主要是和request相关。

* ImageCtx类中，包含两个成员AioImageRequestWQ\* aio\_work\_queue和ContextWQ\* op\_work\_queue，这两个队列共用一个线程池，并且线程池只有一个线程，
前一个队列主要用于IO提交，后一个队列用于callback的回调，单线程可以保证IO的顺序。

* ImageState类中，有一个成员ImageUpdateWatchers\* m\_update\_watchers，它包含一个ContextWQ\* m\_work\_queue，此队列有一个线程。
image的state可能会发生变化，这时候需要通知已经注册的watchers，调用已经注册的callback，主要用于场景rbd-nbd，见文件src/tools/rbd\_nbd/rbd\_nbd.cc。
注意区分，这里的watchers不同于ImageWatcher类的功能。
