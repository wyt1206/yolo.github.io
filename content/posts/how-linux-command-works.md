---
title: "How Do Linux Commands Actually Work? From Terminal to Kernel"
date: 2026-07-22
draft: false
tags:
  - Linux
  - Shell
  - Terminal
  - Process
  - ELF
  - Syscall
  - Kernel
categories:
  - Many WHYs
---
Every Linux user types commands:

```bash
ls
```

or:

```bash
lsof
```

or:

```bash
netstat
```

It feels like:

```
Command
    |
    v
Linux
```

But the real execution chain is:

```
Keyboard

↓

Terminal

↓

Shell

↓

Command Program

↓

ELF Loader

↓

Process

↓

System Call

↓

Kernel

↓

Hardware
```

The question:

> Who actually executes a Linux command?

---

# 1. When We Type ls, What Happens?

The command begins with a user keystroke:

```bash
ls
```

You press Enter.

A complete flow:

```
User Input

↓

Terminal

↓

Shell

↓

fork()

↓

exec()

↓

ELF Loader

↓

ls Process

↓

syscall

↓

Kernel

↓

Filesystem
```

---

The important idea:

`ls` itself does not read the disk.

It asks the kernel.

---

Example:

```text
ls
 |
 |
 v

get directory entries

 |
 |
 v

Kernel

 |
 |
 v

Filesystem
```

---

# 2. Terminal, Shell, Command: What Is The Difference?

These three are often confused.

---

## Terminal

Terminal is the interface.

Examples:

* GNOME Terminal
* iTerm2
* Windows Terminal

Its job:

```
Receive keyboard input

Display output
```

It does not understand:

```bash
ls
```

---

## Shell

Shell is a command interpreter.

Examples:

* bash
* zsh
* fish

Its job:

```
Read command

Understand syntax

Start programs
```

Example:

Input:

```bash
ls -l
```

Shell parses:

```
program:

ls


argument:

-l
```

---

## Command

Command is usually a program.

Example:

```bash
ls
```

is usually:

```
/bin/ls
```

It is an executable file.

---

Relationship:

```
Terminal

↓

Shell

↓

Command Program
```

---

# 3. What Exactly Is ls?

Check:

```bash
which ls
```

Example:

```
/bin/ls
```

So:

```
ls
```

is just a binary executable.

---

Check:

```bash
file /bin/ls
```

Output:

```
ELF 64-bit executable
```

Meaning:

```
ls

=

compiled machine code
```

---

It contains:

* CPU instructions
* data
* metadata

---

The CPU eventually executes:

```
machine instructions
```

---

# 4. What Exists Inside Linux Kernel?

Kernel is not a command collection.

It provides services.

Main components:

```
Linux Kernel

├── Process Scheduler
├── Memory Management
├── Virtual File System
├── Network Stack
├── Device Drivers
├── System Call Interface
```

---

Example:

`ls` needs:

```
directory information
```

Kernel provides:

```
filesystem subsystem
```

---

# 5. How Does Shell Execute A Command?

Suppose:

```bash
ls
```

Shell does:

---

## Step 1: Parse command

Shell understands:

```
command = ls
```

---

## Step 2: Find executable

Search:

```
PATH
```

Example:

```
/bin
/usr/bin
```

Find:

```
/bin/ls
```

---

## Step 3: Create Process

Shell calls:

```
fork()
```

Result:

```
Shell

 |

 +---- Child Process
```

---

## Step 4: Replace Process Image

Child calls:

```
exec()
```

Before:

```
Child = shell copy
```

After:

```
Child = ls program
```

---

Now:

```
ls process
```

is running.

---

# 6. What Is ELF Loader?

When kernel receives:

```
exec("/bin/ls")
```

it needs to load the program.

This is where:

```
ELF Loader
```

appears.

---

ELF:

```
Executable and Linkable Format
```

Linux executable format.

---

The loader:

1. Reads ELF header
2. Creates memory layout
3. Maps program sections
4. Loads libraries
5. Sets entry point

---

Memory:

```
Process Address Space


Code Segment

Data Segment

Heap

Stack
```

---

Finally:

CPU starts:

```
instruction pointer

↓

ELF entry address
```

---

# 7. Why Does ls Need Kernel?

Because applications cannot directly access hardware.

---

For example:

ls needs:

```
directory content
```

It calls:

```
getdents()
```

system call.

---

Flow:

```
ls

 |

syscall

 |

Kernel VFS

 |

Filesystem Driver

 |

Disk
```

---

The kernel handles:

* permissions
* filesystem
* caching
* hardware

---

# 8. What Is lsof?

`lsof`

means:

```
List Open Files
```

---

Linux idea:

```
Everything is a file
```

includes:

* regular files
* sockets
* pipes
* devices

---

Example:

```bash
lsof -i :8080
```

asks:

```
Which process owns this socket?
```

---

Internally:

```
lsof

↓

/proc filesystem

↓

Kernel process information
```

---

Example:

```
Process

↓

File Descriptor Table

↓

Socket
```

---

This connects to your previous articles:

```
socket()

↓

fd

↓

struct file

↓

kernel object
```

---

# 9. What Is netstat?

`netstat`

shows:

```
Network statistics
```

Examples:

```bash
netstat -tulnp
```

shows:

* listening ports
* connections
* processes

---

Internally:

```
netstat

↓

/proc/net

↓

Kernel network subsystem
```

---

It reads kernel-maintained information:

```
TCP state

Socket table

Network interfaces
```

---

Modern Linux often uses:

```bash
ss
```

instead.

---

# 10. Why Is cd Not A Program?

Interesting question.

Try:

```bash
which cd
```

Usually:

```
no output
```

Why?

Because:

```
cd changes current directory
```

---

Current directory belongs to:

```
Process state
```

Example:

```
Shell Process

cwd = /home/user
```

---

If cd were a program:

```
Shell

fork()

↓

cd process

chdir()

↓

exit()
```

Only the child changes directory.

Parent shell stays:

```
/home/user
```

---

Therefore:

```
cd must run inside shell
```

It is a:

```
shell builtin
```

---

# 11. Can We Write Our Own Linux Command?

Yes.

A command is just:

```
Executable program
```

---

Example:

hello.c:

```c
#include <stdio.h>

int main()
{
    printf("Hello Linux\n");
    return 0;
}
```

Compile:

```bash
gcc hello.c -o hello
```

Run:

```bash
./hello
```

---

Add to PATH:

```
~/bin
```

Now:

```bash
hello
```

works.

---

Your program can also call:

```
syscall
```

Example:

```c
open()

read()

write()
```

---

# Complete Linux Command Execution Chain

```
User

 |

 v

Terminal

 |

 v

Shell

 |

 v

fork()

 |

 v

exec()

 |

 v

ELF Loader

 |

 v

Process

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

A Linux command is not magic.

Most commands are:

```
Executable Files
```

running as:

```
Processes
```

using:

```
Kernel Services
```

The key boundary:

```
User Space

    |
    |
 syscall

    |
    |

Kernel Space
```

---

# Conclusion

When you type:

```bash
ls
```

you are not directly talking to Linux.

You are starting a user-space program.

The real chain is:

```
ls

↓

Shell launches process

↓

ELF loader loads binary

↓

CPU executes instructions

↓

Program requests kernel services

↓

Kernel accesses hardware
```

Understanding this explains:

* why commands exist
* why some commands are builtins
* why syscalls matter
* why Linux separates user space and kernel space
