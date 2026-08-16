---
title: "Why Can epoll Handle Millions of Connections?"
date: 2026-08-11
draft: false
tags:
  - Linux
  - Networking
  - epoll
  - Kernel
  - Reactor
categories:
  - Systems
description: "Understanding why Linux epoll scales to millions of TCP connections and how it differs from select and poll."
---
You may have heard that:

> Nginx can handle hundreds of thousands or even millions of TCP connections.
> But a natural question comes up:
> **How can one process manage so many sockets efficiently?**
> Does it check every connection repeatedly?
> The short answer:
> No.
> The reason is that **epoll changes the way we discover I/O events.**

---

## Why doesn't scanning all connections scale?

A simple approach would be:

```text
Check socket 1
Check socket 2
Check socket 3
...
Check socket 1,000,000

This is basically how select and poll work.

The problem is that most connections are usually idle.

For example:

1,000,000 TCP connections
999,900 idle connections
100 active connections

But the server still needs to walk through all file descriptors.

The cost grows with the number of connections:

O(N)

More connections mean more unnecessary work.

⸻

What does epoll change?

Instead of repeatedly asking:

“Is this socket ready?”

epoll tells the kernel:

“Watch these sockets. Let me know when something happens.”

The application registers interested file descriptors:

epoll_ctl()
        |
        v
Linux Kernel
        |
        v
Socket event happens
        |
        v
Ready event is recorded
        |
        v
epoll_wait() returns

The application only receives sockets that are actually ready.

⸻

What happens inside the kernel?

epoll maintains two important structures.

1. Interest List

The interest list stores:

Which file descriptors does the application care about?

Example:

fd 10
fd 11
fd 12
...
fd 1000000

The application registers these sockets through:

epoll_ctl()

⸻

2. Ready List

The ready list stores:

Which file descriptors currently have events?

Example:

fd 23456
fd 87654
fd 99999

When the application calls:

epoll_wait()

the kernel returns the ready events directly.

It does not scan one million connections again.

⸻

Who wakes epoll up when data arrives?

The Linux networking stack.

The simplified path:

Network packet arrives
        |
        v
NIC
        |
        v
Linux TCP/IP stack
        |
        v
Socket receive buffer
        |
        v
Wake up epoll
        |
        v
epoll_wait() returns

A socket becoming readable creates an event.

The application does not need to constantly check whether data has arrived.

⸻

Does epoll completely remove scanning?

Not exactly.

The important difference is:

What do we scan?

With select and poll:

Scan all connections

With epoll:

Process active events

The cost depends on:

Number of ready events

instead of:

Number of monitored connections

This is the key reason epoll scales much better.

⸻

Why does epoll support edge-triggered mode?

epoll provides two triggering modes:

* Level Triggered (LT)
* Edge Triggered (ET)

⸻

Level Triggered (LT)

The kernel keeps notifying while the condition exists.

Example:

Receive buffer has data
        |
        v
Application reads some data
        |
        v
Still has data
        |
        v
Notify again

LT is easier to implement because the kernel keeps reminding the application.

⸻

Edge Triggered (ET)

The kernel notifies only when the state changes.

Example:

No data
   |
   v
Data arrives
   |
   v
Notify once

ET reduces repeated notifications.

High-performance servers usually combine:

epoll + ET + non-blocking sockets

⸻

Is epoll alone enough for millions of connections?

No.

epoll solves one specific problem:

Efficiently finding ready I/O events.

A production-grade server also needs other design choices.

⸻

1. Non-blocking I/O

A traditional blocking model looks like:

One connection
        |
One thread

This does not scale.

For one million connections:

1 million connections
        ≠
1 million threads

Threads consume memory and introduce scheduling overhead.

⸻

2. Reactor Pattern

Many high-performance servers use the Reactor pattern:

              Event Loop
                   |
             epoll_wait()
                   |
            Handle events
                   |
              Repeat

The event loop waits for ready events and dispatches callbacks.

This is the architecture behind many scalable network servers.

⸻

3. Memory Management

A million TCP connections also consume system resources:

* file descriptors
* socket buffers
* TCP states
* kernel memory

Connection scalability is not only an I/O problem.

⸻

Why doesn’t Nginx create one thread per connection?

The old model:

One connection
        |
One thread

does not work at a large scale.

Instead, Nginx uses:

Few worker processes
        |
Event loop
        |
Millions of sockets

A small number of workers can efficiently manage a huge number of concurrent connections.

⸻

What does epoll have to do with High-Frequency Trading?

The goals are different.

Internet servers focus on:

Connection scale
Throughput

High-frequency trading systems focus on:

Latency
Jitter
Deterministic performance

A typical web server stack:

epoll
+
Reactor
+
Async I/O

A low-latency trading system may go further:

DPDK
+
Kernel bypass
+
Busy polling
+
Zero copy

⸻

The Key Idea

select and poll work like this:

“I keep checking every connection to see if something happened.”

epoll works like this:

“The kernel tells me which connections changed.”

The fundamental shift is:

Polling everything
        ↓
Receiving events

This is why Linux can efficiently support millions of concurrent TCP connections.

⸻

Further Questions

If epoll can handle millions of connections:

* Where does epoll_wait() sleep inside the Linux kernel?
* How does a network packet wake up an epoll event loop?
* Why can Redis achieve high performance with a single thread?
* How does a Reactor-based server manage connection lifecycle?

These questions lead deeper into Linux networking and high-performance server design.
```
