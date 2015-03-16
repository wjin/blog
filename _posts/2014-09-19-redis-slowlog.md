---
layout: post
title: "Redis Slowlog"
description: ""
category: redis
tags: []
---
{% include JB/setup %}


# Introduction

Slowlog, as its name, is used to record commands whose execution time exceeds limitation (a little slow). It is useful to optimize server performance.

There are two config options for this feature: `slowlog-log-slower-than` and `slowlog-max-len`.

```text
slowlog-log-slower-than: time limitation
slowlog-max-len: maximum log entry
```

# Implementation

**Data Structure**

File: `slowlog.h` and `slowlog.c`.

```cpp
typedef struct slowlogEntry {
    robj **argv;
    int argc;
    long long id;       /* Unique entry identifier. */
    long long duration; /* Time spent by the query, in nanoseconds. */
    time_t time;        /* Unix time at which the query was executed. */
} slowlogEntry;
```

All entries are stored in a single list and each entry has an unique ID.

```cpp
struct redisServer {
    ...
    
    list *slowlog;                  /* SLOWLOG list of commands */
    long long slowlog_entry_id;     /* SLOWLOG current entry ID */
    long long slowlog_log_slower_than; /* SLOWLOG time limit (to get logged) */
    unsigned long slowlog_max_len;     /* SLOWLOG max number of items logged */
    
    ...
};
```

**Init**

When redis server starts up, it will call `initServer` function, this function will call `slowlogInit` to initialize slowlog related properties.

It also registers a callback `slowlogFreeEntry` to release slowlog entry.

```cpp
void slowlogInit(void) {
    server.slowlog = listCreate();
    server.slowlog_entry_id = 0; // id start from 0
    listSetFreeMethod(server.slowlog,slowlogFreeEntry); // free entry callback
}
```

**Add Entry**

Inserts new node to list head and delete old node when exceeding max length. `slowlogCreateEntry` will increate object reference normally.

However it will also create new object sometimes to save memory: 

1. string is too long 

2. two many arguments, using the last argument to identify more arguments

This is feasible because it is just a log infomation.

```cpp
void slowlogPushEntryIfNeeded(robj **argv, int argc, long long duration) {
    if (server.slowlog_log_slower_than < 0) return; /* Slowlog disabled */
    if (duration >= server.slowlog_log_slower_than)
        listAddNodeHead(server.slowlog,slowlogCreateEntry(argv,argc,duration));

    /* Remove old entries if needed. */
    while (listLength(server.slowlog) > server.slowlog_max_len)
        listDelNode(server.slowlog,listLast(server.slowlog));
}


slowlogEntry *slowlogCreateEntry(robj **argv, int argc, long long duration) {
    slowlogEntry *se = zmalloc(sizeof(*se));
    int j, slargc = argc;

    if (slargc > SLOWLOG_ENTRY_MAX_ARGC) slargc = SLOWLOG_ENTRY_MAX_ARGC;
    se->argc = slargc;
    se->argv = zmalloc(sizeof(robj*)*slargc);
    for (j = 0; j < slargc; j++) {
        /* increase obj reference or create new object */
        ....
    }
    se->time = time(NULL);
    se->duration = duration;
    se->id = server.slowlog_entry_id++;
    return se;
}
```

**Related Commands**

Three sub commands have been given so far: `reset`, `len` and `get`.

```cpp
void slowlogCommand(redisClient *c) {
    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"reset")) {
        slowlogReset();
        addReply(c,shared.ok);
    } else if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"len")) {
        addReplyLongLong(c,listLength(server.slowlog));
    } else if ((c->argc == 2 || c->argc == 3) &&
               !strcasecmp(c->argv[1]->ptr,"get"))
    {
        /* generate each entry info and return it to client */
        ... 
    } else {
        addReplyError(c,
            "Unknown SLOWLOG subcommand or wrong # of args. Try GET, RESET, LEN.");
    }
}
```

# Example

```bash
Weis-MacBook-Pro:redis eric$ redis-cli
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 5
OK
127.0.0.1:6379> set msg "hello, world"
OK
127.0.0.1:6379> set idx 1
OK
127.0.0.1:6379> set x y
OK
127.0.0.1:6379> slowlog get
1) 1) (integer) 4
   2) (integer) 1411105058
   3) (integer) 8
   4) 1) "set"
      2) "x"
      3) "y"
2) 1) (integer) 3
   2) (integer) 1411105037
   3) (integer) 11
   4) 1) "set"
      2) "idx"
      3) "1"
3) 1) (integer) 2
   2) (integer) 1411105027
   3) (integer) 30
   4) 1) "set"
      2) "msg"
      3) "hello, world"
4) 1) (integer) 1
   2) (integer) 1411105016
   3) (integer) 9
   4) 1) "config"
      2) "set"
      3) "slowlog-max-len"
      4) "5"
5) 1) (integer) 0
   2) (integer) 1411104979
   3) (integer) 26
   4) 1) "config"
      2) "set"
      3) "slowlog-log-slower-than"
      4) "0"
127.0.0.1:6379> slowlog get
1) 1) (integer) 5
   2) (integer) 1411105073
   3) (integer) 31
   4) 1) "slowlog"
      2) "get"
2) 1) (integer) 4
   2) (integer) 1411105058
   3) (integer) 8
   4) 1) "set"
      2) "x"
      3) "y"
3) 1) (integer) 3
   2) (integer) 1411105037
   3) (integer) 11
   4) 1) "set"
      2) "idx"
      3) "1"
4) 1) (integer) 2
   2) (integer) 1411105027
   3) (integer) 30
   4) 1) "set"
      2) "msg"
      3) "hello, world"
5) 1) (integer) 1
   2) (integer) 1411105016
   3) (integer) 9
   4) 1) "config"
      2) "set"
      3) "slowlog-max-len"
      4) "5"
127.0.0.1:6379>
```