---
title: "Why Does socket() Return an Integer? Understanding File Descriptor in Linux"
date: 2026-07-09
draft: false
tags:
  - Linux
  - Socket
  - File Descriptor
  - Kernel
  - System Programming
categories:
  - Many WHYs
---
# Why Does socket() Return an Integer? Understanding File Descriptor in Linux

When writing network programs in Linux, we often see:

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

The return value of `socket()` is an integer.

This raises several questions:

* Why does a socket return an integer?
* Why doesn't Linux return a socket object directly?
* Why do many tutorials show socket file descriptors like `3`?
* Can file descriptors run out?
* What does `ulimit` actually control?

To answer these questions, we need to understand:

```
socket()
   |
   v
File Descriptor
   |
   v
Process FD Table
   |
   v
Kernel Object
   |
   v
Resource Management
```

---

# 1. Why Does socket() Return an Integer?

A simple socket program:

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

The result:

```
fd = 3
```

At first glance, this seems strange.

A socket is a complex kernel object:

```
Socket

- IP address
- port
- TCP state
- receive buffer
- send buffer
- protocol information
```

Why return only:

```
3
```

?

---

The reason is:

> User space does not directly access kernel objects.

Linux separates:

```
User Space

     |

System Call

     |

Kernel Space
```

Applications cannot directly manipulate:

```
struct socket
struct tcp_sock
network buffers
```

inside the kernel.

Instead, Linux gives the process a small identifier:

```
File Descriptor
```

---

# 2. What Is a File Descriptor?

A file descriptor (FD) is:

> A small integer used by a process to reference an open resource.

Examples:

```
File

Socket

Pipe

Terminal

Device
```

All can be represented by:

```
integer fd
```

Example:

```c
int fd = open("file.txt", O_RDONLY);
```

or:

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

Both return:

```
integer
```

because Linux provides a unified resource abstraction.

---

# 3. Why Doesn't Linux Return a Socket Object?

In many programming languages:

```cpp
Socket socket = new Socket();
```

looks natural.

Why doesn't Linux do this?

Because the kernel and application have different responsibilities.

---

Application side:

```
User Space

int fd
```

Kernel side:

```
Kernel

struct file

        |

        v

struct socket

        |

        v

struct sock
```

The application only needs a handle.

The kernel manages the actual object.

---

This design provides:

## 1. Security

Applications cannot directly modify kernel memory.

---

## 2. Abstraction

The same interface works for:

```
read(fd)
write(fd)
close(fd)
```

regardless of whether:

```
fd -> file

fd -> socket

fd -> pipe
```

---

## 3. Compatibility

The kernel can change internal structures without breaking applications.

The application only depends on:

```
integer fd
```

---

# 4. How Does Linux Find the Real Socket From an Integer?

Now the important question:

If we only have:

```
fd = 5
```

how does Linux find the actual socket?

Each process has an:

```
File Descriptor Table
```

Conceptually:

```
Process

   FD Table

+-------+----------------+
| FD    | Object         |
+-------+----------------+
| 0     | stdin          |
| 1     | stdout         |
| 2     | stderr         |
| 3     | file           |
| 4     | socket         |
| 5     | pipe           |
+-------+----------------+
```

When we call:

```c
read(fd, buffer, size);
```

Linux:

```
fd

 |

 v

Process FD Table

 |

 v

struct file

 |

 v

Actual kernel object
```

For a socket:

```
struct file

        |

        v

struct socket

        |

        v

struct sock
```

---

# 5. Why Are Sockets Usually 3?

Many tutorials show:

```c
int sockfd = socket(...);

printf("%d", sockfd);
```

Output:

```
3
```

Why?

Because every Linux process starts with three default file descriptors:

```
FD 0 -> stdin

FD 1 -> stdout

FD 2 -> stderr
```

The first available FD is:

```
3
```

So:

```c
socket()
```

usually returns:

```
3
```

Example:

```
Process FD Table


0  stdin

1  stdout

2  stderr

3  socket
```

---

But this is not guaranteed.

If a program opens a file first:

```c
open("test.txt");
```

then:

```
FD 3 -> file
```

The socket becomes:

```
FD 4
```

---

# 6. Are File Descriptors Limited?

Yes.

A process cannot create unlimited file descriptors.

Example:

A server:

```
Client 1  -> fd 10

Client 2  -> fd 11

Client 3  -> fd 12

...

Client 100000
```

Each connection consumes resources:

```
socket

memory buffers

kernel structures
```

Eventually the process reaches its FD limit.

---

When this happens:

```c
socket()
```

fails.

Example:

```
Too many open files
```

The error:

```
EMFILE
```

means:

> The process has reached its maximum number of open file descriptors.

---

# 7. What Is ulimit?

`ulimit` controls resource limits for a process.

Check:

```bash
ulimit -a
```

Example:

```
open files (-n) 1024
```

This means:

A process can open at most:

```
1024 file descriptors
```

---

For a server:

```
1024 connections
```

may not be enough.

Administrators often increase:

```bash
ulimit -n 100000
```

allowing more concurrent connections.

---

# 8. Why Is File Descriptor Important for High Performance Servers?

Modern Linux servers are built around file descriptors.

Examples:

## epoll

```c
epoll_ctl(
    epfd,
    EPOLL_CTL_ADD,
    sockfd,
    &event
);
```

epoll does not monitor:

```
socket objects
```

It monitors:

```
file descriptors
```

---

Your C++ server:

```
TcpConnection

        |

        v

socket fd

        |

        v

Channel

        |

        v

epoll
```

Everything is connected through FD.

---

# 9. The Complete Knowledge Chain

Why does socket return an integer?

```
Socket is a kernel object

        |

        v

User space cannot access kernel memory

        |

        v

Linux provides a handle

        |

        v

Handle = File Descriptor

        |

        v

FD indexes process FD table

        |

        v

FD points to kernel object

        |

        v

epoll and servers use FD abstraction
```

---

# Conclusion

The integer returned by:

```c
socket()
```

is not the socket itself.

It is a reference:

```
integer fd
```

that allows a process to access a kernel-managed resource.

This simple design enables Linux to provide a unified interface:

```
read()
write()
close()
epoll()
```

for many different resources:

```
file

socket

pipe

device
```

Understanding file descriptors is the foundation for understanding:

* Linux I/O model
* socket programming
* epoll
* high-performance servers
* resource limits
