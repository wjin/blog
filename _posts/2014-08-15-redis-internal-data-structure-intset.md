---
layout: post
title: "Redis Internal Data Structure: Intset"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
Intset is used to store **sorted**, **non-duplicate** integer data. 

It can store three types of integers: int16, int32 and int64. It can upgrade automatically if a new added value is beyond current encoding (overflow). Actually, two types of upgrade: **int16->int32** and **int32->int64**. 

During add or delete, it will move elements so time complexity is **O(n)**, However, it uses **memmove** function to move elements, and many runtime libraries have optimisation to this memory operation, so it is faster than general move int element one by one. 

Besides, it supports both **little endian** and **big endian**.

Here is intset data structure overview:

```text
+----------+--------+-----------+------------+-----------+
| encoding | length | first int | second int |    ...    |
+----------+--------+-----------+------------+-----------+
                    |
                    `-> data stored here
```

# Implementation

Related files: **intset.h** and **intset.c**

## Data Structure

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
    uint32_t encoding; // encoding
    uint32_t length; // number of integers
    int8_t contents[]; // data store here
} intset;
```

## Initialization
Default encoding is int16, it can be upgraded to int32 and then int64 automatically.

```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```
Function *intrev32ifbe* is used to deal with endian problem. It will reverse int32 byte by byte in big endian and do nothing in little endian. There are many other similar functions.

```c
#if (BYTE_ORDER == LITTLE_ENDIAN)
#define memrev16ifbe(p)
#define memrev32ifbe(p)
#define memrev64ifbe(p)
#define intrev16ifbe(v) (v)
#define intrev32ifbe(v) (v)
#define intrev64ifbe(v) (v)
#else
#define memrev16ifbe(p) memrev16(p)
#define memrev32ifbe(p) memrev32(p)
#define memrev64ifbe(p) memrev64(p)
#define intrev16ifbe(v) intrev16(v)
#define intrev32ifbe(v) intrev32(v)
#define intrev64ifbe(v) intrev64(v)
#endif
```

## Destroy
Actually, there is no destroy API in intset. According to initialization process, what you only need to do is to release dynamic memory when destroying.

## Find
As data are sorted in intset, so we can use **binary search** to find an element. **intsetSearch** will get the right position for *value* even if *value* is not in the set.

```c
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value); // get encoding for value
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}

static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    // binary search
    while(max >= min) {
        mid = min+((max-min) >> 1); // (min+max)/2;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    // update position
    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

## Add
Function *intsetAdd* first verify whether current encoding can store *value*, if not, it will automatically upgrade encoding. Otherwise, it will check whether *value* is already in the array as it does not store duplicate data. After that, it will adjust array memory, move elements, and then insert value. As element movement, so time complexity is **O(n)**. However, it uses **memmove** to move elements, so it is pretty fast as there are many optimisations against those mem* functions in libc. Also, we should not use **memcpy** here because of data overlap.

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value); // get corresponding encoding for value
    uint32_t pos;
    if (success) *success = 1;

    if (valenc > intrev32ifbe(is->encoding)) { // cannot store, upgrade encoding
        return intsetUpgradeAndAdd(is,value); 
    } else { // can store value with current encoding
        if (intsetSearch(is,value,&pos)) { // duplicate
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1); // realloc one more entry

        // if insert position is not the last pos, need to move elements
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); 
    }

    _intsetSet(is,pos,value); // set value
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // length++
    return is;
}

static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    uint32_t bytes = intrev32ifbe(is->length)-from; // calculate move bytes
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    memmove(dst,src,bytes);
}
```


## Delete
Move elements after *delete* position, so time complexity is O(n).

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos); // move tail element
        is = intsetResize(is,len-1); // length--
        is->length = intrev32ifbe(len-1); // update length
    }
    return is;
}
```

## Upgrade
When calling function *intsetUpgradeAndAdd*, it means that current encoding cannot store *value*(overflow). So after upgrade, *value* must be the **min** or **max** element. Local variable **prepend** is used to control insert position either at the beginning(min) or ending(max).


```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding); // get current encoding
    uint8_t newenc = _intsetValueEncoding(value); // new encoding
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0; // very tricky :(

    is->encoding = intrev32ifbe(newenc); // set new encoding
    is = intsetResize(is,intrev32ifbe(is->length)+1); // allocate one more entry memory

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */

    // get data according to old encoding and then insert
    // them as new encoding, no override
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value); // minimum element
    else
        _intsetSet(is,intrev32ifbe(is->length),value); // max element

    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // update length
    return is;
}
```
