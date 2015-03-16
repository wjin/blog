---
layout: post
title: "Concurrent Server Model"
description: ""
category: network
tags: []
---
{% include JB/setup %}

# Introduction

Develop large-scale, high concurrent networking service application is challenging. Here are  a few models. PPC (process per connection), TPC (thread per connection), and I/O multiplexing, such as select, poll and epoll.

# PPC and TPC

PPC and TPC create a new process/thread for each connection. Therefore, this mode has its inherent drawbacks：when connecting increases dramatically, the overhead of create a new process/thread cannot be ignored.

Even though **process/thread pool** may alleviate it, switch and resource contention between processes/threads can obviously slow down the system response. That is the famous problem of **apache webserver avalanche**.

# I/O Multiplexing

## Select and Poll

Select and poll allows a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready". A file descriptor is considered ready if it is possible to perform the corresponding I/O operation without blocking.

There are some drawbacks for select:

 * The max number of concurrency is limited by FD (set by FD_SETSIZE)
 * O(n) time complexity when scanning FD
 * Memory Copy between kernel and user space

## Linux Epoll

**Epoll** standards for event poll. It is a variant of poll that can be used either as an **edge-triggered** or a **level-triggered** interface and scales well to large numbers of watched file descriptors. It overcomes the disadvantages of select and poll.

 * No limitation of FD, the concurrent number is related to how many files this process can open
 * O(1) time complexity when scanning FD
 * Memory Sharing

To develop a concurrent and high performance service in linux environment, we should choose epoll with no doubt, and its related sytem calls are also easy to use.

### epoll_create

```c
int epoll_create(int size);
```

### epoll_ctl

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

### epoll_wait

```c
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);

struct epoll_event {
    __uint32_t events; // Epoll events
    epoll_data_t data; // User data variable
};

typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;
```