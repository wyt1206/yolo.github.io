---
title: "CPU Cache: The Small Warehouse Inside the CPU"
date: 2026-07-11
draft: true
tags:
  - CPU
  - Cache
  - Memory
  - Computer Architecture
  - Performance
categories:
  - Many WHYs
---

# CPU Cache: The Small Warehouse Inside the CPU

When we write programs, we often think:

```

CPU executes instructions
|
v
Memory provides data

```

But modern CPUs are much faster than memory.

A CPU can execute billions of instructions per second, while accessing main memory may take hundreds of CPU cycles.

This creates a fundamental problem:

> How can CPU avoid waiting for slow memory?

The answer is:

> CPU Cache

This leads to a chain of questions:

```

What is CPU Cache?

```
    |
```

Why does CPU need Cache?

```
    |
```

Why isn't Cache as large as Memory?

```
    |
```

Why do we have L1, L2, L3?

```
    |
```

How does CPU find data?

```
    |
```

What is Cache Miss?

```
    |
```

Why does CPU load blocks instead of bytes?

```
    |
```

Why is the block size 64 bytes?

```
    |
```

Why don't we have L4 Cache?

```

---

# 1. What Is CPU Cache?

CPU Cache is a small and fast memory located close to the CPU core.

The memory hierarchy looks like:

```

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

             |

           Disk
```

```

The closer the storage is to the CPU:

- the faster it is
- the smaller it is
- the more expensive it is

---

A simple analogy:

Imagine you are working at a desk.

You have:

```

Desk

|
|
Bookshelf

|
|
Warehouse

```

The desk contains the things you use frequently.

The warehouse contains everything.

CPU Cache works the same way:

```

Cache = CPU's small warehouse

```

---

# 2. Why Does CPU Need Cache?

The main reason:

> CPU is too fast compared with memory.

Consider:

```

CPU

```
|
|
v
```

Memory

```

Every memory access requires waiting.

The CPU pipeline may become:

```

Execute instruction

```
    |
```

Request data

```
    |
```

Wait for memory

```
    |
```

Continue execution

```

The CPU spends time doing nothing.

---

Cache solves this problem.

Instead of:

```

CPU
|
|
Memory

```

we use:

```

CPU

|

Cache

|

Memory

```

Frequently used data stays close to the CPU.

---

# 3. Why Isn't Cache As Large As Memory?

A natural question:

> If Cache is faster, why don't we make all memory Cache?

Because:

## 1. Cache is expensive

Cache uses very fast memory technology:

```

SRAM

```

Main memory uses:

```

DRAM

```

SRAM is:

- faster
- more expensive
- larger in hardware area


---

## 2. Cache consumes more power

A huge cache would require:

- more transistors
- more energy
- more CPU area


---

## 3. Bigger is not always faster

A larger cache means:

- more complex lookup
- longer access time

The CPU needs a balance between:

```

Speed

Capacity

Cost

```

---

# 4. Why Do We Have L1, L2, and L3 Cache?

Because one cache cannot satisfy all requirements.

CPU needs:

- extremely fast access
- reasonable storage size
- acceptable cost


So cache is organized into levels.

---

# L1 Cache

Closest to CPU core.

Characteristics:

```

Very small

Very fast

```

Usually:

```

tens of KB

```

Each CPU core usually has its own L1.

---

# L2 Cache

Larger than L1.

Slower than L1.

Usually:

```

hundreds of KB

```

Often private to each core.

---

# L3 Cache

Shared among CPU cores.

Characteristics:

```

Much larger

Slower

Shared

```

Usually:

```

several MB

```

Example:

```

Core 0
|
L1

Core 1
|
L1

Core 2
|
L1

```
  |
  v

L3 Cache
```

```

---

# 5. How Does CPU Know Where Data Is?

Suppose the CPU wants:

```

variable x

```

Where is it?

Possible locations:

```

L1 Cache?

L2 Cache?

L3 Cache?

Memory?

```

CPU uses cache lookup mechanisms.

The process is:

```

CPU requests address

```
    |

    v
```

Check L1 Cache

```
    |

    v
```

If missing

```
    |

    v
```

Check L2

```
    |

    v
```

Check L3

```
    |

    v
```

Access Memory

```

---

This creates:

```

Cache Hit

or

Cache Miss

```

---

# 6. What Is Cache Miss?

A Cache Miss happens when requested data is not found in cache.

Example:

CPU requests:

```

x

```

But:

```

L1:

not found

L2:

not found

L3:

not found

```

Then CPU must access:

```

Main Memory

```

This is much slower.

---

The performance difference:

```

Cache Hit:

few CPU cycles

Cache Miss:

hundreds of cycles

````

Therefore:

> High-performance systems care deeply about cache misses.

---

# 7. Why Doesn't CPU Read One Byte At A Time?

Suppose:

```c
char x;
````

Why not load:

```
1 byte
```

only?

Because programs usually access nearby data.

This is called:

## Spatial Locality

Example:

```c
for(i=0;i<1000;i++)
{
    sum += array[i];
}
```

When CPU reads:

```
array[0]
```

it expects:

```
array[1]
array[2]
array[3]
```

will probably be used soon.

---

Therefore CPU loads a block of data.

This block is:

```
Cache Line
```

---

# 8. Why Is Cache Line Usually 64 Bytes?

Modern CPUs usually use:

```
64 bytes
```

as the cache line size.

Why?

Because it is a balance.

Too small:

```
More memory requests

Less benefit from locality
```

Too large:

```
Waste bandwidth

Load unnecessary data
```

64 bytes provides a good tradeoff between:

```
Memory efficiency

Hardware complexity

Performance
```

---

Example:

Instead of:

```
Read:

int a
```

CPU reads:

```
Cache Line:

a
+
nearby variables
+
array elements
```

---

# 9. Why Don't We Have L4 Cache?

Some systems have experimented with L4 cache.

But mainstream CPUs usually stop at:

```
L1
L2
L3
```

Why?

Because after L3:

The benefits become smaller.

The distance from CPU matters.

A larger cache:

* increases lookup complexity
* increases latency
* increases power consumption

Modern designs often prefer:

```
More efficient L1/L2/L3

+

Better memory controllers

+

Prefetching
```

instead of adding another cache level.

---

# 10. Why Does This Matter for System Engineers?

Cache behavior affects:

* operating systems
* databases
* networking
* high-frequency trading
* lock-free programming

For example:

A program with random memory access:

```
Node A

 |

Node X

 |

Node B
```

causes many cache misses.

A program with sequential access:

```
array[0]

array[1]

array[2]
```

is cache friendly.

---

# Final Knowledge Chain

The complete reasoning chain:

```
CPU is faster than Memory

        |

        v

Memory access becomes bottleneck

        |

        v

CPU introduces Cache

        |

        v

Multiple Cache Levels

        |

        v

L1 / L2 / L3 hierarchy

        |

        v

Cache Lookup

        |

        v

Cache Hit or Cache Miss

        |

        v

Data loaded by Cache Line

        |

        v

64-byte optimization
```

---

# Conclusion

CPU Cache looks like a simple optimization:

> Put frequently used data closer to CPU.

But it affects almost every layer of systems:

```
Application

     |

Operating System

     |

CPU Architecture

     |

Hardware
```

Understanding CPU Cache explains:

* why memory layout matters
* why cache misses hurt performance
* why cache lines exist
* why high-performance systems care about hardware details

The code you write is eventually executed by hardware.

To build high-performance systems, you need to understand the path from:

```
Code

 |

CPU

 |

Cache

 |

Memory
```