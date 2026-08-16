---
title: "CPU Cache Line: Why Does a 64-Byte Block Matter?"
date: 2026-07-08
draft: false
tags:
  - CPU
  - Cache
  - MESI
  - False Sharing
  - Performance
categories:
  - Many WHYs
---
Many engineers know:

> CPU has cache because memory is slow.

But a deeper question is:

> Why does CPU cache data in fixed-size blocks?
>
> Why is this block usually 64 bytes?
>
> Why can two unrelated variables affect each other's performance?

To answer these questions, we need to understand:

```

Cache Line
|
v
Multi-core Cache Coherence
|
v
MESI Protocol
|
v
False Sharing
|
v
Prefetching
```

---

# 1. Why Does CPU Need Cache?

Modern CPUs are extremely fast.

However, main memory is much slower.

If the CPU had to wait for memory access every time:

```

Instruction
```

  |

```

Read Memory
```

  |

```

Wait
```

the CPU pipeline would frequently stall.

To reduce this gap, CPUs introduce multiple levels of cache:

```

CPU Core

|

L1 Cache

|

L2 Cache

|

L3 Cache

|

Main Memory
```

The closer the cache is to the CPU core, the faster it is.

---

# 2. Why Doesn't CPU Load Only One Variable?

Suppose we execute:

```c
int x = array[0];
```

A simple idea would be:

> Just load array[0].

But CPUs do not work this way.

They exploit two important properties of programs.

---

## 2.1 Temporal Locality

If data was accessed recently:

```
x
```

it is likely to be accessed again soon.

---

## 2.2 Spatial Locality

If we access:

```
array[0]
```

we are likely to access:

```
array[1]
array[2]
array[3]
```

soon.

Therefore, loading nearby data together improves performance.

This leads to the concept of:

> Cache Line

---

# 3. What Is a Cache Line?

A cache line is the smallest unit of data transferred between:

```
Memory

   <-->

CPU Cache
```

Modern CPUs usually use:

```
64 bytes
```

as a cache line size.

For example:

Memory:

```
Address

1000  variable A
1004
1008
1012
1016
...
103F
```

The CPU does not load only:

```
variable A
```

Instead, it loads:

```
1000 ~ 103F
```

as one block.

This block is called:

```
Cache Line
```

---

# 4. Why Does Cache Line Size Matter?

Consider:

```c
struct Data {

    int a;

    int b;

};
```

Memory layout:

```
+----------------+
| a              |
| b              |
| other data     |
+----------------+
```

When CPU loads `a`:

it also loads `b`.

Even if the program does not immediately need `b`.

This is beneficial because nearby data is likely to be used soon.

---

# 5. What Happens With Multiple CPU Cores?

Modern servers have multiple CPU cores:

```
        CPU

+----------------+
| Core 0         |
| Cache          |
+----------------+

+----------------+
| Core 1         |
| Cache          |
+----------------+

        |

     Memory
```

Each core has its own cache.

Now consider:

```
Core 0:

x = 10


Core 1:

reads x
```

Both cores may have their own copy of `x`.

The system needs a way to keep these copies consistent.

This is called:

> Cache Coherence

---

# 6. Why Do We Need MESI?

MESI is a cache coherence protocol.

It defines four states:

```
M - Modified

E - Exclusive

S - Shared

I - Invalid
```

---

## Modified

A cache line has been changed locally.

Example:

```
Core 0 Cache:

x = 20

State:

Modified
```

Memory may still contain:

```
x = 10
```

---

## Shared

Multiple cores have the same data.

Example:

```
Core 0:

x = 10


Core 1:

x = 10
```

---

## Invalid

The cached copy is no longer valid.

Example:

Core 0 updates:

```
x = 20
```

Core 1's old copy:

```
x = 10
```

must become:

```
Invalid
```

---

# 7. What Is False Sharing?

Now we reach the interesting problem.

Consider:

```c
struct Counter {

    int counterA;

    int counterB;

};
```

Memory:

```
Cache Line

+----------------+
| counterA       |
| counterB       |
| padding        |
+----------------+
```

Two threads:

Thread 1:

```c
counterA++;
```

Thread 2:

```c
counterB++;
```

They modify different variables.

It looks safe.

But:

Both variables are inside the same cache line.

---

What happens?

Thread 1 modifies:

```
counterA
```

The entire cache line becomes:

```
Modified
```

Thread 2 wants to modify:

```
counterB
```

But its cache line is now invalid.

The CPU must synchronize:

```
Core 0

   |
   v

Invalidate Cache Line

   |
   v

Core 1
```

The two cores keep invalidating each other's cache.

This is called:

> False Sharing

---

# 8. How Do We Avoid False Sharing?

The common solution is:

## Cache Line Padding

Example:

```c
struct Counter {

    int counterA;

    char padding[60];

    int counterB;

};
```

Now:

```
Cache Line 1

counterA


Cache Line 2

counterB
```

Each thread modifies a different cache line.

Linux kernel and high-performance systems often use cache-line alignment techniques.

---

# 9. Can CPU Prepare Data Before We Need It?

Cache misses are expensive.

So another question:

> Can CPU predict what data we need next?

The answer is:

Yes.

This mechanism is called:

> Hardware Prefetching

---

Example:

```c
for(int i = 0; i < 10000; i++)
{
    sum += array[i];
}
```

The CPU observes:

```
array[0]

array[1]

array[2]

array[3]
```

The access pattern is predictable.

So it may load:

```
array[10]
array[11]
```

before the program requests them.

When the program needs them:

they are already in cache.

---

# 10. Why Doesn't Prefetch Always Work?

Prefetch works well with predictable memory access.

Example:

Array:

```
A
B
C
D
```

Easy to predict.

But linked lists:

```
Node A

   |

Node F

   |

Node C

   |

Node X
```

have unpredictable memory locations.

The CPU cannot easily guess the next node.

Therefore:

Cache-friendly structures:

```
array
vector
ring buffer
```

often outperform:

```
linked list
```

---

# 11. Why Does This Matter for High Performance Systems?

Cache Line concepts directly affect:

* Operating systems
* Networking systems
* Lock-free data structures
* HFT systems

For example, a lock-free queue:

```
Producer Thread

        |

    Cache Line

        |

Consumer Thread
```

A poor memory layout can create unnecessary cache invalidations.

A good layout can significantly reduce latency.

---

# Final Knowledge Chain

The complete reasoning chain:

```
Memory is slow

        |

        v

CPU introduces Cache

        |

        v

Cache transfers data in blocks

        |

        v

Block = Cache Line (64 bytes)

        |

        v

Multiple cores need consistency

        |

        v

MESI Protocol

        |

        v

False Sharing

        |

        v

Padding and Memory Alignment

        |

        v

High Performance Computing
```

---

# Conclusion

A cache line looks like a simple hardware detail:

> CPU loads 64 bytes at a time.

But this small design decision affects:

* how CPUs communicate
* how multi-threaded programs scale
* why false sharing happens
* how lock-free systems are designed
* why HFT engineers care about memory layout

Understanding cache lines is the bridge between:

```
Software Code

      |

Operating System

      |

CPU Hardware
```

This is why performance engineering requires understanding not only algorithms, but also how code executes on real hardware.
