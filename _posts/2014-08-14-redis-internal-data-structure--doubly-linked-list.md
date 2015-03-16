---
layout: post
title: "Redis Internal Data Structure : Doubly Linked List"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
Almost every programmer uses list in their code, so does Redis Author. And there is a simple implementation for this widely used data structure. 

It looks like:

![img](/assets/img/post/redis_list.png)

# Implementation
In Redis, it was called adlist. I guess 'adlist' stands for 'a doubly linked list'. Related files are: **src/adlist.h** and **src/adlist.c**.

## Data Structure
Here is the list node structure, be careful about the value type, it is void*.

```c
// list node
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value; // list node data
} listNode;
```

And here is the list itself. There is a member **len** to record list node numbers so that we can get list length in O(1) time. Also, three function pointers used to deal with **value** in list node.

```c
// list itself
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr); // used to copy list node
    void (*free)(void *ptr); // used to release list node
    int (*match)(void *ptr, void *key); // used to compare list node
    unsigned long len; // list length
} list;
```

For convenience, adlist provides an iterator to traverse list node.

```c
// list iterator
typedef struct listIter {
    listNode *next;
    int direction; // AL_START_HEAD or AL_START_TAIL
} listIter;
```

We can use functions **listGetIterator** and **listNext** to traverse list, like this:

```c
iter = listGetIterator(list,direction);
while ((node = listNext(iter)) != NULL) {
     doSomething(listNodeValue(node));
```

## Initialization

```c
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

## Destroy

```c
void listRelease(list *list)
{
    unsigned long len;
    listNode *current, *next;

    current = list->head;
    len = list->len;
    while(len--) {
        next = current->next;
        if (list->free) list->free(current->value);
        zfree(current);
        current = next;
    }
    zfree(list);
}
```

# ALL APIs
This simple data structure is easy to use and understand compared with Linux kernel list. All APIs are here:

```c
list *listCreate(void);
void listRelease(list *list);
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value);
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
void listDelNode(list *list, listNode *node);
listIter *listGetIterator(list *list, int direction);
listNode *listNext(listIter *iter);
void listReleaseIterator(listIter *iter);
list *listDup(list *orig);
listNode *listSearchKey(list *list, void *key);
listNode *listIndex(list *list, long index);
void listRewind(list *list, listIter *li);
void listRewindTail(list *list, listIter *li);
void listRotate(list *list);
```
