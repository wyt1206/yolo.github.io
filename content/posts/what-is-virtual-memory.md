---
title: "Virtual Memory Explained: From Address Translation to Huge Pages"
date: 2026-07-19
draft: false
tags:
  - Linux
  - Virtual Memory
  - Page Table
  - TLB
  - Page Fault
  - Huge Page
  - Memory
categories:
  - Many WHYs
---
Every program uses memory.

When we write:

```cpp
int x = 10;
```

we think:

```
x exists somewhere in memory
```

But the address used by the program is not the real RAM address.

The CPU sees:

```
Virtual Address
```

not:

```
Physical Address
```

The question is:

> How does Linux create the illusion that every program has its own large, continuous memory?

The answer:

```
Virtual Memory
```

---

# 1. What Is Virtual Memory?

Virtual Memory is an abstraction layer between:

```
Application
```

and:

```
Physical RAM
```

The application sees:

```
Virtual Address Space
```

The hardware sees:

```
Physical Memory
```

The mapping:

```
Virtual Address

        |

        v

Page Table

        |

        v

Physical Address
```

---

Example:

Process A:

```
Virtual Address

0x1000

        |

        v

RAM

0x800000
```

Process B:

```
Virtual Address

0x1000

        |

        v

RAM

0x900000
```

Both processes use:

```
0x1000
```

but access different memory.

---

# 2. How Is Continuous Virtual Memory Created?

A common question:

> My program memory looks continuous. Is RAM also continuous?

No.

Virtual memory creates this illusion.

---

Example:

Application sees:

```
Virtual Memory

Page 1
Page 2
Page 3
Page 4
```

But physical RAM may be:

```
Frame 100

Frame 500

Frame 20

Frame 900
```

The pages are scattered.

The Page Table connects them.

```
Virtual Page

      |

      v

Physical Frame
```

---

So:

Virtual memory is continuous.

Physical memory does not need to be.

---

# 3. How Does CPU Translate Addresses So Quickly?

Every memory access needs translation:

```
Virtual Address

        |

        v

Physical Address
```

Doing a full page table lookup every time would be too slow.

---

Therefore CPU has:

```
TLB
Translation Lookaside Buffer
```

TLB is a cache for address translation.

Example:

```
Virtual Page

      |

      v

Physical Frame
```

stored inside CPU.

---

Fast path:

```
CPU

 |

TLB lookup

 |

Physical Memory
```

Very fast.

---

# 4. What Is TLB Miss?

TLB only stores frequently used translations.

If translation is not inside TLB:

```
TLB Miss
```

happens.

Flow:

```
CPU

 |

TLB lookup

 |

Miss

 |

Page Table lookup

 |

Update TLB

 |

Access Memory
```

---

Why is it slower?

Because now CPU needs extra memory accesses.

---

Without TLB:

```
Memory access

=

Page Table lookup

+

Memory access
```

---

With TLB:

```
Memory access

=

TLB lookup

+

Memory access
```

---

# 5. Why Is Page Fault So Slow?

Important:

TLB miss != Page fault

They are different.

---

TLB Miss:

```
Page exists in RAM

but translation is missing
```

Cost:

```
extra lookup
```

---

Page Fault:

```
Page is not available
```

Example:

```
Virtual Page

        |

        v

Not in RAM
```

CPU triggers:

```
Page Fault Exception
```

---

Kernel handles:

```
CPU

 |

Page Fault

 |

Kernel

 |

Find page

 |

Load from disk

 |

Update Page Table

 |

Resume process
```

---

If disk access happens:

RAM:

```
~100 ns
```

Disk:

```
milliseconds
```

Huge difference.

---

# 6. How Do We Avoid Page Fault?

The best page fault is:

```
the one that never happens
```

Methods:

---

## Keep Working Set in RAM

Programs should reuse memory.

This improves:

```
Locality
```

---

## Use Memory Mapping Efficiently

Example:

```
mmap()
```

allows controlled memory usage.

---

## Avoid Swap

Swap causes:

```
RAM

 |

 v

Disk

 |

 v

RAM
```

Very slow.

---

## Prefault Memory

High-performance systems may allocate memory before workload starts.

Example:

Trading systems.

---

# 7. What Are Huge Pages For?

Normal page size:

```
4KB
```

Large memory requires many pages.

Example:

1GB memory:

```
1GB / 4KB

= 262144 pages
```

Each page needs:

```
Page Table Entry
```

---

More pages mean:

* larger page tables
* more TLB pressure

---

Huge Pages increase page size.

Example:

```
2MB

or

1GB
```

---

Now:

1GB memory:

Using 2MB pages:

```
512 pages
```

Much fewer translations.

---

Benefits:

```
Less Page Table overhead

Fewer TLB misses

Better memory performance
```

---

This is why:

* databases
* DPDK
* HFT systems

often use Huge Pages.

---

# 8. Does Vector Have Anything To Do With Virtual Memory?

A common misunderstanding:

```
std::vector
```

is a data structure.

Virtual Memory is an OS mechanism.

They are different.

---

But they interact.

Example:

```cpp
std::vector<int> v;

v.push_back(1);
```

Internally:

```
vector

 |

malloc()

 |

heap

 |

virtual memory

 |

physical memory
```

---

When vector grows:

```
old memory

        |

        v

new larger memory
```

The allocator uses:

```
Virtual Memory
```

---

So:

vector does not implement virtual memory.

It uses memory provided by the OS.

---

# 9. What Is The Essence Of Virtual Memory?

The core idea:

> Separate what programs see from what hardware provides.

Application sees:

```
Simple continuous memory
```

Linux manages:

```
Complex physical resources
```

---

Virtual Memory provides:

## Isolation

Process A cannot directly access:

```
Process B memory
```

---

## Flexibility

Programs can use more memory than physical RAM.

---

## Efficient Management

Linux can:

* move pages
* swap pages
* share pages
* protect pages

---

# Final Knowledge Chain

How does Virtual Memory work?

```
Application

 |

Virtual Address

 |

MMU

 |

TLB

 |

Page Table

 |

Physical Address

 |

RAM
```

When memory is missing:

```
Page Fault

 |

Kernel

 |

Load Page

 |

Resume Execution
```

When performance matters:

```
Reduce TLB Miss

        |

        v

Use Huge Pages
```

---

# Conclusion

Virtual Memory is not just a memory feature.

It is a fundamental operating system abstraction.

It allows Linux to provide:

* process isolation
* flexible memory allocation
* efficient RAM usage

The key idea:

> Programs see virtual memory. The operating system manages reality.
