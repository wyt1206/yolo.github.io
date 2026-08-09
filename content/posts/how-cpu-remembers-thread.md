---
title: "How Does CPU Remember a Thread? Understanding Context Switch"
date: 2026-07-20
draft: true
tags:
  - Linux
  - CPU
  - Thread
  - Process
  - Context Switch
  - Scheduling
  - Kernel
categories:
  - Many WHYs
---
```

---

# How Does CPU Remember a Thread? Understanding Context Switch

When we run multiple programs:

```
Browser

Database

Server

Compiler
```

It looks like they are running simultaneously.

But:

A single CPU core can execute only one instruction stream at a time.

So the question:

> How can CPU switch between different threads and continue exactly where it stopped?

The answer:

```
Context Switch
```

---

# 1. What Is A Running Context?

A thread is not just code.

A running thread needs:

```
Execution State
```

including:

* Program Counter
* CPU Registers
* Stack Pointer
* Memory Mapping
* Kernel Scheduling Information

Together:

```
Thread Context
```

---

Example:

A thread is executing:

```c
sum += value;
```

CPU currently knows:

```
Instruction address:

0x401234
```

Registers:

```
RAX = 100
RBX = 200
```

Stack:

```
function call information
```

---

These values define:

> Where the thread is and what it is doing.

---

# 2. What Happens During Context Switch?

Imagine:

Thread A is running:

```
CPU Core

   |
   v

Thread A
```

Scheduler decides:

```
Run Thread B
```

The CPU cannot simply jump away.

Because Thread A would lose:

* current instruction
* register values
* stack state

---

The kernel does:

```
Thread A

   |

   v

Save Context

   |

   v

Scheduler

   |

   v

Restore Thread B Context

   |

   v

Thread B continues
```

---

Important:

The CPU does not save the thread.

The OS saves the CPU state.

---

# 3. Why Are Registers So Important?

Registers are the fastest storage inside CPU.

Examples:

x86-64:

```
RAX
RBX
RCX
RDX
RSP
RIP
```

---

Most important:

## Instruction Pointer

```
RIP
```

It tells CPU:

> Which instruction should execute next?

Example:

Before switch:

```
RIP = 0x401000
```

After restore:

CPU continues:

```
0x401000
```

---

## Stack Pointer

```
RSP
```

points to:

* function stack
* local variables
* return addresses

Without RSP:

The thread cannot continue.

---

## General Registers

Example:

```
RAX = calculation result
```

If lost:

Program logic breaks.

---

Therefore:

Context switch is mainly:

> Save registers + restore registers.

---

# 4. What Does It Look Like In Assembly?

Conceptually:

Save current thread:

```asm
push rax
push rbx
push rcx
push rdx
push rip
```

Store into:

```
Thread Control Block
```

---

Switch:

```asm
load new thread context
```

Restore:

```asm
pop rdx
pop rcx
pop rbx
pop rax
```

Return:

```asm
iret
```

---

Real Linux implementation is more complicated:

* architecture specific
* kernel stack switching
* scheduler interaction

But the idea is:

```
Save CPU state

Load another CPU state
```

---

# 5. Where Is Thread Context Stored?

Linux maintains structures.

For example:

```
task_struct
```

contains information about:

* process state
* scheduling
* memory
* files
* credentials

---

Each thread has:

```
Kernel Stack
```

which stores:

* saved registers
* syscall information
* execution state

---

Conceptually:

```
Thread A

task_struct

    +

kernel stack


Thread B

task_struct

    +

kernel stack
```

---

# 6. Difference Between Process Switch And Thread Switch

Important interview question.

## Thread Switch

Same process:

```
Process A

 Thread 1

 Thread 2
```

They share:

* address space
* code
* heap
* files

Only need:

```
CPU context switch
```

---

## Process Switch

Different processes:

```
Process A

      |

      v

Process B
```

Need:

1. Save CPU registers

2. Switch memory space

3. Change page table

---

Example:

Before:

```
CR3 -> Process A Page Table
```

After:

```
CR3 -> Process B Page Table
```

---

This affects:

* TLB
* cache locality

---

Therefore:

Process switch is usually more expensive.

---

# 7. Why Is Context Switch Expensive?

A context switch involves several costs.

---

## 1. Save / Restore Registers

CPU must:

```
Store registers

Load registers
```

---

## 2. Scheduler Overhead

Kernel needs:

* choose next task
* update state
* maintain queues

---

## 3. Cache Pollution

Example:

Thread A uses:

```
Data A
```

Cache contains:

```
A A A
```

Switch to Thread B:

```
Data B
```

Cache gradually changes.

---

This causes:

```
Cache Miss
```

---

## 4. TLB Effects

Process switch changes:

```
Page Table
```

Potentially causing:

```
TLB Miss
```

---

# 8. Complete Context Switch Flow

Full picture:

```
Running Thread A

        |

        v

Timer Interrupt

        |

        v

Kernel Entry

        |

        v

Save Registers

        |

        v

Scheduler

        |

        v

Select Thread B

        |

        v

Restore Registers

        |

        v

Return to User Mode

        |

        v

Running Thread B
```

---

# 9. Why Does Linux Need Context Switch?

Because CPU sharing requires:

```
One CPU

+

Many Tasks
```

Linux provides:

```
Time Sharing
```

---

Example:

CPU:

```
0ms

Thread A


10ms

Thread B


20ms

Thread C
```

Users perceive:

```
Everything runs together
```

---

# Final Knowledge Chain

How does CPU remember a thread?

```
Thread

 |

 v

Execution Context

 |

 v

Registers

 |

 v

Kernel saves state

 |

 v

Scheduler switches

 |

 v

Kernel restores state

 |

 v

Thread continues
```

---

# Conclusion

CPU does not remember threads.

CPU only executes:

```
Instructions
```

The operating system creates the illusion of threads by:

```
Saving execution state

+

Restoring execution state
```

The core idea:

> A thread is not a running CPU. A thread is a saved execution context that can be resumed by the CPU.