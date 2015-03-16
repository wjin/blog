---
layout: post
title: "Redis Internal Data Structure : Dictionary"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
In Redis, dictionary is a widely used data structure as well as sds and adlist. And it is the most important data structure as Redis is a key-value store.

It is implemented by means of hash table and there are two hash tables in dictionary to implement **incremental rehashing**. Dictionary can **auto-resize** if needed, and the internal hash table size is always **power of two**. And the internal hash table uses **chaining** to deal with collision.

Here is an overview of dictionary. ht[1] is always empty unless it is in the process of rehashing.

![img](/assets/img/post/redis_dict.png)

# Implementation

Related files: **dict.h** and **dict.c**

## Data Structure
To make dictionary more flexible and universal, **dictType** structure includes 6 hooks to deal with key and value. 

```c
typedef struct dictType {
    unsigned int (*hashFunction)(const void *key); // hash function
    void *(*keyDup)(void *privdata, const void *key); // copy key
    void *(*valDup)(void *privdata, const void *obj); // copy value
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // compare keys
    void (*keyDestructor)(void *privdata, void *key); // free key
    void (*valDestructor)(void *privdata, void *obj); // free value
} dictType;
```

This is dictionary data structure:

```c
typedef struct dict {
    dictType *type; // hooks
    void *privdata; // private data used in hooks function
    dictht ht[2]; // two hash tables for incremental rehashing
    int rehashidx; // rehashing tag
    int iterators; // number of iterators
} dict;
```
Member **rehashidx** is pretty important because it identifies whether dict is in the process of rehashing. If it is not *-1*, dictionary is rehashing and *ht[1]* will not be NULL. And all new pairs <key, val> will be added into ht[1] table.

Below two structures are related to hash table itself. According to structure *dictEntry*, we know that it will chain all conflicted nodes using pointer *next*.

```c
typedef struct dictEntry {
    void *key; // key
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v; // value
    struct dictEntry *next; // next node in the same slot
} dictEntry;

typedef struct dictht {
    dictEntry **table; // hash node array (buckets)
    unsigned long size; // total slot size
    unsigned long sizemask; // size mask
    unsigned long used; // slot in use
} dictht;
```

The member *sizemask* is used to get an index or slot in hash table by **dictHashKey(key) & sizemask**. So it is always initialized with **size - 1** because array **table** subscription starts from zero in C language. For example, if size is 16, and sizemask will be 15, slot index will be from 0 to 15, that is the result of **dictHashKey(key) & sizemask**.

## Core API
Following I will analyse some core APIs, including init, destroy, add, delete and replace operations. Code does not exactly match the original code. I have removed some trivial code.

### Init
Allocate memory for dictionary structure and initialize with default values. Be careful both hash tables ht[0] and ht[1] in dictionary are NULL at present.

```c 
dict *dictCreate(dictType *type,
        void *privDataPtr)     
{
    dict *d = zmalloc(sizeof(*d));  
    _dictInit(d,type,privDataPtr);  
    return d;
}

int _dictInit(dict *d, dictType *type,
        void *privDataPtr)     
{
    _dictReset(&d->ht[0]);     
    _dictReset(&d->ht[1]);     
    d->type = type;
    d->privdata = privDataPtr; 
    d->rehashidx = -1;
    d->iterators = 0;          
    return DICT_OK;
}

static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

### Destroy
Clear hash tables ht[0] and ht[1] and then release dict memory.

```c
void dictRelease(dict *d)
{
    _dictClear(d,&d->ht[0],NULL);
    _dictClear(d,&d->ht[1],NULL);
    zfree(d);
}

int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    // traverse all slots
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        if (callback && (i & 65535) == 0) callback(d->privdata);

        if ((he = ht->table[i]) == NULL) continue; // null slot, continue
        while(he) { // iterate current slot
            nextHe = he->next;
            dictFreeKey(d, he);
            dictFreeVal(d, he);
            zfree(he);
            ht->used--;
            he = nextHe;
        }
    }
    zfree(ht->table); // free bucket array
    _dictReset(ht); // reset
    return DICT_OK;
}
```

### Add
Add pair <key, val> to dictionary: it calls function *dictAddRaw* to get an entry pointer first and then call function *dictSetVal* to set value.

```c
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key); // get entry
    dictSetVal(d, entry, val); // set value
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key)
{
    // get the index
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    // if rehashing, add to the second hash table
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];

    // allocate entry memory
    entry = zmalloc(sizeof(*entry));

    // add entry to slot index, similar to insert node to list head
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    // set the hash entry field
    dictSetKey(d, entry, key);
    return entry;
}

// be careful about the hook function
// all other hooks are used in the same way
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        entry->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        entry->v.val = (_val_); \
} while(0)
```

### Delete
There are two versions of delete node. The difference is whether free key memory.

```c
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0);
}

// no free tags means do not free key memory
int dictDeleteNoFree(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,1);
}

static int dictGenericDelete(dict *d, const void *key, int nofree)
{
    h = dictHashKey(d, key); // get hash result
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask; // get slot
        he = d->ht[table].table[idx]; // get first entry in slot idx

        // iterate current slot
        while(he) {
            if (dictCompareKeys(d, key, he->key)) {
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                }
                zfree(he);
                d->ht[table].used--;
                return DICT_OK;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break; // no need to traverse table 1 if no rehashing
    }
    return DICT_ERR; // not found
}
```

### Replace
*Replace* operation tries to add a pair <key, value> first, if *add* fails, it will find an entry and then set a new value.

```c
int dictReplace(dict *d, void *key, void *val)
{
    if (dictAdd(d, key, val) == DICT_OK) // try add
        return 1;

    entry = dictFind(d, key); // get entry
    /* Set the new value and free the old one. Note that it is important
     * to do that in this order, as the value may just be exactly the same
     * as the previous one. In this context, think to reference counting,
     * you want to increment (set), and then decrement (free), and not the
     * reverse. */
    auxentry = *entry;
    dictSetVal(d, entry, val);
    dictFreeVal(d, &auxentry);
    return 0;
}
```

## Rehashing
We need to adjust buckets(slot) size according to the number of entries used in hash table. That is called **auto-resize** or **rehashing**. For example, when hash table is in highly use (entries / slot_size is too large), we need to enlarge slot size to reduce conflict. Vice verse, if hash table is nearly not used, we need to shrink slot size to save memory.

Rehashing is controlled by two variables `dict_can_resize` and `dict_force_resize_ratio`.

```c
static int dict_can_resize = 1;
static unsigned int dict_force_resize_ratio = 5;
```

When `dict_can_resize` is set to 0, not all resizes are prevented: a hash table is still allowed to grow if the ratio between the number of elements and the `buckets > dict_force_resize_ratio`. This is just for efficiency. It can guarantee that there are at most 5 entries in a slot on average.

When dict is in the process of rehashing, ht[1] is in use. All newly added pairs will be added into this hash table. The core function to execute rehash operation is function **dictRehash**. It moves n slots from ht[0] to ht[1] each time according to the second parameter.

```c
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table. */
int dictRehash(dict *d, int n) {
    if (!dictIsRehashing(d)) return 0;

    while(n--) {
        dictEntry *de, *nextde;

        /* Check if we already rehashed the whole table... */
        if (d->ht[0].used == 0) {
            zfree(d->ht[0].table);
            d->ht[0] = d->ht[1];
            _dictReset(&d->ht[1]);
            d->rehashidx = -1;
            return 0;
        }

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }
    return 1;
}
```

However, for server response time, we should not finish rehash in just only one call. What we need is **incremental rehashing**.

Function `_dictRehashStep` moves one slot from ht[0] to ht[1] each time. 

```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```

**Incremental hashing** happens during following operations: add, delete, find, and getRandomKey.

```text
redis/src/dict.c|344| <<dictAddRaw>> if (dictIsRehashing(d)) _dictRehashStep(d);
redis/src/dict.c|408| <<dictGenericDelete>> if (dictIsRehashing(d)) _dictRehashStep(d);
redis/src/dict.c|487| <<dictFind>> if (dictIsRehashing(d)) _dictRehashStep(d);
redis/src/dict.c|622| <<dictGetRandomKey>> if (dictIsRehashing(d)) _dictRehashStep(d);
```

Except for controlling how many slots will be moved in a rehashing process. Here is another function **dictRehashMilliseconds** to control how much time rehashing will last.

```c
/* Rehash for an amount of time between ms milliseconds and ms+1 milliseconds */
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```
