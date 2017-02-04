---
layout: post
title: "Mutex of pthread_cond_wait"
description: ""
category: linux
tags: []
---
{% include JB/setup %}

# Introduction

Question: `Why does pthread_cond_wait need a lock?`

We know that, when we use API `pthread_cond_wait`, we need to lock a mutex.

Like this:

```cpp
int gx = 0;
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;  
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;  

// thread A
...
pthread_mutex_lock(&mtx);
while (gx == 0) {
    pthread_cond_wait(&cond, &mtx);
}
...
pthread_mutex_unlock(&mtx);
```

```cpp
// thread B
...
pthread_mutex_lock(&mtx);
gx = 1;
pthread_mutex_unlock(&mtx);
pthread_cond_signal(&cond);
...
```
Why we need such a lock? Can we eliminate it? The answer is `NO` because
pthread library itself hides the potential problem for you.

# Analysis

According to man page:

```cpp
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
         pthread_mutex_t *restrict mutex,
         const struct timespec *restrict abstime);

int pthread_cond_wait(pthread_cond_t *restrict cond,
         pthread_mutex_t *restrict mutex);

These functions atomically release mutex and cause the calling thread to block
on the condition variable cond; atomically here means "atomically with respect
to access by another thread to the mutex and then the condition variable". That is,
if another thread is able to acquire the mutex after the about-to-block thread
has released it, then a subsequent call to  pthread_cond_broadcast() or pthread_cond_signal()
in that thread shall behave as if it were issued after the about-to-block thread has blocked.
```

The key point is `atomically`. That means `unlock and wait` should not be interrupted.
If we can guarantee this atomic property, we can guarantee that `behave as if it were issued
after the about-to-block thread has blocked.` Otherwise, there might be `signal lost`.

If there is no mutex in `pthread_cond_wait` function, we may write code like this:

```cpp
// thread A
...
pthread_mutex_lock(&mtx);
while (gx == 0) {
	pthread_mutex_unlock(&mtx); // step1
	pthread_cond_wait(&cond); // step2
	pthread_mutex_lock(&mtx);
}
...
pthread_mutex_unlock(mtx);
```

```cpp
// thread B
...
pthread_mutex_lock(&mtx);
gx = 1;
pthread_mutex_unlock(&mtx);
pthread_cond_signal(&cond); // step3
...
```

However, above code snippet has a problem as step1 and step2 are not `atomic` any more.
This could lead to a bad effect: `signal lost`.

For example, when finish executing step1, control switches back to thread B. And thread A won't
get control until thread B executes step3. After thread A gets control again, the signal was lost.

So how does the C library avoid it? 

# Library Implementaion

As I am familiar with Bionic C code in Android, I will use code from there directly.

```cpp
int pthread_cond_wait(pthread_cond_t* cond, pthread_mutex_t* mutex) {
	return __pthread_cond_timedwait(cond, mutex, NULL, COND_GET_CLOCK(cond->value));
}

int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t * mutex, const timespec *abstime) {
	return __pthread_cond_timedwait(cond, mutex, abstime, COND_GET_CLOCK(cond->value));
}

int __pthread_cond_timedwait(pthread_cond_t* cond, pthread_mutex_t* mutex, const timespec* abstime, clockid_t clock) {
	timespec ts;
	timespec* tsp;

	if (abstime != NULL) {
		if (__timespec_from_absolute(&ts, abstime, clock) < 0) {
			return ETIMEDOUT;
		}
		tsp = &ts;
	} else {
		tsp = NULL;
	}

	return __pthread_cond_timedwait_relative(cond, mutex, tsp);
}

int __pthread_cond_timedwait_relative(pthread_cond_t* cond, pthread_mutex_t* mutex, const timespec* reltime) {
	int old_value = cond->value; // ***this is the key point***

	pthread_mutex_unlock(mutex);

	// call system call with ***old value***
	int status = __futex_wait_ex(&cond->value, COND_IS_SHARED(cond->value), old_value, reltime);

	pthread_mutex_lock(mutex);

	if (status == -ETIMEDOUT) {
		return ETIMEDOUT;
	}
	return 0;
}

static inline int __futex_wake_ex(volatile void* ftx, bool shared, int count) {
	return __futex(ftx, shared ? FUTEX_WAKE : FUTEX_WAKE_PRIVATE, count, NULL);
}

static inline __always_inline int __futex(volatile void* ftx, int op, int value, const struct timespec* timeout) {
	// Our generated syscall assembler sets errno, but our callers (pthread functions) don't want to.
	int saved_errno = errno;
	int result = syscall(__NR_futex, ftx, op, value, timeout);
	if (__predict_false(result == -1)) {
		result = -errno;
		errno = saved_errno;
	}
	return result;
}
```

We can see that, the key point in implementation is that: `before unlock, it saves a copy of old value of
condition variable, and then use this old value to call system call futex. Even if there is a switch after
unlock, it can still know that signal was happened before.` 

Here is futex system call explanation:

```cpp
int futex(int *uaddr, int op, int val, const struct timespec *timeout,
          int *uaddr2, int val3);

FUTEX_WAIT
       This operation atomically verifies that the futex address uaddr still contains the value val, and sleeps awaiting FUTEX_WAKE
       on this futex address. If the timeout argument is non-NULL, its contents describe the maximum duration of the wait, which
       is infinite otherwise. The arguments uaddr2 and val3 are ignored.
```

# Conclusion

Library uses this mutex to guarantee that `unlock and wait` is atomic and not interrupted.
`Atomic` here is not like what we have already known before, it uses a trick way to avoid
`signal lost` bug. The library considers it carefully, hides the potential problem and provides easy API to use.
