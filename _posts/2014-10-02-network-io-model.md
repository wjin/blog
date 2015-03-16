---
layout: post
title: "Network I/O Model"
description: ""
category: network
tags: []
---
{% include JB/setup %}


# Synchronous vs Asynchronous

**Synchronous I/O** would block the calling process until the I/O operation is done.
The **calling process** will finish I/O operation itself within **process context**.

**Asynchronous I/O** would not block the calling process. Calling process just sends request,
and then continue to work. The *opearting system* will do the I/O operation, after it has done,
it will notify the calling process.

# I/O Model

There are two stages in I/O operation and four I/O models in general.

## Stage

1. Wait for data to be ready

2. Copy data from kernel space to user space

## Model

**Blocking**

Block on the first stage like recvfrom system call.

**Non-blocking**

Poll on the first stage.

**I/O Multi-plexing**

Block on select/poll/epoll system calls.

Above three models belong to synchronous I/O.

**Asynchronous**

It is not good so far. There is a long story about Asynchronous I/O in Linux mail list :(
