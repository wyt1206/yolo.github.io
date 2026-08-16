---
title: "How Does Linux Find the Real Socket From an Integer FD?"
date: 2026-07-10
draft: false
tags:
  - Linux
  - File Descriptor
  - Socket
  - Kernel
  - System Programming
categories:
  - Many WHYs
---
In the previous article, we learned:

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

returns an integer.

But this creates another question:

> If a socket is represented only by an integer, how does Linux find the real socket object inside the kernel?

For example:

```c
read(fd, buffer, size);
```

The application only provides:

```
fd = 3
```

But inside the kernel, Linux needs to locate:

```
socket
TCP state
receive buffer
send buffer
protocol information
```

The complete lookup chain is:

```
File Descriptor

      |

      v

Current Process

      |

      v

File Descriptor Table

      |

      v

struct file

      |

      v

struct socket

      |

      v

struct sock
```

Each step in the lookup path is straightforward once the surrounding objects are identified.

---

# 1. FD Is Only a Number

The first important idea:

> A file descriptor is not the resource itself.

When:

```c
int fd = socket(...);
```

returns:

```
3
```

Linux does not create:

```
fd = socket object
```

Instead:

```
3
```

is only an index.

Think of it like an array position:

```
FD number

0
1
2
3
4
5
```

The kernel uses this number to find another object.

---

# 2. Linux First Finds the Current Process

Every system call runs in the context of a process.

Example:

```c
read(fd, buffer, size);
```

When this happens:

```
User Space

read()

   |

   v

System Call

   |

   v

Kernel Space
```

The kernel already knows:

> Which process is making this system call.

Linux represents a running process with:

```
task_struct
```

Conceptually:

```
task_struct

    |

    v

files_struct
```

The process stores information about its open resources.

---

# 3. Linux Looks Up the File Descriptor Table

Inside each process:

```
task_struct
       |
       v
files_struct
       |
       v
FD Table
```

The FD table maps:

```
integer
   |
   v
kernel object reference
```

Example:

```
Process FD Table


FD       Object

0        stdin

1        stdout

2        stderr

3        socket

4        file

5        pipe
```

When Linux receives:

```
fd = 3
```

it performs:

```
FD 3

 |

 v

FD Table

 |

 v

kernel object
```

---

# 4. Why Doesn't FD Directly Point to Socket?

A common question:

> Why not make FD directly point to a socket?

Because Linux wants a unified abstraction.

A process can open:

```
Regular file

Socket

Pipe

Device

Terminal
```

They are all different kernel objects.

If FD directly pointed to every possible type:

```
fd
 |
 +--> file

fd
 |
 +--> socket

fd
 |
 +--> pipe
```

the kernel would need separate handling everywhere.

Instead Linux introduces:

```
struct file
```

as a common layer.

---

# 5. struct file: The Universal Kernel Object

The real chain is:

```
FD

 |

 v

struct file

 |

 +----------------+
 |
 v

struct inode       (file)

or

struct socket      (network)
```

`struct file` is Linux's generic representation of an opened resource.

It contains things like:

```c
struct file {

    file_operations *f_op;

    private_data;

    ...
};
```

The important part:

```
f_op
```

defines:

> How should this object handle operations?

---

# 6. How Does struct file Find the Socket?

For a socket:

```
fd

 |

 v

struct file

 |

 v

struct socket

 |

 v

struct sock
```

The socket information is stored through kernel structures.

Conceptually:

```
struct file

      |

      v

struct socket

      |

      v

struct sock
```

`struct sock` contains TCP/IP related state:

```
TCP state

receive queue

send queue

IP address

port

connection status
```

---

# 7. Why Can read() Work With Both Files and Sockets?

Now we reach the most important design idea.

Consider:

```c
read(fd, buffer, size);
```

The application does not know:

```
fd -> file?

fd -> socket?

fd -> pipe?
```

It does not matter.

Why?

Because Linux uses:

```
file_operations
```

as an abstraction layer.

---

For a regular file:

```
read()

 |

 v

file_operations

 |

 v

filesystem read()
```

---

For a socket:

```
read()

 |

 v

file_operations

 |

 v

socket receive()
```

The same system call:

```
read()
```

invokes different implementations.

---

This is similar to polymorphism:

```
read()

     |
     |
     +---- regular file read()

     |
     |
     +---- socket read()

     |
     |
     +---- pipe read()
```

---

# 8. The Complete Journey of read(fd)

The complete lookup path can be summarized as follows.

Application:

```c
read(3, buffer, size);
```

Step 1:

Kernel receives:

```
fd = 3
```

---

Step 2:

Find current process:

```
task_struct
```

---

Step 3:

Find FD table:

```
files_struct
```

---

Step 4:

Lookup:

```
FD 3
```

---

Step 5:

Get:

```
struct file
```

---

Step 6:

Follow:

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

Step 7:

Execute socket read operation.

The complete path:

```
read(fd)

    |

    v

Current Process

    |

    v

FD Table

    |

    v

struct file

    |

    v

struct socket

    |

    v

struct sock

    |

    v

TCP/IP Stack
```

---

# 9. Why Is This Design Powerful?

Because Linux provides one unified I/O model.

The application only needs:

```
integer fd
```

and common APIs:

```
read()

write()

close()

epoll()
```

The kernel handles the complexity.

This allows:

```
Application

      |

      v

File Descriptor

      |

      v

Different Kernel Resources
```

including:

```
Files

Sockets

Pipes

Devices
```

---

# Final Knowledge Chain

The whole idea can be summarized as:

```
FD is only an index

        |

        v

Linux finds current process

        |

        v

Process contains FD table

        |

        v

FD table points to struct file

        |

        v

struct file provides abstraction

        |

        v

struct file reaches real resource

        |

        v

Socket becomes accessible
```

---

# Conclusion

A file descriptor is not a socket.

It is a small integer handle created to hide kernel complexity.

When Linux receives:

```c
read(fd)
```

it follows a single chain:

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

struct socket

 |

 v

struct sock
```

This simple abstraction is one of the reasons Linux can support:

* files
* sockets
* pipes
* devices

with the same programming model.

Understanding this chain is the foundation for understanding:

* epoll
* event-driven servers
* Linux I/O architecture
* high-performance networking
