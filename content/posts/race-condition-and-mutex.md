---
title: "Why Can Multiple Threads Produce Wrong Results? From Race Condition to Mutex"
date: 2026-08-01
draft: false
tags:
  - Linux
  - Thread
  - Concurrency
  - Race Condition
  - Mutex
  - Futex
  - CPU Cache
categories:
  - Many WHYs
---
Modern applications use multiple threads.

Example:

```text
Web Server

Thread 1 -> Request A

Thread 2 -> Request B

Thread 3 -> Request C
```

It looks faster.

But a simple question:

> If two threads execute correctly, why can the final result still be wrong?

The answer:

```text
Race Condition
```

---

# 1. Why Can Multi-threaded Programs Be Wrong?

Consider:

```cpp
counter++;
```

Many people think:

> This is one line, so it must be atomic.

But CPU does not execute C++ statements.

The compiler converts:

```cpp
counter++;
```

into machine instructions.

Conceptually:

```text
LOAD counter

ADD 1

STORE counter
```

---

Now two threads:

Initial:

```text
counter = 0
```

Thread A:

```text
LOAD counter
```

gets:

```text
0
```

Thread B:

```text
LOAD counter
```

gets:

```text
0
```

---

Thread A:

```text
ADD 1

STORE 1
```

Thread B:

```text
ADD 1

STORE 1
```

Final:

```text
counter = 1
```

Expected:

```text
counter = 2
```

---

The program is wrong.

But every instruction executed correctly.

---

# 2. Isn't counter++ One CPU Instruction?

Usually no.

High-level code:

```cpp
counter++;
```

does not mean:

```text
one CPU instruction
```

---

CPU sees:

```text
Read memory

Modify value

Write memory
```

These are separate operations.

---

Even if an instruction looks atomic:

The question is:

> Is the whole operation protected from other CPUs?

---

For example:

```asm
inc counter
```

may still involve:

```text
Read cache line

Modify value

Write cache line
```

Multiple cores can interfere.

---

# 3. What Is Race Condition?

Race Condition means:

> The result depends on the timing/order of thread execution.

Example:

Thread A:

```text
Read X

Update X
```

Thread B:

```text
Read X

Update X
```

The final result depends on:

```text
Who runs first?
```

---

The "race":

```text
Thread A

      vs

Thread B
```

Both compete for shared data.

---

# 4. Does Every Shared Data Cause Race Condition?

No.

Only when:

```text
Shared Data

+

Concurrent Access

+

At least one Write
```

happens.

---

Examples:

## Safe

Two threads only read:

```text
Config File

Thread A read

Thread B read
```

No problem.

---

## Unsafe

One writes:

```text
counter++

Thread A write

Thread B write
```

Race condition.

---

The formula:

```text
Shared Mutable State

        +

Concurrent Access

        =

Potential Race Condition
```

---

# 5. What Is The Critical Section?

The code area accessing shared resources is:

```text
Critical Section
```

Example:

```cpp
mutex.lock();

counter++;

mutex.unlock();
```

The critical section:

```cpp
counter++;
```

---

The rule:

Only one thread should enter:

```text
Critical Section
```

at a time.

---

# 6. How Do We Protect Critical Section?

Common solutions:

## Mutex

```text
Mutual Exclusion
```

Only one thread enters.

---

## Spinlock

Thread waits actively:

```text
while(lock)
{
}
```

Useful for very short critical sections.

---

## Atomic Operation

CPU provides atomic instructions:

Example:

```text
compare-and-swap
```

---

## Semaphore

Controls number of allowed users.

---

# 7. Why Can Mutex Guarantee Only One Thread Enters?

Mutex has a state:

```text
Unlocked

or

Locked
```

Example:

Before:

```text
mutex = unlocked
```

Thread A:

```text
lock()
```

changes:

```text
mutex = locked
```

---

Thread B:

```text
lock()
```

sees:

```text
already locked
```

must wait.

---

The important question:

> How does changing lock state itself avoid race condition?

---

# 8. How Does Mutex Solve Its Own Race Condition?

Because mutex acquisition uses atomic hardware operations.

Example:

```text
Compare And Swap (CAS)
```

CPU provides instructions that guarantee:

```text
Only one CPU can modify this value successfully
```

---

Example:

Initial:

```text
lock = 0
```

Two CPUs:

CPU A:

```text
CAS(0 -> 1)
```

CPU B:

```text
CAS(0 -> 1)
```

Only one succeeds.

---

Result:

```text
CPU A:

lock acquired


CPU B:

failed
```

---

Mutex itself is built on atomic primitives.

---

# 9. What Happens If Mutex Cannot Be Acquired?

Two strategies.

---

## Spin

Keep checking:

```text
while(lock)
{
    retry
}
```

Advantages:

* no context switch

Disadvantage:

* wastes CPU

---

## Sleep

Thread blocks:

```text
Running

 |

 v

Waiting
```

Kernel removes it from CPU.

---

Later:

```text
unlock()

 |

 v

wake waiting thread
```

---

# 10. Is Mutex Implemented Inside Kernel?

Depends.

Modern Linux mutex is hybrid.

Fast path:

```text
User Space
```

Slow path:

```text
Kernel
```

---

If lock is free:

```text
atomic operation

return immediately
```

No syscall.

---

If contention happens:

```text
Thread waits

↓

Kernel involvement
```

---

This avoids unnecessary context switches.

---

# 11. What Is Futex?

Futex:

```text
Fast Userspace Mutex
```

Linux mechanism.

Idea:

Most of the time:

```text
No contention
```

So avoid kernel calls.

---

Fast path:

```text
Thread

 |

atomic operation

 |

continue
```

---

Slow path:

```text
Thread

 |

futex syscall

 |

sleep in kernel
```

---

Flow:

```text
Mutex

 |

 v

Atomic instruction

 |

 v

Futex

 |

 v

Kernel scheduler
```

---

Many C++ pthread mutex implementations use futex internally.

---

# 12. Does Mutex Affect CPU Cache?

Yes.

Very important.

Modern CPUs have:

```text
L1 Cache

L2 Cache

L3 Cache
```

Each core has its own cache.

---

Suppose:

Thread A:

```text
Core 1

counter in cache
```

Thread B:

```text
Core 2

counter in cache
```

Both modify:

```text
same cache line
```

---

CPU must maintain consistency.

Using:

```text
MESI Protocol
```

---

The cache line moves:

```text
Core 1

Modified

        |

        v

Core 2

Modified
```

---

This causes:

* cache invalidation
* memory traffic
* latency

---

This is why:

```text
Lock contention
```

is expensive.

Not only because of waiting.

Also because of:

```text
Cache coherence traffic
```

---

# Complete Race Condition Chain

```text
Multiple Threads

        |

        v

Shared Data

        |

        v

Concurrent Write

        |

        v

Race Condition

        |

        v

Critical Section

        |

        v

Mutex

        |

        v

Atomic CPU Instruction

        |

        v

Futex

        |

        v

Kernel Scheduler
```

---

# Conclusion

Threads are powerful because they share memory.

But sharing memory creates danger.

The core problem:

> Multiple CPUs can observe and modify the same data at unpredictable times.

Mutex solves this by creating:

```text
Mutual Exclusion
```

The deeper chain:

```text
Thread

↓

Shared Memory

↓

Race Condition

↓

Synchronization

↓

Atomic Operation

↓

Mutex/Futex

↓

Kernel Scheduling
```

Understanding this is the foundation of:

* Linux concurrency
* C++ multithreading
* database locks
* high-performance systems
* HFT low-latency programming
