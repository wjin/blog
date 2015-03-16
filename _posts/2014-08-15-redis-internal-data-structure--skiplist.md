---
layout: post
title: "Redis Internal Data Structure : Skiplist"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
Skip List gets **O(log n)** time complexity on average. And it is easy to implement compared to AVL tree or Red-Black tree. So Redis uses it to implement ordered set.
 
# Implementation
## Data Structure
Related File: **redis.h**

Redis adds a *backward* pointer for each skip list node to traverse reversely. Also there is a *span* variable in level entry to record how many nodes must be crossed when reaching to next node. Actually, when traverse list, we can accumulate span to get the **rank** of a node in sorted set.

Here is an example of skip list without dumb head node. There are three nodes and they are sorted by score. There are two lists: level 1 and level2. We can reach to node3 from node1 by level2 list directly, and its span is 2. Also, we can cross node2 to get to node3 with level1 list.

![img](/assets/img/post/redis_skiplist.png)

```c
typedef struct zskiplistNode {
    robj *obj; // redis generic object
    double score;
    struct zskiplistNode *backward; // backward pointer, only exist in level zero list
    struct zskiplistLevel {
        struct zskiplistNode *forward; // next node, may skip a lot of nodes
        unsigned int span; // number of nodes need be crossed to reach to next node
    } level[]; // level array to make up lists
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length; // number of nodes
    int level; // current level
} zskiplist;
```

## Core API
Related file: **t_zset.c**

### Init
Allocate memory for skip list and create a **dumb head** skip list node. This dumb head has the max levels(`ZSKIPLIST_MAXLEVEL`), all level's pointer is initialised to null, so the skip list is empty. Actually, it has `ZSKIPLIST_MAXLEVEL` empty single lists.

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1; // only one level
    zsl->length = 0; // no node at present

    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);

    // initialise each level to empty list
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }

    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}

zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->obj = obj;
    return zn;
}
```

### Destroy
In Redis, there is an abstract data type: robj. It can be shared by many structures using reference count to save memory. So when destroy node, just decrease robj reference count.

```c
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl->header->level[0].forward, *next;

    zfree(zsl->header); // free dumb head

    // free all nodes one by one
    // just use level[0] list because all nodes must be in level[0] list
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    zfree(zsl); / free skiplist itself
}

void zslFreeNode(zskiplistNode *node) {
    decrRefCount(node->obj); // decrease reference count
    zfree(node);
}
```

### Insert
**zslInsert** is the most important function of skip list. The **update** array stores previous pointers for each level, new node will be added after them. **rank** array stores the rank value of each skiplist node.

Steps:

1. generate update and rank array
2. create a new node with random level
3. insert new node according to *update* and *rank* info
4. update other necessary infos, such as span, backward pointer, length.

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    // get update and rank array
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    // create new node
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,obj);

    // insert
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // update info
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

### Delete
Similar to insert, we need to get **update** array.

```c
int zslDelete(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;
    
    // get update info
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0)))
            x = x->level[i].forward;
        update[i] = x;
    }

    // find it, delete
    x = x->level[0].forward;
    if (x && score == x->score && equalStringObjects(x->obj,obj)) {
        zslDeleteNode(zsl, x, update);
        zslFreeNode(x);
        return 1;
    }
    return 0; /* not found */
}

void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
        // previous pointer pointed to this node
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
        // previous pointer pointed to nodes behind this node or nullptr
        // just decrease span
            update[i]->level[i].span -= 1;
        }
    }

    // update backward
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }

    // top level lists may be null
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;

    zsl->length--;
}
```

### Other APIs
There are a lot of useful APIs, however, they are easy to understand if you have already known about **update** and **rank** array in *Insert* operation.

Remember there are MAX_LEVEL single lists in skip lists. **update** records previous pointer in each list. **rank** is the position of a node (node are sorted in some way).

# Reference
* [http://epaperpress.com/sortsearch/download/skiplist.pdf](http://epaperpress.com/sortsearch/download/skiplist.pdf)
* [http://en.wikipedia.org/wiki/Skip_list](http://en.wikipedia.org/wiki/Skip_list)
