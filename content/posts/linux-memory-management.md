---
title: "How Does Linux Manage Memory When Multiple Processes Use RAM?"
date: 2026-07-18
draft: false
tags:
  - Linux
  - Memory Management
  - Virtual Memory
  - Page Table
  - TLB
  - Swap
categories:
  - Many WHYs
---

# How Does Linux Manage Memory When Multiple Processes Use RAM?

Modern systems run many processes at the same time.

For example:
```

Browser

Database

Nginx

Redis

Compiler

```

All of them need memory.

But physical RAM is limited.

A simple question:

> If multiple processes need more memory than the machine has, how does Linux manage it safely?

The answer is:

```

Process

|

v

Virtual Memory

|

v

Page Table

|

v

Physical Memory

|

v

Memory Management

```

---

# 1. What Happens If Multiple Processes Need More Memory Than RAM?

Imagine:

Machine:

```

RAM = 8GB

```

Processes:

```

Process A = 4GB

Process B = 4GB

Process C = 4GB

```

Total:

```

12GB

```

More than:

```

8GB RAM

```

What happens?

Linux does not immediately fail.

Instead:

- use virtual memory
- reclaim unused pages
- use page cache
- move inactive pages to swap

---

The key idea:

> Processes do not directly own physical RAM.

They own:

```

Virtual Address Space

```

---

# 2. Why Do We Need Virtual Memory?

Without virtual memory:

Process directly accesses physical memory.

Example:

```

Process A

address 1000

```
  |

  v
```

RAM address 1000

```

Problem:

Another process can access the same address.

```

Process B

address 1000

```
  |

  v
```

same RAM location

```

This causes:

- security problems
- memory corruption
- difficult allocation

---

Virtual memory introduces abstraction:

```

Process A

Virtual Address

```
    |

    v
```

Physical Address

Process B

Virtual Address

```
    |

    v
```

Different Physical Address

```

---

# 3. Why Can Different Processes Have The Same Address?

Because every process has its own virtual address space.

Example:

Process A:

```

Virtual address:

0x1000

```

Process B:

```

Virtual address:

0x1000

```

They are not the same memory.

The mapping is different:

```

Process A Page Table

0x1000

|

v

Physical Frame 10

Process B Page Table

0x1000

|

v

Physical Frame 200

```

---

Same virtual address.

Different physical memory.

---

# 4. How Are Physical Frames Prevented From Overlapping?

Physical memory is divided into:

```

Physical Frames

```

Example:

RAM:

```

Frame 0

Frame 1

Frame 2

Frame 3

```

The kernel maintains:

```

Free Frame List

```

When allocating memory:

```

Process A

needs page

```
    |

    v
```

Kernel selects free frame

```

The mapping is stored in:

```

Page Table

```

---

Therefore:

Two processes cannot normally map the same writable frame.

---

# 5. What Is The Relationship Between Page and Physical Frame?

Important distinction:

## Page

Virtual memory unit.

Example:

```

Virtual Memory:

Page 0

Page 1

Page 2

```

---

## Frame

Physical memory unit.

Example:

```

RAM:

Frame 0

Frame 1

Frame 2

```

---

Mapping:

```

Virtual Page

```
    |

    v
```

Physical Frame

```

Example:

```

Process A:

Page 5

|

v

Frame 100

```

---

Usually:

```

Page size = Frame size

```

Common:

```

4KB

```

---

# 6. How Does CPU Translate Virtual Address To Physical Address?

The CPU does not directly access RAM.

The flow:

```

CPU

|

v

Virtual Address

|

v

MMU

|

v

Physical Address

|

v

RAM

```

The component responsible:

```

MMU
(Memory Management Unit)

```

---

Example:

CPU wants:

```

Virtual Address:

0x1234

```

MMU checks:

```

Page Table

```

Finds:

```

Virtual Page 1

```
  |

  v
```

Physical Frame 100

```

Then:

```

Physical Address

Frame 100 + Offset

```

---

# 7. What Is Page Table? Where Is It?

Page Table stores:

```

Virtual Page

```
    ->
```

Physical Frame

```

Example:

```

Virtual Page     Physical Frame

0                 20

1                 35

2                 100

```

---

Each process has its own page table.

Linux stores:

```

Page Table

inside kernel memory

```

The CPU knows the current page table through:

```

CR3 register

```

(on x86)

---

When switching processes:

```

Process A

CR3 -> Page Table A

Context Switch

Process B

CR3 -> Page Table B

```

---

# 8. What Is TLB?

Page table lookup is expensive.

Every memory access would require:

```

CPU

|

Page Table lookup

|

RAM access

```

Too slow.

---

CPU introduces:

```

TLB
Translation Lookaside Buffer

```

It is a cache for:

```

Virtual Address

```
    ->
```

Physical Address

```

---

Fast path:

```

CPU

|

TLB hit

|

RAM

```

---

Slow path:

```

CPU

|

TLB miss

|

Page Table

|

RAM

```

---

# 9. When RAM Is Not Enough, How Does Linux Handle It?

Linux has several strategies.

---

## Memory Reclaim

First:

remove unnecessary memory.

Examples:

```

Page Cache

Buffer Cache

```

---

## Swap

Move inactive memory pages:

```

RAM

|

v

Swap

```

---

## OOM Killer

If everything fails:

```

Memory exhausted

```
    |

    v
```

OOM Killer

```
    |

    v
```

Kill process

```

---

# 10. Which Pages Are Swapped Out?

Linux does not randomly choose pages.

It considers:

- inactive pages
- rarely accessed pages
- anonymous memory

---

Example:

```

Active page

used frequently

```

Less likely to swap.

---

```

Inactive page

not used recently

```

More likely to swap.

---

# 11. What Is Swap?

Swap is:

> Disk space used as an extension of RAM.

It stores memory pages that are temporarily removed from RAM.

---

Example:

RAM:

```

Hot pages

```

Swap:

```

Cold pages

````

---

Swap can be:

## Swap partition

Dedicated disk area.

---

## Swap file

A normal file used as swap.

---

# 12. Where Is Swap?

Linux exposes:

```bash
swapon --show
````

Example:

```
/swapfile
```

---

The kernel manages:

```
RAM

+

Swap
```

as memory resources.

---

# 13. What Happens When Accessing A Swapped Page?

Suppose:

Process accesses:

```
Virtual Page X
```

But:

```
Page not in RAM
```

CPU triggers:

```
Page Fault
```

---

Flow:

```
CPU

 |

Page Fault

 |

Kernel

 |

Find page in Swap

 |

Load into RAM

 |

Update Page Table

 |

Continue execution
```

---

This is called:

```
Major Page Fault
```

---

# 14. What If Both RAM And Swap Are Full?

Then Linux cannot satisfy allocations.

Flow:

```
Memory request

        |

        v

No RAM

        |

        v

No Swap

        |

        v

OOM Killer
```

---

The kernel selects:

```
victim process
```

and kills it.

---

# 15. Where Does Memory Management Actually Happen?

Memory management exists across the system.

## User Space

Application requests:

```
malloc()
```

---

## System Library

glibc manages:

```
heap allocation
```

---

## Kernel

Handles:

```
Virtual Memory

Page Allocation

Swap

Page Fault

Reclaim
```

---

## Hardware

CPU provides:

```
MMU

TLB
```

---

Complete picture:

```
Application

    |

 malloc()

    |

 libc

    |

 syscall

    |

 Linux Kernel

    |

 Memory Manager

    |

 Page Table

    |

 MMU

    |

 RAM
```

---

# Final Knowledge Chain

How does Linux manage memory?

```
Process

 |

 v

Virtual Address Space

 |

 v

Page Table

 |

 v

MMU

 |

 v

Physical Frame

 |

 v

RAM


Memory Pressure

 |

 v

Reclaim

 |

 v

Swap

 |

 v

OOM Killer
```

---

# Conclusion

Linux Memory Management is an abstraction layer between:

```
Programs

and

Physical Hardware
```

It allows:

* multiple processes to safely share RAM
* each process to have its own address space
* memory overcommit
* efficient resource usage

The core idea:

> Processes see virtual memory. Linux decides where the data actually lives.
