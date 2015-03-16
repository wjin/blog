---
layout: post
title: "Redis Introduction"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Background
There are increasing data generated on the Internet every day. How to store, analyse and deal with them efficiently is becoming pretty important for those Internet Giants.

It is no surprise that many solutions coexist, such as well-known commercial RDBMS Oracle, Open Source RDBMS Mysql (Acquired by Oracle), and several kinds of NoSql, like MongoDB, Redis, HBase and so on. See more details here: [DB ranking](http://db-engines.com/en/ranking).

In this article, I will give a brief introduction about the most popular key-value store **Redis**.

# Introduction
Redis is an ***in-memory persistent key-value*** store. It can be used as a cache level for backend DB. Actually, it can be used anywhere you use Memcached.

It is **blazing fast** because all of its data  are in memory. Also, it writes data back to disk for persistence using either **RDB** or **AOF** (I will explain it in my following articles). Last but not least, it is much more than a key-value store because it exposes five different data structures (***string, list, set, sorted-set, hash***), which makes it be more popular than Memcached.

Besides, Redis Cluster is ongoing: [cluster specification](http://redis.io/topics/cluster-spec). 

# Install
It conforms to general *nix program installation:

```bash
git clone https://github.com/antirez/redis.git
cd redis
make
make test (optional)
make install (optional)
```


# Run
After build, you can directly start redis server by:

```bash
Weis-MacBook-Pro:redis eric$ ./src/redis-server
[5512] 01 May 01:18:52.548 # Warning: no config file specified, using the default config. In order to specify a config file use ./src/redis-server /path/to/redis.conf
[5512] 01 May 01:18:52.549 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.9.11 (11d9ecb7/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 5512
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[5512] 01 May 01:18:52.550 # Server started, Redis version 2.9.11
[5512] 01 May 01:18:52.550 * The server is now ready to accept connections on port 6379
```

And then run client to connect to redis server:

```bash
Weis-MacBook-Pro:redis eric$ ./src/redis-cli
127.0.0.1:6379>
```

There is a benchmark you can run to verify how fast redis is:

```bash
Weis-MacBook-Pro:src eric$ ./src/redis-benchmark
```

# Usage
As Redis is a key-val store, so keys and values are fundamental concepts. Below is a simple example to operate keys and values using Redis SET and GET command. You could reference all commands at : [http://redis.io/commands](http://redis.io/commands).

```bash
Weis-MacBook-Pro:~ eric$ redis-cli
127.0.0.1:6379> set test eric
OK
127.0.0.1:6379> get test
"eric"
127.0.0.1:6379>
```

# Notes
* Keys are always strings which identify pieces of data (values)
* Values are arbitrary byte arrays, they can be any other types.
