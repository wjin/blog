---
layout: post
title: "Ceph Librbd Feature"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Feature List

* **layering**

image的克隆操作。可以对image创建快照并保护，然后从快照克隆出新的image出来，父子image之间采用COW技术，共享对象数据。

* **striping v2**

条带化对象数据，类似raid 0，可改善顺序读写场景较多情况下的性能。

* **exclusive lock**

保护image数据一致性，对image做修改时，需要持有此锁。这个可以看做是一个分布式锁，在开启的时候，确保只有一个客户端在访问image，否则锁的竞争会导致io急剧下降。
主要应用场景是qemu live-migration。

* **object map**

此特性依赖于exclusive lock。因为image的对象分配是thin-provisioning，此特性开启的时候，会记录image所有对象的一个位图，用以标记对象是否真的存在，在一些场景下可以加速io。

* **fast diff**

此特性依赖于object map和exlcusive lock。快速比较image的snapshot之间的差异。

* **deep-flatten**

layering特性使得克隆image的时候，父子image之间采用COW，他们之间的对象文件存在依赖关系，flatten操作的目的是解除父子image的依赖关系，但是子image的快照并没有解除依赖，deep-flatten特性使得快照的依赖也解除。

* **journaling**

依赖于exclusive lock。将image的所有修改操作进行日志化，并且复制到另外一个集群（mirror)，可以做到块存储的异地灾备。这个特性在部署的时候需要新部署一个daemon进程，目前还在试验阶段，不过这个特性很重要，可以做跨集群/机房容灾。

创建image的时候，jewel默认开启的特性包括: layering/exlcusive lock/object map/fast diff/deep flatten

# Exclusive Lock

从上面可以看出，很多特性都依赖于exclusive lock，重点介绍一下。

exclusive lock 是分布式锁，实现的时候默认是客户端在第一次写的时候获取锁，并且在收到其他客户端的锁请求时自动释放锁。这个特性在jewel默认开启后，本身没什么问题，
客户端可以自动获取和释放锁，在客户端crash后也能够正确处理。

但是qemu社区的人员，将这个feature和以前旧的librbd API(rbd\_lock\_exclusive/rbd\_lock\_shared) 联系在了一起，发了一个错误的patch在qemu社区，
参见[maillist](https://lists.gnu.org/archive/html/qemu-devel/2016-04/msg02422.html)讨论。大致情况是当qemu线程已经获取锁的情况下，
librbd线程想要获取锁的时候就会一直获取不到，导致锁的冲突，进而变为只读，因为实现锁的方式是一样的。鉴于此，ceph社区新增加了librbd API，用来显示地获取和释放exclusive lock的锁，
见[pr](https://github.com/ceph/ceph/pull/9592)，个人认为旧的API是一个advisory lock，应该会被deprecated。

同时，目前k8s使用ceph krbd作为存储卷，并且通过加锁的方式将卷挂载在指定的pod上，锁的使用方式和上面qemu的误导patch原理一样，
见[issue](https://github.com/kubernetes/kubernetes/issues/33013)，如果将来krbd enable了exclusive lock feature，也会导致冲突。改进方式是增加选项来控制锁，由k8s自己控制谁该拥有锁:

* krbd禁止exclusive lock的自动转换，[issue](http://tracker.ceph.com/issues/17524)

* k8s支持rbd-nbd [issue]( https://github.com/kubernetes/kubernetes/issues/32266), rbd-nbd已经支持禁止exclusive lock的自动转换, [pr](https://github.com/ceph/ceph/pull/11438)

k8s社区有人建议容器可以考虑不走krbd，因为krbd严重依赖于内核，feature落后librbd太多，并且生产环境不宜随时升级kernel版本。
替代方案是用rbd-nbd方式，nbd是kernel的网络块设备，已经很稳定，ceph社区已经实现了rbd-nbd 用户空间客户端，用以连接kernel nbd设备，
客户端使用librbd实现，非常简单，性能损耗也不大。
