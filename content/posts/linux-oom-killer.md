---
title: "Linux OOM Killer: How Does Linux Survive When Memory Runs Out?"
date: 2026-07-17
draft: false
tags:
  - Linux
  - Memory Management
  - OOM
  - Kernel
  - Kubernetes
categories:
  - Many WHYs
---
# Linux OOM Killer: How Does Linux Survive When Memory Runs Out?

Memory is one of the most important resources in an operating system.

Every process needs memory:

- code
- heap
- stack
- buffers
- kernel structures

But memory is limited.

A simple question:

> What happens when Linux runs out of memory?

Does the kernel crash?

Does the machine freeze?

The answer is:

Linux tries to recover first.

Only when recovery fails:

> Linux activates the OOM Killer.

The complete chain:

```

Memory Pressure

```

    |

    v

```

Memory Reclaim

```

    |

    v

```

Reclaim Failed

```

    |

    v

```

OOM Killer

```

    |

    v

```

Kill Selected Process

```

---

# 1. What Is OOM?

OOM means:

> Out Of Memory

It means:

The system cannot satisfy a memory allocation request.

Example:

A process requests:

```c
malloc(1GB);
```

But the kernel cannot provide enough memory.

---

Important:

OOM does not simply mean:

> RAM is 100% full.

Linux memory includes:

* physical RAM
* page cache
* swap
* reclaimable kernel memory

The kernel continuously manages these resources.

---

# 2. What Does Linux Do When Memory Becomes Low?

Linux does not immediately kill processes.

First:

> It tries memory reclaim.

The kernel attempts to free memory.

---

## Page Cache Reclaim

Linux uses unused memory as cache.

Example:

```
Application

    |
    v

Disk data

    |
    v

Page Cache
```

When memory pressure appears:

Linux can remove cached pages.

---

## Swap

If swap exists:

Anonymous memory may be moved:

```
RAM

 |

 v

Swap Disk
```

---

## Drop Reclaimable Kernel Objects

Linux can reclaim:

* dentries
* inode cache
* buffers

---

The process:

```
Memory Pressure

       |

       v

Try reclaim

       |

       v

Free memory
```

---

# 3. Why Does Linux Trigger OOM Killer?

Because memory reclaim may fail.

Example:

All memory contains:

```
Active Processes

+

Anonymous Memory

+

Locked Memory
```

Nothing useful can be reclaimed.

The kernel reaches:

```
No available memory

        |

        v

Allocation cannot continue
```

At this point:

Linux needs a last resort.

---

# 4. What Does OOM Killer Do?

OOM Killer:

> Selects one or more processes and terminates them to release memory.

Example:

Before:

```
Process A   500MB

Process B   20GB

Process C   200MB
```

Memory exhausted.

OOM Killer chooses:

```
Kill Process B
```

Memory becomes available.

System survives.

---

The goal:

Not fairness.

The goal:

> Protect system availability.

---

# 5. How Does Linux Choose Which Process to Kill?

Linux calculates:

```
oom_score
```

for processes.

A higher score means:

> More likely to be killed.

---

Factors include:

## Memory Usage

Large memory consumers are candidates.

Example:

```
Process A

20GB RAM
```

is more suspicious.

---

## Process Importance

Some processes should be protected.

Example:

```
init/systemd
```

should not be easily killed.

---

## oom_score_adj

Administrators can influence priority.

Example:

```bash
echo -500 > /proc/PID/oom_score_adj
```

Lower value:

Less likely to be killed.

---

The selection idea:

```
Memory Usage

+

Process Priority

+

oom_score_adj

        |

        v

Kill Decision
```

---

# 6. What Happens After OOM Killer Kills a Process?

Example:

```
Process

 |

 v

SIGKILL

 |

 v

Process exits
```

The kernel releases:

* virtual memory
* page tables
* file descriptors
* kernel resources

---

Memory returns:

```
Freed Pages

      |

      v

Available Memory
```

---

The system continues running.

---

# 7. How Can We Check Who Triggered OOM?

The kernel logs OOM events.

Common commands:

```bash
dmesg | grep -i oom
```

or:

```bash
journalctl -k | grep -i oom
```

---

Example:

```
Out of memory: Killed process 1234 (java)
```

Meaning:

The kernel killed:

```
PID 1234

Program: java
```

---

Useful information:

```
Memory usage

oom_score

Process name
```

---

# 8. Why Do Kubernetes Pods Often Show OOMKilled?

Kubernetes adds another memory boundary.

Example:

Pod:

```yaml
resources:
  limits:
    memory: 1Gi
```

The container can only use:

```
1GB memory
```

---

If application uses:

```
1.5GB
```

The container runtime detects:

```
Memory limit exceeded
```

Then:

```
Container

      |

      v

Killed

      |

      v

OOMKilled
```

---

Important:

Kubernetes OOMKilled can happen before the whole node runs out of memory.

Because:

```
Container limit

        <

Node memory
```

---

# 9. Why Do Low Latency Systems Care About OOM?

High performance systems care deeply about memory behavior.

Examples:

* HFT systems
* trading engines
* databases

Why?

Because memory pressure causes:

## Page Fault

```
Memory access

      |

      v

Disk access
```

Huge latency difference.

---

## Reclaim Activity

Kernel may spend time:

```
Scanning pages

Reclaiming memory
```

---

## Unexpected Process Death

A trading service killed by OOM:

```
Application

 |

 v

SIGKILL

 |

 v

Service unavailable
```

is unacceptable.

---

Therefore low latency systems prefer:

* memory reservation
* huge pages
* avoiding swap
* monitoring memory usage

---

# 10. Where Does OOM Killer Exist in Linux Memory Management?

Complete Linux Memory Management:

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

MMU

 |

 v

Physical Memory

 |

 v

Memory Management

 |

 +----------------+

 |                |

Reclaim       OOM Killer

 |                |

 v                v

Free Pages    Kill Process
```

---

OOM Killer is:

> The final recovery mechanism when normal memory management cannot satisfy allocations.

---

# Final Knowledge Chain

Why does Linux need OOM Killer?

```
Applications allocate memory

        |

        v

Physical memory becomes limited

        |

        v

Kernel tries reclaim

        |

        v

Reclaim fails

        |

        v

OOM Killer selects victim

        |

        v

Free memory

        |

        v

Keep system alive
```

---

# Conclusion

OOM Killer represents an important Linux design principle:

> When resources become unavailable, sacrifice one component to protect the whole system.

Linux does not simply crash.

It tries:

```
Recover

    |

    v

Protect

    |

    v

Survive
```

Understanding OOM Killer helps explain:

* Linux Memory Management
* Kubernetes reliability
* Production troubleshooting
* High-performance system design
