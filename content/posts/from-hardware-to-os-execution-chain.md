---
title: "From CPU Hardware to Operating System: A Complete Execution Chain"
date: 2026-07-20
draft: false
tags:
  - Computer Architecture
  - CPU
  - ISA
  - MMU
  - Operating System
  - Kernel
  - Syscall
  - Cache
categories:
  - Many WHYs
---
When we run:

```bash
./server
```

many layers start working together.

Most developers only see:

```
Application
```

but underneath:

```
Application

↓

Library

↓

System Call

↓

Operating System

↓

CPU

↓

Hardware Circuit
```

The question:

> How does a piece of code finally become electrical signals inside CPU?

---

# 1. The Big Picture

A computer system can be divided into layers:

```
Application

    |
    v

Operating System

    |
    v

Instruction Set Architecture (ISA)

    |
    v

CPU Microarchitecture

    |
    v

Digital Circuit

    |
    v

Transistors
```

Each layer hides complexity from the layer above.

---

For example:

A programmer writes:

```c
x = a + b;
```

The CPU eventually executes:

```
ADD instruction
```

which becomes:

```
Control Signals
```

which become:

```
Electrical operations
```

inside transistor circuits.

---

# 2. Starting From Hardware Circuit

At the lowest level:

CPU is built from:

* transistors
* logic gates
* registers
* arithmetic circuits

---

A transistor is basically:

```
ON

or

OFF
```

which represents:

```
1

or

0
```

---

Combining transistors creates:

```
Logic Gates
```

Examples:

```
AND

OR

NOT
```

---

Combining logic gates creates:

```
Adder

Register

ALU

Control Unit
```

---

The CPU is essentially:

```
Data Path

+

Control Logic
```

---

# 3. CPU Needs Rules: ISA Appears

Hardware alone cannot understand:

```
x = a + b
```

The CPU needs a contract:

```
Instruction Set Architecture
```

ISA defines:

* instructions
* registers
* memory model
* privilege levels

Examples:

* x86-64
* ARM64
* RISC-V

---

Example:

Source:

```c
a + b
```

Compiler generates:

```
ADD R1, R2
```

ISA defines:

```
ADD means:

Take values from registers

Perform addition

Store result
```

---

ISA is the boundary between:

```
Software

     |

     v

Hardware
```

---

# 4. Operating System Enters

Without OS:

Every program would need:

* hardware knowledge
* memory management
* device drivers

Impossible.

---

OS creates abstractions:

```
Process

Thread

File

Virtual Memory

Socket
```

---

Example:

Application:

```c
read(fd, buffer, size);
```

does not know:

* disk controller
* memory address
* hardware interrupt

Linux handles everything.

---

# 5. Complete Execution Flow

The system can be read as a single execution path:

```c
read(fd, buffer, size);
```

---

## Step 1: User Program Runs

CPU executes:

```
User Mode
```

Program instructions are loaded into memory.

CPU:

```
Fetch

Decode

Execute
```

---

## Step 2: Program Calls syscall

Example:

```c
read()
```

The library eventually executes:

```
syscall instruction
```

---

## Step 3: What Happens Inside CPU?

CPU performs:

### Save user context

Store:

* Program Counter
* Registers
* Stack information

---

### Change privilege level

From:

```
User Mode
```

to:

```
Kernel Mode
```

---

### Jump to kernel entry

CPU uses:

```
System Call Handler
```

---

Flow:

```
User Program

      |

      v

syscall instruction

      |

      v

CPU privilege transition

      |

      v

Kernel Entry
```

---

# 6. What Does Kernel Do?

Kernel receives:

```
system call number

arguments
```

Example:

```
read(fd, buffer, size)
```

Kernel checks:

* Is fd valid?
* Does process have permission?
* Where is the data?

---

Then:

```
Kernel

 |

 v

Device Driver

 |

 v

Hardware
```

---

After completion:

```
Kernel

 |

 v

Return value

 |

 v

User Program
```

---

# 7. What Does MMU Do?

MMU:

```
Memory Management Unit
```

is the bridge between:

```
Virtual Address

        |

        v

Physical Address
```

---

Application thinks:

```
0x1000
```

MMU translates:

```
Virtual Address

↓

Page Table

↓

Physical Frame
```

---

Every memory access:

```
CPU

↓

MMU

↓

TLB

↓

Page Table

↓

RAM
```

---

Without MMU:

* no process isolation
* no virtual memory
* no safe multitasking

---

# 8. What Does Cache Do?

CPU is much faster than RAM.

Speed gap:

```
CPU

↓

Cache

↓

RAM

↓

Disk
```

---

When CPU needs data:

First:

```
L1 Cache
```

then:

```
L2

L3

RAM
```

---

Cache stores:

```
Recently used data
```

---

Because programs have:

## Temporal locality

Recently used data may be used again.

## Spatial locality

Nearby data may be used soon.

---

Example:

```cpp
for(int i=0;i<1000;i++)
{
    sum += array[i];
}
```

Cache loves this pattern.

---

# 9. Where Does SMM Appear?

SMM:

```
System Management Mode
```

is a special CPU mode on x86.

It is different from:

```
User Mode

Kernel Mode
```

---

Privilege levels:

Normal:

```
User

 |

Kernel
```

---

SMM:

```
Hardware Management Mode
```

---

SMM is entered by:

```
SMI
System Management Interrupt
```

---

Examples:

* firmware control
* power management
* hardware monitoring

---

When SMI happens:

```
CPU

 |

save current state

 |

enter SMM

 |

execute firmware code

 |

return
```

---

The OS usually does not directly control SMM.

---

# 10. Putting Everything Together

A program execution:

```
Application

    |

    v

Compiler

    |

    v

ISA Instructions

    |

    v

CPU Fetch/Decode/Execute

    |

    v

MMU translates address

    |

    v

Cache checks data

    |

    v

Memory Access

    |

    v

Syscall

    |

    v

Kernel

    |

    v

Hardware
```

---

# Final Mental Model

A modern computer is a stack:

```
Application
     |
     |
Operating System
     |
     |
ISA
     |
     |
CPU Architecture
     |
     |
Digital Circuit
     |
     |
Transistors
```

---

# Conclusion

Understanding systems programming means understanding the whole chain:

```
Code

↓

Instruction

↓

CPU

↓

Memory Translation

↓

Kernel

↓

Hardware
```

High-performance systems are built by understanding every layer:

* Cache affects latency
* MMU affects memory access
* Syscall affects execution boundary
* OS manages resources
* Hardware executes instructions

This is the foundation behind:

* Linux kernel
* databases
* high-performance servers
* HFT systems
* cloud infrastructure
