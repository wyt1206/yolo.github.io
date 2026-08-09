---
title: "How Do CPU, Process, Thread, Syscall, and Context Switch Work Together?"
date: 2026-06-11
draft: true
tags:
  - Linux
  - CPU
  - Process
  - Thread
  - Syscall
  - Context Switch
  - Operating System
categories:
  - Many WHYs
---
# How Do CPU, Process, Thread, Syscall, and Context Switch Work Together?

When we run a program:

```bash
./server
```

Many things happen:

* CPU executes instructions
* Linux creates a process
* Scheduler chooses which thread runs
* System calls enter the kernel
* Context switches happen

A simple question:

> How does Linux manage many running programs on limited CPUs?

The answer connects:

```
CPU

 |

 v

Thread

 |

 v

Process

 |

 v

Scheduler

 |

 v

Context Switch

 |

 v

System Call
```

---

# 1. Can CPU Only Do One Thing at a Time?

At the hardware level:

A single CPU core executes:

```
one instruction stream
```

at one moment.

Example:

```
CPU Core

execute instruction A

then

execute instruction B
```

It cannot truly execute two instructions from two different programs at exactly the same time.

---

But modern CPUs have:

* multiple cores
* hyper-threading
* superscalar execution

Example:

4 cores:

```
Core 0 -> Thread A

Core 1 -> Thread B

Core 2 -> Thread C

Core 3 -> Thread D
```

Now multiple things can execute simultaneously.

---

# 2. Are Programs Running at the Same Time or Alternating?

Depends.

## Multiple cores

True parallelism:

```
Core 0

Process A


Core 1

Process B
```

---

## One core

The OS creates illusion of concurrency.

Example:

```
time

0ms

Process A


5ms

Process B


10ms

Process A
```

The switching happens very fast.

Humans feel:

```
everything is running
```

---

The mechanism:

```
Scheduler

      |

      v

Context Switch
```

---

# 3. What Is Context Switch?

A context switch means:

> CPU switches from executing one thread to another.

Example:

Before:

```
CPU

running Thread A
```

After:

```
CPU

running Thread B
```

---

But CPU must remember:

Where was Thread A?

It needs to save:

* registers
* program counter
* stack pointer
* CPU state

---

Conceptually:

```
Thread A

registers

PC

stack pointer

        |

        v

saved


Load Thread B state

        |

        v

continue Thread B
```

---

# 4. What Are Process and Thread?

Many beginners confuse them.

## Process

A process is:

> A running program with its own resources.

Example:

```
nginx process

memory space

file descriptors

security context
```

---

Each process has:

```
Virtual Memory

+

Resources
```

---

## Thread

A thread is:

> The execution unit scheduled by CPU.

A process can have:

```
Process

 |
 +-- Thread 1
 |
 +-- Thread 2
 |
 +-- Thread 3
```

---

Threads share:

```
Code

Heap

Files
```

but have their own:

```
Stack

Registers

Execution state
```

---

# 5. What Are fork() and exec()?

They are Unix process creation mechanisms.

---

## fork()

fork creates a new process.

Example:

```c
pid = fork();
```

Before:

```
Parent Process
```

After:

```
Parent Process

        +

Child Process
```

---

The child initially has:

* copied memory view
* same code
* inherited file descriptors

Modern Linux uses:

```
Copy On Write
```

so memory is not immediately copied.

---

## exec()

exec replaces the current process image.

Example:

```c
exec("./program");
```

Before:

```
bash process
```

After:

```
new program
```

Same process ID.

Different program.

---

Typical shell:

```
bash

 |

 fork()

 |

child

 |

exec(ls)

 |

ls program
```

---

# 6. Are fork/exec Related to Context Switch?

They are related but different.

fork:

creates a new process.

exec:

loads a new program.

Context switch:

changes CPU execution target.

---

Example:

```
Parent

fork()

 |

 v

Child created


Scheduler chooses Child


Context Switch


Child runs
```

---

Process creation may cause scheduling changes.

But:

```
fork != context switch
```

---

# 7. What Is a System Call?

A system call is:

> A controlled way for user programs to request kernel services.

User programs cannot directly:

* access hardware
* modify kernel memory
* control devices

So they call:

```
syscall
```

Example:

```c
read(fd, buffer, size);
```

Behind:

```
Application

    |

    v

System Call

    |

    v

Kernel

    |

    v

Hardware
```

---

Examples:

Process:

```
fork()
exec()
```

File:

```
open()
read()
write()
```

Network:

```
socket()
bind()
epoll_wait()
```

---

# 8. Relationship Between Syscall and Context Switch

Important:

Many people think:

> syscall always causes context switch.

Not exactly.

A syscall causes:

```
user mode

        |

        v

kernel mode
```

This is:

```
privilege transition
```

---

Context switch is:

```
Thread A

        |

        v

Thread B
```

Different concept.

---

However:

A syscall may trigger scheduling.

Example:

```c
sleep()
```

Flow:

```
Thread A

syscall

 |

kernel

 |

block

 |

scheduler chooses Thread B

 |

context switch
```

---

So:

```
syscall

can cause

context switch
```

but:

```
syscall != context switch
```

---

# 9. Why Do Syscalls Often Look Slow?

Because crossing the boundary has cost.

Normal function:

```
user

 |

function call

 |

return
```

---

Syscall:

```
user mode

 |

save state

 |

switch privilege

 |

kernel execution

 |

return to user mode
```

---

Additional costs:

* CPU pipeline disruption
* security checks
* argument validation
* kernel bookkeeping

---

But syscall is not extremely slow.

Modern CPUs optimize it.

The problem appears when:

```
millions of syscalls
```

happen repeatedly.

---

Example:

Bad:

```
write()

write()

write()

write()
```

Better:

```
buffer

batch write()
```

---

# 10. What Are pthread and clone()?

Linux does not have a separate thread object.

Linux implements threads using:

```
clone()
```

---

Example:

pthread:

```c
pthread_create()
```

internally:

```
pthread

 |

 v

clone()

 |

 v

Linux task
```

---

clone allows sharing:

```
memory

files

signals
```

between execution units.

---

Process:

```
clone(flags=0)
```

separate resources.

Thread:

```
clone(
CLONE_VM |
CLONE_FILES |
CLONE_THREAD
)
```

share resources.

---

# 11. What Is the Lowest-Level Thread Model?

Linux scheduler manages:

```
task_struct
```

Everything is a task.

Including:

* processes
* threads

---

Kernel view:

```
task_struct

      |

      v

Scheduler

      |

      v

CPU Core
```

---

A thread has:

```
task_struct

+

stack

+

register state
```

---

The scheduler does not really care:

```
process

or

thread
```

It schedules:

```
runnable tasks
```

---

# Final Knowledge Chain

How does Linux run everything?

```
CPU Core

    |

    v

Scheduler

    |

    v

Thread

    |

    v

Process

    |

    v

System Call

    |

    v

Kernel

    |

    v

Context Switch
```

---

# Conclusion

Understanding Linux performance requires understanding the relationship:

```
CPU

 |

Thread

 |

Process

 |

Syscall

 |

Context Switch
```

A high-performance server is not only about APIs like:

```
epoll()
```

It is built on top of:

* CPU scheduling
* thread model
* kernel boundary
* execution switching

This is the foundation behind:

* Nginx workers
* Redis event loop
* thread pools
* C++ server frameworks

````

---

## 这篇在你的 ManyWhys 里的位置

建议顺序：

```text
01 CPU Cache
        |
02 Cache Line
        |
03 Process / Thread / CPU / Context Switch
        |
04 User Mode / Kernel Mode
        |
05 Syscall
        |
06 socket()
        |
07 fd
        |
08 epoll
        |
09 Callback
        |
10 Reactor
````
