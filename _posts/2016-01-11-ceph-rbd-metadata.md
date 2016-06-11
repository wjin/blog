---
layout: post
title: "Ceph Rbd Metadata"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

ceph RBD块存储可以通过librbd库供qemu虚拟机使用，也可以通过krbd内核模块直接给linux块设备驱动使用。
不论哪种情况，rbd对外提供的都是一块虚拟的磁盘，那么，这个虚拟磁盘内部是怎么存储的？有些什么元数据？

这篇文章简单总结下内部数据的组织，测试利用vstart方式启动一个测试集群，
可以参考[官方文档](http://docs.ceph.com/docs/master/dev/#development-mode-cluster)。

# Create Image

```bash
[jw@localhost src]$ ./rados -p rbd ls # 列出所有对象
[jw@localhost src]$ ./rbd create foo --size 1024 --image-format 2 # 创建镜像foo
[jw@localhost src]$ ./rados -p rbd ls
rbd_header.101c190e05d5
rbd_directory
rbd_id.foo
```

当向一个空的pool新建一个image后，发现会新建三个对象，其中rbd\_directory保存当前pool的所有image名字与id的双向映射，方便列出当前pool所有的image:

```bash
[jw@localhost src]$ ./rados -p rbd listomapvals rbd_directory # 获取directory对象omap信息 
id_101c190e05d5
value (7 bytes) :
0000 : 03 00 00 00 66 6f 6f                            : ....foo

name_foo
value (16 bytes) :
0000 : 0c 00 00 00 31 30 31 63 31 39 30 65 30 35 64 35 : ....101c190e05d5
```

对象rbd\_id.foo保存image的ID：

```bash
[jw@localhost src]$ ./rados -p rbd get rbd_id.foo foo # 获取对象内容
[jw@localhost src]$ hexdump -C foo # dump对象内容
00000000  0c 00 00 00 31 30 31 63  31 39 30 65 30 35 64 35  |....101c190e05d5|
```

对象rbd\_header.101c190e05d5 保存image的元数据信息(可以用命令rbd info image\_name查看)：

```bash
[jw@localhost src]$ ./rados -p rbd listomapvals rbd_header.101c190e05d5 # 获取header对象omap信息
features
value (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value (25 bytes) :
0000 : 15 00 00 00 72 62 64 5f 64 61 74 61 2e 31 30 31 : ....rbd_data.101
0010 : 63 31 39 30 65 30 35 64 35                      : c190e05d5

order
value (1 bytes) :
0000 : 16                                              : .

size
value (8 bytes) :
0000 : 00 00 00 40 00 00 00 00                         : ...@....

snap_seq
value (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........
```

以后每新加一个image，就会出现两个对象rbd\_header.xx 和 rbd\_id.xx，并且更新rbd\_directory中的信息。
同时需要注意，header 和 directory两个对象都是记录的kv信息，所以实际size是0，查看的时候使用的是listomapvals，
而rbd\_id对象存放的是image的id，内容存放在对象文件内部，查看的时候，先通过get命令获取对象内容，然后再dump出来。

```bash
[jw@localhost src]$ ./rados -p rbd stat rbd_header.101c190e05d5 # 获取header对象文件的信息
rbd/rbd_header.101c190e05d5 mtime 2016-01-16 15:17:19.000000, size 0
[jw@localhost src]$ ./rados -p rbd stat rbd_directory # 获取directory对象文件的信息
rbd/rbd_directory mtime 0.000000, size 0
[jw@localhost src]$ ./rados -p rbd stat rbd_id.foo # 获取rbd_id对象文件的信息
rbd/rbd_id.foo mtime 2016-01-12 10:33:51.000000, size 16
```

# Create Snapshot

## Create

紧接着前面的实验，对镜像foo创建一个快照：

```bash
[jw@localhost src]$ ./rbd snap create rbd/foo@snap # 创建快照, 快照的名称为snap
[jw@localhost src]$ ./rados -p rbd ls # 列出所有对象
rbd_header.101c190e05d5
rbd_directory
rbd_id.foo
```

发现没有增加任何对象，那快照的信息保存在什么地方？答案在header对象中：

```bash
[jw@localhost src]$ ./rados -p rbd listomapvals rbd_header.101c190e05d5 # 获取header对象omap信息
features
value (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value (25 bytes) :
0000 : 15 00 00 00 72 62 64 5f 64 61 74 61 2e 31 30 31 : ....rbd_data.101
0010 : 63 31 39 30 65 30 35 64 35                      : c190e05d5

order
value (1 bytes) :
0000 : 16                                              : .

size
value (8 bytes) :
0000 : 00 00 00 40 00 00 00 00                         : ...@....

snap_seq
value (8 bytes) :
0000 : 02 00 00 00 00 00 00 00                         : ........

snapshot_0000000000000002  # 快照ID
value (81 bytes) :
0000 : 04 01 4b 00 00 00 02 00 00 00 00 00 00 00 04 00 : ..K.............
0010 : 00 00 73 6e 61 70 00 00 00 40 00 00 00 00 01 00 : ..snap...@......
0020 : 00 00 00 00 00 00 01 01 1c 00 00 00 ff ff ff ff : ................
0030 : ff ff ff ff 00 00 00 00 fe ff ff ff ff ff ff ff : ................
0040 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 : ................
0050 : 00                                              : .
```
snap_seq为image当前最新的snapshot的snapid。

## Copy On Write

ceph创建快照是秒极的，然后通过COW机制进行维护，可以对镜像进行读写看看发生的变化。先借助于krbd，
将刚才的镜像foo map到/dev下面：

```bash
[jw@localhost src]$ sudo ./rbd map rbd/foo # map镜像foo
/dev/rbd0
[jw@localhost src]$ sudo ./rbd showmapped # 列出所有map的信息
id pool image snap device    
0  rbd  foo   -    /dev/rbd0 
```

然后对镜像foo写入4M的数据：

```bash
[jw@localhost src]$ sudo dd if=/dev/random of=/dev/rbd0 bs=1M count=4 # 写入数据
dd: warning: partial read (78 bytes); suggest iflag=fullblock
0+4 records in
0+4 records out
312 bytes (312 B) copied, 0.0270896 s, 11.5 kB/s

[jw@localhost src]$ ./rados -p rbd ls # 查看所有对象
rbd_data.101c190e05d5.0000000000000000 # 新增加的对象文件
rbd_header.101c190e05d5
rbd_directory
rbd_id.foo

[jw@localhost src]$ sudo ./rados -p rbd stat rbd_data.101c190e05d5.0000000000000000
rbd/rbd_data.101c190e05d5.0000000000000000 mtime 2016-01-16 15:53:37.000000, size 4096
```

因为之前已经创建了快照，然后才写入数据，按理说这时候应该有快照的数据存在，但是实际情况却不是这样的:

```bash
[jw@localhost src]$ find ./dev -name "*101c190e05d5.*" # 三个一样的对象，因为是三副本
./dev/osd0/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
./dev/osd1/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
./dev/osd2/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
```

没有快照的对象存在，说明快照的数据没有保存，这肯定不是bug，原因是在最开始创建快照的时候，镜像是空数据或全零数据，
然后写的时候，触发了COW，COW在复制数据的时候，发现是全零数据，就不会创建对象去存储了，浪费存储空间。

```bash
[jw@localhost src]$ ./rbd snap create rbd/foo@snap2 # 再次创建快照，名称为snap2
[jw@localhost src]$ ./rados -p rbd ls
rbd_data.101c190e05d5.0000000000000000
rbd_header.101c190e05d5
rbd_directory
rbd_id.foo

[jw@localhost src]$ sudo dd if=/dev/random of=/dev/rbd0 bs=1M count=4 # 再次写入第一个object
dd: warning: partial read (78 bytes); suggest iflag=fullblock
0+4 records in
0+4 records out
305 bytes (305 B) copied, 0.0329425 s, 9.3 kB/s

[jw@localhost src]$ find ./dev -name "*101c190e05d5.*"
./dev/osd0/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
./dev/osd0/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__3_5A1CE208__0 # COW发生了, 3是快照的ID
./dev/osd1/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
./dev/osd1/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__3_5A1CE208__0
./dev/osd2/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
./dev/osd2/current/0.0_head/rbd\udata.101c190e05d5.0000000000000000__3_5A1CE208__0
```

find查找比较粗暴，也可以直接查找：

```bash
[jw@localhost src]$ ./ceph osd map rbd rbd_data.101c190e05d5.0000000000000000 # 找到对象文件对应的pg
osdmap e22 pool 'rbd' (0) object 'rbd_data.101c190e05d5.0000000000000000' -> pg 0.5a1ce208 (0.0) -> up ([0,2,1], p0) acting ([0,2,1], p0) # 0.0 是 pg ID

[jw@localhost src]$ ls ./dev/osd0/current/0.0_head/ # pg 0.0对应的目录
__head_00000000__0  rbd\udata.101c190e05d5.0000000000000000__3_5A1CE208__0  rbd\udata.101c190e05d5.0000000000000000__head_5A1CE208__0
```

# Clone Image

还是利用vstart脚本启动一个新的集群，然后创建镜像:

```bash
[jw@localhost src]$ ./rbd create foo --size 1024 --image-format 2 #创建镜像
[jw@localhost src]$ ./rados -p  rbd ls
rbd_directory
rbd_id.foo
rbd_header.10146b8b4567

[jw@localhost src]$ ./rbd snap create rbd/foo@snap #创建快照
[jw@localhost src]$ ./rados -p rbd ls
rbd_directory
rbd_id.foo
rbd_header.10146b8b4567
```
紧接着将快照保护起来，然后clone出一个新的镜像:

```bash
[jw@localhost src]$ ./rbd snap protect rbd/foo@snap #保护快照
[jw@localhost src]$ ./rbd clone rbd/foo@snap rbd/bar #克隆镜像
[jw@localhost src]$ ./rbd ls
bar
foo
[jw@localhost src]$ ./rados -p rbd ls
rbd_id.bar
rbd_children #存放父子关系
rbd_directory
rbd_id.foo
rbd_header.10146b8b4567
rbd_header.101e6b8b4567
```

可以看到，clone的时候除了新的image自身的两个对象，还多了一个对象rbd\_children，这个对象用来保存clone的时候image的父子关系:

```bash
[jw@localhost src]$ ./rados -p rbd listomapvals rbd_children
key (32 bytes):
00000000  00 00 00 00 00 00 00 00  0c 00 00 00 31 30 31 34  |............1014|
00000010  36 62 38 62 34 35 36 37  04 00 00 00 00 00 00 00  |6b8b4567........|
00000020

value (20 bytes) :
00000000  01 00 00 00 0c 00 00 00  31 30 31 65 36 62 38 62  |........101e6b8b|
00000010  34 35 36 37                                       |4567|
00000014
```

如果对克隆的镜像做flatten，即解除父子关系，则rbd\_children对象将会删除父子的信息:

```bash
[jw@localhost src]$ ./rbd flatten bar #flatten 镜像
[jw@localhost src]$ ./rados -p rbd listomapvals rbd_children
[jw@localhost src]$ 
```

大部分元数据就是以上的对象，另外需要注意的是，如果新的特性object map打开(jewel版本以后默认会开启)，还会有存放object map元数据的对象。
