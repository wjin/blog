---
layout: post
title: "TCP/IP Review"
description: ""
category: network
tags: []
---
{% include JB/setup %}

# Network

## Terms

**OSI**: Open Systems Interconnection

* Application layer

* Presentation Layer

* Session Layer

* Transport layer

* Network layer

* Data Link layer

* Physical layer

**TCP/IP**: Transmission Control Protocol and Internet Protocol

* Application layer

* Transport layer

* Internet layer

* Link layer

* Physical layer

**Abbreviation**

RFC: Request For Comment

MTU: Maximum Transmission Unit (Link)

RTT: Round-Trip Time

TTL: Time-to-Live (IP)

MSS: Maximum Segment Size (TCP)

MSL: Maximum Segment Lifetime (TCP)

IP datagram

UDP datagram

TCP segment

## IP Address

![ip class](/assets/img/post/ip_class.png)

![ip range](/assets/img/post/ip_range.png)

The IPv4 address space was originally divided into **five classes**. Classes A, B, and C were used for
assigning addresses to interfaces on the Internet (**unicast** addresses) and for some other special-case uses. 

The classes are defined by the first few bits in the address: 0 for class A, 10 for class B, 110 for class C, and so on.
Class D addresses are for **multicast** use, and class E addresses remain reserved.

**Host portion** can be divided into a `subnetwork (subnet) number` and a `host number`.
And subnet mask is used to partition host (zero bit is host bit).

## IP (ICMP, IGMP, ARP, RARP)

![ip](/assets/img/post/ip_protocol.png)

Big endian

Header is 20 bytes if no optional entry

TTL: Time-to-Live, sets an upper limit on the number of routers through which a datagram can pass (hop number).

IP fragmentation and reassembly:

IP Routing Order:

> host -> subnet -> net number -> default routing or fail

IGP: RIP

EGP: 

ARP: IP -> MAC

RARP: MAC -> IP

ICMP:

`Ping`: send icmp type 0 packet; receive icmp type 8 packet

`Traceroute`: using ip ttl, each time ttl + 1, distinguish result: Time Exceeded and Port Unreachable.

The traceroute tool is used to determine the routers used along a path from a sender to a destination.

The approach involves sending datagrams *first with an IPv4 TTL field set to 1* and allowing the expiring datagrams
to induce routers along the path to send ICMPv4 **Time Exceeded (code 0) messages**.

Each round, the sending TTL value is **increased by 1**, causing the routers that are one hop farther to
expire the datagrams and generate ICMP messages.

## UDP
	
![udp](/assets/img/post/udp.png)

* Simple, Unreliable, Fixed header (8 bytes)

* Broadcast and multicast use UDP as it is one to many relationship

* Path MTU Discovery with UDP

## TCP

![tcp](/assets/img/post/tcp.png)

TCP provides a **reliable**, **connection-oriented**, **byte stream**, transport-layer service.

**Reliable:** 

* Acknowledge data packet

* Time out retransmission

* Flow control and congestion control

### Connection

![set up connection](/assets/img/post/initiate_connection.png)

![close connection](/assets/img/post/close_conection.png)

normal open: three-way handshake

normal close: four-way handshake

simultaneous open and close: both are four-way handshake

half close: can still receive data

**How about the third packet (ACK) lost when initiating a connection?**

1. if client send data to server, server replys with RST.

2. if client does not send data until server retransmission time out, server will re-send the second packet (SYN-ACK).

### State Transition

![tcp](/assets/img/post/tcp_state.png)

### Interactive Communication

**Delayed acknowledgments**

Interactive data is normally transmitted in segments smaller than the MSS. **Delayed acknowledgments** (200ms) may
be used by the **receiver** of these small segments to see if the acknowledgment can be piggybacked along with data
going back to the sender.

This often reduces the number of segments, however, it may introduce additional delay.

**Nagle algorithm**

This algorithm **limits the sender to a single small packet of unacknowledged data at anytime**. 
On connections with relatively large round-trip times, such as WANs, the **Nagle algorithm** is often used to **reduce the number of small segments**.
However, it adds delay that is sometimes unacceptable to applications. We can use `TCP_NODELAY` to disable Nagle algorithm.

### Bulk Data Communication


![sliding window](/assets/img/post/sliding_window.png)
	
`Sliding Window` is used to deal with bulk data.

1. The window closes as the **left edge advances** to the right. This happens when data that has been sent is **acknowledged** and the window size gets smaller.

2. The window opens when the **right edge moves to the right**, allowing more data to be sent. This happens when the receiving process on the other end
**reads acknowledged data**, freeing up space in its TCP receive buffer (kernel).

3. The window shrinks when the right edge moves to the left. RFC strongly discourages this.

**Two problems:**

1. Zero Window and then following Window Update packet lost

2. Silly Window Syndrome (SWS)

**Solution:**

For the first problem, there is a `Persist Timer` to deal with the loss of `Window Update` packet.

For SWS, **small data segments** are exchanged across the connection instead of full-size segments.
This leads to undesirable **inefficiency** because each segment has relatively **high overhead**—a small
number of data bytes relative to the number of bytes in the headers. We can fix it on both receiver and sender sides:

* For receiver, small windows are not advertised.

* For sender, small segments are not sent and the Nagle algorithm governs when to send.

### Congestion Control

Congestion is a situation that a router is forced to **discard data** because it cannot handle the arriving traffic rate.

In TCP, an assumption is made that **a lost packet** is an indicator of congestion.

`Slow Start` and `Congestion Avoidance` algorithms are triggered when loss is detected,
either by the fast retransmit algorithm or by retransmission timeouts.

**Slow Start**

The purpose of slow start is to help TCP **find a value for cwnd** before probing for more available bandwidth using congestion avoidance.

Typically, a TCP begins a new connection in slow start, eventually drops a packet, and then settles into steady-state operation using the congestion avoidance algorithm.

It can also be triggered when a timeout-based retransmission occurs.

**Congestion Avoidance**

There is a congestion window at the sender called `cwnd`. A standard TCP limits its window to the minimum of cwnd and sliding window.

**Selecting between Slow Start and Congestion Avoidance**

Slow start grows the value of the congestion window **exponentially with time**, and congestion avoidance grows it about **linearly with time**.

Only **one** of the two algorithms is in operation at any one time, and this decision is made by comparing the current value of the
congestion window to the slow start threshold **ssthresh**: 

* cwnd < ssthresh, slow start is used

* cwnd > ssthresh, congestion avoidance is used

The value of **ssthresh** is not fixed but instead **varies** over time. Its main purpose is to remember the last “best” estimate
of the operating window when no loss was present.

### Timer

In my humble opinion, timer is used to deal with `exception`, such as time out, packet loss, crash or failure, to guarantee TCP Reliability.

Pure `ACK` packet and `Window Update` packet do not include data, so they are not acknowledged by the peer and not reliable either.

**Retransmission**

Deal with `time out`.

TCP measures the RTT and then uses these measurements to keep track of a smoothed RTT estimator and a smoothed mean deviation estimator.
These two estimators are then used to calculate the next retransmission timeout value.

Without the Timestamps option, a TCP measures only a single RTT per window of data. Karn’s algorithm removes the retransmission
ambiguity problem by preventing the use of RTT measurements for segments that have been lost.

Today, most TCPs use the Timestamps option, which permits each segment to be individually timed.

**2MSL**

Deal with the `loss of ACK` that would acknowledge FIN.

When entering TIME_WAIT state, wait for 2 MSL time so that it can send the last ACK due to its loss.
The other side will send the FIN again due to time out, so when receiving the second FIN , this side can still send ACK.

**Persist**

Deal with the `loss of Window Updates Packet` that would open the window.

Considering sliding window, when the receiver once again has space available, it provides a window update
to the sender to indicate that data is permitted to flow once again. Because such updates do not generally
contain data** (they are a form of “pure ACK”), they are not reliably delivered by TCP.

TCP must therefore handle the case where such window updates that would open the window are lost.

If an acknowledgment is lost, we could end up with both sides waiting for
the other. To prevent this form of **deadlock** from occurring, the `sender` uses a persist timer to
query the receiver periodically, **to find out if the window size has increased**.

**Keepalive**

Deal with one side `crash or failure`.

The keepalive feature was originally intended for **server applications** that might **tie up resources**
and want to know **if the client host crashes or goes away**. Of course, it can be used on client side as well
even though it is uncommon.

# Reference

* TCP/IP Illustrated, Volume 1, Second Version

