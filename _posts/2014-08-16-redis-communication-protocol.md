---
layout: post
title: "Redis Communication Protocol"
description: ""
category: redis
tags: []
---
{% include JB/setup %}

# Introduction
Redis clients communicate with Redis server using a protocol called **RESP** (REdis Serialization Protocol).

RESP is simple, fast and human readable. Also, it is binary safe and can serialize different data types like integers, strings, arrays. It is an application layer protocol over tcp as client connects to Redis server using a **TCP connection** to the default port 6379.

Note: RESP is only used for client-server communication. Redis Cluster uses a different binary protocol in order to exchange messages between nodes.

# Protocol Specification

## Data Type
In RESP, there are five basic types to identify different data, the difference is the first byte character:

 * Simple Strings : +
 * Errors : -
 * Integers : :
 * Bulk Strings : $
 * Bulk Arrays : *

## Bulk String
Bulk Strings are encoded in the following way: 

 * A "$" byte
 * Number of bytes composing the string
 * CRLF
 * Actual string data
 * CRLF

Example:

 * normal bulk string: "$6\r\nfoobar\r\n"
 * empty string: "$0\r\n\r\n"
 * null string:  "$-1\r\n"

## Bulk Array
Bulk Arrays are sent using the following format: 

  * A "*" byte
  * Number of elements in the array
  * CRLF

Following with additional RESP type for every element of the Array.

Example:

 * null array: "*-1\r\n"
 * empty array : "*0\r\n"
 * Integer array: "*3\r\n:1\r\n:2\r\n:3\r\n"
 * string array: "*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"
 * mix array: "*2\r\n$3\r\nfoo\r\n:1\r\n" 
 * array array:

```text
*2\r\n
*3\r\n
:1\r\n
:2\r\n
:3\r\n
*2\r\n
+Foo\r\n
-Bar\r\n
```

 * null element in array:

```text
*3\r\n
$3\r\n
foo\r\n
$-1\r\n
$3\r\n
bar\r\n
```

# Request and Response
 * Clients send commands to a Redis server as a **RESP Array** of Bulk Strings.
 * Server replies with **one of the RESP types** according to the specific command.

# Pipelining
Clients and Servers are connected via networking link. Whatever the network latency is, there is a time for the packets to travel from the client to the server, and back from the server to the client to carry the reply. This time is called **RTT** (Round Trip Time). This can heavily affect the server performance.

Pipelining sends **multiple commands** to the server without waiting for the replies at all, and finally read the replies in a single step. So there is no RTT overhead for each command, just one RTT for all commands together.

NOTE: while the client sends commands using pipelining, the server will be forced to queue the replies, using memory. For saving memory, just send a reasonable number of commands, like **10k** commands each time.

# Reference
 * [http://redis.io/topics/protocol](http://redis.io/topics/protocol)
 * [http://redis.io/topics/pipelining](http://redis.io/topics/pipelining)
