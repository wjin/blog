---
layout: post
title: "Redis Internal Data Structure : SDS"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
SDS means **Simple Dynamic Strings**. It is the simplest basic data structure and widely used in many modules in Redis. Its purpose is to replace char* in C language.

Redis provides SDS because it supports efficient functions to get the **string length** and **append** another string to the end without allocating memory each time. 

Also, it is **binary safe** because it does not care about whether this string is ending with '\0' or not. 

```text
+--------+-------------------------------+-----------+
| Header | Binary safe C alike string... | Null term |
+--------+-------------------------------+-----------+
         |
         `-> Pointer returned to the user.
```

Now, it is extracted and forked as a stand alone project at :  [https://github.com/antirez/sds](https://github.com/antirez/sds).

# Implementation
SDS data structure related files: **src/sds.h** and **src/sds.c**

## Data Structure

```c
typedef char *sds;

struct sdshdr {
    int len; // current string length
    int free; // available space in buffer
    char buf[]; // string content stored here, c99 grammar
};
```

## Initialization
The core function to create a new SDS string is **sdsnewlen**. The string is always null-termined. Also, it is binary safe and can contain '\0' characters in the middle, as the length is stored in the sds header.

```c
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;

    // +1 means there is always a '\0' in the end
    if (init) {
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }

    if (sh == NULL) return NULL; // no memory
    sh->len = initlen; // init string length
    sh->free = 0; // init available space to zero
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    sh->buf[initlen] = '\0';

    // return the real string content, not including header
    // user can use it as usual
    return (char*)sh->buf;
}
```

**Note**: sdsnewlen always returns the **real string content** to user, not including sds header. Actually, so do all other APIs. So it does not break the API use.

According to above implementation, it is easy to get string length and extra available buffer size in O(1) time.

```c
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}

static inline size_t sdsavail(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->free;
}
```

## Destroy
SDS memory (including sds header) is dynamically allocated. So just call free to release memory to system.

```c
void sdsfree(sds s) {
    if (s == NULL) return;
    zfree(s-sizeof(struct sdshdr));
}
```

## Memory Allocation Strategy
SDS uses function **sdsMakeRoomFor** to adjust the buf size. This function accepts a second parameter *addlen*. It guarantees that after calling it, there is at least *addlen* bytes free space in the end of buf.

It **pre-allocates** memory to reduce memory allocation times. Actually, it just simply doubles the original size when it is less than `SDS_MAX_PREALLOC(1MB)`. This is similar to C++ vector allocation strategy when memory is not enough. This is why string append operation does not need to allocate memory every time.

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);
    size_t len, newlen;

    if (free >= addlen) return s;
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2; // double size
    else
        newlen += SDS_MAX_PREALLOC;
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
    if (newsh == NULL) return NULL;

    newsh->free = newlen - len;
    return newsh->buf;
}
```

Function **sdsRemoveFreeSpace** used to remove the free space in the end of buf.

```c
sds sdsRemoveFreeSpace(sds s) {
    struct sdshdr *sh;

    sh = (void*) (s-(sizeof(struct sdshdr)));
    sh = zrealloc(sh, sizeof(struct sdshdr)+sh->len+1);
    sh->free = 0;
    return sh->buf;
}
```

# All APIs
Most SDS APIs are similar to standard c string.

```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
size_t sdslen(const sds s);
sds sdsdup(const sds s);
void sdsfree(sds s);
size_t sdsavail(const sds s);
sds sdsgrowzero(sds s, size_t len);
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const char *t);
sds sdscatprintf(sds s, const char *fmt, ...);
sds sdscatfmt(sds s, char const *fmt, ...);
sds sdstrim(sds s, const char *cset);
void sdsrange(sds s, int start, int end);
void sdsupdatelen(sds s);
void sdsclear(sds s);
int sdscmp(const sds s1, const sds s2);
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);
void sdsfreesplitres(sds *tokens, int count);
void sdstolower(sds s);
void sdstoupper(sds s);
sds sdsfromlonglong(long long value);
sds sdscatrepr(sds s, const char *p, size_t len);
sds *sdssplitargs(const char *line, int *argc);
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);
sds sdsjoin(char **argv, int argc, char *sep);

/* Low level functions exposed to the user API */
sds sdsMakeRoomFor(sds s, size_t addlen);
void sdsIncrLen(sds s, int incr);
sds sdsRemoveFreeSpace(sds s);
size_t sdsAllocSize(sds s);
```
