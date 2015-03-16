---
layout: post
title: "Redis Event Library"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
Redis as a network service must wait for client connection and deal with client request. Considering this kind of socket event, redis has its own simple event library instead of using the well-known libevent library for simplicity.

The redis server process is like this:

```c
int main(int argc, char **argv) {
    ...
    // create event loop handle
    server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);
    ...

    aeSetBeforeSleepProc(server.el,beforeSleep); // set function ptr
    aeMain(server.el); // loop

    // stop
    aeDeleteEventLoop(server.el); 
    return 0;
}

void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        
        // deal with events
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```
Above code snippet shows how redis server process starts and waits for connection. It creates an event loop handle, and then loop until explicit stop. It will deal with client request when looping.

# Event Library
There are two kinds of events that redis cares about: **file event** and **time event**. Macro `AE_DONT_WAIT` is used for **non-blocking**, which means return to the caller immediately. 

```c
#define AE_FILE_EVENTS 1 
#define AE_TIME_EVENTS 2 
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)
#define AE_DONT_WAIT 4  
```

There are two kinds of operations for event library: **read** and **write**.

```c
#define AE_NONE 0        
#define AE_READABLE 1    
#define AE_WRITABLE 2    
```

According to different OS, event library wrappers different low level implementation, such as epoll, kqueue, select and so on. Those low level implementations start with api*.

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

## Data Structure
Related file: **ae.h**

### File event

```c
typedef struct aeFileEvent {
    int mask; // AE_READABLE or AE_WRITABLE
    aeFileProc *rfileProc; // read call back function
    aeFileProc *wfileProc; // write call back function
    void *clientData; // specific api data
} aeFileEvent;
```

Fired file event:

```c
typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;
```

### Time event

All time event will be **linked in a single list**. New time event will be inserted in after the head, just like insert an element to a list reversely.

```c
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

### Event handle

*setsize* is the total number of file descriptors this event handle tracked. For example, if setsize is 1024, this event handle can deal with fd from 0 to 1023. If setsize is 10, it can only deal with fd from 0 to 9. 

*maxfd* is the current max fd. It is always little than *setsize*.

```c
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead; // time event single list head
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;
```

## Event Library API

Related file: **ae.c**

### Init

Create event loop handle so that we can use it to add/delete event. Allocate memory according to *setsize*.

```c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    // allocate memory for file events
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize); 
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);

    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL; // null list
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1; // no event added at present, so set it -1

    eventLoop->beforesleep = NULL;
    if (aeApiCreate(eventLoop) == -1) goto err;

    // all file event's initial state is AE_NONE
    // it will be changed when adding event
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;

err:
    ...
    return NULL;
}
```

### Destroy

Just release all dynamically allocated memory.

```c
void aeDeleteEventLoop(aeEventLoop *eventLoop) {
    aeApiFree(eventLoop);
    zfree(eventLoop->events);
    zfree(eventLoop->fired);
    zfree(eventLoop);
}
```

### Add/Delete Event

According to mask and fd, call function *aeApiAddEvent* to set event to epoll.

```c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    // check fd
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;

    // update maxfd
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

Delete Event is similar to add.

### Deal with Event

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

	    /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

## Low Level API

These low level APIs are used by the event library and hide the details specific to operating system.

```c
static int aeApiCreate(aeEventLoop *eventLoop);
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask);
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask);
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp);
```

Take epoll implementation as a example, function map relationship:

> aeApiCreate ------> epoll_create

> aeApiAddEvent/aeApiDelEvent ------>epoll_ctl

> aeApiPoll ------> epoll_wait

### Epoll wrapper data structure

```c
typedef struct aeApiState {
    int epfd;
    struct epoll_event *events;
} aeApiState;
```

### Create epoll handle:

```c
static int aeApiCreate(aeEventLoop *eventLoop) {
    // allocate memory
    aeApiState *state = zmalloc(sizeof(aeApiState));
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);

    // create epoll fd
    state->epfd = epoll_create(1024);

    // initialize specific api data
    eventLoop->apidata = state;
    return 0;
}
```

### Add/Delete event to epoll fd

It converts event library mask to epoll mask and then call epoll_ctl.

```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee;
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;

    // epoll system call
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```
Delete is similar to add operation.

### Deal with event

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```

# Reference
 * [http://redis.io/topics/internals-eventlib](http://redis.io/topics/internals-eventlib)
 * [http://redis.io/topics/internals-rediseventlib](http://redis.io/topics/internals-rediseventlib)
