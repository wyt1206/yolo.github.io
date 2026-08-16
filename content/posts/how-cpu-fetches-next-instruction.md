---
title: "How Does CPU Find the Next Instruction? Understanding Fetch and Cache"
date: 2026-07-23
draft: false
tags:
  - CPU
  - Computer Architecture
  - Instruction Pipeline
  - Cache
  - Memory
  - ISA
categories:
  - Many WHYs
---
Every program eventually becomes:

```text
Machine Instructions
```

For example:

```c
a = b + c;
```

becomes:

```text
ADD instruction
```

But an important question:

> How does CPU know which instruction to execute next?

The answer involves:

* Program Counter
* Instruction Fetch
* Cache
* Memory Hierarchy

---

# 1. What Stages Does An Instruction Go Through?

A CPU does not simply:

```text
Read instruction

↓

Execute
```

Modern CPUs use pipelines.

A simplified pipeline:

```text
Fetch

↓

Decode

↓

Execute

↓

Memory Access

↓

Write Back
```

---

## Fetch

CPU obtains the next instruction.

Example:

```text
Instruction Address:

0x401000
```

CPU asks:

> What instruction is stored here?

---

## Decode

CPU understands:

```asm
ADD RAX, RBX
```

meaning:

* which registers?
* what operation?

---

## Execute

ALU performs:

```text
RAX + RBX
```

---

## Memory Access

If instruction needs memory:

```text
load/store
```

CPU accesses data.

---

## Write Back

Result goes into:

```text
Register
```

---

Complete:

```text
Instruction Address

        |

        v

Fetch

        |

        v

Decode

        |

        v

Execute

        |

        v

Write Back
```

---

# 2. Does Fetch Get Instructions From Registers?

No.

This is a common misunderstanding.

CPU does not store the whole program inside registers.

Registers store:

* temporary values
* addresses
* execution state

---

The register responsible for instruction location:

```text
Program Counter (PC)

or

Instruction Pointer (IP)
```

---

Example:

```text
PC = 0x401000
```

means:

> The next instruction is located at address 0x401000.

---

CPU uses:

```text
PC

↓

Memory System

↓

Instruction
```

---

So:

PC stores:

```text
Where to fetch
```

not:

```text
The instruction itself
```

---

# 3. After Getting Address, Does CPU Go Directly to DRAM?

No.

CPU first checks cache.

The real path:

```text
CPU

 |

 v

L1 Instruction Cache

 |

 v

L2 Cache

 |

 v

L3 Cache

 |

 v

DRAM
```

---

Modern CPUs have:

```text
Instruction Cache (I-Cache)
```

because instructions are also data.

---

Example:

CPU wants:

```text
Address 0x401000
```

First:

```text
Is it in L1 I-Cache?
```

---

If yes:

```text
Cache Hit

↓

Return instruction
```

Very fast.

---

If no:

```text
Cache Miss
```

go deeper.

---

# 4. Why Do We Need Multiple Levels of Cache?

Because there is a speed and size tradeoff.

---

CPU:

Very fast.

DRAM:

Much slower.

---

A rough hierarchy:

```text
        Faster

        CPU Register

             ↓

        L1 Cache

             ↓

        L2 Cache

             ↓

        L3 Cache

             ↓

        DRAM

             ↓

        Disk

        Slower
```

---

Why not make L1 huge?

Because:

## Larger cache is slower

A small cache:

```text
quick lookup
```

A large cache:

```text
more hardware

more search time
```

---

Therefore:

```text
Small + Fast

Large + Slow
```

---

# 5. What Happens If Cache Does Not Have The Instruction?

This is:

```text
Instruction Cache Miss
```

---

Example:

CPU:

```text
Need instruction X
```

Checks:

```text
L1

miss
```

Then:

```text
L2

miss
```

Then:

```text
L3

miss
```

---

CPU requests:

```text
Memory Controller
```

---

Flow:

```text
CPU

↓

Cache Miss

↓

Lower Cache

↓

DRAM Request

↓

Instruction Loaded

↓

Cache Filled

↓

CPU Continues
```

---

Important:

CPU usually does not fetch one byte.

It fetches:

```text
Cache Line
```

Typically:

```text
64 Bytes
```

---

Because programs have:

```text
Spatial Locality
```

Meaning:

If CPU needs:

```text
Instruction A
```

it probably soon needs:

```text
Instruction A+1
```

---

# 6. What If DRAM Does Not Have It?

Interesting question.

Normally:

If a program is running, its code pages should be in memory.

But possible:

The page is not in RAM.

Example:

```text
Virtual Address

        |

        v

Page not present
```

---

Then:

```text
Page Fault
```

happens.

---

Flow:

```text
CPU

↓

MMU

↓

Page Table

↓

Page Missing

↓

Page Fault Exception

↓

Kernel

↓

Load Page

↓

Update Page Table

↓

Resume Program
```

---

The page may come from:

* executable file
* shared library
* swap

---

This connects to your previous article:

```text
Virtual Memory

↓

Page Table

↓

Page Fault
```

---

# 7. How Does CPU Know The Next Instruction?

This is the most important question.

The answer:

```text
Program Counter
```

---

Normal instruction:

Example:

Current:

```text
PC = 0x1000
```

Instruction length:

```text
4 Bytes
```

After execution:

```text
PC = 0x1004
```

---

CPU automatically updates:

```text
PC += instruction length
```

---

But what about:

```c
if(condition)
    goto label;
```

?

Then:

```text
Branch Instruction
```

changes PC.

---

Example:

```asm
JMP 0x2000
```

CPU:

```text
PC = 0x2000
```

---

The CPU does not always know.

It predicts.

---

# 8. What About Branch Prediction?

Modern CPUs execute ahead.

Example:

```c
if(x > 0)
{
    A();
}
else
{
    B();
}
```

CPU guesses:

```text
Probably A()
```

and fetches:

```text
A instructions
```

---

If correct:

```text
Performance gain
```

---

If wrong:

```text
Pipeline Flush
```

CPU throws away speculative work.

---

This is why modern CPU performance depends on:

* branch prediction
* instruction cache
* pipeline

---

# Complete Instruction Fetch Chain

Putting everything together:

```text
Program Counter

        |

        v

Instruction Address

        |

        v

Instruction Cache

        |

        v

L2 Cache

        |

        v

L3 Cache

        |

        v

DRAM

        |

        v

Page Table / MMU

        |

        v

Instruction

        |

        v

Decode

        |

        v

Execute
```

---

# Final Mental Model

CPU does not magically know instructions.

It follows a mechanism:

```text
PC tells CPU where to look

↓

Cache tries to provide instruction

↓

Memory hierarchy finds data

↓

CPU decodes instruction

↓

CPU executes

↓

PC moves to next instruction
```

---

# Conclusion

The core idea:

> CPU execution starts from an address, not from an instruction.

The chain is:

```text
Program Counter

↓

Instruction Fetch

↓

Cache Hierarchy

↓

Memory

↓

Decode

↓

Execute
```

Understanding this explains:

* why Cache matters
* why instruction locality matters
* why branch prediction exists
* why memory latency affects CPU performance

This is the foundation of:

* operating systems
* compilers
* high-performance computing
* HFT latency optimization
