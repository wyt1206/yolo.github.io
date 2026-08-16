---
title: "Why Is epoll So Important? Why Do Nginx and Redis Depend on It?"
date: 2026-07-15
draft: false
tags:
  - Linux
  - epoll
  - Socket
  - Networking
  - I/O Multiplexing
  - Event Driven
categories:
  - Many WHYs
---
A high-performance server usually needs to handle thousands or millions of connections.

Examples:

- Nginx
- Redis
- Node.js
- Netty

A simple question:

> If thousands of clients connect at the same time, how does one server know which socket has data?

This leads to:

```

Socket

|

v

I/O Multiplexing

|

v

epoll

|

v

Event Loop
```

---

# 1. Why Does a TCP Server Need I/O Multiplexing?

A simple server shows the bottleneck directly.

Traditional blocking model:

```cpp
while(true)
{
    client = accept();

    read(client);

    process(client);
}
```

The problem:

While waiting for one client:

```
Client A

waiting for data
```

the server cannot handle:

```
Client B
Client C
Client D
```

---

A common solution:

One thread per connection.

```
Connection A -> Thread A

Connection B -> Thread B

Connection C -> Thread C
```

But with:

```
100,000 connections
```

we get:

* too many threads
* memory overhead
* context switch overhead
* scheduler pressure

---

The real question becomes:

> Can one thread monitor many sockets?

This is:

```
I/O Multiplexing
```

---

# 2. What Is I/O Multiplexing?

I/O multiplexing means:

> One thread monitors multiple file descriptors and handles whichever one becomes ready.

Instead of:

```
Thread
 |
 +-- socket 1
 |
 +-- socket 2
 |
 +-- socket 3
```

We have:

```
          Event Monitor

               |

   +-----------+-----------+

   |           |           |

 socket1    socket2     socket3
```

The application asks:

> Tell me which fd is ready.

---

Example:

```cpp
events = epoll_wait();
```

The kernel returns:

```
fd 5 readable

fd 8 writable

fd 10 error
```

The application handles them.

---

# 3. What Are the Implementations of I/O Multiplexing?

Linux provides several mechanisms:

```
select

   |

   v

poll

   |

   v

epoll

   |

   v

io_uring
```

They represent different generations.

---

# 4. How Does select() Work?

The oldest model:

```c
select(
    max_fd,
    readfds,
    writefds,
    exceptfds,
    timeout
);
```

The application provides:

```
fd set
```

Example:

```
fd:

3
4
5
6
7
```

Kernel checks:

```
Is fd 3 ready?

Is fd 4 ready?

Is fd 5 ready?

...
```

Then returns.

---

The problem:

Every call requires:

```
User Space

copy fd list

        |

        v

Kernel Space

scan everything
```

---

# 5. Why Did select() Become Insufficient?

## Problem 1: FD limit

select has:

```
FD_SETSIZE
```

Usually:

```
1024
```

Not enough for modern servers.

---

## Problem 2: O(n) scanning

Suppose:

```
100000 sockets
```

Only:

```
10 sockets ready
```

select still scans:

```
100000 entries
```

every time.

---

## Problem 3: Copy overhead

Every call:

```
Application

    |

    v

Kernel
```

copies fd sets.

---

# 6. Why Was poll() Introduced?

poll solved some problems.

Instead of:

```
fixed fd array
```

it uses:

```c
struct pollfd
```

Example:

```c
pollfd fds[10000];
```

Advantages:

* no 1024 limit
* more flexible

---

But the fundamental problem remains:

The kernel still scans everything.

Complexity:

```
O(n)
```

---

Example:

```
100000 sockets

10 ready

kernel checks all 100000
```

---

# 7. Why Was epoll Created?

The Linux kernel introduced:

```
epoll
```

to solve scalability problems.

The key idea:

> Do not ask the kernel to scan everything. Let the kernel maintain the ready list.

---

select/poll:

```
Application

send all fds

        |

        v

Kernel scans all
```

---

epoll:

```
Application

register fds once

        |

        v

Kernel remembers them


        |

event happens


        |

ready list returned
```

---

# 8. What Is the Biggest Difference Between epoll and select?

The fundamental difference:

## select/poll

```
Interest list

is managed by user
```

Every time:

```
send list
scan list
return result
```

---

## epoll

```
Interest list

is managed by kernel
```

Register once:

```
epoll_ctl()
```

Wait:

```
epoll_wait()
```

---

The model:

```
select:

User
 |
 | fd list
 v
Kernel
 |
 | scan
 v
Ready events


epoll:

User
 |
 | register
 v
Kernel

(saved state)

 |
 v

Ready events
```

---

# 9. How Is epoll Implemented Internally?

Inside kernel, epoll maintains:

## Interest List

Which fd are monitored.

Implemented using:

```
Red Black Tree
```

Why?

Because:

* fast insert
* fast delete
* fast lookup

---

## Ready List

Which fd are ready.

Implemented using:

```
linked list
```

---

Conceptually:

```
epoll instance


      |

      +----------------+

      |                |

Interest Tree     Ready List

(all fd)          (ready fd)
```

---

# 10. Complete epoll Workflow

## Step 1: Create epoll instance

```c
epoll_create()
```

Kernel creates:

```
epoll object
```

---

## Step 2: Register socket

```c
epoll_ctl(
    EPOLL_CTL_ADD,
    fd
)
```

Kernel stores:

```
fd

+

events to monitor
```

---

## Step 3: Wait

```c
epoll_wait()
```

Thread sleeps.

---

## Step 4: Event happens

Example:

Network packet arrives:

```
NIC

 |

 v

TCP Stack

 |

 v

Socket buffer
```

Kernel marks:

```
fd ready
```

and adds it:

```
Ready List
```

---

## Step 5: Return events

```c
epoll_wait()
```

returns:

```
fd 8 readable
fd 10 writable
```

---

## Step 6: Execute callback

User space:

```
fd

 |

 v

Channel

 |

 v

callback()
```

---

# 11. Why Is epoll Faster?

Because it avoids unnecessary work.

---

select:

```
Every time:

scan all fd
```

Cost:

```
O(n)
```

---

epoll:

```
Only return ready fd
```

Cost:

```
O(number of ready events)
```

---

Example:

```
100000 connections

10 active

```

select:

```
check 100000
```

epoll:

```
return 10
```

---

# 12. What Are epoll Working Modes?

epoll supports:

## Level Triggered (LT)

Default mode.

Meaning:

> As long as data exists, notify repeatedly.

Example:

Socket buffer:

```
100 bytes available
```

Application reads:

```
50 bytes
```

Remaining:

```
50 bytes
```

Next epoll_wait:

```
still returns readable
```

---

## Edge Triggered (ET)

Meaning:

> Notify only when state changes.

Example:

Data arrives:

```
0 bytes

 |

 v

100 bytes
```

Trigger event.

But:

Application must read everything.

Usually:

```c
while(read())
{
}
```

until:

```
EAGAIN
```

---

# 13. Relationship Between epoll and io_uring

epoll:

```
Event notification model
```

Application:

```
submit interest

wait event
```

---

io_uring:

```
Asynchronous submission/completion model
```

Instead of:

```
wait then execute
```

it allows:

```
submit operation

kernel completes it

receive completion
```

---

Comparison:

|                    | epoll                  | io_uring           |
| ------------------ | ---------------------- | ------------------ |
| Model              | readiness notification | async completion   |
| Kernel involvement | notify ready fd        | execute operations |
| Socket support     | yes                    | yes                |
| Disk IO            | limited                | excellent          |

---

# Final Knowledge Chain

Why does epoll exist?

```
Many connections

        |

        v

One thread cannot block

        |

        v

Need I/O multiplexing

        |

        v

select

        |

        v

poll

        |

        v

epoll

        |

        v

Event Loop

        |

        v

High Performance Server
```

---

# Conclusion

epoll is not just an API.

It represents a design change:

From:

```
Ask the kernel:

"Which socket is ready?"
```

to:

```
Kernel:

"I already know which sockets are ready."
```

This design enables:

* Nginx high concurrency
* Redis event loop
* modern C++ network frameworks

Understanding epoll is the bridge between:

```
Linux Kernel

        |

        v

Network Programming

        |

        v

High Performance System Design
```

````

---

加入后你的 ManyWhys 链：

```text
8. User Mode / Kernel Mode
          |
          v
9. Callback
          |
          v
10. Callback Registration
          |
          v
11. epoll
          |
          v
12. Reactor Pattern
          |
          v
13. C++ EventLoop Server
````
